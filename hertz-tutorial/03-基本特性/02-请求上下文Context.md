# 请求上下文 Context

> 一句话定位：Hertz 的 handler 同时拿到标准 `context.Context` 和 `*app.RequestContext`，前者管生命周期与传值，后者管请求、响应与 handler 链。

## 本篇目标

- 分清 `context.Context` 与 `*app.RequestContext` 的职责、生命周期和并发限制。
- 熟练读取 Query、Form、路径参数、Header、Cookie、Body。
- 会用 `c.Set/Get` 在中间件与 handler 之间传值。
- 避免“Body 读两次”“把 c 丢进 goroutine”等高频事故。

## 前置知识

- 学过 Go 的 `context` 包，知道取消与超时。
- 了解 HTTP 请求由方法、路径、Query、Header、Body 组成。
- 已完成 `03-基本特性/01-路由`。

## 两个 Context 参数的区别

```go
func handler(ctx context.Context, c *app.RequestContext) {
	// ctx：标准库上下文
	// c  ：Hertz 请求上下文
}
```

| 维度 | `context.Context` | `*app.RequestContext` |
| --- | --- | --- |
| 来源 | 框架为每次请求创建 | 从 `sync.Pool` 复用 |
| 主要用途 | 取消、超时、跨层传值 | 读写请求、响应、参数、错误 |
| 是否可跨 goroutine | 可以传递，但要在生命周期内 | 不可以，handler 返回后会被重置复用 |
| 传给下游 | HTTP Client、DB、RPC 调用 | 只用于当前请求处理 |

实践原则：

- 调用下游（HTTP、DB、RPC）时把 `ctx` 传下去，让超时和取消生效；需要请求级自定义值时优先用 `c.Set/Get`。

## 请求生命周期

请求生命周期：解析 HTTP → 从池中取 `RequestContext` → 路由匹配（写入 Params / FullPath）→ 执行 handler 链 → 写响应 → 重置并放回池。

正因为最后一步会“重置并复用”，任何在 handler 返回后仍持有 `c` 的代码都会读到别的请求的数据，甚至引发数据竞争。

## 读取 Query 参数

```go
h.GET("/search", func(ctx context.Context, c *app.RequestContext) {
	keyword := c.Query("q")                    // 不存在返回 ""
	page := c.DefaultQuery("page", "1")        // 不存在返回默认值
	sort, ok := c.GetQuery("sort")             // 区分“不存在”和“空值”
	c.JSON(consts.StatusOK, utils.H{
		"q": keyword, "page": page, "sort": sort, "hasSort": ok,
	})
})
```

需要多值参数时用 `c.QueryArgs().PeekAll("tag")`，可拿到 `?tag=a&tag=b` 的全部值。

## 读取 Form 与文件

```go
h.POST("/login", func(ctx context.Context, c *app.RequestContext) {
	username := c.PostForm("username")
	remember := c.DefaultPostForm("remember", "false")
	c.JSON(consts.StatusOK, utils.H{"username": username, "remember": remember})
})
```

- `PostForm` 同时兼容 `application/x-www-form-urlencoded` 与 `multipart/form-data`。
- 原始表单：`c.FormValue("key")` 返回 `[]byte`；`c.MultipartForm()` 返回 `*multipart.Form`。

## 读取路径参数

路径参数用 `id := c.Param("id")` 读取，对应路由 `/users/:id`；它只读取路由匹配结果，不读取 Query，取不到时返回空字符串。

## 读取 Header 与 Cookie

```go
h.GET("/whoami", func(ctx context.Context, c *app.RequestContext) {
	token := string(c.GetHeader("Authorization"))
	ua := string(c.UserAgent())
	session := string(c.Cookie("session"))
	c.JSON(consts.StatusOK, utils.H{
		"token": token, "ua": ua, "session": session,
	})
})
```

设置 Cookie：

```go
c.SetCookie("session", "abc123", 3600, "/", "localhost",
	protocol.CookieSameSiteLaxMode, false, true)
```

> `c.Header(key, value)` 设置的是**响应头**，不是请求头。读取请求头请用 `c.GetHeader`。

## 读取 Body

```go
body, err := c.Body()          // 一次性读取全部 Body
if err != nil {
	c.AbortWithStatusJSON(400, utils.H{"message": "read body failed"})
	return
}

raw := c.GetRawData()          // 另一条读取入口，返回 []byte
```

注意：

- Body 只能被成功读取一次，读完后再读会得到空内容。
- 大文件不要用 `c.Body()` 全量读入内存，应使用 `c.RequestBodyStream()` 流式处理。
- JSON 场景优先用 `c.Bind/BindJSON`，不要先 `Body()` 再手动 `json.Unmarshal`，避免重复读取。

