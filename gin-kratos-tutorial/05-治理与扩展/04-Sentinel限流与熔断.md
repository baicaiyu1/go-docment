# Sentinel 限流与熔断

> 一句话定位：Sentinel Go 用一套规则模型覆盖流控、熔断、热点与系统自适应；本篇给出 `InitDefault()` 初始化、Gin 与 Kratos gRPC 两个完整中间件、规则热加载与压测定阈值。

## 本篇目标

- 分清 flow / circuitbreaker / hotspot / system 四类规则各保护什么。
- 写出 Gin 限流中间件：资源名 `GET:/api/v1/orders`，超限 429 + `Retry-After`。
- 写出 Kratos gRPC 中间件：`grpc:/user.v1.UserService/GetUser`，限流 `ResourceExhausted`、熔断 `Unavailable`，规则从 YAML 热加载。

## 前置知识

- 已读 [中间件](../03-基本特性/05-中间件.md) 与 [配置管理 Viper](03-配置管理Viper.md)。
- 建议先读 [Kratos 客户端与 Gin 集成](../12-微服务与Kratos/04-Kratos客户端与Gin集成.md)。

## 一、四类规则：先想清楚保护谁

| 规则 | 触发依据 | 动作 | 保护对象 |
| --- | --- | --- | --- |
| flow | 资源 QPS / 并发 | Reject 429、排队、预热 | 自己 |
| circuitbreaker | 错误率、慢调用比例 | 打开熔断，快速失败 503 | 调用链 |
| hotspot | 单个参数值的 QPS | 只限该热 key | 热点数据 |
| system | Load、CPU、并发 | BBR 式整体限流 | 整个进程 |

## 二、初始化

Entry 必须在 `InitDefault()` 之后创建，否则 panic。初始化放在启动最前面，失败直接退出。

```go
// pkg/sentinel/init.go
package sentinelx

import sentinel "github.com/alibaba/sentinel-golang/api"

func Init() error {
    if err := sentinel.InitDefault(); err != nil {
        return err
    }
    return LoadFile("configs/sentinel-rules.yaml") // 见第六节
}
```

## 三、Gin 中间件：429 + Retry-After

```go
// app/gateway/internal/middleware/sentinel.go
package middleware

import (
    "net/http"

    sentinel "github.com/alibaba/sentinel-golang/api"
    "github.com/alibaba/sentinel-golang/core/base"
    "github.com/gin-gonic/gin"
)

func Sentinel() gin.HandlerFunc {
    return func(c *gin.Context) {
        path := c.FullPath()
        if path == "" {
            path = "unmatched" // 404 不能拼出 "GET:" 这种漂移资源名
        }
        e, b := sentinel.Entry(c.Request.Method+":"+path, sentinel.WithTrafficType(base.Inbound))
        if b != nil {
            status, retry := http.StatusTooManyRequests, "1"
            if b.BlockType() == base.BlockTypeCircuitBreaking {
                status, retry = http.StatusServiceUnavailable, "5"
            }
            c.Header("Retry-After", retry)
            c.AbortWithStatusJSON(status, gin.H{"code": status*100 + 1, "message": "rejected by sentinel", "request_id": c.GetString("request_id")})
            return
        }
        defer e.Exit() // 必须 Exit，否则并发统计不清零
        c.Next()
    }
}
```

`c.FullPath()` 是注册模板，不随 `?page=2` 或路径参数漂移；`defer e.Exit()` 先注册再 `c.Next()`，handler panic 时也能归还。

## 四、Kratos gRPC 中间件

