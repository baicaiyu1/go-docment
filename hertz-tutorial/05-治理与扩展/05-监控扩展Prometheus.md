# 监控扩展 Prometheus
> 一句话定位：把 Hertz 请求数据变成时间序列，配出 QPS、延迟分位、错误率三块核心看板与告警。

## 本篇目标

- 分清 Counter、Gauge、Histogram、Summary 的用途
- 用 `hertz-contrib/monitor-prometheus` 暴露指标
- 写出 Prometheus 抓取配置与 Grafana 核心查询
- 掌握 QPS、P99、错误率的标准 PromQL
- 配一条可用的告警规则，并控制标签基数

## 前置知识

- 已读 [03-埋点Instrumentation](../04-可观测性/03-埋点Instrumentation.md)
- 了解 Prometheus pull 模型
- 会用 Docker 起 Prometheus + Grafana

## 一、四种指标类型

| 类型 | 特点 | 用途 | Hertz 例子 |
| --- | --- | --- | --- |
| Counter | 只增不减 | 请求数、错误数 | `hertz_server_throughput` |
| Gauge | 可增可减 | 连接数、goroutine 数 | 在途请求 |
| Histogram | 分桶 + sum/count | 延迟分位 | `hertz_server_latency_us` |
| Summary | 客户端算分位 | 精确但不可聚合 | 少用 |

原则：算“每秒多少”用 Counter + `rate()`；看“现在多少”用 Gauge；算 P95/P99 且跨实例聚合用 Histogram。

## 二、接入 monitor-prometheus

```bash
go get github.com/hertz-contrib/monitor-prometheus
```

```go
h := server.Default(
	server.WithHostPorts("127.0.0.1:8888"),
	// 参数：监听地址、metrics 路径
	server.WithTracer(prometheus.NewServerTracer(":9091", "/hertz")),
)

h.GET("/ping", func(ctx context.Context, c *app.RequestContext) {
	c.String(200, "pong")
})
```

要点：

- tracer 会**单独监听一个端口**（这里 `:9091`），避免业务端口暴露指标
- 抓取地址是 `http://127.0.0.1:9091/hertz`
- 它通过 `Tracer` 接口实现，属于框架扩展，不是中间件
- 若已有 OTel 指标导出，不要再叠加，避免重复计数

| 指标 | 类型 | 标签 |
| --- | --- | --- |
| `hertz_server_throughput` | Counter | `method`、`statusCode` |
| `hertz_server_latency_us` | Histogram | `method`、`statusCode` |

```bash
curl http://127.0.0.1:9091/hertz
```

```powershell
curl.exe http://127.0.0.1:9091/hertz
```

