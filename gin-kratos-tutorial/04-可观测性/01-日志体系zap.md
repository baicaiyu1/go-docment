# 日志体系 zap

> 一句话定位：业务日志、访问日志、框架日志统一走 zap 的结构化 JSON，字段固定、可脱敏、可采样，再用 `trace_id` 与 `request_id` 把日志、指标、追踪串成一条线。

## 本篇目标

- 说清为什么不直接用标准库 `log`，以及 `NewProduction` 与 `NewDevelopment` 的差异
- 写出可复用的 `pkg/logger`：JSON Encoder、级别控制、lumberjack 轮转、错误堆栈
- 实现 Gin 访问日志中间件，固定输出 `service/env/request_id/trace_id/latency_ms/http.*`
- 在 Kratos 侧用 `contrib/log/zap/v2` 复用同一个 zap Logger，并掌握脱敏、采样与测试

## 前置知识

- 已读 [05-中间件](../03-基本特性/05-中间件.md)，知道 `c.Next()` 之后才能拿到最终状态码
- 了解 `context.Context` 的传递方式；知道结构化日志指固定字段的键值对，不是拼字符串

## 一、为什么不用标准库 log

标准库 `log` 只有「时间 + 文本」：没有级别、没有字段、不能 JSON、定位不到调用点。用 `log.Printf("uid=%d cost=%d")` 拼出来的日志，检索时只能写正则，告警规则也无法按 `status>=500` 过滤。logrus 基于反射、比 zap 慢一个量级；标准库 `slog` 可用，但 Kratos 官方适配是 zap，整栈统一最省事。

| 需求 | 标准库 log | zap |
| --- | --- | --- |
| 按级别过滤 | 无级别，只能全打 | Debug~Fatal，可动态调整 |
| 按字段检索 | 正则拆字符串 | JSON 字段直接建索引 |
| 性能 | 每次 `fmt` 反射 + 分配 | 强类型 `zapcore.Field`，热点路径近乎零分配 |

## 二、Production / Development 与自定义 EncoderConfig

| 维度 | `NewDevelopment` | `NewProduction` |
| --- | --- | --- |
| 编码 / 级别 | console / Debug | json / Info |
| 采样 | 无 | 每秒前 100 条全量，之后 100 条采 1 条 |

线上要求「stdout + JSON + 字段固定 + 轮转」，所以不用快捷函数，自己组装 core：

```go
// pkg/logger/logger.go
type Options struct {
    Service, Env, Level, File string
    MaxSize, MaxBackups, MaxAge       int
}
func New(o Options) (*zap.Logger, error) {
    lvl := zap.NewAtomicLevel()
    if err := lvl.UnmarshalText([]byte(o.Level)); err != nil {
        return nil, err
    }
    encCfg := zapcore.EncoderConfig{
        TimeKey: "ts", LevelKey: "level", CallerKey: "caller", MessageKey: "msg",
        StacktraceKey: "stacktrace", LineEnding: zapcore.DefaultLineEnding,
        EncodeLevel: zapcore.LowercaseLevelEncoder, EncodeTime: zapcore.ISO8601TimeEncoder,
        EncodeDuration: zapcore.MillisDurationEncoder, EncodeCaller: zapcore.ShortCallerEncoder,
    }
    enc := zapcore.NewJSONEncoder(encCfg)
    ws := zapcore.AddSync(os.Stdout)
    if o.File != "" {
        ws = zapcore.NewMultiWriteSyncer(ws, zapcore.AddSync(&lumberjack.Logger{
            Filename: o.File, MaxSize: o.MaxSize, MaxBackups: o.MaxBackups, MaxAge: o.MaxAge, Compress: true,
        }))
    }
    core := zapcore.NewSamplerWithOptions(zapcore.NewCore(enc, ws, lvl), time.Second, 100, 100)
    return zap.New(core, zap.AddCaller(), zap.AddStacktrace(zapcore.ErrorLevel),
        zap.Fields(zap.String("service", o.Service), zap.String("env", o.Env))), nil
}
```

`zap.Fields` 是全局字段，之后每条日志都带 `service`/`env`，业务里不用重复传。

## 三、字段规范

