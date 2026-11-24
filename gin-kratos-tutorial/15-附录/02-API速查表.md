# 02-API速查表

> 一句话定位：把 Gin、Kratos、GORM、Viper、zap、Sentinel Go、go-redis 的高频 API 压成「用途 | API | 说明」表格，写代码时按主题直接查，不用翻全文。

## 本篇目标

- 一张表查完 Gin 的路由、中间件、绑定、响应与错误处理。
- 覆盖 Kratos 的服务端、客户端、errors 与 middleware 四类 API。
- 给出 GORM、Viper、zap、Sentinel、go-redis 的最常用方法，并标出易错返回值与参数。

## 前置知识

- 已读 [Gin 路由](../03-基本特性/01-路由.md)、[Gin 中间件](../03-基本特性/05-中间件.md)、[Kratos 快速入门与 Proto](../12-微服务与Kratos/02-Kratos快速入门与Proto.md)；版本以教程基线为准（Gin v1.10 / Kratos v2.8 / GORM v1.25）。

## 一、Gin：路由与中间件

| 用途 | API | 说明 |
| --- | --- | --- |
| 创建引擎 | `gin.New()` / `gin.Default()` | 前者无中间件，后者含 Logger + Recovery |
| 全局中间件 | `r.Use(gin.Recovery(), requestID())` | 必须在注册路由前调用 |
| 路由组 | `api := r.Group("/api/v1")` | 组内共享前缀，`api.Use` 只作用该组 |
| 注册路由 | `r.GET("/users/:id", h)` | 还有 POST/PUT/DELETE/PATCH，`r.Any` 匹配全部 |
| 路径参数 | `"/files/*filepath"` + `c.Param("filepath")` | `:` 匹配单段，`*` 匹配剩余路径 |
| 兜底 | `r.NoRoute(h)` / `r.NoMethod(h)` | 405 需先设 `r.HandleMethodNotAllowed = true` |
| 中间件签名 | `func(c *gin.Context)` | 不是 `http.HandlerFunc`，标准库中间件要包一层 |
| 控制链 | `c.Next()` / `c.Abort()` | 引擎最外层已调用 Next，`Abort` 阻止后续 handler |
| 跨协程 | `c2 := c.Copy()` | 禁止在 goroutine 里直接用原 Context |

## 二、Gin：绑定、校验与响应

| 用途 | API | 说明 |
| --- | --- | --- |
| JSON | `c.ShouldBindJSON(&req)` | 失败只返回 error，不写响应 |
| Query | `c.ShouldBindQuery(&req)` | tag 用 `form` |
| 路径 | `c.ShouldBindUri(&req)` | tag 用 `uri` |
| 自动选择 | `c.ShouldBind(&req)` | 按 Content-Type 选来源 |
| 校验 | `binding:"required,min=1,email"` | validator v10，切片校验加 `dive` |
| 错误翻译 | `err.(validator.ValidationErrors)` | 转成业务码与中文提示 |
| 统一响应 | `c.JSON(200, gin.H{"code": 0, "data": v})` | 全站固定 `{code,message,data,request_id}` |
| SSE | `c.SSEvent("message", data)` | 需自行 `Flush`，见 03-15 |
| 错误出口 | `c.Error(err)` + `c.Abort()` | 由 `ErrorHandler()` 中间件统一输出 |

## 三、Kratos：服务端与客户端

| 用途 | API | 说明 |
| --- | --- | --- |
| 应用 | `kratos.New(kratos.Name("user-service"), kratos.Logger(l), kratos.Server(gs), kratos.Registrar(reg))` | 返回 `*kratos.App` |
| 启停 | `app.Run()` / `app.Stop(ctx)` | 自带信号处理，不要另写 `signal.Notify` |
| gRPC 服务 | `grpc.NewServer(grpc.Address(":9001"), grpc.Timeout(2*time.Second), grpc.Middleware(...))` | 微服务标准入口 |
| 注册实现 | `v1.RegisterUserServiceServer(gs, svc)` | protoc 生成，禁止手写 |
| 统一响应 | `ctx.Result(200, reply)` / `ctx.JSON(200, v)` | HTTP transport 专用 |
| 服务发现客户端 | `grpc.DialInsecure(ctx, grpc.WithEndpoint("discovery:///user-service"), grpc.WithDiscovery(reg))` | 启动时建连一次 |
| 客户端选项 | `grpc.WithTimeout(2*time.Second)` / `grpc.WithMiddleware(...)` | 超时预算要小于上游 |
| 注册中心 | `etcdreg.New(cli, etcdreg.RegisterTTL(15*time.Second))` | 构造函数只返回 `*Registry` |

