# 链路追踪 OpenTelemetry 与 Skywalking
> 一句话定位：用 OpenTelemetry 给 Hertz 服务装上“全链路 GPS”，并说清 SkyWalking 场景下的可取方案。
## 本篇目标
- 理解 Trace、Span、Context 与 W3C `traceparent`
- 用 `hertz-contrib/obs-opentelemetry` 完成服务端/客户端埋点
- 导出到 Jaeger 或 Tempo，完成跨服务串联
- 掌握采样率配置，控制存储与带宽成本
- 了解 SkyWalking 与 Hertz 的现状与替代方案
## 前置知识
- 已读 [01-日志体系hlog](01-日志体系hlog.md)
- 了解 HTTP Header 透传与 Go 的 `context.Context`
- 本地有 Docker（用于起 Jaeger/Tempo）
## 一、四个核心概念
| 概念 | 含义 | Hertz 中的对应物 |
| --- | --- | --- |
| Trace | 一次完整调用链，同一个 trace_id | 入口请求到所有下游 |
| Span | 链路中的一段操作 | 一次 HTTP 请求、一次 DB 查询 |
| Context | 携带当前 span 与传播信息 | `context.Context` |
| Propagation | 跨进程传递上下文 | HTTP Header（W3C/B3） |

```text
trace_id = 4bf92f3577b34da6a3ce929d0e0e4736
├── span A  GET /api/order      (Hertz server, 120ms)
│   ├── span B  GET /user/42    (Hertz client → 用户服务, 30ms)
│   │   └── span C  SELECT users (MySQL, 8ms)
│   └── span D  PUB order.created (Kafka, 2ms)
```

W3C 标准传播头：

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^  ^                                ^                ^
             |  trace-id (32 hex)                span-id (16 hex) flags(01=采样)
             version
```

`obs-opentelemetry` 默认使用 W3C propagator，自动在客户端注入、服务端提取。
## 二、服务端接入
```bash
go get github.com/hertz-contrib/obs-opentelemetry
go get go.opentelemetry.io/otel/sdk
```

```go
package main

import (
	"context"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/common/utils"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
	"github.com/hertz-contrib/obs-opentelemetry/provider"
	hertztracing "github.com/hertz-contrib/obs-opentelemetry/tracing"
)

func main() {
	p := provider.NewOpenTelemetryProvider(
		provider.WithServiceName("order-service"),
		provider.WithExportEndpoint("localhost:4317"), // OTLP gRPC
		provider.WithInsecure(),
	)
	defer p.Shutdown(context.Background())

	tracer, cfg := hertztracing.NewServerTracer()
	h := server.Default(tracer) // tracer 必须作为 server 选项
	h.Use(hertztracing.ServerMiddleware(cfg))

	h.GET("/ping", func(ctx context.Context, c *app.RequestContext) {
		c.JSON(consts.StatusOK, utils.H{"message": "pong"})
	})
	h.Spin()
}
```

两个容易混淆的点：

- `tracer` 是 `server.Option`，决定请求开始/结束时是否记录 span
- `ServerMiddleware(cfg)` 是应用层中间件，负责补充路由、状态码等属性

只做其中一个，链路要么没有 span，要么缺路由和状态码。
## 三、客户端接入与跨服务串联
```go
import hertztracing "github.com/hertz-contrib/obs-opentelemetry/tracing"

c, _ := client.NewClient()
c.Use(hertztracing.ClientMiddleware())

status, body, err := c.Get(ctx, nil, "http://user-service:8888/user/42")
```

串联成立的三个前提：

1. 两端使用同一种传播格式（默认 W3C）
2. 调用必须传链路 ctx，不能新建 `context.Background()`
3. 下游也接了 OTel，且采样能继承（ParentBased）

`obs-opentelemetry` 还会记录 `peer.service`，因此在 Jaeger 里可以直接看到服务拓扑。
## 四、导出到 Jaeger / Tempo
Jaeger（OTLP 直收）：

```yaml
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports: ["16686:16686", "4317:4317", "4318:4318"]
    environment: ["COLLECTOR_OTLP_ENABLED=true"]
```

Tempo 关键配置：

```yaml
server:
  http_listen_port: 3200
distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
```

```bash
docker compose up -d
```

Jaeger UI：`http://localhost:16686`，Service 选 `order-service` 即可查链路。

切换后端用 OTel 标准环境变量，不改代码：

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo:4317
export OTEL_SERVICE_NAME=order-service
```

```powershell
$env:OTEL_EXPORTER_OTLP_ENDPOINT = "http://tempo:4317"
$env:OTEL_SERVICE_NAME = "order-service"
```

生产由 K8s Deployment 注入，代码不写死 endpoint。
## 五、采样率
| 策略 | 说明 | 适用 |
| --- | --- | --- |
| AlwaysSample | 全量采集 | 开发、低流量 |
| TraceIDRatioBased | 按 trace_id 比例 | 生产常用，1%~10% |
| ParentBased | 跟随上游决定 | 微服务链路，避免断链 |

```go
import (
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	"github.com/hertz-contrib/obs-opentelemetry/provider"
)

