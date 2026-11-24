# 依赖注入 Wire 与配置分层 Viper

> 一句话定位：用 wire 在编译期把「配置 → 日志 → MySQL/Redis/Kafka → repo → usecase → service → server → app」装配好，用 Viper 做「基础 yaml + 环境 yaml + `MALL_` 环境变量」三层合并并强校验，装配失败直接退出。

## 本篇目标

- 写出按包分组的 `ProviderSet`，用 `wireinject` 标签生成 `wire_gen.go`
- 让 provider 统一返回 `(资源, cleanup, error)`，管住 DB/Redis/Kafka 的关闭
- 按 Server/Registry/MySQL/Redis/Log/Sentinel 分层定义配置并校验，讲清装配顺序与测试替换

## 前置知识

- [07-工程化与测试/01-项目分层与目录规范](01-项目分层与目录规范.md)：分层职责；[12-微服务与Kratos/03-Kratos服务端最佳实践](../12-微服务与Kratos/03-Kratos服务端最佳实践.md)：`app` 生命周期

## 一、ProviderSet：按包分组，返回 cleanup

包级 `var db *gorm.DB` 不可替换、用例互相污染；fx 靠反射，错误要到启动期才暴露。wire 在**编译期**生成普通 Go 代码，缺类型直接编译失败。约定：`ProviderSet` 放在提供者所在包，provider 返回 `(*T, func(), error)`。

```go
// app/user-service/internal/data/data.go
package data

// 每个资源都返回 (实例, cleanup, error)，wire 把它们汇总成 injector 的 cleanup
var ProviderSet = wire.NewSet(NewData, NewUserRepo, NewDB, NewRedis, NewKafka,
    wire.Bind(new(biz.UserRepo), new(*userRepo)), // 绑定只在实现方声明
)

func NewDB(c *config.Config) (*gorm.DB, func(), error) {
    db, err := gorm.Open(mysql.Open(c.MySQL.DSN), &gorm.Config{})
    if err != nil {
        return nil, nil, err
    }
    sqlDB, err := db.DB()
    if err != nil {
        return nil, nil, err
    }
    sqlDB.SetMaxOpenConns(c.MySQL.MaxOpenConns)
    sqlDB.SetMaxIdleConns(c.MySQL.MaxIdleConns)
    return db, func() { _ = sqlDB.Close() }, nil
}
```

`NewRedis`、`NewKafka` 同形：构造后立刻 `Ping`/`InitProducer` 验证连通，失败返回 error，cleanup 里 `Close()`。其余三组：`biz.ProviderSet = wire.NewSet(NewUserUsecase)`、`service.ProviderSet = wire.NewSet(NewUserService)`、`server.ProviderSet = wire.NewSet(NewGRPCServer)`。

## 二、injector 与 wire_gen.go

```go
//go:build wireinject

// app/user-service/cmd/wire.go
package main

import (
    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/log"
    "github.com/google/wire"

    "mall-lite/app/user-service/internal/biz"
    "mall-lite/app/user-service/internal/data"
    "mall-lite/app/user-service/internal/server"
    "mall-lite/app/user-service/internal/service"
    "mall-lite/pkg/config"
)

func wireApp(*config.Config, log.Logger) (*kratos.App, func(), error) {
    panic(wire.Build(server.ProviderSet, service.ProviderSet,
        biz.ProviderSet, data.ProviderSet, newApp))
}
```

```bash
go install github.com/google/wire/cmd/wire@latest
wire ./app/user-service/cmd       # 生成 wire_gen.go，之后必须 go build 通过
```

`wire.go` 带 `//go:build wireinject` 不参与常规构建；`wire_gen.go` 提交但**禁止手改**。

## 三、配置结构体分层

```go
// pkg/config/config.go
package config

type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Registry RegistryConfig `mapstructure:"registry"`
    MySQL    struct {
        DSN          string `mapstructure:"dsn"`
        MaxOpenConns int    `mapstructure:"max_open_conns"`
        MaxIdleConns int    `mapstructure:"max_idle_conns"`
    } `mapstructure:"mysql"`
    Redis    RedisConfig    `mapstructure:"redis"`
    Log      LogConfig      `mapstructure:"log"`
    Sentinel SentinelConfig `mapstructure:"sentinel"`
}

// ServerConfig / Endpoint / RegistryConfig 的定义与 05 章一致，此处省略。

func (c *Config) Validate() error {
    switch {
    case c.Server.GRPC.Addr == "" && c.Server.HTTP.Addr == "":
        return fmt.Errorf("server.grpc.addr and server.http.addr are both empty")
    case c.Registry.Type == "etcd" && len(c.Registry.Etcd.Endpoints) == 0:
        return fmt.Errorf("registry.etcd.endpoints is empty")
    case c.MySQL.DSN == "":
        return fmt.Errorf("mysql.dsn is empty")
    }
    return nil
}
```

## 四、Viper：三层合并与 BindEnv

优先级：默认值 < `config.yaml` < `config.<env>.yaml` < `MALL_*` 环境变量 < 代码 `Set`。`mysql.dsn` 对应 `MALL_MYSQL_DSN`。

