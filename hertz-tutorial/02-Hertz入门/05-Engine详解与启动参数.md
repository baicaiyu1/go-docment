# Engine 详解与启动参数

> 一句话定位：`server.Hertz` 是框架门面，`route.Engine` 才是路由、中间件、配置与生命周期的核心；选对启动参数，等于提前避开一半生产问题。

## 本篇目标

- 理解 `server.Hertz` 与 `route.Engine` 的组合关系，以及 `Engine` 持有的路由树。
- 掌握 `server.Default` 与 `server.New` 的差异和选用场景。
- 逐一说明监听、超时、请求体大小、网络库、优雅退出等常用启动参数。
- 区分 `Run()` 与 `Spin()`，知道什么时候不能只用 `Run()`。
- 学会用自定义结构体包装 Hertz，为依赖注入和分层做准备。

## 前置知识

- 已跑通一个 Hertz 服务。
- 了解 `context`、超时、HTTP 长连接的基本概念。
- 知道 Netpoll 与 Go 原生网络库的区别（可回看 `02-Hertz入门/01`）。

## server.Hertz 与 route.Engine

源码里 `Hertz` 的定义非常短：

```go
type Hertz struct {
	*route.Engine
	// 用于接收信号以实现优雅退出
	signalWaiter func(err chan error) error
}
```

- `*route.Engine` 被嵌入，所以 `h.GET`、`h.Group`、`h.Use`、`h.Static`、`h.Shutdown` 这些方法其实都来自 `Engine`。
- `signalWaiter` 只服务于 `Spin()`，负责等待退出信号。
- `Engine` 内部持有 `options`、按 HTTP Method 组织的路由树 `trees`、`ctxPool`、`OnRun` / `OnShutdown` 钩子等。

`Engine` 的定义（节选）：

```go
type Engine struct {
	Name       string
	options    *config.Options
	RouterGroup          // 嵌入路由组，提供 GET/POST/Group/Use 等
	trees      MethodTrees
	maxParams  uint16
	ctxPool    sync.Pool
	OnRun      []CtxErrCallback
	OnShutdown []CtxCallback
}
```

## Default 与 New

| 构造函数 | 是否注册 Recovery | 适用场景 |
| --- | --- | --- |
| `server.Default(opts...)` | 是，自动 `h.Use(recovery.Recovery())` | 绝大多数服务，尤其是入门 |
| `server.New(opts...)` | 否，完全空白 | 想自定义 Recovery、统一错误处理或替换中间件栈 |

需要自定义 Recovery 时用 `New`，避免 `Default` 的 Recovery 先于你的中间件执行：

```go
h := server.New(server.WithHostPorts(":8080"))
h.Use(recovery.Recovery(recovery.WithRecoveryHandler(
	func(ctx context.Context, c *app.RequestContext, err interface{}, stack []byte) {
		hlog.SystemLogger().CtxErrorf(ctx, "[Recovery] %v\n%s", err, stack)
		c.AbortWithStatusJSON(500, utils.H{"code": 500, "message": "internal error"})
	},
)))
```

## 常用启动参数

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `WithHostPorts` | `:8888` | 监听地址与端口，`":8080"` 表示所有网卡 |
| `WithTransport` | 受平台影响 | 切换网络库，Linux 默认 Netpoll，Windows 自动 Go net |
| `WithReadTimeout` | 3 分钟 | 底层读取数据超时 |
| `WithWriteTimeout` | 0（不限制） | 写超时，仅 > 0 时在 Netpoll 中生效 |
| `WithIdleTimeout` | 3 分钟 | 长连接空闲超时，通常比 ReadTimeout 更值得关注 |
| `WithMaxRequestBodySize` | 4 MB | 请求体上限，超出返回 413 |
| `WithExitWaitTime` | 5 秒 | 优雅退出等待时间 |
| `WithHandleMethodNotAllowed` | false | 开启后路径存在但方法不匹配返回 405 |

### 切换网络库

```go
import (
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/network/netpoll"
	"github.com/cloudwego/hertz/pkg/network/standard"
)

// 强制使用 Go 原生网络库（TLS、大包上传场景推荐）
h1 := server.Default(server.WithTransport(standard.NewTransporter))

// 强制使用 Netpoll（Linux 小包高并发场景）
h2 := server.Default(server.WithTransport(netpoll.NewTransporter))
```

Windows 上 `netpoll.NewTransporter` 不可用，框架会通过条件编译回退到标准网络库；代码里也不要硬编码只依赖 Netpoll 的特性。

### 超时的实际含义

- `ReadTimeout` 覆盖“从连接读到完整请求”的时间，上传大文件时不要设得太短。
- `WriteTimeout` 为 0 表示不限制，生产环境建议显式设置，避免慢客户端长期占用连接。
- `IdleTimeout` 决定 Keep-Alive 连接空闲多久后关闭，直接影响连接复用率。
- 业务处理超时不是这三个参数能解决的，需要在 handler 或中间件里用 `context.WithTimeout` 控制。

## 完整启动示例

下面这份 `main.go` 把常用生产参数串起来，可直接运行：