## Set / Get 传值

```go
func AuthMiddleware() app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		c.Set("user_id", "u-1001")
		c.Next(ctx)
	}
}

h.GET("/profile", func(ctx context.Context, c *app.RequestContext) {
	uid := c.GetString("user_id")     // 类型安全读取，不存在返回 ""
	raw, exists := c.Get("user_id")   // 原始值与是否存在
	c.JSON(consts.StatusOK, utils.H{"uid": uid, "exists": exists, "raw": raw})
}, AuthMiddleware())
```

常用读取方法：`GetString`、`GetBool`、`GetInt`、`GetDuration`、`MustGet`。

- key 是普通字符串，团队内要约定命名，避免中间件之间覆盖。
- `MustGet` 在 key 不存在时会 panic，只适合确定存在的场景。
- 不要用 `c.Set` 存放大对象或跨请求共享的数据。

## Abort 与 Next

```go
func AuthMiddleware() app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		if string(c.GetHeader("Authorization")) == "" {
			c.AbortWithStatusJSON(401, utils.H{"message": "unauthorized"})
			return // Abort 只终止后续 handler，不会自动 return
		}
		c.Next(ctx)
	}
}
```

- `c.Abort()`：终止后续 handler，当前函数继续执行。
- `c.AbortWithStatus(code)`：终止并设置状态码。
- `c.AbortWithStatusJSON(code, obj)`：终止并返回 JSON。
- `c.IsAborted()`：判断是否已终止。
- `c.Next(ctx)`：执行后续 handler；在中间件中用于“前后置逻辑”。

## 路径与方法信息

```go
c.Method()   // GET
c.Path()     // /users/123
c.FullPath() // /users/:id
c.Host()     // 127.0.0.1:8080
```

- `Path()` 是实际请求路径，参数已被替换；`FullPath()` 是注册模板，未匹配时为空。

## 完整示例

```go
package main

import (
	"context"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/common/utils"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
)

func RequestInfo() app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		c.Set("request_id", string(c.Method())+" "+string(c.Path()))
		c.Next(ctx)
	}
}

func main() {
	h := server.Default(server.WithHostPorts("127.0.0.1:8080"))
	h.Use(RequestInfo())

	h.GET("/echo/:name", func(ctx context.Context, c *app.RequestContext) {
		c.JSON(consts.StatusOK, utils.H{
			"name": c.Param("name"), "q": c.DefaultQuery("q", "none"),
			"request_id": c.GetString("request_id"), "full_path": c.FullPath(),
		})
	})

	h.Spin()
}
```

验证：

Windows PowerShell：

```powershell
go run .
curl.exe -i "http://127.0.0.1:8080/echo/hertz?q=hello"
curl.exe -i -X POST -d "username=tom" http://127.0.0.1:8080/login
```

Linux / macOS bash：

```bash
go run .
curl -i "http://127.0.0.1:8080/echo/hertz?q=hello"
```

## 常见坑

1. **把 `c` 传给 goroutine**：handler 返回后 `c` 会被复用，异步代码里读取会拿到别的请求数据。先把需要的数据拷贝成普通变量。
2. **Body 读两次**：先 `c.Body()` 再 `c.BindJSON()` 会失败或得到空对象，二选一。
3. **用 `c.Header` 读请求头**：它写的是响应头；读请求头用 `c.GetHeader`。
4. **`Abort` 后忘记 `return`**：当前 handler 会继续执行，可能重复写响应。
5. **`c.Set` 的 key 冲突**：多个中间件用同一个 key 会互相覆盖，统一加前缀（如 `auth.user_id`）。

## 小结

- `context.Context` 管生命周期与下游调用，`RequestContext` 管当前请求的读写，二者不能互相替代。
- Query、Form、Param、Header、Cookie、Body 各有专用读取方法，Body 只能读一次。
- `c.Set/Get` 是中间件与 handler 传值的标准方式，key 要约定命名。
- `Next` 决定链是否继续，`Abort` 只终止后续 handler，不终止当前函数。

## 练习

1. 实现 `/echo`，同时返回 Query、Header、Cookie 和请求方法，并用 curl 带自定义 Header 测试。
2. 写一个中间件，把 `X-Request-Id` 写入 `c.Set("request_id", ...)`，在 handler 中读取并返回。
3. 故意在 handler 里启动一个 goroutine 并在 100ms 后读取 `c.Param("id")`，观察并发请求下会发生什么，并给出修复方案。

## 官方文档

- 请求上下文：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/context/>
- 请求：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/context/request/>
- 响应：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/context/response/>
