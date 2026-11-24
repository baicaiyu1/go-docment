# SSE 与流式响应

> 一句话定位：SSE 是「chunked + 文本事件」的长连接，Gin 用 `c.SSEvent` / `c.Stream` 加 `Flush` 就能实现；真正的难点是心跳、断开检测、`WriteTimeout` 与代理缓冲。

## 本篇目标

- 记住 SSE 协议格式与 `data` / `id` / `event` / `retry` / 注释行的含义。
- 会用 `c.SSEvent` 推送事件，用 `c.Stream` 做心跳与持续流。
- 能同时用「写失败」与 `c.Request.Context().Done()` 检测客户端断开并释放资源。
- 会处理 `WriteTimeout`、Nginx/APISIX 缓冲、gzip 三大工程坑。
- 会做 NDJSON 流与「逐字输出」的 LLM 转发，并用 `curl -N` 验证。

## 前置知识

- [响应与渲染](04-响应与渲染.md)
- 了解 HTTP/1.1 chunked 编码、`http.Flusher` 与请求 context 的取消语义。

## 一、SSE 协议速览

SSE 不是新协议，而是 HTTP 响应的一种用法：`Content-Type: text/event-stream`，body 是一行行文本，事件之间用**空行**分隔。

```text
Content-Type: text/event-stream; charset=utf-8
Cache-Control: no-cache

id: 1
event: message
data: hello

data: 第一行
data: 第二行
```

规则：`data:` 可出现多次（客户端拼成多行）；`id:` 用于断线后 `Last-Event-ID` 续传；`event:` 指定事件类型（默认 `message`）；`retry:` 设置浏览器重连间隔（毫秒）；以 `:` 开头是注释行，常用于心跳；必须持续 `Flush`，否则会被框架或代理缓冲。

## 二、最小可用：c.SSEvent

```go
// internal/handler/sse.go
func SSEOnce(c *gin.Context) {
    c.Header("Cache-Control", "no-cache")
    c.Header("X-Accel-Buffering", "no") // 关闭 Nginx 缓冲

    for i := 1; i <= 5; i++ {
        c.SSEvent("message", gin.H{"seq": i, "ts": time.Now().Unix()})
        c.Writer.Flush() // 立即推给客户端
        time.Sleep(time.Second)
    }
}
```

`c.SSEvent` 会自动写 `event:` 与 `data:`（`interface{}` 会被 JSON 序列化），它只写一条事件，不结束请求也不循环。

## 三、心跳 + 断开检测：c.Stream

长连接必须发心跳（15~30s），否则 NAT / 网关空闲超时会断链：

```go
func SSEStream(c *gin.Context) {
    c.Header("Content-Type", "text/event-stream; charset=utf-8")
    c.Header("Cache-Control", "no-cache")
    c.Header("X-Accel-Buffering", "no")

    dataCh := time.Tick(time.Second)
    hb := time.Tick(15 * time.Second)
    seq := 0 // c.Stream 的回调串行执行，无需加锁

    c.Stream(func(w io.Writer) bool {
        select {
        case <-c.Request.Context().Done():
            return false // 客户端断开，结束循环并释放资源
        case <-hb:
            fmt.Fprint(w, ": ping\n\n")
            return true
        case <-dataCh:
            seq++
            fmt.Fprintf(w, "id: %d\ndata: {\"ts\":%d}\n\n", seq, time.Now().Unix())
            return true
        }
    })
}
```

`c.Stream` 每轮自动 `Flush`；`c.Request.Context()` 在客户端断开时被取消，比「等写失败」更早。两者都要用：写失败证明连接已断，`ctx.Done()` 负责唤醒阻塞中的 `select`。

## 四、三个工程陷阱

**1）`WriteTimeout` 会杀死 SSE。** 全局不设 `WriteTimeout`，或只对 SSE 路由清除写截止时间（Go 1.20+）：

```go
func NoWriteDeadline() gin.HandlerFunc {
    return func(c *gin.Context) {
        // 清除本请求的写截止时间，避免长连接被 WriteTimeout 切断
        if err := http.NewResponseController(c.Writer).SetWriteDeadline(time.Time{}); err != nil {
            log.Printf("clear write deadline: %v", err)
        }
    }
}
// r.GET("/events", NoWriteDeadline(), SSEStream)
```

**2）代理缓冲。** Nginx 默认 `proxy_buffering on`，事件会被攒成大块下发；必须设 `X-Accel-Buffering: no`，或让 APISIX 路由关闭缓冲。

**3）gzip。** 压缩器内部还有缓冲，会让 `Flush` 失效；SSE 路由必须跳过 gzip 中间件。

## 五、NDJSON 流

客户端不用 `EventSource` 时（移动端、命令行），NDJSON（每行一个 JSON）更简单：

