# 02-用户服务 Gin 与 Kratos 落地

> 一句话定位：用户域只做注册、登录、取用户、刷新令牌四件事——Kratos gRPC + DDD 四层承载业务，Gin 网关只保留一层薄 HTTP 适配。

## 本篇目标

- 用 `internal/{biz,data,service,server,conf}` 落地 Kratos DDD，data 实现 biz 的仓储接口
- `user.proto` 定义 Register / Login / GetUser / RefreshToken 并生成 gRPC 代码
- GORM + 唯一索引兜住并发注册，bcrypt 存密码，错误统一为 Kratos `errors`
- access 15 分钟 / refresh 7 天，jti 落 Redis 原子消费；服务注册到 etcd（:9001）

## 前置知识

- [01-项目全景与架构设计](01-项目全景与架构设计.md)：统一响应与错误码区间
- [09-MySQL与ORM/02-GORM核心用法与模型设计](../09-MySQL与ORM/02-GORM核心用法与模型设计.md)

## 一、先分层，再写代码

网关只做 HTTP 绑定、JWT 校验与统一响应；user-service 只暴露 gRPC。代码落在 `app/user-service` 下：`cmd/main.go` 组装 App，`internal/conf` 读配置，`internal/server` 配中间件，`internal/service` 做 proto 适配，`internal/biz` 写用例与仓储接口，`internal/data` 实现 GORM 与 Redis。依赖方向 data → biz，`data` 实现 `biz` 定义的接口，用例才能用 mock 单测。

## 二、user.proto：唯一契约

```proto
// api/user/v1/user.proto
syntax = "proto3";
package user.v1;
option go_package = "mall-lite/api/user/v1;v1";

service UserService {
    rpc Register(RegisterRequest) returns (RegisterReply);
    rpc Login(LoginRequest) returns (LoginReply);
    rpc GetUser(GetUserRequest) returns (User);
    rpc RefreshToken(RefreshTokenRequest) returns (LoginReply);
}
message User { int64 id = 1; string email = 2; string nickname = 3; }
message RegisterRequest { string email = 1; string password = 2; string nickname = 3; }
message RegisterReply { int64 user_id = 1; }
message LoginRequest { string email = 1; string password = 2; }
message LoginReply { string access_token = 1; string refresh_token = 2; int64 expires_in = 3; }
message GetUserRequest { int64 user_id = 1; }
message RefreshTokenRequest { string refresh_token = 1; }
```

`User` 里永远不放 `password`、`token`，字段只增不改序号。

```bash
kratos proto client api/user/v1/user.proto
kratos proto server api/user/v1/user.proto -t app/user-service/internal/service
```

## 三、users 表与 data 层

`users` 只需 `uk_email` 唯一索引兜住并发注册；`gorm.Open` 开 `TranslateError: true`，冲突会被翻译成 `gorm.ErrDuplicatedKey`，不用解析 1062。

```go
// app/user-service/internal/data/user.go
type UserModel struct {
    ID           int64  `gorm:"column:id;primaryKey"`
    Email        string `gorm:"column:email;size:128;uniqueIndex"`
    PasswordHash string `gorm:"column:password_hash;size:72"`
    Nickname     string `gorm:"column:nickname;size:32"`
    Status       int8   `gorm:"column:status"`
}

func (UserModel) TableName() string { return "users" }

func (r *userRepo) Create(ctx context.Context, u *biz.User) error {
    m := &UserModel{Email: u.Email, PasswordHash: u.Password, Nickname: u.Nickname, Status: 1}
    if err := r.db.WithContext(ctx).Create(m).Error; err != nil {
        if errors.Is(err, gorm.ErrDuplicatedKey) {
            return kerrors.Conflict("USER_EMAIL_EXISTS", "email already registered")
        }
        return err
    }
    u.ID = m.ID
    return nil
}
```

`FindByEmail` 用 `Where("email = ? AND status = 1").First`，`ErrRecordNotFound` 转成 `nil, nil` 交给 biz；所有查询必须 `WithContext(ctx)`，否则超时与 trace 断在这一层。

## 四、biz：注册与登录

