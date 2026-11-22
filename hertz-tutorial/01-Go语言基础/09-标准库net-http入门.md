# 标准库 net/http 入门

> 不用任何框架，用标准库写一个带路由、JSON、中间件的 HTTP 服务；写完你就知道 Hertz 到底帮你省了什么。

## 本篇目标

- 用 `http.ListenAndServe` 起一个可访问的 HTTP 服务，理解 `ServeMux` 路由
- 掌握 `http.Handler` 接口与 `HandlerFunc` 适配器，能手写处理器
- 正确使用 `http.Request` 与 `ResponseWriter`，避开响应头顺序陷阱
- 用函数包装写出日志、鉴权、recover 三个中间件
- 对比 Hertz 的写法，说清迁移时哪些概念一一对应

## 前置知识

- 已完成 [08-包管理与项目结构](08-包管理与项目结构.md)
- 理解接口、闭包、错误处理与 `context`
- 会用 `encoding/json` 的基本序列化

## 一、最小 HTTP 服务

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "ok")
	})

	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
```

```powershell
go run .
curl.exe -i http://127.0.0.1:8080/healthz
```

```bash
go run .
curl -i http://127.0.0.1:8080/healthz
```

输出中包含 `HTTP/1.1 200 OK` 与 `ok`，服务就跑起来了。`nil` 表示使用 `http.DefaultServeMux`（全局默认路由）。**生产代码不要用全局 mux**，因为它是一个全局变量，任何导入的包都能往里注册路由，容易冲突，也不利于测试；应该自己 `http.NewServeMux()`。

## 二、ServeMux 与路径参数

Go 1.22 起，`ServeMux` 支持方法匹配与路径参数：

```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id")
	fmt.Fprintf(w, "user id = %s", id)
})
mux.HandleFunc("POST /users", createUser)
```

规则：`GET /users/{id}` 只匹配 GET，`{id}` 捕获一段路径；`{path...}` 捕获剩余全部路径（如静态文件）；精确路径优先于通配，最长匹配优先；注册冲突的模式会在 `Handle` 时 panic。

这与 Hertz 的 `h.GET("/users/:id", ...)` 很接近，区别是语法：标准库用 `{id}`，Hertz 用 `:id`。

## 三、Handler 接口与 HandlerFunc

`http.Handler` 只有一个方法：

```go
type Handler interface {
	ServeHTTP(w http.ResponseWriter, r *http.Request)
}
```

自定义类型实现它：`type UserHandler struct{ repo *UserRepo }`，然后写 `func (h *UserHandler) ServeHTTP(...)`。普通函数通过 `http.HandlerFunc` 适配成 Handler：

```go
mux.Handle("/users", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}))
```

`HandlerFunc` 的定义是「一个函数类型 + 一个 ServeHTTP 方法」，这正是 [04-结构体-方法-接口](04-结构体-方法-接口.md) 里讲的「函数适配接口」技巧。

## 四、Request：读请求

| 字段 / 方法 | 说明 |
| --- | --- |
| `r.Method` | HTTP 方法 |
| `r.URL.Path` | 路径（不含查询串） |
| `r.URL.Query()` | 解析后的查询参数 |
| `r.Header.Get("X")` | 读取请求头，大小写不敏感 |
| `r.Body` | 请求体，**用完必须 Close** |
| `r.Context()` | 请求级 context，随客户端断开而取消 |
| `r.PathValue("id")` | 取路径参数（Go 1.22+） |

读取 JSON 请求体：

```go
type CreateUserReq struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}

func createUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()

	var req CreateUserReq
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}
	if req.Name == "" {
		http.Error(w, "name is required", http.StatusBadRequest)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	_ = json.NewEncoder(w).Encode(map[string]any{"id": 1, "name": req.Name})
}
```

`http.Error` 会自动设置 `Content-Type: text/plain` 并写入状态码，适合快速返回错误。

## 五、ResponseWriter：写响应

顺序非常关键，必须遵守：先设置响应头 `w.Header().Set(...)` → 再调用 `w.WriteHeader(status)` → 最后 `w.Write(body)`。

一旦调用 `WriteHeader`，状态码和响应头就发出去了，之后再改无效，且**重复调用会打印警告**：

```go
w.Header().Set("Content-Type", "application/json")
w.WriteHeader(http.StatusBadRequest)
w.Write([]byte(`{"msg":"bad"}`))
// w.WriteHeader(500) // 无效，还会告警
```

如果没显式调用 `WriteHeader`，第一次 `Write` 会隐式写入 200。所以错误场景一定要先写状态码。工具函数：

```go
func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(status)
	_ = json.NewEncoder(w).Encode(v)
}
```

## 六、中间件：用函数包装 Handler

中间件的本质是 `func(http.Handler) http.Handler`：

```go
func logging(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		next.ServeHTTP(w, r)
		log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
	})
}

