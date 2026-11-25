# 埋点 Instrumentation
> 一句话定位：理解 Hertz 自己的“请求生命周期埋点”，知道它和日志、链路追踪、Prometheus 指标如何分工。
## 本篇目标
- 分清“埋点 / 链路追踪 / 指标 / 日志”四件事的边界
- 掌握 `LevelDisabled`、`LevelBase`、`LevelDetailed` 三档埋点
- 看懂 `TraceInfo` 与 `stats.Event`，能取到耗时、包大小、错误、panic
- 写一个自定义 Tracer，把耗时与状态码变成业务指标
- 理解埋点的性能成本与采样策略
## 前置知识
- 已读 [02-链路追踪OpenTelemetry与Skywalking](02-链路追踪OpenTelemetry与Skywalking.md)
- 了解 Hertz 的 `server.WithTracer` 与 `Tracer` 接口
- 知道 Histogram / Counter 的基本含义（不要求用过 Prometheus）
## 一、四个概念先分清
| 能力 | 回答的问题 | Hertz 中的入口 |
| --- | --- | --- |
| 埋点 | 一个请求内部各阶段花了多久、多大、有没有错 | `TraceInfo` / `stats.Event` |
| 链路追踪 | 这次请求经过了哪些服务、哪一步慢 | `Tracer` + OTel/SkyWalking |
| 指标 | 整体 QPS、P99、错误率是多少 | `Tracer` 里记录 Counter/Histogram |
| 日志 | 具体发生了什么、参数是什么 | `hlog` |

埋点是“原料”：链路追踪和指标都从埋点数据里取信息。Hertz 框架自身只负责采集，不负责上报。
## 二、Hertz 的 Tracer 接口
```go
// Tracer 在请求开始和结束时各执行一次
type Tracer interface {
	Start(ctx context.Context, c *app.RequestContext) context.Context
	Finish(ctx context.Context, c *app.RequestContext)
}
```

注册方式：

```go
h := server.Default(server.WithTracer(&myTracer{}))
```

注意时序（官方文档明确指出）：

- `Start` 在**读包之前**执行，此时 `RequestContext` 还是空的
- `Finish` 在**写回数据之后**执行，此时能拿到状态码、包大小、错误
- 多个 Tracer 会被依次执行；任何一个出错/阻塞都会拖慢请求

因此 `Start` 里不要尝试读取 header、path、body；需要这些信息请在中间件里做。
## 三、三档埋点粒度
| 级别 | 记录内容 | 默认行为 |
| --- | --- | --- |
| `LevelDisabled` | 不记录任何事件 | 没有注册 Tracer 时默认 |
| `LevelBase` | 请求开始/结束 | 手动指定时使用 |
| `LevelDetailed` | 基本埋点 + 读 header/body、业务处理、写响应 | 注册了 Tracer 时默认 |

显式指定：

```go
import (
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/common/tracer/stats"
)

h := server.Default(server.WithTraceLevel(stats.LevelBase))
```

事件清单：

| 事件 | 阶段 | 典型用途 |
| --- | --- | --- |
| `HTTPStart` / `HTTPFinish` | 请求整体 | 总耗时 |
| `ReadHeaderStart` / `ReadHeaderFinish` | 读请求头 | 慢客户端、超大 header |
| `ReadBodyStart` / `ReadBodyFinish` | 读请求体 | 大文件上传耗时 |
| `ServerHandleStart` / `ServerHandleFinish` | 业务 handler | 业务处理耗时 |
| `WriteStart` / `WriteFinish` | 写响应 | 网络回写耗时 |

官方特别提醒：**把某节点设为 `LevelDisabled` 或不注册 Tracer，会让该节点的 span/metrics 丢失，并导致链路中断。** 全线服务要么统一开 Base 以上，要么统一使用外部 SDK 的埋点。
## 四、读取埋点数据
在 `Finish` 中读取：

```go
func (t *myTracer) Finish(ctx context.Context, c *app.RequestContext) {
	ti := c.GetTraceInfo()
	st := ti.Stats()

	start := st.GetEvent(stats.HTTPStart)
	finish := st.GetEvent(stats.HTTPFinish)
	if start == nil || finish == nil {
		return
	}
	cost := finish.Time().Sub(start.Time())

	// 其它可用信息
	_ = st.SendSize() // 响应字节数
	_ = st.RecvSize() // 请求字节数
	_ = st.Error()    // 框架层错误
	if panicked, val := st.Panicked(); panicked {
		_ = val // panic 值，注意不要直接打印敏感内容
	}
	_ = cost
}
```

在中间件里读取：

```go
h.Use(func(ctx context.Context, c *app.RequestContext) {
	start := time.Now()
	c.Next(ctx)

	st := c.GetTraceInfo().Stats()
	hlog.CtxInfof(ctx,
		"cost=%s send=%d recv=%d err=%v",
		time.Since(start), st.SendSize(), st.RecvSize(), st.Error())
})
```

`GetEvent` 返回 nil 是常态（该级别未记录该事件），必须先判空再取 `Time()`，否则会 panic。
## 五、自定义指标 Tracer
下面是一个“按方法与状态码记录请求耗时”的 Tracer 骨架，指标库用 Prometheus 客户端：