```go
package main

import (
	"context"
	"time"

	"github.com/cloudwego/hertz/pkg/app"
	"github.com/cloudwego/hertz/pkg/app/middlewares/server/recovery"
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/common/hlog"
	"github.com/cloudwego/hertz/pkg/common/utils"
	"github.com/cloudwego/hertz/pkg/network/standard"
	"github.com/cloudwego/hertz/pkg/protocol/consts"
)

func main() {
	h := server.New(
		server.WithHostPorts("0.0.0.0:8080"),
		server.WithReadTimeout(10*time.Second),
		server.WithWriteTimeout(10*time.Second),
		server.WithIdleTimeout(60*time.Second),
		server.WithMaxRequestBodySize(8<<20), // 8MB
		server.WithExitWaitTime(10*time.Second),
		server.WithHandleMethodNotAllowed(true),
		server.WithTransport(standard.NewTransporter),
	)

	h.Use(recovery.Recovery())

	h.GET("/ping", func(ctx context.Context, c *app.RequestContext) {
		c.JSON(consts.StatusOK, utils.H{"message": "pong"})
	})

	if err := h.Run(); err != nil {
		hlog.Fatalf("server exited: %v", err)
	}
}
```

运行与验证：

Windows PowerShell：

```powershell
go run .
curl.exe -i http://127.0.0.1:8080/ping
```

Linux / macOS bash：

```bash
go run .
curl -i http://127.0.0.1:8080/ping
```

## Run 与 Spin 的区别

| 方法 | 启动服务 | 信号处理 | 优雅退出 |
| --- | --- | --- | --- |
| `h.Run()` | 是 | 否 | 否 |
| `h.Spin()` | 是 | 是（SIGINT/SIGTERM/SIGHUP） | 是 |

`Spin()` 的内部流程可以简化为：

```go
go func() { errCh <- h.Run() }()
if err := waitSignal(errCh); err != nil {
	_ = h.Close() // 异常，立即关闭
	return
}
_ = h.Shutdown(context.Background()) // 正常信号，优雅退出
```

所以：

- 业务服务几乎总是用 `Spin()`。
- 只有在你自己接管信号、或者在测试里手动控制生命周期时，才用 `Run()`，并自行调用 `Shutdown`。
- `Run()` 返回 `error`，适合在 `main` 里记录日志后退出；`Spin()` 不返回错误。

## 自定义 Server 结构

真实项目通常不会把所有 handler 写成闭包，而是把依赖挂在结构体上：

```go
type HTTPServer struct {
	*server.Hertz
	db *sql.DB
}

func NewHTTPServer(db *sql.DB) *HTTPServer {
	h := server.Default(server.WithHostPorts(":8080"))
	s := &HTTPServer{Hertz: h, db: db}
	h.GET("/users/:id", s.getUser)
	return s
}

func (s *HTTPServer) getUser(ctx context.Context, c *app.RequestContext) {
	c.JSON(consts.StatusOK, utils.H{"id": c.Param("id")})
}
```

这样既保留了 `Spin()`、`Shutdown()` 等能力，又为依赖注入、单元测试留出了入口。

## Engine 与路由树

- `Engine` 嵌入了 `RouterGroup`，所以 `h.GET` 实际调用的是 `RouterGroup.GET`。
- `trees MethodTrees` 按 HTTP Method 各维护一棵路由树，注册时写入，匹配时按“静态 > 命名参数 > 通配”查。
- `maxParams` 记录整棵树的最大参数个数，用于分配 `RequestContext.Params`。
- 路由注册发生在 `Spin/Run` 之前；启动后再注册会有并发风险，也不保证生效。
- `h.Routes()` 可以拿到所有已注册路由，适合启动时打印或做健康检查。

## 常见坑

1. **以为 `WriteTimeout` 默认有值**：默认是 0，表示不限制，慢连接可能长期占用资源。
2. **上传接口沿用默认 4MB**：超过 `WithMaxRequestBodySize` 会直接 413，上传服务要显式调大，并同步考虑内存。
3. **`WithBasePath` 写成 `api/v1`**：前后都必须有 `/`，正确写法是 `/api/v1/`。
4. **用 `Run()` 却期待优雅退出**：`Run()` 不处理信号，进程被 kill 时在途请求会中断。
5. **在 Windows 上强制 Netpoll**：会编译失败或行为不符预期，跨平台代码不要绑定单一网络库。
6. **端口冲突**：启动报 `bind: address already in use`，先确认 8080/8888 是否被占用。

## 小结

- `server.Hertz` 是门面，`route.Engine` 承担路由树、中间件链、配置和生命周期。
- `Default` 自带 Recovery，`New` 完全空白；需要自定义错误与 panic 处理时选 `New`。
- 启动参数里最该关注的是超时、请求体大小、网络库和退出等待时间。
- 生产服务用 `Spin()`；需要自己掌控生命周期时才用 `Run()` + `Shutdown()`。

## 练习

1. 用 `server.New` 重写 `02-快速开始` 的服务，手动注册 Recovery，并验证 panic 时返回 500。
2. 把 `WithMaxRequestBodySize` 设为 1KB，用 curl 发送 2KB 的 POST Body，观察状态码。
3. 用 `h.Routes()` 在启动时打印所有路由，解释输出中的 Method、Path、Handler 分别来自哪里。

## 官方文档

- Engine：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/engine/>
- 配置说明：<https://www.cloudwego.io/zh/docs/hertz/reference/config/>
- 网络库：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/network-lib/>
- 优雅退出：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/graceful-shutdown/>