```go
// app/user-service/internal/server/middleware/sentinel.go
package middleware

import (
    "context"

    sentinel "github.com/alibaba/sentinel-golang/api"
    "github.com/alibaba/sentinel-golang/core/base"
    kerrors "github.com/go-kratos/kratos/v2/errors"
    "github.com/go-kratos/kratos/v2/middleware"
    "github.com/go-kratos/kratos/v2/transport"
    "google.golang.org/grpc/codes"
)

func Sentinel() middleware.Middleware {
    return func(handler middleware.Handler) middleware.Handler {
        return func(ctx context.Context, req any) (any, error) {
            resource := "grpc:unknown"
            if info, ok := transport.FromServerContext(ctx); ok {
                resource = "grpc:" + info.Operation() // grpc:/user.v1.UserService/GetUser
            }
            e, b := sentinel.Entry(resource, sentinel.WithTrafficType(base.Inbound))
            if b != nil {
                if b.BlockType() == base.BlockTypeCircuitBreaking {
                    return nil, kerrors.New(int(codes.Unavailable), "CIRCUIT_OPEN", "circuit breaker is open")
                }
                return nil, kerrors.New(int(codes.ResourceExhausted), "TOO_MANY_REQUESTS", "rate limited")
            }
            defer e.Exit()
            return handler(ctx, req)
        }
    }
}
```

装配：`grpc.Middleware(recovery.Recovery(), logging.Server(klog), tracing.Server(), sentinelmw.Sentinel())`。网关按 [错误处理](../03-基本特性/06-错误处理.md) 的表把 `ResourceExhausted` 映射成 429xx，不把 gRPC 错误直接吐给浏览器。

## 五、熔断与降级

```go
// pkg/sentinel/breaker.go
package sentinelx

import "github.com/alibaba/sentinel-golang/core/circuitbreaker"

func LoadBreakers() error {
    _, err := circuitbreaker.LoadRules([]*circuitbreaker.Rule{
        {Resource: "grpc:/user.v1.UserService/GetUser", Strategy: circuitbreaker.ErrorRatio,
            Threshold: 0.5, MinRequestAmount: 20, StatIntervalMs: 10000, RetryTimeoutMs: 5000},
        {Resource: "GET:/api/v1/users/:id", Strategy: circuitbreaker.SlowRequestRatio,
            Threshold: 0.5, MinRequestAmount: 10, StatIntervalMs: 10000, RetryTimeoutMs: 10000, MaxAllowedRtMs: 300},
    })
    return err
}
```

`MinRequestAmount` 小于阈值（这里 20/10）不判定，避免小样本误开；`RetryTimeoutMs` 到期进入半开，探测失败立刻回到 Open。熔断打开后网关必须降级：捕获 `codes.Unavailable` / `DeadlineExceeded`，返回缓存快照并带 `"degraded": true`——降级的价值在「旧数据可用」，否则不如直接报错。

## 六、规则来源：YAML → 配置中心

规则写死在代码里等于每次调阈值都要发版；用 DTO 承接 YAML 再映射成 `flow.Rule`，换 etcd 只改读取那一行。

```yaml
# configs/sentinel-rules.yaml
- {resource: "GET:/api/v1/orders", threshold: 100, behavior: reject}
- {resource: "GET:/api/v1/products/:id", threshold: 200, behavior: warmup, warm_up_period_sec: 10}
- {resource: "POST:/api/v1/payments/:id", threshold: 50, behavior: throttling}
```

```go
// pkg/sentinel/loader.go
package sentinelx

import (
    "os"

    "github.com/alibaba/sentinel-golang/core/flow"
    "gopkg.in/yaml.v3"
)

type flowItem struct {
    Resource        string  `yaml:"resource"`
    Threshold       float64 `yaml:"threshold"`
    Behavior        string  `yaml:"behavior"`
    WarmUpPeriodSec uint32  `yaml:"warm_up_period_sec"`
}

func LoadFile(path string) error {
    raw, err := os.ReadFile(path)
    if err != nil {
        return err
    }
    var items []flowItem
    if err := yaml.Unmarshal(raw, &items); err != nil {
        return err
    }
    rules := make([]*flow.Rule, 0, len(items))
    for _, it := range items {
        r := &flow.Rule{Resource: it.Resource, Threshold: it.Threshold, ControlBehavior: flow.Reject, StatIntervalInMs: 1000}
        switch it.Behavior {
        case "warmup":
            r.TokenCalculateStrategy, r.WarmUpPeriodSec = flow.WarmUp, it.WarmUpPeriodSec
        case "throttling":
            r.ControlBehavior, r.MaxQueueingTimeMs = flow.Throttling, 200
        }
        rules = append(rules, r)
    }
    _, err = flow.LoadRules(rules)
    return err
}
```

