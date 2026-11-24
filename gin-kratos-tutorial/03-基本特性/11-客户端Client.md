# 客户端 Client

> 一句话定位：生产可用的 Go HTTP 客户端只有一条路径——复用同一个 `http.Client`、把超时拆到每一层、用 `context` 控制取消、按错误类型分类处理；本文给出可直接抄进 `pkg/httpclient` 的封装。

## 本篇目标

- 理清 `http.Client`、`http.Transport`、`RoundTripper` 三者的职责边界。
- 掌握 `Client.Timeout` 与 Transport 各阶段超时的差异，避免「调了 Timeout 还是卡住」。
- 会配置连接池、重定向策略、Header 与 User-Agent，并正确用 `context` 取消请求。
- 能把错误分成「网络 / 超时 / 4xx / 5xx / 业务」五类分别处理。
- 得到一套含重试、熔断、trace、日志的企业级封装思路。

## 前置知识

- [网络库 net/http 与性能模型](10-网络库net-http与性能模型.md)
- [请求上下文 Context](02-请求上下文Context.md)
- 了解 `context.WithTimeout`、`errors.Is/As` 与 `net.Error`。

## 一、三层结构，先分清职责

| 层 | 类型 | 负责 |
| --- | --- | --- |
| 客户端 | `*http.Client` | 总超时、重定向策略、Cookie Jar |
| 传输层 | `*http.Transport` | 连接池、拨号、TLS 握手、HTTP/2、代理 |
| 拦截层 | `http.RoundTripper` | 日志、trace、重试、限流等横切逻辑 |

`http.Client` 内部就是「构造 `*http.Request` → 交给 `Transport.RoundTrip` → 返回 `*http.Response`」。想加日志或 trace，实现 `RoundTripper` 比包 `Client` 更通用（见 [重试与 Hooks](12-重试与Hooks.md)）。

## 二、超时体系：一个 Timeout 不够用

```go
// pkg/httpclient/client.go
tr := &http.Transport{
    Proxy: http.ProxyFromEnvironment,
    DialContext: (&net.Dialer{
        Timeout:   2 * time.Second,  // 建连
        KeepAlive: 30 * time.Second, // TCP 保活探测
    }).DialContext,
    TLSHandshakeTimeout:   3 * time.Second, // TLS 握手
    ResponseHeaderTimeout: 3 * time.Second, // 发出请求到收到响应头
    IdleConnTimeout:       90 * time.Second,
    MaxIdleConns:          200,
    MaxIdleConnsPerHost:   100,
    ForceAttemptHTTP2:     true,
}
cli := &http.Client{
    Transport: tr,
    Timeout:   5 * time.Second, // 含读 body 的总时长；0 表示不限
}
```

| 超时 | 覆盖范围 | 典型值 |
| --- | --- | --- |
| `Dialer.Timeout` | TCP 建连 | 1~2s |
| `TLSHandshakeTimeout` | TLS 握手 | 3s |
| `ResponseHeaderTimeout` | 请求发出 → 响应头到达 | 读操作的业务 SLA |
| `Client.Timeout` | 全过程，**含读完 body** | 略大于下游 SLA |
| `context` deadline | 整个调用链，可跨服务传递 | 上游剩余预算 |

关键结论：`Client.Timeout` 只是兜底，真正的超时控制要用 `context`——只有它能跨服务传递预算。

## 三、连接池：默认值就是坑

`MaxIdleConnsPerHost` 默认只有 **2**。并发 100 的调用方会不停建连、关连，QPS 上不去还制造 `TIME_WAIT`。规则：

- `MaxIdleConnsPerHost` ≥ 单实例最大并发下游请求数，通常 50~200。
- `MaxIdleConns` ≥ `MaxIdleConnsPerHost × 下游服务数量`。
- `IdleConnTimeout` 小于下游/网关的空闲超时，避免复用到即将关闭的连接。
- **全进程只创建一次 Client**，放进依赖注入，禁止在 handler 里 `&http.Client{}`。

## 四、context 取消、重定向、Header 与 UA

```go
// 带超时的 GET，并透传 trace id
func GetJSON(ctx context.Context, cli *http.Client, url, traceID string, out any) error {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
    if err != nil {
        return err
    }
    req.Header.Set("Accept", "application/json")
    req.Header.Set("User-Agent", "mall-lite-gateway/1.0")
    if traceID != "" {
        req.Header.Set("X-Request-ID", traceID)
    }

    resp, err := cli.Do(req)
    if err != nil {
        return err // ctx 到期时返回 *url.Error，可 errors.Is(err, context.DeadlineExceeded)
    }
    defer resp.Body.Close()

    if resp.StatusCode >= 400 {
        body, _ := io.ReadAll(io.LimitReader(resp.Body, 4<<10)) // 限长，防打爆内存
        return fmt.Errorf("downstream %s: status=%d body=%s", url, resp.StatusCode, body)
    }
    return json.NewDecoder(resp.Body).Decode(out)
}
```

