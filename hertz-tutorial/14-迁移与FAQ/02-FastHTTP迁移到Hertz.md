# 02-FastHTTP迁移到Hertz

> 一句话定位：fasthttp 与 Hertz 都走零拷贝与对象池，但前者是 HTTP 引擎、后者是完整框架；迁移的关键是重写路由与中间件，并守住 byte slice 的生命周期。

## 本篇目标

- 看清 fasthttp 与 Hertz/netpoll 在设计、网络模型与生态上的差异。
- 掌握 `RequestCtx` ↔ `RequestContext` 的参数、Body、响应写法对照。
- 理解 Body 只能读一次、零拷贝字符串生命周期、连接复用与对象池。
- 按“先跑通再优化”的步骤迁移，并设计可执行的回归测试。

## 前置知识

- 已读 [网络库 Netpoll 与标准库](../03-基本特性/10-网络库Netpoll与标准库.md) 与 [客户端 Client](../03-基本特性/11-客户端Client.md)。
- 用过 fasthttp，或至少用过 `fasthttp/router`、`fasthttprouter` 之一。

## 一、设计差异

| 维度 | fasthttp | Hertz |
| --- | --- | --- |
| 定位 | HTTP 引擎，不含路由/中间件/绑定 | 完整 HTTP 框架 |
| 网络模型 | `net.Listener` + 每连接一个 goroutine，bufio 读写 | Linux 默认 netpoll 事件循环；Windows/TLS 回退标准库 |
| 路由 | 需 `fasthttp/router` 或 `fasthttprouter` | 内置 radix 路由树 |
| Handler | `func(*fasthttp.RequestCtx)` | `func(context.Context, *app.RequestContext)` |
| 上下文 | `RequestCtx` 池化 | `RequestContext` 池化 |
| 中间件 | 自行包装成 `func(next) next` | `app.HandlerFunc` + `Use/Group` |
| 绑定校验 | 无，自己解析 | `Bind` / `BindAndValidate` |
| 客户端 | `fasthttp.HostClient` | Hertz Client，内置连接池与中间件 |
| 响应 | `SetBody/Write`，JSON 自己 marshal | `c.JSON/c.String/c.Data` |
| TLS | 支持 | 支持；netpoll 不支持 TLS，会自动切标准库 |

netpoll 用 epoll 水平触发、连接由网络库托管，适合小包高并发；fasthttp 的 goroutine-per-connection 模型更易调试，但连接数很高时 goroutine 与栈成本更明显。请求体大于 1MB 时，Hertz 建议切标准库并做流式处理。

## 二、API 对照表

| 能力 | fasthttp | Hertz |
| --- | --- | --- |
| 路由注册 | `router.GET("/users/{id}", h)` | `h.GET("/users/:id", h)` |
| 路径参数 | `ctx.UserValue("id")` | `c.Param("id")` |
| Query | `ctx.QueryArgs().Peek("page")` | `c.Query("page")` |
| 表单 | `ctx.PostArgs().Peek("name")` | `c.PostForm("name")` |
| 表单+Query | `ctx.FormValue("k")` | `c.FormValue("k")` |
| Body | `ctx.Request.Body()` | `c.Body()` / `c.GetRawData()` |
| Header | `ctx.Request.Header.Peek("X")` | `c.GetHeader("X")` |
| Cookie | `ctx.Request.Header.Cookie("sid")` | `c.Cookie("sid")` |
| 请求级存储 | `ctx.SetUserValue/UserValue` | `c.Set/Get` |
| 状态码 | `ctx.SetStatusCode(code)` | `c.SetStatusCode(code)` |
| Content-Type | `ctx.SetContentType(v)` | `c.SetContentType(v)` |
| 写 Body | `ctx.SetBodyString/WriteString` | `c.SetBodyString/WriteString` |
| JSON | `json.Marshal` + `ctx.SetBody` | `c.JSON(code, obj)` |
| 重定向 | `ctx.Redirect(url, code)` | `c.Redirect(code, []byte(url))` |
| 文件 | `ctx.SendFile(path)` | `c.File(path)` |
| 流式 Body | `ctx.SetBodyStream(r, n)` | `c.SetBodyStream(r, n)` |
| 中间件 | `func(next fasthttp.RequestHandler)` | `app.HandlerFunc` |
| 关闭 | `srv.Shutdown()` | `h.Shutdown(ctx)` |

## 三、逐项代码对照