```go
// app/user-service/internal/biz/user.go
type UserRepo interface {
    Create(ctx context.Context, u *User) error
    FindByEmail(ctx context.Context, email string) (*User, error)
}

func (uc *UserUsecase) Register(ctx context.Context, email, password, nickname string) (int64, error) {
    if len(password) > 72 { // bcrypt 只取前 72 字节
        return 0, kerrors.BadRequest("USER_PASSWORD_TOO_LONG", "password must be <= 72 bytes")
    }
    hash, err := bcrypt.GenerateFromPassword([]byte(password), uc.cost) // 生产 cost=12
    if err != nil {
        return 0, kerrors.InternalServer("USER_BCRYPT_ERROR", "hash password failed")
    }
    u := &User{Email: strings.ToLower(email), Password: string(hash), Nickname: nickname, Status: 1}
    err = uc.repo.Create(ctx, u) // data 已把冲突翻译成 USER_EMAIL_EXISTS
    return u.ID, err
}

func (uc *UserUsecase) Login(ctx context.Context, email, password string) (*TokenPair, error) {
    u, err := uc.repo.FindByEmail(ctx, strings.ToLower(email))
    if err != nil {
        return nil, err
    }
    if u == nil || bcrypt.CompareHashAndPassword([]byte(u.Password), []byte(password)) != nil {
        return nil, kerrors.Unauthorized("USER_AUTH_FAILED", "email or password incorrect")
    }
    return uc.tokens.Issue(ctx, u.ID)
}
```

「查不到用户」与「密码错误」共用 `USER_AUTH_FAILED`，否则接口等于枚举已注册邮箱；线上再加一次 dummy hash 比对抹平时间差。

## 五、JWT 双令牌与 Redis jti

access 15 分钟、refresh 7 天；签发时把 refresh 的 `jti` 写进 Redis，刷新时先 `DEL` 再签新对，`DEL` 返回 0 即视为重放。

```go
// app/user-service/internal/biz/token.go
func (t *TokenIssuer) Issue(ctx context.Context, uid int64) (*TokenPair, error) {
    jti := uuid.NewString()
    at, err := t.sign(uid, "access", jti, 15*time.Minute)
    if err != nil {
        return nil, err
    }
    rt, err := t.sign(uid, "refresh", jti, 7*24*time.Hour)
    if err != nil {
        return nil, err
    }
    if err := t.rdb.Set(ctx, refreshKey(uid, jti), "1", 7*24*time.Hour).Err(); err != nil {
        return nil, kerrors.InternalServer("USER_REDIS_ERROR", "issue token failed")
    }
    return &TokenPair{AccessToken: at, RefreshToken: rt, ExpiresIn: 900}, nil
}
```

`sign` 内部固定 `jwt.SigningMethodHS256`，claims 是自定义 `Claims{Type, jwt.RegisteredClaims}`，`ID` 放 jti；`parse` 用 `jwt.ParseWithClaims` + `jwt.WithValidMethods([]string{"HS256"})` 并校验 `Type`——只 `Parse` 会接受 `alg=none`。`Refresh` 先 `DEL refreshKey(uid, claims.ID)`：返回 0 报 `USER_TOKEN_REPLAY`，返回 1 才 `Issue` 新对，旧 refresh 立即失效。密钥从 `MALL_JWT_SECRET` 注入，≥ 32 字节。

## 六、service 层与错误码

service 只做协议转换，不碰业务：`Register` 调 `s.uc.Register` 包成 `v1.RegisterReply`；`GetUser` 查不到返回 `kerrors.NotFound("USER_NOT_FOUND", "user not found")`；biz 返回的 `*errors.Error` 原样返给网关，禁止再包一层 `fmt.Errorf`。

```go
// app/user-service/internal/service/user.go
type UserService struct {
    v1.UnimplementedUserServiceServer
    uc *biz.UserUsecase
}
```

| 场景 | Kratos 错误 | HTTP | 业务码 |
| --- | --- | --- | --- |
| 参数不合法 | `BadRequest("USER_INVALID_PARAM")` | 400 | 40001 |
| 邮箱重复 | `Conflict("USER_EMAIL_EXISTS")` | 409 | 40901 |
| 邮箱或密码错 | `Unauthorized("USER_AUTH_FAILED")` | 401 | 40101 |
| 用户不存在 | `NotFound("USER_NOT_FOUND")` | 404 | 40401 |
| 依赖故障 | `InternalServer("USER_REDIS_ERROR")` | 500 | 50001 |

网关 handler 不写业务分支：绑定失败返回 40001，RPC 错误交给 `errs.FromGRPC(err, requestID)` 输出统一响应，堆栈与下游服务名不出网。