字段乱起名，Loki/Grafana 就无法聚合。约定：顶层 snake_case，HTTP 相关用点号分组，总数控制在 15 个以内。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `service` / `env` | string | 服务名与注册中心一致；环境 `dev/test/prod` |
| `request_id` | string | 单次请求 ID，来自 `X-Request-Id` 或生成 |
| `trace_id` / `span_id` | string | 来自 OTel `SpanContext`，见 [02-链路追踪](02-链路追踪OpenTelemetry与Jaeger.md) |
| `user_id` | string | 鉴权后写入，禁止写手机号/身份证 |
| `latency_ms` | float64 | 统一毫秒，不要混用 us |
| `http.method` / `http.path` / `http.status` | string/int | 路径用路由模板，不用原始 URL |

## 四、Gin 访问日志中间件

```go
// app/gateway/internal/middleware/accesslog.go
const CtxRequestID, CtxLogger, CtxUserID = "meta.request_id", "meta.logger", "meta.user_id"
func AccessLog(base *zap.Logger) gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        rid := cmp.Or(c.GetHeader("X-Request-Id"), uuid.NewString())
        c.Set(CtxRequestID, rid)
        c.Header("X-Request-Id", rid)
        lg := base.With(zap.String("request_id", rid))
        if sc := trace.SpanFromContext(c.Request.Context()).SpanContext(); sc.IsValid() {
            lg = lg.With(zap.String("trace_id", sc.TraceID().String()),
                zap.String("span_id", sc.SpanID().String()))
        }
        c.Set(CtxLogger, lg)
        c.Next() // 先跑完后续，才能拿到最终状态码与字节数
        fields := []zap.Field{
            zap.String("http.method", c.Request.Method),
            zap.String("http.path", cmp.Or(c.FullPath(), "unmatched")), // 404 时 FullPath 为空
            zap.Int("http.status", c.Writer.Status()), zap.Int("http.bytes", c.Writer.Size()),
            zap.Float64("latency_ms", float64(time.Since(start).Microseconds())/1000),
        }
        if uid, ok := c.Get(CtxUserID); ok { fields = append(fields, zap.Any("user_id", uid)) }
        if len(c.Errors) > 0 { fields = append(fields, zap.String("error", c.Errors.Last().Error())) }
        if c.Writer.Status() >= 500 {
            lg.Error("http.access", fields...)
            return
        }
        lg.Info("http.access", fields...)
    }
}
```

业务 handler 里通过 `c.MustGet(CtxLogger).(*zap.Logger)` 取带上下文的 logger。注意 `AccessLog` 必须注册在 `otelgin.Middleware` 之后，否则取不到 span；`c.Writer.Size()` 在 `Next` 前是 -1。

## 五、Kratos 侧复用同一个 zap

Kratos 用 `log.Logger` 接口，官方 contrib 提供 zap 适配，不要再自造日志抽象：

```go
// app/user-service/cmd/main.go
import (
    klog "github.com/go-kratos/kratos/contrib/log/zap/v2"
    "github.com/go-kratos/kratos/v2"
    "github.com/go-kratos/kratos/v2/log"
)
func main() {
    zl, _ := logger.New(logger.Options{Service: "user-service", Env: "dev", Level: "info"})
    defer zl.Sync()
    lg := log.With(klog.NewLogger(zl), "service", "user-service")
    app := kratos.New(
        kratos.Name("user-service"), kratos.Version("v1.0.0"), kratos.Logger(lg),
        kratos.Server(newGRPCServer(lg)), // 内部 grpc.Middleware(logging.Server(lg), tracing.Server())
    )
    if err := app.Run(); err != nil {
        panic(err)
    }
}
```

`klog.NewLogger` 已处理 `level`/`caller`/`keyvals`，`logging.Server(lg)` 会自动为每次 gRPC 调用打一条带 `span_id`、`kind`、`args` 的日志，框架日志与业务日志共享同一份输出配置。

## 六、脱敏、级别与采样

泄密大多不是故意打印密码，而是 `zap.Any("req", req)` 把整个结构体序列化了。规则：请求体默认不打印；Header 只打白名单；手机号/邮箱/身份证用固定函数处理。