func auth(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		if r.Header.Get("Authorization") == "" {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}
		next.ServeHTTP(w, r)
	})
}

func recovery(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("panic: %v\n%s", rec, debug.Stack())
				http.Error(w, "internal error", http.StatusInternalServerError)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

组合：`handler := logging(auth(recovery(mux)))`。执行顺序是**从外到内**：logging 最先进入、最后退出；recovery 在最内层兜底。Hertz 的 `h.Use(mw1, mw2)` 同样是洋葱模型，只是 API 更简洁。

## 七、生产配置：超时

把服务换成 `http.Server`，设置三个超时：

```go
srv := &http.Server{
	Addr:         ":8080",
	Handler:      logging(mux),
	ReadTimeout:  5 * time.Second,
	WriteTimeout: 10 * time.Second,
	IdleTimeout:  60 * time.Second,
}
// srv.ListenAndServe() 返回 http.ErrServerClosed 时属于正常退出
```

三个超时是生产服务的底线：没有 `ReadTimeout`，慢连接会耗尽文件描述符；没有 `WriteTimeout`，慢客户端会拖住 goroutine。优雅退出的完整写法见 [03-基本特性/07-优雅退出](../03-基本特性/07-优雅退出.md)。

## 八、对比 Hertz：概念一一对应

| 能力 | 标准库 net/http | Hertz |
| --- | --- | --- |
| 创建服务 | `http.NewServeMux()` + `http.Server` | `server.Default()` 或 `hertz.New()` |
| 路由 | `mux.HandleFunc("GET /users/{id}", h)` | `h.GET("/users/:id", h)` |
| 路径参数 | `r.PathValue("id")` | `c.Param("id")` |
| 请求上下文 | `*http.Request` | `*app.RequestContext` |
| 底层 context | `r.Context()` | `c context.Context`（处理器参数） |
| 读 JSON | `json.NewDecoder(r.Body).Decode` | `c.BindAndValidate(&req)` |
| 写 JSON | 手动设置头 + Encode | `c.JSON(200, obj)` |
| 中间件 | `func(http.Handler) http.Handler` | `func(ctx context.Context, c *app.RequestContext)` |
| 启动 | `srv.ListenAndServe()` | `h.Spin()` |

Hertz 并没有发明新概念，而是把「绑定、校验、渲染、路由树、连接池」做得更高效、更省代码。学完标准库再看 Hertz，你会有「只是换了一套更顺手的 API」的感觉。

## 常见坑

1. **重复调用 `WriteHeader`**：第二次无效并告警，状态码以第一次为准。
2. **先 `Write` 再设置 Header**：头已经发出，设置无效。
3. **忘记 `defer r.Body.Close()`**：连接无法复用，高并发下耗尽资源。
4. **用全局 `DefaultServeMux`**：路由注册分散、测试困难，生产应用自建 mux。
5. **不设置超时**：慢连接攻击可以直接拖垮服务。
6. **`ListenAndServe` 返回 `http.ErrServerClosed` 当错误**：优雅退出时这是正常现象，要排除。
7. **在处理器里直接 `panic` 没有 recover**：一个请求的 panic 会终止整个进程。

## 小结

标准库已经能写出完整服务：`ServeMux` 管路由、`Handler` 管处理、`Request/ResponseWriter` 管输入输出、函数包装管中间件。下一篇 [10-测试与基准测试](10-测试与基准测试.md) 会给这些处理器写测试；之后进入 [02-Hertz入门/02-快速开始-第一个服务](../02-Hertz入门/02-快速开始-第一个服务.md)，你会看到同样的服务用 Hertz 只要一半代码。

## 练习

1. 用标准库实现 `GET /users/{id}` 与 `POST /users` 两个接口，JSON 输入输出，错误统一返回 `{"code":...,"msg":...}`。
2. 实现 `logging`、`auth`、`recovery` 三个中间件，并用 `curl` 验证：无 token 返回 401，正常请求有耗时日志。
3. 给服务加上 `ReadTimeout` 与优雅退出，按 `Ctrl+C` 后确认正在处理的请求能完成、进程正常退出。

## 官方文档

- `net/http` 包：<https://pkg.go.dev/net/http>
- Go 1.22 路由增强：<https://go.dev/blog/routing-enhancements>
- `http.Server` 配置：<https://pkg.go.dev/net/http#Server>
- 优雅退出示例：<https://pkg.go.dev/net/http#Server.Shutdown>
