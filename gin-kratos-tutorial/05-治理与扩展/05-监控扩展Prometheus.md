# 监控扩展 Prometheus

> 一句话定位：指标不是「加个 /metrics 就完事」，而是先定命名与标签规范，再让 Gin 与 Kratos 各自暴露 RED 指标，最后用 PromQL 画出 QPS、P99、5xx 比例并配告警。

## 本篇目标

- 分清 Counter / Gauge / Histogram，并按规范命名 `http_server_*`、`grpc_server_*`。
- 在独立端口暴露 `/metrics`，写出 Gin 指标中间件与 Kratos gRPC 指标拦截器。
- 掌握 QPS、P99、5xx 比例的标准 PromQL，以及桶、标签基数与告警窗口的取舍。

## 前置知识

- 已读 [埋点 Instrumentation](../04-可观测性/03-埋点Instrumentation.md)，了解 RED 与四黄金信号。
- 已读 [链路追踪 OpenTelemetry 与 Jaeger](../04-可观测性/02-链路追踪OpenTelemetry与Jaeger.md)。

## 一、指标模型与命名

| 类型 | 特点 | 适用 | 本教程指标 |
| --- | --- | --- | --- |
| Counter | 只增不减 | 请求数、错误数 | `http_server_requests_total` |
| Gauge | 可增可减 | 在途请求、连接数 | `http_server_inflight_requests` |
| Histogram | 分桶 + sum/count | 延迟分位 | `http_server_request_duration_seconds` |

命名三段式 `<namespace>_<subsystem>_<name>_<unit>`：延迟统一秒（`_seconds`），计数统一 `_total`。不要造 `http_qps`——QPS 是 `rate()` 算出来的，不是打点打出来的。标签只放低基数枚举：`method`、`route`、`status`、`grpc_service`、`grpc_method`、`code`。

## 二、独立端口暴露 /metrics

```go
// app/gateway/cmd/main.go（片段）
mux := http.NewServeMux()
mux.Handle("/metrics", promhttp.HandlerFor(reg, promhttp.HandlerOpts{EnableOpenMetrics: true}))
go func() { _ = http.ListenAndServe(":9101", mux) }()
```

端口按约定 9101/9102/9103，与业务端口分离；`/metrics` 不要挂进 Gin 路由树，否则会被鉴权拦住、还会被计入业务 QPS。`HandlerOpts.EnableOpenMetrics` 打开 exemplars 后，指标可以跳到对应 trace。

## 三、Gin 指标中间件

```go
// app/gateway/internal/middleware/metrics.go
package middleware

import (
    "strconv"
    "time"

    "github.com/gin-gonic/gin"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    httpRequests = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "http_server_requests_total", Help: "Total handled HTTP requests.",
    }, []string{"method", "route", "status"})
    httpDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name: "http_server_request_duration_seconds", Help: "HTTP latency in seconds.",
        Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5},
    }, []string{"method", "route"})
)

func Metrics() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        c.Next()

        route := c.FullPath()
        if route == "" {
            route = "unmatched" // 不能用原始 URL，否则序列爆炸
        }
        httpRequests.WithLabelValues(c.Request.Method, route, strconv.Itoa(c.Writer.Status())).Inc()
        httpDuration.WithLabelValues(c.Request.Method, route).Observe(time.Since(start).Seconds())
    }
}
```

`promauto` 自动注册到默认 Registry，省掉 `MustRegister`；在途请求用 Gauge 在 `c.Next()` 前后 `Inc/Dec`，本篇从简省略。`status` 保留原始码便于排查，若 4xx 细分类太多，可先归类成 `2xx/4xx/5xx` 再打点。

## 四、Kratos gRPC 指标拦截器

```go
// app/user-service/internal/server/middleware/metrics.go
package middleware

import (
    "context"
    "strings"
    "time"

    kerrors "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/transport"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
)

var (
    rpcHandled = promauto.NewCounterVec(prometheus.CounterOpts{
        Name: "grpc_server_handled_total", Help: "Total handled RPCs.",
    }, []string{"grpc_service", "grpc_method", "code"})
    rpcSeconds = promauto.NewHistogramVec(prometheus.HistogramOpts{
        Name: "grpc_server_handling_seconds", Help: "gRPC latency in seconds.",
        Buckets: []float64{0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2},
    }, []string{"grpc_service", "grpc_method"})
)

func GRPCMetrics() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req any) (any, error) {
            service, method := "unknown", "unknown"
            if info, ok := transport.FromServerContext(ctx); ok {
                service, method, _ = strings.Cut(strings.TrimPrefix(info.Operation(), "/"), "/")
            }
            start := time.Now()
            reply, err := handler(ctx, req)

            code := "OK"
            if err != nil {
                code = "UNKNOWN"
                if se := kerrors.FromError(err); se != nil {
                    code = se.Reason // 大写下划线，基数可控
                }
            }
            rpcHandled.WithLabelValues(service, method, code).Inc()
            rpcSeconds.WithLabelValues(service, method).Observe(time.Since(start).Seconds())
            return reply, err
        }
    }
}
```

