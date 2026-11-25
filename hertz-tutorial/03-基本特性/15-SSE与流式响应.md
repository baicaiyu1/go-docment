# 15-SSE与流式响应
> 从 `text/event-stream` 协议到 Hertz 的 `sse.Writer`，再到心跳、断线检测与大文件流式下载。

## 本篇目标

- 理解 SSE 协议格式：`data`、`id`、`event`、`retry` 与注释行。
- 用 `pkg/protocol/sse` 在 Hertz 中实现 SSE 服务端与客户端。
- 掌握心跳保活、客户端断线检测与资源释放的正确姿势。
- 区分 SSE、普通分块响应与大文件流式下载的适用场景。
- 避开代理缓冲、Handler 返回后继续写、`c.Stream` 不存在等坑。

## 前置知识

- 熟悉 HTTP/1.1 chunked 传输与 `Flush` 的概念。
- 了解 `text/event-stream` 与浏览器 `EventSource` 的基本行为。
- 已读 03-基本特性/02，知道 Handler 的 `ctx` 与 `c` 的生命周期。

## 1. SSE 协议速览

SSE 是 HTML 标准的一部分，不是独立协议。服务端返回：

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream; charset=utf-8

id: 1
event: message
data: hello

data: 第一行
data: 第二行

: keep-alive

```

规则要点：

- 每个事件以**空行**结束；
- `data:` 可以出现多次，客户端用换行拼接；
- `id:` 用于断线重连时通过 `Last-Event-ID` 续传；
- `event:` 指定事件类型，默认 `message`；
- `retry:` 告诉浏览器重连间隔（毫秒）；
- 以 `:` 开头的是注释行，浏览器忽略，常用来做心跳。
- 由于是长连接，必须使用 chunked 编码并持续 `Flush`，否则会被缓冲。

## 2. Hertz 的 SSE 支持

v0.10.6 的 SSE 实现在 `github.com/cloudwego/hertz/pkg/protocol/sse`，核心是 `sse.Writer`。注意：

- **没有 `c.SetSSE` 这个方法**，也不存在 `c.Stream`（旧资料或别的框架的 API）；
- 正确入口是 `sse.NewWriter(c)`，它会自动设置 `Content-Type: text/event-stream`、`Cache-Control: no-cache`，并接管 chunked writer。

### 2.1 服务端最小示例

```go
import (
	"context"
	"fmt"
	"time"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/protocol/sse"
)

