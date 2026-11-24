# Kratos 服务端最佳实践

> 一句话定位：把 user-service 从「能跑」调到「敢上生产」——配置、日志、注册、中间件、错误码、DDD 分层与优雅退出一次配齐。

## 本篇目标

- 用 Viper→struct 注入配置，避开 `AutomaticEnv` 对 `Unmarshal` 不生效的坑
- 用 zap + `contrib/log/zap/v2` 统一日志，接入 Kratos 的 Logger 接口
- 正确构造 etcd / Consul 注册器：先建 client，再 `New`，注册器只返回 `*Registry`
- 配好中间件链、超时、健康检查、优雅退出与稳定错误码
- 按 service → biz → data 分层，控制并发与资源上限，日志脱敏

## 前置知识

- [02-Kratos快速入门与Proto](02-Kratos快速入门与Proto.md)

## 一、完整入口 main.go

`kratos.App` 负责 Server、Registrar、Logger 的组装与信号处理，`grpc.Server` 只负责协议与中间件。

```go
// app/user-service/cmd/main.go
package main

import (
    kratoszap "github.com/go-kratos/kratos/contrib/log/zap/v2"
    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/middleware/logging"
    "github.com/go-kratos/kratos/v2/middleware/metadata"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
    "github.com/go-kratos/kratos/v2/middleware/validate"
    kratosgrpc "github.com/go-kratos/kratos/v2/transport/grpc"
    "go.uber.org/zap"
    ggrpc "google.golang.org/grpc"
    "google.golang.org/grpc/health"
    healthpb "google.golang.org/grpc/health/grpc_health_v1"
    v1 "mall-lite/api/user/v1"
    "mall-lite/app/user-service/internal/conf"
    "mall-lite/app/user-service/internal/server"
    "mall-lite/app/user-service/internal/service"
    "mall-lite/pkg/discovery"
)

func main() {
    cfg := conf.MustLoad("configs/user-service.yaml")
    zl, _ := zap.NewProduction(zap.AddCaller())
    logger := kratoszap.NewLogger(zl) // zap → Kratos log.Logger，只保留一套日志
    reg, closeReg, err := discovery.NewRegistrar(cfg.Registry)
    if err != nil {
        panic(err)
    }
    defer closeReg()

    gs := kratosgrpc.NewServer(
        kratosgrpc.Address(cfg.Server.Grpc.Addr),
        kratosgrpc.Timeout(cfg.Server.Grpc.Timeout),
        kratosgrpc.Middleware(
            recovery.Recovery(), // 最外层：panic 也不漏 trace
            tracing.Server(),
            logging.Server(logger),
            metadata.Server(),
            validate.Validator(),
            server.Sentinel(), // 最内层：只统计真实业务调用
        ),
        kratosgrpc.Options(ggrpc.MaxConcurrentStreams(1000)),
    )
    healthpb.RegisterHealthServer(gs.Server, health.NewServer())
    v1.RegisterUserServiceServer(gs, service.NewUserService(cfg))

    app := kratos.New(kratos.Name("user-service"), kratos.Logger(logger), kratos.Server(gs), kratos.Registrar(reg))
    if err := app.Run(); err != nil {
        panic(err)
    }
}
```

中间件顺序由外到内固定为 recovery → tracing → logging → metadata → validate → Sentinel：recovery 最外才能兜住 panic；logging 在 tracing 之内才拿得到 `trace_id`；Sentinel 靠内，避免把 panic 与校验失败算进业务指标。

## 二、配置：Viper → struct

配置文件里 `server.grpc.addr/timeout`、`registry.type/etcd.endpoints`、`mysql.dsn` 与 struct 的 `mapstructure` tag 一一对应；环境变量前缀 `MALL`，`.` 换 `_`。

```go
// internal/conf/conf.go（核心片段）
v := viper.New()
v.SetConfigFile(path)
v.SetEnvPrefix("MALL")
v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
v.AutomaticEnv()
_ = v.BindEnv("mysql.dsn")     // AutomaticEnv 对 Unmarshal 不生效
_ = v.BindEnv("registry.type") // 关键 key 必须显式绑定，MALL_MYSQL_DSN 才会覆盖文件值
if err := v.ReadInConfig(); err != nil {
    panic(err)
}
var cfg Config // 字段带 mapstructure tag，如 MySQL.DSN `mapstructure:"dsn"`
if err := v.Unmarshal(&cfg); err != nil {
    panic(err)
}
```

## 三、注册中心：etcd 与 Consul 的正确构造

```go
// pkg/discovery/registry.go（etcd 分支；Consul 同构）
func NewRegistrar(c conf.Registry) (registry.Registrar, func(), error) {
    cli, err := clientv3.New(clientv3.Config{Endpoints: c.Etcd.Endpoints, DialTimeout: 3 * time.Second})
    if err != nil {
        return nil, func() {}, err
    }
    // 构造函数只返回 *Registry，没有 error；连接错误在上一步处理
    return etcdreg.New(cli, etcdreg.RegisterTTL(15*time.Second)), func() { _ = cli.Close() }, nil
}
```

**最容易写错的地方**：`etcdreg.New` 的入参是 `*clientv3.Client`，不是 `clientv3.Config`；先 `clientv3.New`（这一步才返回 error），再交给它。Consul 同理，先 `consulapi.NewClient`。含 Consul 分支、Namespace 与 TLS 的完整实现在 [05-服务注册与发现etcd与Consul实践](05-服务注册与发现etcd与Consul实践.md)。

## 四、错误码与参数校验

