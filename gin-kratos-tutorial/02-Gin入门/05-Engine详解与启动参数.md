# Engine 详解与启动参数

> 一句话定位：`gin.New()` 给你一个干净引擎，`gin.Default()` 只是多挂两个中间件；真正决定生产可靠性的是 `http.Server` 的超时四件套、可信代理设置与优雅退出。

## 本篇目标

- 说清 `gin.New()` 与 `gin.Default()` 的差异与选用标准。
- 会设置运行模式与可信代理，避免 Debug 开销与 `X-Forwarded-For` 伪造。
- 会用 `http.Server` 承载 Engine 并配置四个超时与 `MaxHeaderBytes`。
- 理解 `r.Run()` 的局限，会写优雅退出，并处理路由冲突 panic 与 404/405。

## 前置知识

- 已跑通第一个服务，见[快速开始：第一个服务](./02-快速开始-第一个服务.md)；建议先读[网络库 net/http 与性能模型](../03-基本特性/10-网络库net-http与性能模型.md)与[优雅退出](../03-基本特性/07-优雅退出.md)。

## 一、gin.New 与 gin.Default

`gin.Default()` 的实现只有三行：`gin.New()` + `gin.Logger()` + `gin.Recovery()`。

| 构造方式 | 中间件 | 适用场景 |
| --- | --- | --- |
| `gin.Default()` | Logger（文本）+ Recovery | 本地实验、demo |
| `gin.New()` | 无，完全空白 | 生产：自定义日志、错误、链路、限流 |

生产几乎总是用 `gin.New()`。需要 panic 兜底时自己写一个能返回 JSON 的 Recovery：

```go
// internal/middleware/recovery.go
package middleware

import (
    "log/slog"
    "net/http"
    "runtime/debug"

    "github.com/gin-gonic/gin"
)

func Recovery() gin.HandlerFunc {
    return gin.CustomRecovery(func(c *gin.Context, recovered any) {
        slog.Error("panic recovered", "err", recovered, "path", c.Request.URL.Path, "stack", string(debug.Stack()))
        if !c.Writer.Written() { // 已写出的流式响应无法再改状态码
            c.AbortWithStatusJSON(http.StatusInternalServerError,
                gin.H{"code": 50000, "message": "internal server error"})
        }
    })
}
```

注册顺序：Recovery 必须是最外层的第一个 `Use`，否则内层中间件 panic 时收不到。

## 二、运行模式：SetMode

```go
gin.SetMode(gin.ReleaseMode) // DebugMode / TestMode / ReleaseMode
```

- `DebugMode`（默认）：启动打印路由表并做额外断言；`ReleaseMode` 关闭这些行为，生产必须显式设置；`TestMode` 配合 `httptest`。

容器里更推荐环境变量：`GIN_MODE=release go run ./app/gateway/cmd`。

## 三、可信代理：SetTrustedProxies

`c.ClientIP()` 依赖 `X-Forwarded-For` / `X-Real-IP`，而 Gin 默认信任所有代理，任何人都能伪造来源 IP。

```go
// 1) 公网直连、前面没代理：不信任任何转发头
_ = r.SetTrustedProxies(nil)

// 2) 部署在 APISIX 之后：只信任网关所在网段
_ = r.SetTrustedProxies([]string{"10.0.0.0/8", "172.16.0.0/12"})
```

判据：只有你自己的代理出口网段才可信；配成 `0.0.0.0/0` 等于没有防护，限流与审计都会失效。

## 四、用 http.Server 承载 Engine

`*gin.Engine` 实现了 `http.Handler`，直接交给标准库 Server 才能拿到完整的超时能力：

| 字段 | 作用 | 生产建议 |
| --- | --- | --- |
| `ReadHeaderTimeout` | 读完请求头上限，防 Slowloris | 5s |
| `ReadTimeout` | 读完整个请求（含 body） | 10s，上传接口放宽 |
| `WriteTimeout` | 读完头到写完响应 | 15s，SSE 需特殊处理 |
| `IdleTimeout` | Keep-Alive 空闲回收 | 60s |
| `MaxHeaderBytes` | 请求头最大字节数 | 默认 1MB，公网可降到 32KB |

```go
// cmd/gateway/main.go
package main

import (
    "context"
    "errors"
    "log/slog"
    "net"
    "net/http"
    "os/signal"
    "syscall"
    "time"

    "github.com/gin-gonic/gin"

    "mall-lite/app/gateway/internal/middleware"
)

func main() {
    gin.SetMode(gin.ReleaseMode)
    r := gin.New()
    r.Use(middleware.Recovery(), gin.Logger())
    _ = r.SetTrustedProxies(nil)

    srv := &http.Server{
        Handler:           r,
        ReadHeaderTimeout: 5 * time.Second, ReadTimeout: 10 * time.Second,
        WriteTimeout:      15 * time.Second, IdleTimeout: 60 * time.Second,
        MaxHeaderBytes:    1 << 20, // 1MB
    }

    // 先 Listen 再 Serve：端口占用能在启动阶段立刻失败
    ln, err := net.Listen("tcp", ":8080")
    if err != nil {
        slog.Error("listen failed", "err", err)
        return
    }

    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    go func() {
        if err := srv.Serve(ln); err != nil && !errors.Is(err, http.ErrServerClosed) {
            slog.Error("serve failed", "err", err)
        }
    }()

    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := srv.Shutdown(shutdownCtx); err != nil {
        slog.Error("shutdown timeout", "err", err)
    }
}
```