## 七、server 与 etcd 注册

```go
// app/user-service/cmd/main.go
func main() {
    cfg := conf.MustLoad("configs/user-service.yaml")
    logger := kratoszap.NewLogger(zap.Must(zap.NewProduction()))
    uc := biz.NewUserUsecase(data.NewUserRepo(data.NewDB(cfg)), biz.NewTokenIssuer(cfg.JWT.Secret, data.NewRedis(cfg)))
    gs := grpc.NewServer(grpc.Address(cfg.Server.GRPC.Addr), grpc.Timeout(2*time.Second),
        grpc.Middleware(recovery.Recovery(), tracing.Server(), logging.Server(logger), server.Sentinel()))
    v1.RegisterUserServiceServer(gs, service.NewUserService(uc))
    cli, err := clientv3.New(clientv3.Config{Endpoints: cfg.Registry.Etcd.Endpoints, DialTimeout: 3 * time.Second})
    if err != nil { // 注册中心不可用就别启动
        panic(err)
    }
    app := kratos.New(kratos.Name("user-service"), kratos.Logger(logger), kratos.Server(gs),
        kratos.Registrar(etcdreg.New(cli, etcdreg.RegisterTTL(15*time.Second))))
    if err := app.Run(); err != nil {
        panic(err)
    }
}
```

## 八、单测要点

- biz：mock `UserRepo`，覆盖重复邮箱、bcrypt 超长、密码错误；bcrypt 用 `bcrypt.MinCost`
- token：固定时钟，覆盖过期、篡改 payload、`alg=none`、refresh 重放；data 层用 testcontainers 验证唯一键翻译

## 完整示例

```yaml
# configs/user-service.yaml
app: { name: user-service, env: dev }
server: { grpc: { addr: 0.0.0.0:9001, timeout: 2s } }
registry: { type: etcd, etcd: { endpoints: ["127.0.0.1:2379"] } }
mysql: { dsn: "root:root@tcp(127.0.0.1:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local" }
redis: { addr: "127.0.0.1:6379" }
jwt: { secret: "dev-only-at-least-32-bytes-secret" } # 生产用 MALL_JWT_SECRET 覆盖（需 BindEnv）
```

```bash
go run ./app/user-service/cmd
grpcurl -plaintext -d '{"email":"neo@example.com","password":"12345678","nickname":"neo"}' 127.0.0.1:9001 user.v1.UserService/Register
```

```powershell
go run .\app\user-service\cmd
grpcurl -plaintext -d '{"email":"neo@example.com","password":"12345678","nickname":"neo"}' 127.0.0.1:9001 user.v1.UserService/Register
```

## 常见坑

1. **先查后插做查重**：并发下两个请求都查到「不存在」；唯一索引才是最终防线。
2. **JWT 不校验算法与 typ**：只 `Parse` 会接受 `alg=none`，access 也能当 refresh 用。
3. **refresh 不消费旧 jti**：令牌泄露后 7 天可反复换新；`DEL` 成功才签发新对。
4. **用 `context.Background()` 调下游**：超时预算与 trace_id 全断；一律透传 `ctx`。
5. **bcrypt cost 调到 14**：单次登录百毫秒级，压测先崩 CPU；cost 12 是常见折中。

## 小结

- 并发注册靠 `uk_email` 兜底，data 层把唯一键冲突翻译成 409
- access 15 分钟 / refresh 7 天，jti 落 Redis 原子消费，刷新即轮换
- 错误统一用 Kratos `errors`，网关只映射一次，堆栈不出网

## 练习

1. 给 refresh 加设备维度：`jwt:refresh:{uid}:{device}:{jti}`，实现单设备登出。
2. 给 `GetUser` 加 Cache-Aside（`user:info:{id}`，随机 TTL 30±5 分钟），压测对比 QPS。
3. 用 `grpcurl` 验证同一个 refresh token 第二次调用返回 `USER_TOKEN_REPLAY`。

## 官方文档

- Kratos 快速开始：<https://go-kratos.dev/docs/getting-started/start>
- Kratos 错误处理：<https://go-kratos.dev/docs/component/errors>
- Kratos 服务注册：<https://go-kratos.dev/docs/component/registry>
- Kratos 代码生成：<https://go-kratos.dev/docs/component/api>
