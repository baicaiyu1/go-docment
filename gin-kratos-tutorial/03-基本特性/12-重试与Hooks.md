# 重试与 Hooks

> 一句话定位：重试是放大器不是保险丝——只对幂等请求、只对可重试错误、在预算内用「指数退避 + 抖动」；服务端 Hook 用 Gin 中间件的 before/after 实现，客户端 Hook 用 `RoundTripper` 注入。

## 本篇目标

- 明确重试的边界：哪些方法、哪些错误、在哪一层重试。
- 会写带抖动的指数退避，并用「总预算」防止重试拖垮上游。
- 会处理 `Retry-After`、请求体重放（`GetBody`）与重试风暴。
- 会用 Gin 中间件做出 before/after 语义的 Hook，理解 `c.Next()` 的真实作用。
- 会实现可组合的 `RoundTripper`，把日志、trace、重试叠在客户端。

## 前置知识

- [客户端 Client](11-客户端Client.md)
- [中间件](05-中间件.md)
- 了解 `context` 超时预算。

## 一、先确定边界，再谈实现

| 方法 | 幂等 | 默认是否重试 |
| --- | --- | --- |
| `GET` / `HEAD` | 是 | 可重试（网络错误、超时、5xx） |
| `PUT` / `DELETE` | 是 | 可重试 |
| `POST` | 否 | **只有带幂等键时**才可重试 |

1. **4xx 不重试**（`408`、`429` 除外），否则只是在放大错误。
2. **只在链路的某一层重试**：网关、服务、SDK 各重试 3 次 = 放大 27 倍流量。
3. **重试必须有总预算**：等待时间也算进 deadline，否则上游早已超时你还在重试。
4. **重试风暴防护**：全链路重试比例超 10%（Envoy retry budget 思路）就停止；熔断打开时不再消耗重试配额；网关层最多 1 次；退避上限 2~3 秒。

## 二、指数退避 + 抖动 + 可重试判定

```go
// pkg/retry/backoff.go
package retry

import (
    "context"
    "errors"
    "math"
    "math/rand"
    "net"
    "time"
)

// IsRetryable：网络/超时类错误可重试；业务错误与 ctx 主动取消不重试
func IsRetryable(err error) bool {
    var ne net.Error
    return err != nil && !errors.Is(err, context.Canceled) && errors.As(err, &ne)
}

// Backoff：base*2^n，上限 max，带 ±20% 抖动
func Backoff(attempt int, base, max time.Duration) time.Duration {
    d := math.Min(float64(base)*math.Pow(2, float64(attempt)), float64(max))
    return time.Duration(d * (1 + (rand.Float64()*2-1)*0.2))
}
```

固定退避会让同一时刻失败的上千个客户端在同一毫秒重试；抖动把它们打散，这是防「重试风暴」的第一道防线。总预算靠调用方传入的 `context`：每次重试前 `select ctx.Done()`，预算耗尽立即返回最后一次错误。

## 三、Retry-After、状态码与请求体重放

```go
// 优先听服务端的 Retry-After（秒数或 HTTP-date）
func waitFromResp(resp *http.Response, attempt int) time.Duration {
    if v := resp.Header.Get("Retry-After"); v != "" {
        if sec, err := strconv.Atoi(v); err == nil {
            return time.Duration(sec) * time.Second
        }
    }
    return retry.Backoff(attempt, 100*time.Millisecond, 2*time.Second)
}

func retryableStatus(code int) bool {
    return code >= http.StatusInternalServerError || code == http.StatusTooManyRequests ||
        code == http.StatusRequestTimeout
}

// body 一次性读取，重试前必须重建
func cloneReq(req *http.Request) (*http.Request, error) {
    r2 := req.Clone(req.Context())
    if req.GetBody != nil {
        b, err := req.GetBody()
        if err != nil {
            return nil, err
        }
        r2.Body = b
    }
    return r2, nil
}
```

`bytes.NewReader` / `strings.Reader` 构造的请求会自动填充 `GetBody`；自定义 Reader 必须手动实现，否则重试只能发空 body。

## 四、服务端 Hook：Gin 的 before / after

`c.Next()` 的语义与 `Audit` 访问日志的完整实现见 [05 篇](05-中间件.md)，这里只强调两点：后置逻辑（统计、审计）必须写在 `c.Next()` 之后，此时响应已写出、`c.Writer.Status()` 才是最终值；拦截型 Hook 校验失败时 `c.AbortWithStatusJSON(...)` 后必须 `return`，否则后续 handler 仍会执行。