## 三、Prometheus 抓取

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/*.yml"

scrape_configs:
  - job_name: "hertz-order"
    metrics_path: /hertz
    static_configs:
      - targets: ["10.0.0.12:9091"]
        labels:
          env: prod
          service: order-service
```

K8s 环境可用 pod 注解自动发现（`prometheus.io/scrape`、`prometheus.io/port`），把 `metrics_path` 指向 `/hertz`，具体 relabel 配置以 Prometheus 文档为准。

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
    ports: ["9090:9090"]
  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
```

## 四、Grafana 核心查询

QPS：

```promql
sum(rate(hertz_server_throughput[1m])) by (method)
```

错误率：

```promql
sum(rate(hertz_server_throughput{statusCode=~"5.."}[5m]))
/
sum(rate(hertz_server_throughput[5m]))
```

P99 延迟（微秒转毫秒）：

```promql
histogram_quantile(
  0.99,
  sum(rate(hertz_server_latency_us_bucket[5m])) by (le)
) / 1000
```

平均延迟：

```promql
sum(rate(hertz_server_latency_us_sum[5m]))
/
sum(rate(hertz_server_latency_us_count[5m]))
/
1000
```

| 行 | 面板 |
| --- | --- |
| 总览 | QPS、错误率、P99、在途请求 |
| 流量 | 按 method/status 的 QPS 堆叠图 |
| 延迟 | P50/P90/P99 + 慢接口 TopN |
| 饱和度 | CPU、内存、goroutine、连接数 |

## 五、告警规则

```yaml
groups:
  - name: hertz-http
    rules:
      - alert: HertzHighErrorRate
        expr: |
          sum(rate(hertz_server_throughput{statusCode=~"5.."}[5m]))
          / sum(rate(hertz_server_throughput[5m])) > 0.05
        for: 5m
        labels: { severity: critical }
        annotations:
          summary: "服务 5xx 比例超过 5%"
          description: "当前 {{ $value | humanizePercentage }}"

      - alert: HertzHighLatencyP99
        expr: |
          histogram_quantile(0.99,
            sum(rate(hertz_server_latency_us_bucket[5m])) by (le)
          ) / 1000 > 800
        for: 10m
        labels: { severity: warning }
        annotations:
          summary: "P99 超过 800ms"
          description: "当前 {{ $value }}ms"
```

`for` 必须留足时间，避免瞬时抖动触发告警；告警文案要带处置建议。

## 六、自定义业务指标

```go
var orderCreated = promauto.NewCounterVec(
	prometheus.CounterOpts{Name: "order_created_total", Help: "Created orders"},
	[]string{"channel", "status"}, // 低基数
)

var orderAmount = promauto.NewHistogram(
	prometheus.HistogramOpts{
		Name:    "order_amount_yuan",
		Help:    "Order amount in yuan",
		Buckets: []float64{10, 50, 100, 500, 1000, 5000},
	},
)
```

标签三条红线：

1. 不放 `user_id`、`order_id`、`trace_id` 这类高基数
2. 枚举值可控（channel 建议 5~10 个）
3. 新增前估算：`指标数 × 标签组合数 × 实例数`

## 七、成本控制

| 成本项 | 影响 | 控制 |
| --- | --- | --- |
| 时间序列数 | Prometheus 内存 | 控制标签基数 |
| 抓取频率 | 双方 CPU | 15s~30s 足够 |
| 保留时长 | 磁盘 | 原始 15d + 降采样 |
| Histogram 桶 | 序列数 × 桶数 | 每指标 10~15 个桶 |
| 高基数路径 | 序列爆炸 | 用 `c.FullPath()` |

## 常见坑

1. `/metrics` 挂在业务端口：公网可访问，泄露路由与流量信息。
2. 用 `statusCode="500"` 匹配不到：标签是字符串，用 `=~"5.."`。
3. P99 算出来是微秒：`hertz_server_latency_us` 单位是微秒，画图除 1000。
4. 直接对多实例 `histogram_quantile`：必须先 `sum by (le)` 聚合桶。
5. 自定义指标加 userId 标签：几天后 Prometheus 内存暴涨。
6. 同时开 Prometheus Tracer 与 OTel 指标：QPS 翻倍，看板失真。
7. `for` 设为 0：一次抖动就告警，很快没人看。
8. 抓取目标写 `127.0.0.1`：Prometheus 在另一台机器/Pod，抓到的是自己。

## 小结

- Counter 算速率，Histogram 算分位，Gauge 看瞬时值
- `monitor-prometheus` 通过 Tracer 接入，独立端口暴露 `/hertz`
- QPS、错误率、P99 是黄金三指标
- 告警要有 `for` 窗口与处置建议
- 指标治理的核心是控制标签基数

## 练习

1. 接入 `monitor-prometheus`，观察请求前后 `/hertz` 输出变化。
2. 配 QPS、错误率、P99 三个面板，并压测验证曲线。
3. 让接口返回 500，验证告警按 `for: 5m` 触发。
4. 增加一个业务 Counter，标签组合不超过 6 种。

## 官方文档

- Prometheus 扩展：<https://www.cloudwego.io/zh/docs/hertz/tutorials/third-party/prometheus/>
- hertz-contrib/monitor-prometheus：<https://github.com/hertz-contrib/monitor-prometheus>
- PromQL 基础：<https://prometheus.io/docs/prometheus/latest/querying/basics/>
- 告警规则：<https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/>
