# 11-客户端Client
> Hertz 不只是服务端框架：用自带客户端发请求、管连接池、配超时，并和标准库 `http.Client` 做出正确取舍。

## 本篇目标

- 用 `client.NewClient` 创建客户端，完成 GET / POST 与自定义 Header、Body。
- 理解 Hertz Client 的 URL、连接池与 keep-alive 行为，避免"以为有 WithHostPorts"的误区。
- 掌握客户端级、请求级、调用级三种超时设置方式。
- 会写客户端中间件，实现统一加签、打点、日志。
- 能在 Hertz Client 与 `http.Client` 之间做出合理选择。

## 前置知识

- 熟悉 HTTP 报文结构、Header、Content-Type、keep-alive。
- 了解 Go 的 `context.Context` 超时控制。
- 已读 03-基本特性/10，知道 Hertz 客户端默认拨号器受平台影响。

## 1. 最小可用客户端

Hertz 客户端的核心对象是 `*client.Client`，请求与响应分别用 `protocol.Request` / `protocol.Response` 表示。高频路径建议用 `Acquire*` / `Release*` 复用对象，减少分配。

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/cloudwego/hertz/pkg/app/client"
	"github.com/cloudwego/hertz/pkg/protocol"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
)

func main() {
	c, err := client.NewClient()
	if err != nil {
		log.Fatal(err)
	}

	req := protocol.AcquireRequest()
	resp := protocol.AcquireResponse()
	defer protocol.ReleaseRequest(req)
	defer protocol.ReleaseResponse(resp)

	req.SetMethod(consts.MethodGet)
	req.SetRequestURI("http://127.0.0.1:8080/ping")

	if err := c.Do(context.Background(), req, resp); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("status=%d body=%s\n", resp.StatusCode(), resp.Body())
}
```

要点：

- `Do` **不会跟随重定向**，需要跟随请用 `Get` / `Post` 或 `DoRedirects`；
- `Do` 需要请求里至少有一个非零的 `RequestURI`，或非零 Host + RequestURI；
- `resp` 用于接收响应，一般应传入已获取的 `protocol.Response`，不要传 `nil`。

### 关于 `client.WithHostPorts`

v0.10.6 的 Hertz Client **没有** `client.WithHostPorts` 这个选项（该名字是服务端 `server.WithHostPorts`）。客户端的目标地址通过以下方式指定：

- 在 `SetRequestURI` 里写完整 URL（最常用）；
- 使用服务发现时设置 `config.WithSD(true)`，由 discovery 扩展解析目标（具体以官方服务发现文档为准）。

不要因为看到旧资料写了 `client.WithHostPorts` 就照抄，编译会直接报错。

## 2. GET 与 POST

### 2.1 便捷方法 `Get` / `Post`

```go
status, body, err := c.Get(context.Background(), nil, "http://127.0.0.1:8080/ping")
// status 是 int 状态码，body 是 []byte，Get/Post 会自动跟随重定向

var args protocol.Args
args.Set("name", "hertz")
status, body, err = c.Post(context.Background(), nil, "http://127.0.0.1:8080/hello", &args)
```

`Post` 发送的是 `application/x-www-form-urlencoded` 表单；`postArgs` 为 `nil` 时发送空 body。

### 2.2 自定义 Header 与 JSON Body

```go
req.SetMethod(consts.MethodPost)
req.SetRequestURI("http://127.0.0.1:8080/v1/users")
req.Header.SetContentTypeBytes([]byte(consts.MIMEApplicationJSON))
req.Header.Set("Authorization", "Bearer <token>")
req.Header.Set("X-Request-ID", "req-10001")
req.SetBodyString(`{"name":"hertz","age":3}`)

if err := c.Do(ctx, req, resp); err != nil {
	log.Fatal(err)
}
if resp.StatusCode() != consts.StatusOK {
	log.Fatalf("unexpected status=%d body=%s", resp.StatusCode(), resp.Body())
}
```

其他常用 body 设置方式：

- `req.SetBody([]byte)`：直接写字节；
- `req.SetFormData(map[string]string)`：urlencoded 表单；
- `req.SetMultipartFormData(map[string]string)`：multipart 表单；
- `req.SetBodyStream(reader, size)`：流式上传，适合大文件。

读取响应头：`resp.Header.Get("Content-Type")`；读取 body：`resp.Body()`（返回 `[]byte`）。若开启了 `client.WithResponseBodyStream(true)`，应改用 `resp.BodyStream()` 流式读取，避免大响应全部进内存。

## 3. 连接池与复用

Hertz Client 自带 host client 连接池，keep-alive 默认开启。相关配置：

```go
c, err := client.NewClient(
	client.WithMaxConnsPerHost(1024),          // 每个 host 最大连接数，0 表示不限制
	client.WithMaxIdleConnDuration(30*time.Second), // 空闲连接存活时间
	client.WithMaxConnDuration(5*time.Minute),      // 连接最长寿命
	client.WithMaxConnWaitTimeout(2*time.Second),   // 等空闲连接的最长时间
	client.WithKeepAlive(true),
)
```

实践建议：

- 客户端对象 **全局复用**，不要每个请求 `NewClient`，否则连接池形同虚设；
- 批量任务结束后可调用 `c.CloseIdleConnections()` 主动释放空闲连接；
- 下游有连接数配额时，务必设置 `MaxConnsPerHost`，否则突发流量会打满下游。

## 4. 超时：三个层级要分清

| 层级 | 配置 | 说明 |
| --- | --- | --- |
| Client 级 | `client.WithDialTimeout` / `client.WithClientReadTimeout` / `client.WithWriteTimeout` | 对该客户端所有请求生效 |
| Request 级 | `config.WithDialTimeout` / `config.WithReadTimeout` / `config.WithWriteTimeout` / `config.WithRequestTimeout` | 优先级高于 Client 级，覆盖对应项 |
| 调用级 | `DoTimeout` / `DoDeadline` | 直接给本次调用设超时或截止时间 |

```go
import "github.com/cloudwego/hertz/pkg/common/config"

