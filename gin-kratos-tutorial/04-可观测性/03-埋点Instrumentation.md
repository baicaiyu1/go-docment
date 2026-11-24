# 埋点 Instrumentation

> 一句话定位：日志回答「发生了什么」，追踪回答「慢在哪一跳」，指标回答「整体多少」；本篇用 Prometheus client_golang 在 Gin 与 Kratos 上埋点，路由标签用 `c.FullPath()` 防基数爆炸。

## 本篇目标

- 分清日志、指标、追踪三支柱的边界与关联键
- 掌握 Counter/Gauge/Histogram 的选型与 Prometheus 命名规范
- 写出 Gin HTTP 指标中间件与 Kratos gRPC 指标中间件
- 定义下单数、支付成功率等业务指标，用 RED 与四黄金信号组织看板并控制基数

## 前置知识

- 已读 [02-链路追踪](02-链路追踪OpenTelemetry与Jaeger.md)，知道一次请求会有 `trace_id`
- 了解 Prometheus 的 pull 模型（服务暴露 `/metrics`，Prometheus 定时抓取）

## 一、三支柱怎么分工

| 能力 | 回答的问题 | 本项目实现 | 成本 |
| --- | --- | --- | --- |
| 日志 | 这一次请求具体发生了什么 | zap JSON，`request_id`/`trace_id` 关联 | 高（每条都写） |
| 追踪 | 这一次请求经过了哪些服务、哪段慢 | OTel + Jaeger，按比例采样 | 中（采样后存储） |
| 指标 | 整体 QPS、P99、错误率是多少 | Prometheus 时间序列 | 低（聚合后只存数值） |

三者用同一组键串起来：`request_id` 找单条日志，`trace_id` 找完整链路，指标负责发现「哪个时间段、哪个接口异常」，再回查日志与链路。埋点不是第四套东西，而是产生指标数据的那层代码。

## 二、指标类型与命名

| 类型 | 语义 | 典型用法 | PromQL |
| --- | --- | --- | --- |
| Counter | 只增不减 | 请求数、错误数、下单数 | `rate(x_total[5m])` |
| Gauge | 可增可减 | 在途请求、连接池使用数 | 直接取值 |
| Histogram | 分桶累计 + sum/count | 延迟、请求体大小 | `histogram_quantile(0.99, ...)` |
| Summary | 客户端算分位 | 极少用，不可跨实例聚合 | 不推荐 |

命名规范：Counter 以 `_total` 结尾（`http_server_requests_total`）；时间统一秒并带单位后缀（`http_server_request_duration_seconds`）；名字体现「谁 + 什么 + 单位」，标签体现「哪个接口、什么结果」；禁止把 `user_id`、`order_id`、原始 URL 放进标签。

## 三、Gin HTTP 指标中间件

```go
// app/gateway/internal/middleware/metrics.go
var (
    httpRequests = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_server_requests_total",
        Help: "Total HTTP requests by method, route and status.",
    }, []string{"method", "route", "status"})

    httpDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "http_server_request_duration_seconds",
        Help:    "HTTP request latency in seconds.",
        Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
    }, []string{"method", "route", "status"})
)

func Metrics() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()
        route := c.FullPath() // 路由模板，/users/:id 不会因 ID 不同而分裂
        if route == "" {
            route = "unmatched" // 404 兜底，避免空标签
        }
        status := strconv.Itoa(c.Writer.Status())
        httpRequests.WithLabelValues(c.Request.Method, route, status).Inc()
        httpDuration.WithLabelValues(c.Request.Method, route, status).
            Observe(time.Since(start).Seconds())
    }
}
```

`promauto` 会自动注册到默认 Registry，暴露端点只需一行 `r.GET("/metrics", gin.WrapH(promhttp.Handler()))`。`gin.WrapH` 把标准库 `http.Handler` 适配成 Gin handler；`/metrics` 只在内网端口或经网关鉴权后暴露，公网开放等于泄露全部接口结构。

## 四、gRPC 指标中间件（Kratos）

Kratos 中间件签名是 `func(middleware.Handler) middleware.Handler`，从 `transport.FromServerContext` 取方法名：

```go
// pkg/metrics/grpc.go
func GRPCServer() middleware.Middleware {
    handled := promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "grpc_server_handled_total",
        Help: "Total gRPC calls handled by method and code.",
    }, []string{"service", "method", "grpc_code"})

    handling := promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name:    "grpc_server_handling_seconds",
        Help:    "gRPC handling latency in seconds.",
        Buckets: prometheus.DefBuckets,
    }, []string{"service", "method", "grpc_code"})

    return func(next middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req any) (any, error) {
            start := time.Now()
            reply, err := next(ctx, req)
            tr, _ := transport.FromServerContext(ctx)
            svc, method := splitOperation(tr.Operation()) // "/user.v1.UserService/GetUser"
            code := "OK"
            if err != nil {
                code = "ERROR"
            }
            handled.WithLabelValues(svc, method, code).Inc()
            handling.WithLabelValues(svc, method, code).Observe(time.Since(start).Seconds())
            return reply, err
        }
    }
}
```

`splitOperation` 按 `/` 拆成 `user.v1.UserService` 与 `GetUser`。标签控制在同一服务十几个方法以内；如果方法数量很大（如分页查询变体），只保留 service 级标签。

## 五、业务指标

技术指标看「系统是否健康」，业务指标看「生意是否正常」，两者缺一不可：

