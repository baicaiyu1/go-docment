# 链路追踪 OpenTelemetry 与 Jaeger

> 一句话定位：用 OpenTelemetry SDK 统一埋点，Gin 侧挂 `otelgin`、Kratos 侧挂 `tracing.Server/Client`，通过 W3C `traceparent` 跨 HTTP 与 gRPC 串联，最后把 `trace_id` 写回日志。

## 本篇目标

- 讲清 Trace、Span、SpanContext、Propagator 四个概念及 W3C `traceparent` 结构
- 搭一个 OTLP gRPC 的 TracerProvider，导出到 Jaeger，并正确 Shutdown
- 在 Gin 与 Kratos 两侧接入追踪，掌握采样、传播，以及 `trace_id` 写入 zap 的做法
- 说清「未 flush、未传播、采样率、异步」四类高频故障

## 前置知识

- 已读 [01-日志体系zap](01-日志体系zap.md)，知道 `request_id` 与 `trace_id` 的分工
- 了解 HTTP Header 与 gRPC Metadata；本地有 Docker，能起 Jaeger

## 一、四个概念与 traceparent

| 概念 | 含义 | 代码对应 |
| --- | --- | --- |
| Trace | 一次完整调用链，共用一个 32 位十六进制 `trace_id` | 入口请求到所有下游 |
| Span | 链路中的一段操作，有 `span_id`、开始/结束时间、属性 | 一次 HTTP 请求、一次 SQL 查询 |
| SpanContext | 跨进程传播所需的最小信息：`trace_id`、`span_id`、采样标记 | `trace.SpanContextFromContext(ctx)` |
| Propagator | 把 SpanContext 写入/读出载体（Header/Metadata） | `propagation.TraceContext{}` |

```text
trace_id = 4bf92f3577b34da6a3ce929d0e0e4736
└── span GET /api/v1/orders                    gateway        120ms
    ├── span /user.v1.UserService/GetUser      user-service    30ms
    └── span publish order.created             kafka           2ms

traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  │                                │                └ flags=01 已采样
             │  │                                └ parent span-id（16 hex）
             │  └ trace-id（32 hex）
             └ version
```

`Propagator` 是「断链」的第一嫌疑人：调用方写 `traceparent`、下游用 B3 解析，链路就会断成两条。本项目统一 W3C TraceContext，HTTP 用 Header、gRPC 用 Metadata，OTel 与 Kratos 都默认支持。

## 二、TracerProvider 与 OTLP gRPC 导出

```go
// pkg/tracing/tracing.go
func Init(ctx context.Context, service, endpoint string, ratio float64) (*sdktrace.TracerProvider, error) {
    exp, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint(endpoint), otlptracegrpc.WithInsecure(), // endpoint 形如 127.0.0.1:4317
        otlptracegrpc.WithTimeout(3*time.Second),
    )
    if err != nil {
        return nil, err
    }
    res, _ := resource.New(ctx, resource.WithAttributes(
        semconv.ServiceName(service), attribute.String("env", "dev")))
    tp := sdktrace.NewTracerProvider(
        sdktrace.WithBatcher(exp, sdktrace.WithBatchTimeout(5*time.Second)),
        sdktrace.WithResource(res),
        sdktrace.WithSampler(sdktrace.ParentBased(sdktrace.TraceIDRatioBased(ratio))),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{}, propagation.Baggage{},
    ))
    return tp, nil
}
```

三个要点：

- `WithEndpoint` 只写 `host:port`，写了 `http://` 会导致 gRPC 拨号失败；带 TLS 时去掉 `WithInsecure`。
- 必须 `otel.SetTracerProvider`，`otelgin` 与 Kratos 的 `tracing` 中间件都从全局取 Provider。
- `WithBatcher` 是异步批量上报，进程退出前必须 `Shutdown`，否则最后一批 span 全丢；`tp.Shutdown(ctx)` 会阻塞到队列发送完，`ctx` 用 5 秒超时（写法见文末「完整示例」）。

## 三、Gin 与 Kratos 两侧接入

Gin 只需一个中间件，它会提取上游 `traceparent`、创建 server span、把 span 写回 context：

```go
r := gin.New()
r.Use(otelgin.Middleware("gateway"), middleware.AccessLog(zl)) // AccessLog 在后，才能拿到 trace_id
```

注意不要把 `otelgin.Middleware` 放在 `Recovery` 之后：panic 时 span 来不及记录 `status=500`。

Kratos 用官方中间件，服务端与客户端各一个：

```go
// 省略 import：tracing 与 kratosgrpc 分别来自 middleware/tracing、transport/grpc
grpcSrv := kratosgrpc.NewServer(
    kratosgrpc.Address(":9001"),
    kratosgrpc.Middleware(
        recovery.Recovery(),
        tracing.Server(),       // 创建 server span，读取 metadata 里的 traceparent
        logging.Server(lg),
    ),
)

conn, err := kratosgrpc.DialInsecure(ctx,
    kratosgrpc.WithEndpoint("discovery:///user-service"),
    kratosgrpc.WithDiscovery(reg),
    kratosgrpc.WithMiddleware(tracing.Client()), // 创建 client span，注入 traceparent
)
```

`tracing.Server()` 与 `tracing.Client()` 默认使用全局 TracerProvider 与 W3C Propagator，因此只要 `Init` 里设置过一次，整条链路自动打通。

## 四、跨进程传播与采样

HTTP → gRPC 的传播由框架完成，不需要手写 Header。真正需要手写的是消息队列、异步任务这类「非 RPC」载体：