p := provider.NewOpenTelemetryProvider(
	provider.WithServiceName("order-service"),
	provider.WithExportEndpoint("localhost:4317"),
	provider.WithInsecure(),
	provider.WithSampler(
		sdktrace.ParentBased(sdktrace.TraceIDRatioBased(0.05)), // 5%
	),
)
```

建议：入口网关按 1%~10% 采样，下游一律 `ParentBased`；采样率调整纳入发布，不要临时改代码上线。
## 六、SkyWalking 场景怎么接
结论：**SkyWalking Go Agent 官方插件列表目前不包含 Hertz**（已有 gin、net/http、fasthttp、fiber 等，以官方 support-plugins 页面为准）。不要照搬 gin 插件写法。

可选方案，按推荐度排序：

1. **OTLP 上报到 SkyWalking OAP**
   SkyWalking OAP 提供 OpenTelemetry receiver，把 OTLP 数据转换为自身的 trace/metric。Hertz 侧仍用 `obs-opentelemetry`，只改 endpoint：

```go
p := provider.NewOpenTelemetryProvider(
	provider.WithServiceName("order-service"),
	provider.WithExportEndpoint("skywalking-oap:11800"), // 以 OAP 实际接收端口为准
	provider.WithInsecure(),
)
```

2. **用 SkyWalking Go 的 trace toolkit 手动埋点**
   在 Hertz 中间件里创建 entry span、在客户端中间件创建 exit span，并注入 `sw8` header。工作量大，适合必须复用 SkyWalking 原生协议的团队。

3. **统一到 OpenTelemetry 再桥接**
   新项目以 OTel 为事实标准，SkyWalking 只作为后端之一；未来换 Jaeger/Tempo 不动业务代码。

如果公司强制使用 SkyWalking UI 与告警，方案 1 成本最低；方案 2 只在必须使用原生 agent 能力时考虑。
## 七、异步与并发
```go
// 错误：异步任务丢失链路
go func() { doSomething(context.Background()) }()

// 正确：显式传入请求 ctx
go func(ctx context.Context) { doSomething(ctx) }(ctx)
```

请求结束、ctx 被 cancel 后，异步任务里的 span 可能提前结束。Go 1.21+ 可用 `context.WithoutCancel(ctx)` 保留值、去掉取消信号：

```go
asyncCtx := context.WithoutCancel(ctx)
go func() {
	defer span.End()
	doSomething(asyncCtx)
}()
```

消息队列同理：生产端把 `traceparent` 写入消息 header，消费端提取后创建 consumer span，不要假设框架自动完成。
## 常见坑
1. 只传 tracer 没加 ServerMiddleware：有 span 但缺路由、状态码、peer 信息。
2. 客户端调用用 `context.Background()`：下游生成新 trace，链路断成两条。
3. 两端传播格式不一致（W3C vs B3）：下游有 trace 但接不上。
4. 异步 goroutine 丢 ctx：日志有 trace id，异步 span 却是孤儿。
5. 采样率 100% 上生产：把 collector 与后端打满，服务延迟抖动。
6. `defer p.Shutdown` 放在 `h.Spin()` 之后：Spin 阻塞，退出时丢最后一批 span。
7. 直接套用 SkyWalking gin 插件：插件不生效甚至编译失败，Hertz 不实现 gin 接口。
## 小结
- OTel 的核心是 Trace/Span/Context/Propagation，W3C `traceparent` 是串联关键
- Hertz 接入 = Provider + ServerTracer + ServerMiddleware + ClientMiddleware
- Jaeger/Tempo 都吃 OTLP，切换后端靠环境变量
- 采样用 `ParentBased + Ratio`，链路上不要出现“上游采、下游不采”
- SkyWalking 目前无官方 Hertz 插件，优先走 OTLP receiver
## 练习
1. 起 Jaeger，接入示例服务，观察 `/ping` 的 span 结构与耗时。
2. 写两个 Hertz 服务 A→B，验证 Jaeger 中是一条完整链路。
3. 采样率改成 5%，发 200 次请求，统计实际出现的 trace 数。
4. 在 handler 中用 `context.Background()` 启动 goroutine，复现断链并修复。
## 官方文档
- 链路追踪：<https://www.cloudwego.io/zh/docs/hertz/tutorials/observability/tracing/>
- OpenTelemetry 扩展：<https://www.cloudwego.io/zh/docs/hertz/tutorials/third-party/open-telemetry/>
- hertz-contrib/obs-opentelemetry：<https://github.com/hertz-contrib/obs-opentelemetry>
- W3C Trace Context：<https://www.w3.org/TR/trace-context/>
- SkyWalking Go 插件列表：<https://skywalking.apache.org/docs/skywalking-go/latest/en/agent/support-plugins/>