## 五、客户端 Hook：RoundTripper

```go
// pkg/httpclient/retry_rt.go
type RetryRT struct {
    Base        http.RoundTripper
    MaxAttempts int
    Logger      *zap.Logger
}

func (t *RetryRT) RoundTrip(req *http.Request) (resp *http.Response, err error) {
    base := t.Base
    if base == nil {
        base = http.DefaultTransport
    }
    idempotent := req.Method != http.MethodPost || req.Header.Get("Idempotency-Key") != ""
    for i := 0; i < t.MaxAttempts; i++ {
        if i > 0 && !idempotent {
            break // 非幂等请求只发一次
        }
        r2, cerr := cloneReq(req)
        if cerr != nil {
            return nil, cerr
        }
        resp, err = base.RoundTrip(r2) // 可在此处埋 trace span
        t.Logger.Debug("upstream", zap.String("url", req.URL.String()), zap.Int("attempt", i))
        if err == nil && !retryableStatus(resp.StatusCode) {
            return resp, nil
        }
        if resp != nil {
            resp.Body.Close() // 丢弃旧响应体，避免连接泄漏
        }
        if i == t.MaxAttempts-1 {
            break
        }
        wait := time.Duration(0)
        if resp != nil {
            wait = waitFromResp(resp, i)
        }
        select {
        case <-req.Context().Done(): // 预算耗尽
            return nil, req.Context().Err()
        case <-time.After(wait):
        }
    }
    return resp, err
}
```

挂载即可获得重试能力：`&http.Client{Transport: &RetryRT{Base: tr, MaxAttempts: 2, Logger: logger}, Timeout: 5 * time.Second}`。

## 完整示例

```go
// cmd/retry-demo/main.go：先用 9002 起一个返回 500 的 Gin 服务，再运行本程序观察重试
package main

import (
    "log"
    "net/http"
    "time"

    "go.uber.org/zap"

    "mall-lite/pkg/httpclient"
)

func main() {
    cli := &http.Client{
        Transport: &httpclient.RetryRT{MaxAttempts: 3, Logger: zap.NewNop()},
        Timeout:   3 * time.Second,
    }
    resp, err := cli.Get("http://127.0.0.1:9002/flaky")
    if err != nil {
        log.Fatalf("call failed after retries: %v", err)
    }
    defer resp.Body.Close()
    log.Printf("status=%d", resp.StatusCode)
}
```

## 常见坑

1. **多层重试**：网关、服务、SDK 各 3 次，故障时直接把下游打死。
2. **退避不抖动**：所有客户端同时重试，形成周期性尖峰。
3. **重试不设总预算**：超过上游 deadline 还在重试，白占连接与 goroutine。
4. **复用同一个 `*http.Request`**：body 已读完，第二次发空包；必须 `Clone` + `GetBody`。
5. **丢弃响应体不 `Close`**：连接泄漏，最终 `too many open files`。
6. **对 4xx 重试**：参数错误、鉴权失败重试一万次也不会成功。
7. **Gin 中间件不写 `c.Next()` 就写后置逻辑**：后置代码在 handler 之前执行，`Status()` 是错的。

## 小结

- 只对幂等请求与可重试错误重试；4xx（除 408/429）不重试。
- 退避 = 指数 + 上限 + 抖动；等待时间计入总预算，`Retry-After` 优先。
- 重试只放一层，配合熔断与重试预算防风暴。
- 服务端 Hook 用 Gin 中间件的 `c.Next()`；客户端 Hook 用 `RoundTripper` 组合。

## 练习

1. 用 `httptest` 写「前两次 500、第三次 200」的上游，验证重试次数与退避间隔。
2. 对无幂等键的 `POST` 调用 `RetryRT`，确认只发出一次请求。
3. 在 [05 篇](05-中间件.md) 的 `Audit` 中间件里注入 `request_id`，在 handler 里 `c.GetString` 读取验证。

## 官方文档

- `net/http#RoundTripper`：<https://pkg.go.dev/net/http#RoundTripper>
- `net/http#Request.GetBody`：<https://pkg.go.dev/net/http#Request>
- MDN `Retry-After`：<https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Retry-After>
- Gin 中间件：<https://gin-gonic.com/zh-cn/docs/examples/using-middleware/>
- Sentinel Go 熔断：<https://sentinelguard.io/zh-cn/docs/golang/circuit-breaker.html>