```go
// 生产端：把当前 span 上下文写进消息头
carrier := propagation.MapCarrier{}
otel.GetTextMapPropagator().Inject(ctx, carrier)
msg.Headers = append(msg.Headers, sarama.RecordHeader{Key: []byte("traceparent"), Value: []byte(carrier["traceparent"])})
// 消费端：从消息头恢复上下文，再创建 consumer span
ctx = otel.GetTextMapPropagator().Extract(context.Background(), propagation.MapCarrier{"traceparent": string(h.Value)})
ctx, span := otel.Tracer("order-consumer").Start(ctx, "consume order.created")
defer span.End()
```

采样策略决定成本上限：

| 策略 | 行为 | 适用 |
| --- | --- | --- |
| `AlwaysSample` | 全量采集 | 开发、低流量 |
| `TraceIDRatioBased(0.05)` | 按 trace_id 哈希采 5% | 生产入口 |
| `ParentBased(root)` | 有上游就跟随上游，否则按 root 策略 | 所有下游服务必选 |

下游一律 `ParentBased`，否则会出现「上游采了、下游没采」，链路从中间断掉。网关按 1%~10% 采样，错误率高的接口再单独提高比例；采样率调整走配置发布，不要临时改代码。

## 五、把 trace_id 写进 zap，并起 Jaeger

日志与链路的关联键就是 `trace_id`。Gin 侧从 context 取，Kratos 侧由 `logging.Server` 自动带：

```go
func TraceID(ctx context.Context) string {
    sc := trace.SpanFromContext(ctx).SpanContext()
    if !sc.IsValid() {
        return ""
    }
    return sc.TraceID().String() // 32 hex，与 Jaeger UI 搜索框一致
}

// AccessLog 中
lg = lg.With(zap.String("trace_id", TraceID(c.Request.Context())))
```

Jaeger all-in-one 一条命令即可，OTLP 与 UI 端口都在里面：

```bash
docker run -d --name jaeger \
  -p 16686:16686 -p 4317:4317 -p 4318:4318 \
  -e COLLECTOR_OTLP_ENABLED=true \
  jaegertracing/all-in-one:1.62
```

```powershell
docker run -d --name jaeger -p 16686:16686 -p 4317:4317 -p 4318:4318 -e COLLECTOR_OTLP_ENABLED=true jaegertracing/all-in-one:1.62
```

打开 <http://localhost:16686>，Service 选 `gateway`，用日志里的 `trace_id` 直接搜，就能看到完整瀑布图。

## 完整示例

`app/gateway/cmd/main.go` 的初始化顺序：TracerProvider → Gin → 路由。

```go
func main() {
    ctx := context.Background()
    tp, err := tracing.Init(ctx, "gateway", "127.0.0.1:4317", 0.1)
    if err != nil {
        panic(err)
    }
    defer func() {
        sctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
        defer cancel()
        _ = tp.Shutdown(sctx)
    }()

    zl, _ := logger.New(logger.Options{Service: "gateway", Env: "dev", Level: "info"})
    defer zl.Sync()
    r := gin.New()
    r.Use(otelgin.Middleware("gateway"), middleware.AccessLog(zl), gin.Recovery())
    r.Group("/api/v1").GET("/users/:id", userHandler)
    if err := http.ListenAndServe(":8080", r); err != nil {
        zl.Fatal("server exit", zap.Error(err))
    }
}
```

## 常见坑

1. **没调 `tp.Shutdown`**：批量导出器缓冲里的 span 丢失，表现为「最近几分钟的链路查不到」。
2. **endpoint 写成 `http://127.0.0.1:4317`**：`WithEndpoint` 只接受 `host:port`，带 scheme 直接拨号失败。
3. **只接了 Gin 没接 Kratos，或下游没用 `ParentBased`**：前者下游另起一条 trace，后者上游采样、下游重新抽，链路都会断；两侧都挂中间件，下游统一 `ParentBased`。
4. **`c.Request.Context()` 换成 `context.Background()`**：下游生成新 trace，日志里的 `trace_id` 也对不上。
5. **异步 goroutine 直接引用 `*gin.Context`**：请求结束 context 被回收，span 提前结束；用 `c.Copy()` 并传 `context.WithoutCancel(ctx)`。
6. **`otelgin` 放在 `Recovery` 之后**：panic 请求没有 `status=500` 的 span，错误率统计缺失。

## 小结

- Trace/Span/SpanContext 是数据模型，Propagator 决定跨进程能否接上，W3C 是默认选择
- TracerProvider 负责导出，OTLP gRPC 是标准出口，Jaeger 只是后端之一
- Gin 用 `otelgin.Middleware`，Kratos 用 `tracing.Server()/tracing.Client()`，都读全局 Provider
- `trace_id` 必须写进日志，否则有链路也无法从错误日志定位到具体请求

## 练习

1. 起 Jaeger，接入示例网关，请求 `/api/v1/users/42`，在 UI 里确认 span 名称与耗时。
2. 再起一个 Kratos `user-service`，验证 Jaeger 中网关与下游是同一条 trace。
3. 采样率改成 0.05，发 200 次请求，统计实际落库的 trace 数量是否接近 10 条。
4. 在 handler 里用 `go func(){ ... }()` 加 `context.Background()`，复现孤儿 span 后修复。

## 官方文档

- OpenTelemetry Go：<https://opentelemetry.io/docs/languages/go/>
- otelgin 中间件：<https://github.com/open-telemetry/opentelemetry-go-contrib/tree/main/instrumentation/github.com/gin-gonic/gin/otelgin>
- Kratos tracing 中间件：<https://github.com/go-kratos/kratos/tree/main/middleware/tracing>
- Jaeger 部署与 OTLP：<https://www.jaegertracing.io/docs/>
- W3C Trace Context：<https://www.w3.org/TR/trace-context/>