```go
func maskPhone(s string) string {
    if len(s) != 11 { return "***" }
    return s[:3] + "****" + s[7:]
}

func safeHeaders(h http.Header) map[string]string {
    out := map[string]string{}
    for _, k := range []string{"User-Agent", "Content-Type", "X-Request-Id"} {
        if v := h.Get(k); v != "" { out[k] = v }
    }
    return out // Authorization / Cookie 永远不出现
}
```

级别用 `zap.AtomicLevel` 支持运行时调整：把 `lvl` 提升为包级变量后，任意位置 `lvl.SetLevel(zapcore.DebugLevel)` 生效；调级别接口只开在内网管理端口并加鉴权。采样器按 `(level, message)` 计数，所以 `msg` 要稳定，别把变量拼进 `msg`，否则采样形同虚设。

## 七、测试日志

不要断言日志文本，用 `zap/zaptest/observer` 拿结构化条目再断言字段：

```go
func TestAccessLog(t *testing.T) {
    gin.SetMode(gin.TestMode)
    core, observed := observer.New(zapcore.InfoLevel)
    r := gin.New()
    r.Use(AccessLog(zap.New(core)))
    r.GET("/users/:id", func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"code": 0}) })
    r.ServeHTTP(httptest.NewRecorder(), httptest.NewRequest(http.MethodGet, "/users/42", nil))

    require.Equal(t, 1, observed.Len())
    entry := observed.All()[0]
    require.Equal(t, "http.access", entry.Message)
    require.Equal(t, "/users/:id", entry.ContextMap()["http.path"])
}
```

## 完整示例
`app/gateway/cmd/main.go` 的装配顺序（otelgin → AccessLog → 业务路由）：

```go
func main() {
    zl, _ := logger.New(logger.Options{Service: "gateway", Env: "dev", Level: "info"})
    defer zl.Sync()
    zap.ReplaceGlobals(zl) // 三方库用 zap.L() 时也统一输出
    r := gin.New()
    r.Use(otelgin.Middleware("gateway"), middleware.AccessLog(zl), gin.Recovery())
    r.Group("/api/v1").GET("/users/:id", userHandler)
    if err := http.ListenAndServe(":8080", r); err != nil {
        zl.Fatal("server exit", zap.Error(err))
    }
}
```

## 常见坑

1. **采样把错误日志也采样掉**：`NewSamplerWithOptions` 作用于所有级别；只对 Info/Debug 建采样 core，Error 走独立 core。
2. **`c.Writer.Status()` 在 `c.Next()` 之前读**：永远拿到 200，错误率全是 0。
3. **日志没有 `trace_id`**：`AccessLog` 注册在 `otelgin.Middleware` 之前，或 handler 换成了 `context.Background()`。
4. **`zap.Any` 打印整个请求体**：手机号、密码、token 全部落盘；只打白名单字段，需要时先脱敏再截断。
5. **`defer zl.Sync()` 之后调 `os.Exit`**：`os.Exit` 不执行 defer，最后一批日志丢失；先 `Sync` 再退出。

## 小结

- 业务代码禁止 `fmt.Println`/`log.Printf`，统一 zap 结构化输出
- 生产用自定义 EncoderConfig：JSON、ISO8601 时间、毫秒耗时、Error 带堆栈
- 字段名固定，`service/env/request_id/trace_id/latency_ms/http.*` 是排查最小集合
- Kratos 用 `contrib/log/zap/v2` 适配，框架日志与业务日志共用一份输出

## 练习

1. 把 `pkg/logger` 接进网关，请求一次接口，确认 stdout 里能按 `request_id` 搜到访问日志。
2. 给 `AccessLog` 增加 `user_id` 读取，并写用例断言该字段存在。
3. 用 `AtomicLevel.SetLevel` 运行时从 `info` 切到 `debug`，观察 Debug 日志出现；再让 handler `panic`，确认堆栈日志与访问日志共用 `request_id`。

## 官方文档

- zap 使用与配置：<https://pkg.go.dev/go.uber.org/zap>
- zap FAQ（采样、caller、性能）：<https://github.com/uber-go/zap/blob/master/FAQ.md>
- lumberjack 轮转：<https://github.com/natefinch/lumberjack>
- Kratos logging 中间件：<https://github.com/go-kratos/kratos/tree/main/middleware/logging>
- Kratos zap 适配：<https://github.com/go-kratos/kratos/tree/main/contrib/log/zap>