重定向默认跟随 10 次。内部调用建议关闭，否则 307 会把 `POST` 与 body 重放到别的主机：

```go
cli := &http.Client{
    Transport: tr,
    CheckRedirect: func(req *http.Request, via []*http.Request) error {
        return http.ErrUseLastResponse // 返回 3xx 本身，不再跟随
    },
}
```

## 五、错误分类

```go
// pkg/httpclient/errs.go
type Kind int

const (
    KindNetwork Kind = iota + 1 // 建连失败、DNS、连接重置
    KindTimeout                 // 超时（可重试）
    KindBadRequest              // 4xx（不重试）
    KindServer                  // 5xx（按幂等重试）
    KindBiz                     // 2xx 但业务 code != 0
)

func Classify(err error, status int) Kind {
    if err != nil {
        var ne net.Error
        if errors.Is(err, context.DeadlineExceeded) || (errors.As(err, &ne) && ne.Timeout()) {
            return KindTimeout
        }
        return KindNetwork
    }
    switch {
    case status >= 500:
        return KindServer
    case status >= 400:
        return KindBadRequest
    }
    return KindBiz
}
```

只有 `KindTimeout`、`KindServer`、`KindNetwork` 且**方法幂等**时才值得重试；`KindBadRequest` 重试只会放大错误。

## 六、resty 还是自研

**核心链路用标准库封装**（无额外依赖、行为可控、与 OTel/zap 无缝），脚本与内部工具可用 `github.com/go-resty/resty/v2`（链式 API、内置重试与日志）。用 resty 时同样要 `SetTimeout`、`SetTransport`（塞上面的 `tr`）、`SetContext`，否则默认值与标准库一样全踩一遍。

## 完整示例

`tr` 与 `cli` 的配置见第二节，这里只演示调用路径：

```go
// cmd/httpclient-demo/main.go
func main() {
    defer cli.CloseIdleConnections() // 退出前释放连接

    var out map[string]any
    if err := GetJSON(context.Background(), cli, "http://127.0.0.1:8080/ping", "trace-1", &out); err != nil {
        log.Fatalf("call failed: %v", err)
    }
    log.Printf("got: %v", out)
}
```

服务端 sleep 3s、客户端 `-m 2` 时必现 `context deadline exceeded`：`curl.exe -m 2` 或 `curl --max-time 2` 均可验证。

## 常见坑

1. **每个请求 new 一个 Client**：连接池失效，压测时握手开销占一半；Client 必须全局复用。
2. **只设 `Client.Timeout` 不设 context**：超时无法跨服务传递预算，上游早断了你还在等下游。
3. **读完不 `Close()` Body**：连接无法归还池，几分钟后 fd 耗尽。
4. **`io.ReadAll` 读无限 body**：下游返回异常大响应会打爆内存，必须 `io.LimitReader`。
5. **默认跟随重定向**：`POST` 被 307 重放到别的主机，可能重复下单。
6. **重试时复用同一个 `*http.Request`**：body 已被消费，第二次发空包；需要 `GetBody` 或重建。
7. **把 5xx 当唯一重试条件**：`POST /orders` 的 5xx 也可能已落库，必须结合幂等键。

## 小结

- `http.Client` 管总超时与重定向，`Transport` 管连接池，`RoundTripper` 管横切逻辑。
- 超时拆到每一层，业务侧始终用 `context` 传递预算。
- `MaxIdleConnsPerHost` 默认 2，是客户端性能的第一优化点。
- 错误分五类，只有幂等请求的「超时 / 网络 / 5xx」才重试。

## 练习

1. 用同一个 Client 连续请求 10 次，配合服务端 `ConnState` 确认连接复用；再把 `MaxIdleConnsPerHost` 设为 1、并发 10 观察差异。
2. 服务端 sleep 3s，客户端分别用 `Client.Timeout=1s` 与 `context 1s` 调用，对比 `errors.Is(err, context.DeadlineExceeded)`。
3. 用 `httptest.NewServer` 模拟 500 与 307 重定向，验证 `CheckRedirect` 与 `Classify` 的输出。

## 官方文档

- `net/http#Client`：<https://pkg.go.dev/net/http#Client>
- `net/http#Transport`：<https://pkg.go.dev/net/http#Transport>
- `net/http#RoundTripper`：<https://pkg.go.dev/net/http#RoundTripper>
- resty：<https://github.com/go-resty/resty>
- Gin 官方文档：<https://gin-gonic.com/zh-cn/docs/>