// 1) Client 级
c, _ := client.NewClient(
	client.WithDialTimeout(time.Second),
	client.WithClientReadTimeout(3*time.Second),
)

// 2) Request 级：通过 SetOptions 绑定到请求
req.SetOptions(config.WithRequestTimeout(2 * time.Second))

// 3) 调用级
err := c.DoTimeout(ctx, req, resp, 2*time.Second)
deadline := time.Now().Add(2 * time.Second)
err = c.DoDeadline(ctx, req, resp, deadline)
```

注意：`DoTimeout` / `DoDeadline` 与 `WithRequestTimeout` 本质都作用于请求超时字段，同时使用时**最后一次设置生效**；不要指望两个都写会取最小值。

## 5. 重定向

- `Get`、`Post`：自动跟随重定向；
- `Do`：不跟随；
- `DoRedirects(ctx, req, resp, maxRedirectsCount)`：最多跟随 N 次，超过返回 `ErrTooManyRedirects`。

```go
// 服务端 /redirect -> /redirect2 -> /redirect3
err := c.DoRedirects(ctx, req, resp, 2) // 跟两次，最终拿到 /redirect3 的响应
```

## 6. 客户端中间件

中间件类型是 `client.Middleware`：`func(Endpoint) Endpoint`。适合统一注入 trace id、签名、日志、指标。

```go
c.Use(func(next client.Endpoint) client.Endpoint {
	return func(ctx context.Context, req *protocol.Request, resp *protocol.Response) error {
		req.Header.Set("X-Client", "hertz-demo")
		start := time.Now()
		err := next(ctx, req, resp)
		log.Printf("method=%s uri=%s cost=%s err=%v",
			req.Method(), req.URI().String(), time.Since(start), err)
		return err
	}
})
```

注册后，通过该 client 发出的所有请求都会经过中间件。若需要"最后执行"的中间件，可用 `UseAsLast`。

## 7. Hertz Client 还是 http.Client？

| 场景 | 建议 |
| --- | --- |
| 服务内部 RPC / 高频调用，追求性能 | Hertz Client |
| 已使用 Hertz，想复用配置与中间件体系 | Hertz Client |
| 需要与大量标准库生态组件协作（OAuth、云 SDK 回调等） | `http.Client` |
| 需要复杂 TLS、代理、HTTP/2 等标准库成熟能力 | `http.Client` |
| 简单脚本、一次性请求 | 两者皆可，`http.Client` 更通用 |

Hertz Client 的 API 风格偏 fasthttp，与 `net/http` 不同；团队要统一封装，避免两套客户端散落在业务代码里。

## 常见坑

1. **以为有 `client.WithHostPorts`**：v0.10.6 不存在，编译报错；目标地址写在 URL 或走服务发现。
2. **忘记释放 `AcquireRequest` / `AcquireResponse`**：对象池失效、内存上涨；务必 `defer Release`。
3. **每次请求都 `NewClient`**：连接无法复用，性能骤降，还可能耗尽本地端口。
4. **`Do` 不跟随重定向**：拿到 302 空 body 以为接口坏了，改用 `Get` / `DoRedirects`。
5. **混淆 `WithClientReadTimeout` 与 `config.WithReadTimeout`**：前者是客户端级，后者是请求级；请求级优先。
6. **大响应不开流式**：`resp.Body()` 会把整个 body 读进内存，下载大文件应使用 `WithResponseBodyStream(true)` + `resp.BodyStream()`。
7. **HTTPS 忘了标准库拨号器**：报 `not support tls`，需加 `client.WithDialer(standard.NewDialer())`。

## 小结

- `client.NewClient` + `protocol.Request/Response` 是 Hertz 客户端的基本盘，`Get/Post` 是便捷封装。
- 目标地址写在 URL 中，v0.10.6 没有 `client.WithHostPorts`。
- 连接池靠复用 Client 生效，配合 `MaxConnsPerHost`、空闲时长等参数控制。
- 超时分 Client / Request / 调用三层，注意请求级覆盖与"最后一次生效"。
- 客户端中间件适合统一治理；与标准库生态深度协作时优先 `http.Client`。

## 练习

1. 用 Hertz Client 调用自己写的 `/ping` 与 `/echo` 接口，分别用 `Get` 和 `Do` 发送，观察重定向行为差异。
2. 构造一个每秒 1000 次请求的压测脚本，对比"复用 Client"与"每次 NewClient"的耗时和连接数。
3. 给客户端加一个中间件，为每个请求注入 `X-Request-ID` 并打印耗时。
4. 设置 Client 级读超时 1s，再用 `config.WithRequestTimeout(3s)` 覆盖，验证请求级优先级。
5. 用 `DoRedirects` 访问一个两级跳转接口，分别传 1 和 2，记录返回的错误与 body。

## 官方文档

- 客户端：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/client/>
- 网络库（拨号器选择）：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/network-lib/>
- 重试：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/retry/>