服务端只返回 Kratos 错误，禁止裸 `errors.New` 或带堆栈的 `fmt.Errorf`，否则网关拿不到业务码。

| 用法 | HTTP | 业务码 | 场景 |
| --- | --- | --- | --- |
| `kerrors.BadRequest("USER_ID_INVALID", "...")` | 400 | `400xx` | 参数非法 |
| `kerrors.NotFound("USER_NOT_FOUND", "...")` | 404 | `404xx` | 资源不存在 |
| `kerrors.TooManyRequests("RATE_LIMITED", "...")` | 429 | `429xx` | 限流 |
| `kerrors.InternalServer("DB_ERROR", "...")` | 500 | `5xxxx` | 未分类内部错误 |

`reason`（如 `USER_ID_INVALID`）上线后不要改，前端与告警可能按它分支；改文案只改 message。参数校验分两层：proto 里加 `(validate.rules).int64 = {gt: 0}`，由 `protoc-gen-validate` 生成规则交给 `validate.Validator()`；biz 层再校验一次，防止内部调用绕过网关。

## 五、DDD 分层：service → biz → data

| 层 | 只做 | 禁止 |
| --- | --- | --- |
| service | 协议转换、调用 usecase | 写 SQL、拼业务规则 |
| biz | 领域规则、编排、返回 Kratos 错误 | 依赖 gRPC/HTTP、依赖 GORM |
| data | 持久化、缓存、错误翻译 | 感知协议、返回 GORM 原始错误 |

```go
// internal/biz/user.go：领域规则 + 仓储接口
type User struct{ ID int64; Nickname string; Status int32 }

type UserRepo interface{ FindByID(ctx context.Context, id int64) (*User, error) }

type UserUsecase struct{ repo UserRepo }

func (uc *UserUsecase) Get(ctx context.Context, id int64) (*User, error) {
    if id <= 0 {
        return nil, kerrors.BadRequest("USER_ID_INVALID", "user id must be positive")
    }
    return uc.repo.FindByID(ctx, id)
}

// internal/data/user.go：把 GORM 错误翻译成 Kratos 错误，不透传出去
func (r *userRepo) FindByID(ctx context.Context, id int64) (*biz.User, error) {
    var po User
    err := r.db.WithContext(ctx).First(&po, id).Error
    if stderrors.Is(err, gorm.ErrRecordNotFound) {
        return nil, kerrors.NotFound("USER_NOT_FOUND", "user not found")
    }
    if err != nil {
        return nil, kerrors.InternalServer("DB_ERROR", "database error")
    }
    return &biz.User{ID: po.ID, Nickname: po.Nickname, Status: po.Status}, nil
}
```

## 六、超时、健康检查、优雅退出与资源限制

- **超时**：`grpc.Timeout` 是服务端兜底，必须小于 K8s `terminationGracePeriodSeconds`，否则进程被 `SIGKILL`，在途请求与缓冲日志一起丢。
- **健康检查**：`gs.Server` 是内嵌的 `*grpc.Server`，可直接注册标准 health；K8s 1.24+ 用原生 `grpc` 探针，老集群用 `tcpSocket`。
- **优雅退出**：`app.Run()` 阻塞并监听 SIGTERM/SIGINT，收到后停 server → 注销 → 退出；自己再写 `signal.Notify` 会与内置逻辑打架，重复 Stop 还会报错。
- **并发与脱敏**：`MaxConcurrentStreams` 挡慢客户端，DB 侧用 `SetMaxOpenConns` 与 `errgroup.SetLimit` 控并发；日志里手机号、token 只记掩码（`138****8000`），禁止打印请求体、密码与 SQL 参数。

## 完整示例

```bash
go build ./app/user-service/...
MALL_MYSQL_DSN="root:root@tcp(10.0.0.8:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local" go run ./app/user-service/cmd
```

```powershell
$env:MALL_MYSQL_DSN = "root:root@tcp(10.0.0.8:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local"
go run .\app\user-service\cmd
```

## 常见坑

1. **自己再监听 SIGTERM**：与 Kratos 内置信号处理重复，Stop 被调用两次，注册中心报错、日志重复。
2. **recovery 不在最外层**：panic 时 tracing/logging 被跳过，事后查不到现场。
3. **返回裸 `errors.New`**：网关拿不到 code/reason，只能一律映射成 500。
4. **`AutomaticEnv` 当万能**：对 `Unmarshal` 不生效，DSN、密钥等 key 必须显式 `BindEnv`。
5. **注册地址写 `127.0.0.1`**：本机自测正常，跨节点调用全失败；注册的必须是 Pod IP 或可达地址。

## 小结

- `kratos.New` 管生命周期与信号，`grpc.NewServer` 管协议与中间件
- 中间件顺序：recovery → tracing → logging → metadata → validate → Sentinel
- 注册器先建 client 再 `New`；优雅退出靠「停服 → 等在途 → 摘除」
- 错误 reason 稳定、message 可读、堆栈不出网，日志只记掩码

## 练习

1. 给 user-service 加日志中间件，确认每条日志带 `trace_id` 与 `grpc.method`。
2. 在 handler 里故意 panic，对比有无 `recovery.Recovery()` 的日志差异。
3. 用 Consul 起一次服务，只改 `registry.type` 与地址，验证业务代码零改动。

## 官方文档

- Kratos 服务配置：<https://go-kratos.dev/docs/component/server>
- Kratos 中间件：<https://go-kratos.dev/docs/component/middleware>
- Kratos 错误处理：<https://go-kratos.dev/docs/component/errors>
- Sentinel Go 文档：<https://sentinelguard.io/zh-cn/docs/golang/quick-start.html>