```go
var (
    orderCreated = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "order_created_total",
        Help: "Total created orders by channel and result.",
    }, []string{"channel", "result"})

    paymentTotal = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "payment_total",
        Help: "Total payment attempts by result.",
    }, []string{"result"}) // success / failed / timeout
)

// 下单成功后
orderCreated.WithLabelValues(order.Channel, "success").Inc()
// 支付回调里
if err != nil {
    paymentTotal.WithLabelValues("failed").Inc()
} else {
    paymentTotal.WithLabelValues("success").Inc()
}
```

支付成功率不要存成 Gauge 由应用计算（多实例会互相覆盖），用两个 Counter 在 PromQL 里算：

```promql
sum(rate(payment_total{result="success"}[5m]))
/ sum(rate(payment_total[5m]))
```

## 六、RED 与四黄金信号

| 方法论 | 关注点 | 本项目指标 |
| --- | --- | --- |
| RED | Rate 请求速率、Errors 错误率、Duration 耗时 | `http_server_requests_total`、`..._duration_seconds` |
| 四黄金信号 | 延迟、流量、错误、饱和度 | 前三者同上，饱和度看连接池/goroutine/CPU |

Grafana 中三个最常用的查询：

```promql
# QPS（按路由）
sum by (route) (rate(http_server_requests_total[1m]))
# P99 耗时（按路由）
histogram_quantile(0.99,
  sum by (le, route) (rate(http_server_request_duration_seconds_bucket[5m])))
# 5xx 比例
sum(rate(http_server_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_total[5m]))
```

直方图分桶要贴合 SLA：网关关注 5ms~5s，内部 RPC 可以细到 1ms。桶数量控制在 10~15 个，太多会放大存储与计算成本。

## 七、标签基数与成本

| 开销项 | 影响 | 控制手段 |
| --- | --- | --- |
| 标签组合数 | 时间序列数量 = 各标签取值笛卡尔积 | route 用模板、status 只留百位、不放开 userId |
| 直方图桶 | 每次 Observe 遍历桶，序列数 × 桶数 | 10~15 个桶，按 SLA 定制 |
| 抓取频率 | 15s 抓一次是常态，5s 会翻三倍 | 核心服务 15s，边缘 30s |
| 业务标签 | `channel` 从 3 个变 300 个 | 白名单枚举，未知值归 `other` |

经验值：单实例 5k QPS、10 个路由、5 个状态码，序列数在几百级；把 URL 当标签，一天就能产生几十万序列，Prometheus 直接 OOM。

## 完整示例

`deploy/prometheus/prometheus.yml`，按服务抓取：

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: gateway
    static_configs:
      - targets: ["gateway:8080"] # /metrics 与业务同端口
  - job_name: kratos-services
    static_configs:
      - targets: ["user-service:9101", "product-service:9102", "order-service:9103"]
```

Kratos 服务用 `kratos.Server(metricsServer)` 单独起 metrics 端口，并把 `GRPCServer()` 挂进 `grpc.Middleware(...)`；Gin 网关直接 `r.GET("/metrics", gin.WrapH(promhttp.Handler()))`。

## 常见坑

1. **标签用 `c.Request.URL.Path`**：`/users/1`、`/users/2` 各成一条序列，几天后 Prometheus OOM；必须用 `c.FullPath()`。
2. **用 Gauge 存请求数**：多实例互相覆盖、重启归零；计数一律用 Counter。
3. **分位数直接取 Histogram 平均值**：`_sum/_count` 是均值不是 P99，必须 `histogram_quantile`。
4. **业务指标在应用里算成功率**：多副本各算各的再上报，数值随抓取顺序跳变；用 Counter 在 PromQL 聚合。
5. **`/metrics` 暴露在公网**：接口结构、错误码、实例数全部泄露；只开内网或加鉴权。
6. **指标名重复注册**：测试里多次 `promauto` 会 panic，用 `prometheus.NewRegistry()` 隔离或 `testutil.ToFloat64` 断言。

## 小结

- 日志、追踪、指标三支柱互补，用 `request_id`/`trace_id` 关联
- HTTP 用 `http_server_requests_total` + `..._duration_seconds`，gRPC 用 `grpc_server_handled_total` + `..._handling_seconds`
- 路由标签必须用 `c.FullPath()`，否则基数爆炸
- 业务成功率用 Counter + PromQL 计算，不要用 Gauge
- 看板按 RED 或四黄金信号组织，分桶与抓取频率直接决定成本

## 练习

1. 给网关加上指标中间件，压测两个接口，确认 `/metrics` 中 `route` 只有两条序列。
2. 在 Kratos 服务挂上 `GRPCServer()`，用 `ghz` 打 1000 次，核对 `grpc_server_handled_total` 数值。
3. 增加 `order_created_total{channel,result}`，在 Grafana 画出各渠道下单量。
4. 写一条 5xx 比例告警规则，阈值 1%，并验证能触发。

## 官方文档

- client_golang：<https://github.com/prometheus/client_golang>
- 指标命名最佳实践：<https://prometheus.io/docs/practices/naming/>
- Histogram 与分位数：<https://prometheus.io/docs/practices/histograms/>
- PromQL 基础：<https://prometheus.io/docs/prometheus/latest/querying/basics/>
- Kratos 中间件：<https://github.com/go-kratos/kratos/tree/main/middleware>