```go
func NDJSON(c *gin.Context) {
    c.Header("Content-Type", "application/x-ndjson")
    c.Header("X-Accel-Buffering", "no")
    enc := json.NewEncoder(c.Writer) // Encode 自带换行，正好符合 NDJSON
    for i := 0; i < 100; i++ {
        _ = enc.Encode(map[string]any{"seq": i, "v": rand.Intn(100)})
        c.Writer.Flush()
        select {
        case <-c.Request.Context().Done():
            return
        case <-time.After(100 * time.Millisecond):
        }
    }
}
```

Web 端要用 `fetch` + `ReadableStream` 读取 NDJSON，不能用 `EventSource`。

## 六、LLM 场景：转发下游 SSE

网关常见需求：上游大模型返回 SSE，BFF 边读边写。

```go
func ProxyChat(cli *http.Client) gin.HandlerFunc {
    return func(c *gin.Context) {
        req, err := http.NewRequestWithContext(c.Request.Context(), http.MethodPost,
            "http://llm-upstream/v1/chat/completions", c.Request.Body)
        if err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"code": 40001, "message": err.Error()})
            return
        }
        req.Header.Set("Accept", "text/event-stream")
        resp, err := cli.Do(req) // 注意：Client.Timeout 必须为 0，否则流会被切断
        if err != nil {
            c.JSON(http.StatusBadGateway, gin.H{"code": 50201, "message": "upstream error"})
            return
        }
        defer resp.Body.Close()
        c.Header("Content-Type", "text/event-stream; charset=utf-8")
        c.Header("X-Accel-Buffering", "no")
        c.Status(resp.StatusCode)

        sc := bufio.NewScanner(resp.Body)
        sc.Buffer(make([]byte, 64*1024), 1<<20) // 单行可能很长，必须调大
        for sc.Scan() {
            c.Writer.Write(append(sc.Bytes(), '\n')) // 透传原始 SSE 行
            c.Writer.Flush()
            if c.Request.Context().Err() != nil || sc.Err() != nil {
                return
            }
        }
    }
}
```

## 完整示例

```go
// cmd/sse/main.go（SSEStream 见第三节，自带 Content-Type、心跳与断开检测）
package main

import (
    "log"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.New()
    r.GET("/events", SSEStream)
    log.Fatal((&http.Server{Addr: ":8080", Handler: r, ReadHeaderTimeout: 3 * time.Second}).ListenAndServe())
}
```

验证（`-N` 关闭 curl 缓冲）：bash 用 `curl -N -i http://127.0.0.1:8080/events`，PowerShell 用 `curl.exe -N -i http://127.0.0.1:8080/events`。中途 `Ctrl+C`，服务端应立即结束该请求。

## 常见坑

1. **找「推送」方法**：Gin 只有 `c.SSEvent` 与 `c.Stream`，没有广播池；多客户端广播要自己维护 `chan` 列表。
2. **忘记 `Flush`**：数据被缓冲吞掉，客户端长时间无输出；`c.Stream` 自动刷，手写循环必须显式 `c.Writer.Flush()`。
3. **`WriteTimeout` 未清空**：长连接每 N 秒被切断，前端疯狂重连。
4. **gzip 中间件包住 SSE**：`Flush` 失效，事件被攒住。
5. **`bufio.Scanner` 默认 64KB**：长 JSON 行报 `token too long`，务必 `sc.Buffer` 调大；转发场景的 `http.Client.Timeout` 也要设 0。

## 小结

- SSE = `text/event-stream` + 空行分隔事件 + 持续 `Flush`；`id` 支持断线续传；Gin 提供 `c.SSEvent`（单条）与 `c.Stream`（循环 + 自动 Flush）。
- 断线靠 `c.Request.Context().Done()` 与写失败双保险；生产必配心跳、`WriteTimeout=0`、`X-Accel-Buffering: no` 并关闭 gzip。

## 练习

1. 实现 `/events`，每秒推一条、15 秒一次心跳，用 `curl -N` 观察并验证 `Ctrl+C` 后服务端日志立即结束。
2. 加一个 `/events-ndjson`，用 `curl -N ... | head -n 3` 对比两种格式。
3. 写一个中间件在 SSE 结束时打印存活时长与推送条数，验证断开能被感知。

## 官方文档

- WHATWG SSE 规范：<https://html.spec.whatwg.org/multipage/server-sent-events.html>
- `net/http#ResponseController`：<https://pkg.go.dev/net/http#ResponseController>
- Gin `Context.SSEvent`：<https://pkg.go.dev/github.com/gin-gonic/gin#Context.SSEvent>
- Gin `Context.Stream`：<https://pkg.go.dev/github.com/gin-gonic/gin#Context.Stream>
- MDN Server-sent events：<https://developer.mozilla.org/zh-CN/docs/Web/API/Server-sent_events>