热更新用一个 10s 的 `time.Tick` 比较文件 mtime，变化时重入 `LoadFile`；接 etcd 则把轮询换成 `Watch` 回调。`LoadRules` 是**全量覆盖**语义，少写一条等于删掉它，上线前务必 diff 资源名。

## 七、Dashboard 1.8.x、APISIX 分层与压测

Dashboard 1.8.x 的心跳与推送协议是给 Java 客户端设计的，Sentinel Go 不自带完整的上报客户端，Go 侧标准做法是「Dashboard 或配置中心产出规则 → datasource / HTTP 拉取 → `LoadRules`」，上面的加载器就是最小实现；它适合人工调参与看曲线，不作强一致规则源。分层上，APISIX `limit-req` / `limit-count` 在边缘按 IP、Consumer、Route 限流，Sentinel 在服务内按资源限流并熔断下游，两层都要有——网关被绕过时只有服务内规则生效。通过标准：QPS 压在阈值 ±10%，P99 不随并发恶化，429 响应 < 5ms。

验证方式：`docker run -d -p 8858:8858 bladex/sentinel-dashboard:1.8.6` 起 Dashboard（默认 `sentinel/sentinel`），用 `hey -z 60s -c 200 http://127.0.0.1:8080/api/v1/orders` 找 P99 拐点、取 70%~80% 作 `Threshold`；关停 user-service 后 5s 内应熔断打开并返回 `degraded`，重启后 10s 内恢复。

## 常见坑

1. **Entry 没 Exit**：并发统计只增不减，几分钟后所有请求被判超限。`defer e.Exit()` 紧跟成功判断。
2. **资源名漂移**：用 `c.Request.URL.Path` 拼名，路径参数变成上千个资源，规则全不命中；必须 `c.FullPath()`，空值统一 `unmatched`。
3. **阈值拍脑袋**：写「1000 QPS」却从没压测，真实拐点 300，限流形同虚设。
4. **熔断统计不含超时**：只把 5xx 当失败，3s 超时时熔断永远 Closed，线程照样耗尽。
5. **降级返回空数据**：空白页比报错更糟；降级返回可用快照并标 `degraded`。
6. **热更新只加载 flow**：熔断规则还是旧的，出现「限流生效、熔断失效」。
7. **在 goroutine 里用 `*gin.Context`**：context 已取消，统计错乱；异步任务用 `c.Copy()`。

## 小结

- 四类规则各管一段：流控保护自己、熔断保护调用链、热点保护单 key、系统规则兜整机。
- 资源名是主键：HTTP 用 `METHOD:FullPath`，gRPC 用 `grpc:` + `Operation()`，两端逐字一致。
- 限流 429 + `Retry-After`，熔断 503 并走降级；`LoadRules` 全量覆盖，阈值来自压测拐点。

## 练习

1. 给 `POST:/api/v1/orders` 配 Reject 与 Throttling 两套规则，`hey -c 300` 对比 429 比例与 P99。
2. 用 `hotspot.LoadRules` + `sentinel.WithArgs` 给 `product_id` 配热点规则，验证只有该参数被限。
3. 关停 user-service，记录熔断 Open → Half-Open 的实际耗时与自动恢复时间。

## 官方文档

- Sentinel Go 仓库与 Wiki：<https://github.com/alibaba/sentinel-golang>
- Sentinel 规则语义：<https://sentinelguard.io/zh-cn/docs/introduction.html>
- APISIX `limit-req`：<https://apisix.apache.org/docs/apisix/plugins/limit-req/>