### 3.1 Handler 与路由

fasthttp + `fasthttp/router`：

```go
r := router.New()
r.GET("/users/{id}", func(ctx *fasthttp.RequestCtx) {
	id, _ := ctx.UserValue("id").(string)
	ctx.SetContentType("application/json")
	ctx.SetBodyString(`{"id":"` + id + `"}`)
})
log.Fatal(fasthttp.ListenAndServe(":8080", r.Handler))
```

Hertz：

```go
h := server.Default(server.WithHostPorts(":8080"))
h.GET("/users/:id", func(ctx context.Context, c *app.RequestContext) {
	c.JSON(consts.StatusOK, utils.H{"id": c.Param("id")})
})
h.Spin()
```

路由语法必须改：`fasthttp/router` 用 `{id}`、`{filepath:*}`，`fasthttprouter` 用 `:id`、`*filepath`；Hertz 统一 `:id`、`*filepath`。

### 3.2 参数与 Body

fasthttp：

```go
page := string(ctx.QueryArgs().Peek("page"))
name := string(ctx.PostArgs().Peek("name"))
body := ctx.Request.Body()
token := string(ctx.Request.Header.Peek("Authorization"))
```

Hertz：

```go
page := c.Query("page")
name := c.PostForm("name")
body, err := c.Body()
token := string(c.GetHeader("Authorization"))
```

### 3.3 响应

fasthttp：

```go
ctx.SetStatusCode(fasthttp.StatusCreated)
ctx.SetContentType("application/json")
b, _ := json.Marshal(payload)
ctx.SetBody(b)
```

Hertz：

```go
c.JSON(consts.StatusCreated, payload)
```

注意 `Redirect` 参数顺序相反：fasthttp 是 `(url, code)`，Hertz 是 `(code, []byte(url))`；`SendFile` 对应 `c.File`。

### 3.4 中间件适配

fasthttp：

```go
func Auth(next fasthttp.RequestHandler) fasthttp.RequestHandler {
	return func(ctx *fasthttp.RequestCtx) {
		if len(ctx.Request.Header.Peek("Authorization")) == 0 {
			ctx.SetStatusCode(fasthttp.StatusUnauthorized)
			return
		}
		ctx.SetUserValue("uid", "u-1")
		next(ctx)
	}
}
```

Hertz：

```go
func Auth() app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		if len(c.GetHeader("Authorization")) == 0 {
			c.AbortWithStatusJSON(consts.StatusUnauthorized, utils.H{"code": 401})
			return
		}
		c.Set("uid", "u-1")
		c.Next(ctx)
	}
}
```

Hertz 核心库只提供 net/http 的 `adaptor.HertzHandler`，没有 fasthttp 适配器；中间件最稳的做法是按语义重写，社区若有适配器以官方文档为准。

### 3.5 启动、关闭与连接复用

fasthttp：

```go
srv := &fasthttp.Server{Handler: handler}
go func() { _ = srv.ListenAndServe(":8080") }()
_ = srv.Shutdown()
```

Hertz：

```go
h := server.Default(server.WithHostPorts(":8080"), server.WithExitWaitTime(10*time.Second))
h.Spin() // 内部处理信号并优雅退出
```

客户端同理：`fasthttp.HostClient` 要全局复用；Hertz 用 `client.NewClient` 全局复用，不要每请求新建。

## 四、性能与兼容取舍

- **网络模型**：fasthttp 每连接一个 goroutine；Hertz 在 Linux 用小包更省调度，但大包与 TLS 要回退标准库。
- **HTTP/2**：fasthttp 原生不支持；Hertz 支持 TLS + ALPN 与 h2c（`WithH2C`）。
- **WebSocket/Hijack**：fasthttp 可直接 Hijack；Hertz 在 netpoll 下能力有限，需切标准库并验证。
- **生态**：fasthttp 第三方库多；Hertz 有 `hertz-contrib`，但覆盖面小于 net/http。
- **零拷贝**：两者都返回内部 byte slice，生命周期规则一致，不能跨 handler 保存。
- **性能结论**：必须用自己的请求体大小、并发与 TLS 场景压测，不要照搬博客数字。

## 五、迁移步骤（先跑通再优化）