用 `Reason` 而非数字码当标签：`USER_NOT_FOUND`、`TOO_MANY_REQUESTS` 是有限枚举；HTTP 数值码在 Kratos 里可能是 404 也可能是 500，统计含义模糊。装配时放进 `grpc.Middleware(..., middleware.GRPCMetrics())`。

## 五、Histogram 桶与标签基数

| 接口类型 | 建议桶（秒） | 理由 |
| --- | --- | --- |
| 缓存/内存查询 | 0.001 ~ 0.05 | 关注毫秒抖动 |
| 普通 API | 0.005 ~ 1 | 覆盖 SLO 1s |
| 报表/导出 | 0.1 ~ 10 | 长尾能看出拐点 |

桶边界必须包住 SLO（P99 < 500ms 就必须有 0.5 这个桶），否则 `histogram_quantile` 只能线性插值，误差可达 20%。序列数 = 标签组合 × 桶数，15 个桶再叠 3 个标签就已经很贵。

## 六、抓取配置

```yaml
# deploy/prometheus/prometheus.yml
global: { scrape_interval: 15s, evaluation_interval: 15s }
rule_files: ["/etc/prometheus/rules/*.yml"]
scrape_configs:
  - job_name: gateway
    static_configs: [{ targets: ["gateway:9101"], labels: { service: gateway, env: prod } }]
  - job_name: user-service
    static_configs: [{ targets: ["user-service:9102"], labels: { service: user-service, env: prod } }]
```

K8s 用 Pod 注解自动发现（`prometheus.io/scrape`、`port`、`path`），再用 `relabel_configs` 把 `app` 标签映射成 `service`。15s 抓取足够，改 5s 只会让双方更忙。

## 七、PromQL 三件套

```text
# QPS（按路由）
sum by (route) (rate(http_server_requests_total{job="gateway"}[1m]))

# 5xx 比例
sum(rate(http_server_requests_total{job="gateway", status=~"5.."}[5m]))
  / sum(rate(http_server_requests_total{job="gateway"}[5m]))

# P99（秒）：必须先进 sum by (le) 再算分位
histogram_quantile(0.99, sum by (le) (rate(http_server_request_duration_seconds_bucket{job="gateway"}[5m])))
```

`rate()` 窗口至少是抓取间隔的 4 倍；`histogram_quantile` 前不聚合 `le` 会把多实例桶混算错；`status=~"5.."` 是字符串匹配，别写 `status=500`。

## 八、Grafana 与告警

Dashboard 用 JSON 存版本库，四个要点：`templating` 定义 `$service`、`$route`（`label_values` 数据源）；顶部 Stat 面板放 QPS、错误率、P99、在途请求；延迟面板同时画 P50/P90/P99；面板 `datasource` 用变量而不是硬编码 UID。

```yaml
# deploy/prometheus/rules/gateway.yml
groups:
  - name: gateway-slo
    rules:
      - alert: GatewayHighErrorRate
        expr: |
          sum(rate(http_server_requests_total{job="gateway", status=~"5.."}[5m]))
            / sum(rate(http_server_requests_total{job="gateway"}[5m])) > 0.05
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "网关 5xx 比例超过 5%，当前 {{ $value | humanizePercentage }}"

      - alert: GatewayHighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum by (le) (rate(http_server_request_duration_seconds_bucket{job="gateway"}[5m]))) > 0.8
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "网关 P99 超过 800ms，当前 {{ $value }}s，按 route 定位慢接口"
```

`for` 必须留窗口，否则一次 GC 抖动就告警；文案要带处置建议，否则值班还得先查文档。

## 常见坑

1. **`/metrics` 挂在业务端口**：公网可访问，暴露全部路由与流量结构。
2. **P99 单位错**：指标是秒却按毫秒画图，SLO 判断全错；命名写死 `_seconds`。
3. **多实例直接 `histogram_quantile`**：必须先 `sum by (le)`，否则分位被多实例桶污染。
4. **原始 URL 当标签**：`/orders/123456` 每个 ID 一条序列，一天打满内存；用 `c.FullPath()`。
5. **两套 HTTP 指标并存**：OTel 与自研中间件同时打点，QPS 翻倍、看板失真。
6. **Counter 当 Gauge 用**：重启归零却被当累计值，`rate()` 出现负值尖刺。

## 小结

- 先定命名与标签规范，再写打点：`http_server_*` / `grpc_server_*` + 秒 + `_total`。
- Gin 用中间件、Kratos 用 `middleware.Middleware`，都在 `c.Next()` / `handler()` 之后统一记录。
- QPS 用 `rate(Counter)`，P99 用 `histogram_quantile` + `sum by (le)`；成本核心是压标签基数与桶数量。

## 练习

1. 用 `hey -z 30s -c 100` 压网关，核对 `http_server_requests_total` 增量与 QPS 面板一致。
2. 让接口返回 500，确认 5xx 告警在 `for: 5m` 后触发，并按 `route` 定位到具体接口。
3. 给 order-service 增加 `order_created_total{channel,status}`，标签组合不超过 10 种。

## 官方文档

- 指标类型：<https://prometheus.io/docs/concepts/metric_types/>
- PromQL 基础：<https://prometheus.io/docs/prometheus/latest/querying/basics/>
- client_golang：<https://pkg.go.dev/github.com/prometheus/client_golang/prometheus>