## 四、Kratos：errors 与 middleware

| 用途 | API | 说明 |
| --- | --- | --- |
| 构造错误 | `errors.New(400, "USER_NOT_FOUND", "user not found")` | code 与 HTTP/gRPC 码有映射 |
| 常用构造 | `errors.BadRequest/Unauthorized/Forbidden/NotFound/Conflict/InternalServer/ServiceUnavailable(reason, msg)` | reason 全大写、下划线分隔 |
| 限流错误 | `errors.New(429, "RATE_LIMITED", "too many requests")` | 没有内置 429 构造器，需自建 |
| 解析 | `errors.FromError(err)` / `errors.Code(err)` / `errors.Reason(err)` | 网关据此映射业务码 |
| 中间件签名 | `func(middleware.Handler) middleware.Handler` | `Handler = func(ctx, req) (reply, error)` |
| 服务端中间件 | `grpc.Middleware(recovery.Recovery(), logging.Server(l), tracing.Server())` | 顺序即洋葱模型 |
| 客户端中间件 | `grpc.WithMiddleware(recovery.Recovery(), tracing.Client())` | 重试放在熔断之后 |

## 五、GORM

| 用途 | API | 说明 |
| --- | --- | --- |
| 打开 | `gorm.Open(mysql.Open(dsn), &gorm.Config{})` | 建议开 `TranslateError: true` |
| 带上下文 | `db.WithContext(ctx)` | 每个请求都必须带 |
| 连接池 | `sqlDB, _ := db.DB(); sqlDB.SetMaxOpenConns(100)` | 再设 `SetMaxIdleConns`、`SetConnMaxLifetime` |
| 创建 | `db.Create(&u)` | 回填主键与时间戳 |
| 单条 | `db.First(&u, id)` / `db.Take(&u)` | `First` 按主键升序 |
| 列表 | `db.Where("status = ?", 1).Find(&list)` | 占位符传参，禁止拼接 SQL |
| 乐观锁 | `Where("version = ?", v).UpdateColumn("version", gorm.Expr("version + 1"))` | 必须检查 `RowsAffected` |
| 事务 | `db.Transaction(func(tx *gorm.DB) error { ... })` | 返回 error 自动回滚 |
| 预加载 | `db.Preload("Orders").Find(&users)` | 解决 N+1 |

## 六、Viper 与 zap

| 用途 | API | 说明 |
| --- | --- | --- |
| 读配置 | `v.SetConfigFile("configs/gateway.yaml")` + `v.ReadInConfig()` | 失败直接退出进程 |
| 环境变量 | `v.SetEnvPrefix("MALL")` + `v.AutomaticEnv()` | 前缀即 `MALL_` |
| key 映射 | `v.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))` | `mysql.dsn` → `MALL_MYSQL_DSN` |
| 显式绑定 | `v.BindEnv("mysql.dsn")` | `AutomaticEnv` 对 `Unmarshal` 不生效 |
| 解析结构体 | `v.Unmarshal(&cfg)` | 字段 tag 用 `mapstructure` |
| 热更新 | `v.WatchConfig()` + `v.OnConfigChange(fn)` | DSN 等连接参数变更需重建连接 |
| zap 实例 | `zap.NewProduction()` 或 `zap.New(core, zap.AddCaller(), zap.AddStacktrace(zap.ErrorLevel))` | 容器输出 JSON 到 stdout |
| 对接 Kratos | `kratoszap.NewLogger(logger)` | 用 `contrib/log/zap/v2`，不要另起一套 |

## 七、Sentinel 与 go-redis