```go
import (
	"context"
	"strconv"
	"time"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/common/tracer/stats"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

var httpDuration = promauto.NewHistogramVec(
	prometheus.HistogramOpts{
		Name:    "hertz_server_duration_seconds",
		Help:    "HTTP server latency",
		Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5},
	},
	[]string{"method", "route", "status"},
)

type MetricTracer struct{}

func (t *MetricTracer) Start(ctx context.Context, c *app.RequestContext) context.Context {
	return ctx
}

func (t *MetricTracer) Finish(ctx context.Context, c *app.RequestContext) {
	ti := c.GetTraceInfo()
	st := ti.Stats()
	begin, end := st.GetEvent(stats.HTTPStart), st.GetEvent(stats.HTTPFinish)
	if begin == nil || end == nil {
		return
	}
	httpDuration.WithLabelValues(
		c.Request.Method(),
		c.FullPath(), // 路由模板，避免维度爆炸
		strconv.Itoa(c.Response.StatusCode()),
	).Observe(end.Time().Sub(begin.Time()).Seconds())
}
```

注册：

```go
h := server.Default(
	server.WithTraceLevel(stats.LevelDetailed),
	server.WithTracer(&MetricTracer{}),
)
```

三个设计要点：

1. **标签用路由模板而不是原始路径**，`c.FullPath()` 优于 `c.Request.Path()`
2. **标签基数可控**：不要把 user_id、order_id 放进 label
3. **Histogram 分桶贴合业务 SLA**：网关类服务关注 5ms~2s，内部 RPC 可以更细
## 六、与 Prometheus 的关系
`hertz-contrib/monitor-prometheus` 本质就是一个官方写好的 Tracer：

```go
h := server.Default(
	server.WithTracer(prometheus.NewServerTracer(":9091", "/hertz")),
)
```

它暴露两个指标：

- `hertz_server_throughput{method,statusCode}`：请求计数
- `hertz_server_latency_us{method,statusCode}`：耗时直方图（微秒）

所以“埋点”和“监控扩展”不是两套东西，而是**同一条链路上的两层**：

```text
请求生命周期 -> TraceInfo/stats.Event -> Tracer.Finish -> Prometheus/OTel 指标 -> Grafana/告警
```

如果你已经用了 `obs-opentelemetry`，它会同时输出 trace 与 RED 指标，通常不需要再叠一个 Prometheus Tracer，避免重复计数。
## 七、成本与采样
| 开销项 | 影响 | 控制手段 |
| --- | --- | --- |
| Detailed 埋点 | 每请求多记录 8 个事件，分配增加 | 生产用 Base 或 Detailed 但关闭无用事件 |
| 标签数量 | 指标时间序列爆炸，内存/存储上涨 | 控制 label 组合，避免高基数 |
| 直方图桶 | 每次 Observe 遍历桶 | 桶数量控制在 10~15 个 |
| 上报频率 | 抓取/推送带来 CPU 与网络 | 抓取间隔 15s~30s |
| 全量 trace | 存储成本最高 | 比例采样 + ParentBased |

经验值：单机 5k QPS 时，`LevelBase` 的埋点开销通常在 1% 以内；`LevelDetailed` 叠加全量 span 导出才可能到 3%~5%。先用基准测试量化，再决定档位。
## 常见坑
1. **`GetEvent` 返回 nil 直接 `.Time()`**：低级别埋点下必 panic，所有事件读取都要判空。
2. **在 `Start` 里读 header/path**：此时请求还没解包，拿到的是空值。
3. **label 用了原始路径**：`/user/1`、`/user/2` 各成一条时间序列，几天后 Prometheus OOM。
4. **同时注册 Prometheus Tracer 和 OTel 指标**：同一指标被记两遍，QPS 翻倍。
5. **把某个服务设成 `LevelDisabled`**：链路在该服务处断裂，表现为“上下游都有 trace，中间缺一段”。
6. **在 `Finish` 里做同步网络请求上报**：直接拉高尾延迟，上报必须异步或批量。
## 小结
- 埋点是请求生命周期事件的采集，Tracer 是消费这些事件的扩展点
- 三档粒度决定“记不记细节”，生产推荐 Base 起步、按需 Detailed
- `TraceInfo.Stats()` 提供耗时、包大小、错误、panic，取用前必须判空
- 自定义指标要控制标签基数与分桶数量
- Prometheus 扩展与 OTel 指标功能重叠，二选一即可
## 练习
1. 用自定义 Tracer 打印 `ReadHeader`、`ServerHandle`、`Write` 三段耗时，找出最慢阶段。
2. 把 `LevelDetailed` 改成 `LevelBase`，观察哪些事件变为 nil，并补全判空逻辑。
3. 给指标增加 `route` 标签，用压测工具打两个接口，验证 `/metrics` 中的序列数量。
4. 写一个会 panic 的 handler，在 Tracer 中捕获 `Panicked()` 并记录。
## 官方文档
- 埋点：<https://www.cloudwego.io/zh/docs/hertz/tutorials/observability/instrumentation/>
- 监控扩展：<https://www.cloudwego.io/zh/docs/hertz/tutorials/framework-exten/monitor/>
- Prometheus 扩展：<https://www.cloudwego.io/zh/docs/hertz/tutorials/third-party/prometheus/>