```go
func Load(dir, env string) (*Config, error) {
    v := viper.New()
    v.SetConfigName("config")
    v.AddConfigPath(dir)
    if err := v.ReadInConfig(); err != nil {
        return nil, fmt.Errorf("read base config: %w", err)
    }
    v.SetConfigName("config." + env) // 只写差异；用 Merge，Read 会丢掉基础项
    if err := v.MergeInConfig(); err != nil && !errors.As(err, new(viper.ConfigFileNotFoundError)) {
        return nil, fmt.Errorf("merge %s config: %w", env, err)
    }
    v.SetEnvPrefix("MALL")
    v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))
    v.AutomaticEnv()
    for _, k := range []string{"mysql.dsn", "redis.addr", "redis.password", "log.level", "sentinel.enabled"} {
        if err := v.BindEnv(k); err != nil { // AutomaticEnv 对 Unmarshal 不生效
            return nil, fmt.Errorf("bind env %s: %w", k, err)
        }
    }
    var cfg Config
    if err := v.Unmarshal(&cfg); err != nil {
        return nil, fmt.Errorf("unmarshal: %w", err)
    }
    if err := cfg.Validate(); err != nil {
        return nil, fmt.Errorf("invalid config: %w", err)
    }
    return &cfg, nil
}
```

热更新只开放运行时可换项（日志级别、限流阈值）：`WatchConfig()` + `OnConfigChange` 里 `Unmarshal` 到新结构体存 `atomic.Pointer[Config]`；DSN、端口不热更。

## 五、装配顺序与测试替换

顺序固定为 `Load+Validate → zap → MySQL → Redis → repo → usecase → service → server → app.Run`；退出时 `app.Stop(ctx)` 后由 `cleanup()` 逆序关闭 Kafka → Redis → MySQL。

```go
func main() {
    cfg, err := config.Load("configs", os.Getenv("MALL_APP_ENV"))
    if err != nil {
        log.Fatalf("load config: %v", err)
    }
    lg, err := logger.New(cfg.Log) // 项目内 pkg/logger
    if err != nil {
        log.Fatalf("init logger: %v", err)
    }
    app, cleanup, err := wireApp(cfg, lg)
    if err != nil {
        log.Fatalf("wire: %v", err)
    }
    defer cleanup() // 必须 defer，否则滚动发布连接数只增不减
    if err := app.Run(); err != nil {
        log.Fatalf("run: %v", err)
    }
}
```

单测不走 wire，直接 `NewUserUsecase(fakeRepo)`；需要真实装配时在 `_test.go` 写一个带 `wireinject` 的 injector，把 provider 换成内存实现。

## 完整示例

```yaml
app: {name: user-service, env: dev}
server:
    grpc: {addr: 0.0.0.0:9001, timeout: 2s}
    http: {addr: 0.0.0.0:9101, timeout: 3s}
registry:
    type: etcd
    etcd: {endpoints: ["127.0.0.1:2379"]}
mysql:
    dsn: "root:root@tcp(127.0.0.1:3306)/mall?charset=utf8mb4&parseTime=True&loc=Local"
    max_open_conns: 100
redis: {addr: "127.0.0.1:6379", password: "", db: 0}
kafka: {brokers: ["127.0.0.1:9092"]}
log: {level: info, format: json}
sentinel: {enabled: true}
```

## 常见坑

1. **`AutomaticEnv` 后直接 `Unmarshal`**：环境变量被忽略，K8s 注入的 DSN 不生效；关键 key 一律 `BindEnv`。
2. **`BindEnv` 传大写或 `_`**：匹配不到配置项，注入静默失败；必须写小写点分路径 `mysql.dsn`。
3. **provider 的 error 被忽略或丢弃 cleanup**：连接池不关闭，滚动发布打满 MySQL；`main` 必须判 error，`defer cleanup()` 与 `app.Stop` 配对。
4. **同一类型多个 provider**：报 `multiple bindings`；接口用 `wire.Bind`，重复实现拆到独立 ProviderSet。
5. **循环依赖**：A 需要 B、B 又需要 A，wire 直接报错；抽中间层（如 `Data` 聚合）打破环。
6. **改了 `wire.go` 却忘记跑 wire**：`wire_gen.go` 是旧的却仍编译通过；CI 加 `wire ./... && git diff --exit-code`。

## 小结

- wire 把装配提到编译期：`ProviderSet` 按包分组，`wire.go` 只声明，`wire_gen.go` 提交但不手改
- Viper 三层合并 + 显式 `BindEnv` + `Validate`，是配置类线上事故的三个必做项

## 练习

1. 给 gateway 写 injector，注入 `*gin.Engine`、`userv1.UserServiceClient`、`*http.Server`，生成后跑通启动。
2. 只设 `MALL_MYSQL_DSN` 启动 user-service 验证生效；删掉 `BindEnv` 再跑一次做对照。

## 官方文档

- google/wire：<https://github.com/google/wire>
- Viper 官方文档：<https://github.com/spf13/viper>
- 12-Factor 配置原则：<https://12factor.net/zh_cn/config>