`WriteTimeout` 会**截断流式响应**：SSE、大文件下载超过 15s 就会断开；这类接口应单独起 `WriteTimeout` 为 0 的 Server，或用 `http.NewResponseController(c.Writer).SetWriteDeadline(...)` 按需延长。

## 五、Run 的局限

| 调用 | 本质 | 缺失能力 |
| --- | --- | --- |
| `r.Run(":8080")` | `http.ListenAndServe(addr, r)` | 无超时、无优雅退出、地址硬编码 |
| `r.RunTLS(...)` | `http.ListenAndServeTLS` | 同上，证书路径写死 |
| `r.RunUnix(...)` | unix socket 上的 Serve | 无超时、无优雅退出 |

判断标准：**只要需要被 K8s 滚动更新，就必须用 `http.Server` + `Shutdown`**，否则 `SIGTERM` 会直接掐断在途请求。

## 六、路由冲突、404 与 405

两类冲突在**启动时 panic**：同一 Method + Path 重复注册；同一位置参数名不同（如 `/users/:id` 与 `/users/:name`）。`/users/:id` 与 `/users/me` 是合法的，静态优先。用单测锁住路由表：

```go
// internal/router/router_test.go
func TestRouteConflictPanics(t *testing.T) {
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/users/:id", func(c *gin.Context) {})
    defer func() {
        if recover() == nil {
            t.Fatal("expected panic on conflicting param name")
        }
    }()
    r.GET("/users/:name", func(c *gin.Context) {}) // 同位置不同参数名 → panic
}
```

404/405 默认返回纯文本，网关应统一成 JSON：

```go
r := gin.New()
r.HandleMethodNotAllowed = true // 不开启时，方法不匹配也返回 404
r.NoRoute(func(c *gin.Context) {
    c.JSON(http.StatusNotFound, gin.H{"code": 40400, "message": "route not found"})
})
r.NoMethod(func(c *gin.Context) {
    c.JSON(http.StatusMethodNotAllowed, gin.H{"code": 40500, "message": "method not allowed"})
})
```

405 的生效前提是**同一路径注册了其他方法**；只注册了 GET 时，POST 会走 404，这是 Gin 的预期行为。

## 完整示例

```powershell
go run ./app/gateway/cmd; curl.exe -i http://127.0.0.1:8080/api/v1/nope
curl.exe -i -X POST http://127.0.0.1:8080/api/v1/users/1
```

```text
预期：第一条返回 {"code":40400,...}；第二条在开启 HandleMethodNotAllowed 且存在 GET 时返回 40500；
按 Ctrl+C 观察优雅退出日志，进程正常结束。
```

## 常见坑

1. **生产仍是 Debug 模式**：路由表打印与额外断言带来开销，启动第一行就 `SetMode(ReleaseMode)`。
2. **没调用 `SetTrustedProxies`**：默认信任所有代理，`ClientIP` 可被伪造。
3. **`WriteTimeout` 截断 SSE**：流式接口超过该时长即断开，需独立 Server 或 `SetWriteDeadline`。
4. **`ReadTimeout` 过短**：大文件上传在读 body 阶段直接失败。
5. **只用 `r.Run()`**：没有超时、没有优雅退出，滚动更新会丢请求。
6. **`Shutdown` 后进程不退出**：还有后台 goroutine 在跑，需要 `WaitGroup` 收尾。
7. **路由冲突或 405 配置留到线上才发现**：路由冲突在启动 panic，用单测覆盖；只注册一个方法时 405 不会生效。

## 小结

- `gin.New()` 是干净起点，`gin.Default()` 只多了 Logger 与 Recovery；`SetMode`、`SetTrustedProxies`、`HandleMethodNotAllowed` 三个开关先设对。
- 生产入口必须是 `http.Server` + 四个超时 + `Serve(ln)` + `Shutdown`。
- 路由冲突在启动期 panic，用单测提前发现；404/405 统一成 JSON。

## 练习

1. 用自定义 Recovery 替换 `gin.Recovery()`，构造 panic handler，确认返回 JSON。
2. 把 `WriteTimeout` 设为 3s，写一个每 500ms 输出一次的 `c.SSEvent` 接口，观察第 3 秒断开。
3. 注册 `/users/:id` 与 `/users/:name`，运行 `TestRouteConflictPanics`，记录 panic 信息。

## 官方文档

- Gin 官方文档：<https://gin-gonic.com/zh-cn/docs/>
- Gin 优雅重启/停止：<https://gin-gonic.com/zh-cn/docs/examples/graceful-restart-or-stop/>
- `gin.Engine` API：<https://pkg.go.dev/github.com/gin-gonic/gin#Engine>
- `http.Server` 字段：<https://pkg.go.dev/net/http#Server>