| 用途 | API | 说明 |
| --- | --- | --- |
| 初始化 | `sentinel.InitDefault()` | 进程启动时调用一次 |
| 流控规则 | `flow.LoadRules([]*flow.Rule{{Resource: "GET:/api/v1/orders", Threshold: 100, ControlBehavior: flow.Reject}})` | 资源名用「方法:路径」 |
| 埋点 | `e, err := sentinel.Entry(res, sentinel.WithTrafficType(base.Inbound))` | 必须配对 `defer e.Exit()` |
| 熔断规则 | `circuitbreaker.LoadRules([]*circuitbreaker.Rule{{Strategy: circuitbreaker.ErrorRatio, Threshold: 0.5, MinRequestAmount: 10, StatIntervalMs: 10000, RetryTimeoutMs: 5000}})` | 熔断打开快速失败 |
| 判定限流 | `var b *base.BlockError; errors.As(err, &b)` | 返回 429 并带 `Retry-After` |
| Redis 客户端 | `redis.NewClient(&redis.Options{Addr: "127.0.0.1:6379", PoolSize: 50})` | 全局复用，不每请求新建 |
| 读写 | `rdb.Set(ctx, k, v, 10*time.Minute)` / `rdb.Get(ctx, k).Result()` | 不存在返回 `redis.Nil` |

## 完整示例：Gin 网关的一条完整请求

```go
// cmd/demo/main.go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type uriReq struct {
	ID string `uri:"id" binding:"required,numeric"`
}

func main() {
	r := gin.New()
	r.Use(gin.Recovery())
	r.SetTrustedProxies(nil)
	r.GET("/api/v1/users/:id", func(c *gin.Context) {
		var req uriReq
		if err := c.ShouldBindUri(&req); err != nil {
			c.AbortWithStatusJSON(http.StatusBadRequest, gin.H{"code": 40001, "message": "invalid id"})
			return
		}
		c.JSON(http.StatusOK, gin.H{"code": 0, "message": "ok", "data": req.ID})
	})
	_ = r.Run(":8080")
}
```

## 常见坑

1. **`c.Next()` 理解错**：Gin 最外层已调用 Next，中间件不写 Next 时后续 handler 仍会执行；Next 的作用是「先跑后续，再回来做后置逻辑」。
2. **`c.Abort()` 后继续写响应**：Abort 只阻止后续 handler，已写的响应头改不了，务必先判断再输出。
3. **在 goroutine 里用原 Context**：必须 `c.Copy()`，否则 Context 被复用后取到别人的数据。
4. **改 context 只赋局部变量**：要写回 `c.Request = c.Request.WithContext(ctx)`，否则下游拿不到 trace_id。
5. **Viper 环境变量不生效**：`AutomaticEnv` 只影响 `Get*`，`Unmarshal` 前必须对关键 key `BindEnv`。
6. **Redis 的 nil 判错**：`Get` 不存在返回 `redis.Nil` 而非 nil error，用 `errors.Is(err, redis.Nil)` 判定未命中。

## 小结

- Gin 侧记牢三点：中间件是 `func(*gin.Context)`、绑定用 `ShouldBind*`、错误用 `c.Error` + 中间件统一出口。
- Kratos 侧统一走 `kratos.New` 启停、`grpc.DialInsecure` + Discovery 建连、`errors.New` 传业务码。
- 数据层所有查询都带 `WithContext`，事务与行锁写在同一 `Transaction` 闭包里。
- 表格只列高频项，完整签名以 [官方文档索引](01-官方文档索引.md) 的 pkg.go.dev 链接为准。

## 练习

1. 用本篇表格实现一个 `/api/v1/orders/:id` 接口，要求路径参数校验失败返回 40001。
2. 把 GORM 事务示例改造成「扣库存 + 建订单 + 发 Kafka 消息」，思考消息发送失败时如何处理。
3. 给 Sentinel 埋点补一个熔断分支，验证熔断打开时返回 503 而不是 429。

## 官方文档

- [Gin 中文文档](https://gin-gonic.com/zh-cn/docs/)
- [Kratos 官方文档](https://go-kratos.dev/docs/)
- [GORM 中文文档](https://gorm.io/zh_CN/docs/)
- [go-redis 中文文档](https://redis.uptrace.dev/zh/)
- [Sentinel Go 快速开始](https://sentinelguard.io/zh-cn/docs/golang/quick-start.html)

延伸阅读：[官方文档索引](01-官方文档索引.md) · [技术选型与版本基线](03-技术选型与版本基线.md) · [常见问题 FAQ](../14-FAQ与术语表/01-常见问题FAQ.md) · [术语表](../14-FAQ与术语表/02-术语表.md)