func handleSSE(ctx context.Context, c *app.RequestContext) {
	w := sse.NewWriter(c)
	defer w.Close() // 可选，Handler 返回时 Hertz 也会收尾

	for i := 1; i <= 5; i++ {
		err := w.WriteEvent(fmt.Sprintf("id-%d", i), "message", []byte("hello sse"))
		if err != nil {
			// 客户端断开或写入失败，立即退出，释放资源
			return
		}
		time.Sleep(time.Second)
	}
}
```

注册：`h.GET("/events", handleSSE)`。

`Writer` 提供 `WriteEvent`、`Write`、`WriteComment`、`WriteKeepAlive`、`Close` 等方法。

### 2.2 心跳保活与优雅退出

长连接必须发心跳，防止网关/NAT 超时断链。推荐用两个 ticker：

```go
func streamingHandler(ctx context.Context, c *app.RequestContext) {
	w := sse.NewWriter(c)
	defer w.Close()

	dataTicker := time.NewTicker(time.Second)
	defer dataTicker.Stop()
	hbTicker := time.NewTicker(15 * time.Second)
	defer hbTicker.Stop()

	for {
		select {
		case <-ctx.Done():
			// 客户端断开或服务端取消
			return
		case <-hbTicker.C:
			if err := w.WriteKeepAlive(); err != nil {
				return
			}
		case <-dataTicker.C:
			if err := w.WriteEvent("", "message", []byte(time.Now().Format(time.RFC3339))); err != nil {
				return
			}
		}
	}
}
```

注意 `id` 传空字符串表示不写 `id:` 字段；`eventType` 传空表示不写 `event:`。

## 3. 客户端断线检测

两条路可以感知断开：

1. **写失败**：`WriteEvent` / `WriteKeepAlive` 返回错误，通常就是客户端关闭连接（日志里可能是 `connection has been closed when flush`）；
2. **Context 取消**：开启 `server.WithSenseClientDisconnection(true)` 后，客户端断开会让 Handler 的 `ctx` 被取消。

```go
h := server.Default(
	server.WithHostPorts(":8080"),
	server.WithSenseClientDisconnection(true),
)
```

对 SSE Proxy 这类"边读下游边写上游"的场景，开启该选项后把 Handler 的 `ctx` 传给下游 `sse.Reader.ForEach`，客户端一断开就能中断整条链路，避免 goroutine 泄漏。

## 4. SSE 客户端

Hertz 客户端可以像普通请求一样发起 SSE，然后用 `sse.Reader` 逐事件读取：

```go
func consumeSSE(ctx context.Context) error {
	c, _ := client.NewClient()
	req := protocol.AcquireRequest()
	defer protocol.ReleaseRequest(req)
	resp := protocol.AcquireResponse()
	defer protocol.ReleaseResponse(resp)

	req.SetMethod("GET")
	req.SetRequestURI("http://127.0.0.1:8080/events")
	sse.AddAcceptMIME(req)          // 添加 Accept: text/event-stream
	sse.SetLastEventID(req, "id-3") // 断线续传时告诉服务端从哪继续
	if err := c.Do(ctx, req, resp); err != nil {
		return err
	}

	r, err := sse.NewReader(resp)
	if err != nil {
		return err
	}
	defer r.Close()
	return r.ForEach(ctx, func(e *sse.Event) error {
		fmt.Printf("id=%s type=%s data=%s\n", e.ID, e.Type, e.Data)
		return nil
	})
}
```

新版 Hertz 会自动识别 SSE 流，一般不需要手动设置 `client.WithResponseBodyStream(true)`；若遇到读取不到事件的情况，再检查该选项与服务端是否 flush。

## 5. 大文件流式下载

不要把大文件读进内存再用 `c.Data` 返回。使用 `SetBodyStream`，让协议层边读边写：

```go
func download(c *app.RequestContext) {
	path := "./files/report.zip"
	f, err := os.Open(path)
	if err != nil {
		c.String(404, "not found")
		return
	}
	defer f.Close()

	fi, err := f.Stat()
	if err != nil {
		c.String(500, "stat failed")
		return
	}

	c.Response.Header.SetContentType("application/octet-stream")
	c.Response.Header.Set("Content-Disposition", `attachment; filename="report.zip"`)
	// bodySize >= 0 时表示精确长度；< 0 表示读到 EOF（chunked）
	c.SetBodyStream(f, int(fi.Size()))
	// Handler 返回后，Hertz 负责把流写完并关闭
}
```

`SetBodyStream(reader, -1)` 适合长度未知的流；文件有明确大小时传真实 size，客户端能显示进度。

### 5.1 分块写与 Flush

如果数据是动态生成的，可以边写边刷：

```go
func chunked(c *app.RequestContext) {
	c.SetContentType("text/plain; charset=utf-8")
	chunks := []string{"第一段\n", "第二段\n", "第三段\n"}
	for _, s := range chunks {
		if _, err := c.Write([]byte(s)); err != nil {
			return
		}
		if err := c.Flush(); err != nil { // 立即把已缓冲数据推给客户端
			return
		}
		time.Sleep(500 * time.Millisecond)
	}
}
```

需要精细控制 chunked 行为时，可参考 `sse.Writer` 的实现：通过 `c.Response.HijackWriter` 接管底层 writer。普通业务优先使用 `SetBodyStream` 或 `sse.Writer`，不要自己造轮子。

## 常见坑

1. **找 `c.SetSSE` 或 `c.Stream`**：v0.10.6 不存在，正确入口是 `sse.NewWriter(c)` 与 `SetBodyStream`。
2. **忘了 Flush**：数据被缓冲在框架或代理里，客户端长时间收不到事件；SSE Writer 内部已 Flush，自定义流要手动调用。
3. **没有心跳**：网关 60s 空闲超时会切断连接，客户端不断重连；建议 15~30s 一次 `WriteKeepAlive`。
4. **中间代理缓冲 SSE**：Nginx 默认 `proxy_buffering on`，需要 `proxy_buffering off` 或 `X-Accel-Buffering: no`；同时对 SSE 关闭 gzip。
5. **Handler 返回后还在 goroutine 里写 / 客户端断开不退出**：`Writer` 生命周期与 Handler 绑定，且要同时依赖写错误与 `ctx` 取消来释放资源。

## 小结

- SSE 是基于 chunked 的文本协议，事件以空行分隔，注释行用于心跳。
- Hertz 用 `sse.NewWriter` 提供 SSE，`WriteEvent` / `WriteKeepAlive` / `Close` 是核心方法。
- 断线检测靠写错误与 `WithSenseClientDisconnection` + `ctx`。
- 大文件用 `SetBodyStream`，动态输出用 `Write` + `Flush`；`c.SetSSE`、`c.Stream` 在 v0.10.6 不存在。

## 练习

1. 实现一个每秒推送一条消息、每 15 秒心跳的 SSE 接口，用 `curl -N` 观察输出。
2. 在推送过程中强制关闭 curl，观察服务端日志与 Handler 是否及时退出。
3. 用 Hertz Client + `sse.Reader` 消费事件，并保存 `LastEventID` 实现断线续传。
4. 实现一个 1GB 文件的下载接口，分别用 `c.Data` 与 `SetBodyStream`，对比内存占用。
5. 在 Nginx 后面部署 SSE，分别开启/关闭 `proxy_buffering`，记录客户端收到数据的时间差。

## 官方文档

- SSE：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/sse/>
- 响应与渲染：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/render/>
- 客户端：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/client/>