| 步骤 | 动作 | 完成标准 |
| --- | --- | --- |
| 1 冻结基线 | 记录路由表、响应样例、错误码、压测数据 | 有可回放的用例集 |
| 2 抽离业务 | handler 只留参数与响应，逻辑移到 service | service 可单测 |
| 3 搭骨架 | `hz new` 或手写 `server.Default`，先接只读接口 | `/ping` 与一个查询接口跑通 |
| 4 逐接口重写 | 路由语法、取值、响应、错误处理 | 单接口响应与旧版一致 |
| 5 重写中间件 | 鉴权、日志、CORS、限流、Recovery | 失败路径行为一致 |
| 6 回归 | 按下节清单跑自动化与手工用例 | 全绿 |
| 7 灰度与压测 | 按比例切流，对比 P99、内存、连接数 | 指标不劣于基线 |

先跑通再优化：不要在第一步就上 netpoll 调参、对象池或自研协议；先把行为对齐，再针对大包、TLS、长连接逐项优化。

## 六、回归测试要点

| 类别 | 必测项 |
| --- | --- |
| 参数 | Query 数组与空值、表单、multipart、路径参数、通配 |
| Body | 只读一次、空 Body、超大 Body、chunked |
| Header/Cookie | 大小写、多值、`Set-Cookie` 属性 |
| 响应 | JSON 字段与转义、状态码、重定向、文件下载 |
| 路由 | 404/405、尾斜杠、静态资源 |
| 连接 | keep-alive、空闲超时、并发复用 |
| 安全 | TLS 握手、鉴权失败、CORS 预检 |
| 并发 | `go test -race` 下对象池与共享状态 |

## 常见坑

1. **Body 只能读一次**：fasthttp 的 `ctx.Request.Body()` 与 Hertz 的 `c.Body()` 都直接消费底层缓冲；读完要缓存，不要再调 `Bind`。
2. **零拷贝字符串生命周期**：`ctx.QueryArgs().Peek()`、`c.GetHeader()` 返回的 `[]byte` 只在 handler 内有效；需要保留用 `string(b)`、`strings.Clone` 或 `append([]byte(nil), b...)` 拷贝。
3. **路由语法照搬**：`{id}` 是 `fasthttp/router` 语法，Hertz 只认 `:id`。
4. **`Redirect` 参数顺序**：fasthttp 是 `(url, code)`，Hertz 是 `(code, []byte(url))`。
5. **把 `ctx` 传给 goroutine**：两套上下文都会复用，异步任务必须先拷贝所需值。
6. **TLS 与 netpoll 组合**：Hertz 显式 `WithTransport(netpoll.NewTransporter)` 后再配 `WithTLS` 会握手失败；TLS 场景只用 `WithTLS` 或显式标准库。
7. **大 Body 一次性读入**：大于 1MB 的请求切标准库，并用 `c.RequestBodyStream()` 或 `WithStreamBody(true)`。
8. **以为 Hertz 会复用 fasthttp 中间件**：签名不同，必须重写；net/http 中间件也只能通过 `adaptor` 过渡。

## 小结

- 两者共享零拷贝与对象池思想，但 Hertz 多了路由、绑定、中间件与客户端，迁移重点是重写而不是替换类型。
- 参数、Body、响应的 API 可以一一对应，`Redirect` 参数顺序与路由语法是高频错误点。
- 先冻结行为、抽离业务、逐接口重写，再灰度与压测，最后才调网络库与流式参数。
- Body 单次读取与 byte slice 生命周期是迁移期最容易埋下的稳定性问题。

## 练习

1. 把一段 fasthttp + `fasthttp/router` 的 GET/POST 接口改成 Hertz，保持响应完全一致。
2. 写一个中间件，分别用 fasthttp 与 Hertz 实现鉴权，比较两者“未通过时是否继续执行”的差异。
3. 构造 2MB 请求体，分别测试默认与流式读取的内存占用，并解释为什么要切标准库。
4. 故意在 goroutine 中引用 `c.GetHeader` 的返回值，用 `-race` 或数据比对复现生命周期问题。

## 相关章节

- [网络库 Netpoll 与标准库](../03-基本特性/10-网络库Netpoll与标准库.md)
- [请求上下文 Context](../03-基本特性/02-请求上下文Context.md)
- [客户端 Client](../03-基本特性/11-客户端Client.md)
- [适配器与标准库互操作](../03-基本特性/14-适配器与标准库互操作.md)
- [Gin 迁移到 Hertz](01-Gin迁移到Hertz.md)
- [常见问题 FAQ](03-常见问题FAQ.md)
