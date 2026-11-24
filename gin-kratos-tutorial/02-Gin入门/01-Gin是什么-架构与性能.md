# Gin 是什么：架构与性能

> 一句话定位：Gin 是构建在 `net/http` 之上的轻量 HTTP 框架——用 radix 路由树和池化的 `*gin.Context` 换开发效率，同时保留完整的标准库生态兼容。

## 本篇目标

- 说清 Gin 的定位、适用与不适用场景。
- 从网络层、请求对象、生态三个维度对比 Gin、`net/http`、FastHTTP、Echo。
- 理解 radix 路由树、Context 对象池，以及与 `net/http` 的兼容关系。
- 知道性能要看 P99、内存与 GC，而不是只盯 QPS。
- 跑通一个 gin v1.10.x 的最小程序，确认环境可用。

## 前置知识

- Go 基础语法、结构体、接口、goroutine 与 `context`。
- 读过[标准库 net/http 入门](../01-Go语言基础/09-标准库net-http入门.md)。
- 会用 Go Modules，见 [Go Module 与国内代理](../00-开始之前/03-Go-Module与国内代理.md)。

## 一、定位：只解决 HTTP 层

Gin 的官方定位是 "HTTP web framework"，职责边界很窄：负责路由、中间件链、参数绑定与校验、响应渲染、静态文件；不负责服务发现、配置中心、RPC、限流、链路追踪。

- 好处：核心小、上手快，任意 `net/http` 组件都能直接复用，升级风险低。
- 代价：微服务治理能力要自己拼装；没有官方代码生成器，一致性靠分层规范 + lint + 模板。

所以本教程的分工是：**Gin 做对外 BFF，Kratos 做内部 gRPC 微服务**，理由见第七节。

## 二、与 net/http / FastHTTP / Echo 对比

| 维度 | `net/http` | Gin | FastHTTP | Echo |
| --- | --- | --- | --- | --- |
| 底层网络 | Go 原生网络库 | Go 原生网络库 | 自研，复用连接对象 | Go 原生网络库 |
| 请求对象 | `*http.Request` | `*gin.Context` | `*fasthttp.RequestCtx` | `echo.Context` |
| 路由实现 | `ServeMux` 前缀匹配 | radix 树 + 参数 | 自研树 | radix 树 |
| 与 `http.Handler` 兼容 | 原生 | 引擎本身即 `http.Handler` | 不兼容，需适配 | 需适配 |
| 中间件生态 | 标准库生态 | gin-contrib，最活跃 | 较少 | 中等 |

结论：**FastHTTP 用兼容性换吞吐，Echo 用一致性换生态，Gin 用极小的框架成本换最大的中间件生态。**

## 三、内部结构：radix 树 + Context 池 + 标准库兼容

Gin 为每个 HTTP Method 维护一棵压缩前缀树，注册时按公共前缀合并节点，匹配时逐段下降——查找成本与路径段数相关，而不是与路由条数线性相关。匹配优先级固定为 静态 > `:param` > `*catchAll`，所以 `/users/me` 一定优先于 `/users/:id`。

```text
    /api/v1/users          → GET(listUsers)
    /api/v1/users/:id      → GET(getUser)  PUT(updateUser)
    /api/v1/orders/:id/items → GET(listItems)
```

`gin.Engine` 内部用 `sync.Pool` 复用 `*gin.Context`：请求开始时取出并绑定读写对象，结束后重置再放回。因此 `*gin.Context` 只在本次请求的 goroutine 内有效，跨 goroutine 必须 `c.Copy()`：

```go
// internal/handler/async.go
func AsyncAudit(c *gin.Context) {
    cc := c.Copy()                          // 请求结束后 c 会被重置并放回池中
    uid := cc.GetString("uid")
    ctx := context.WithoutCancel(c.Request.Context()) // 异步任务不能跟着请求取消
    go func() { _ = writeAudit(ctx, uid) }()
    c.JSON(http.StatusAccepted, gin.H{"code": 0})
}
```

`*gin.Engine` 实现了 `http.Handler`，所以既能交给 `http.Server`，也能与存量 `http.ServeMux` 共存：

```go
// 在已有 Gin 服务中挂载存量路由
legacy := http.NewServeMux()
legacy.HandleFunc("/legacy/health", func(w http.ResponseWriter, r *http.Request) { _, _ = w.Write([]byte("ok")) })

r := gin.New()
r.Any("/legacy/*path", gin.WrapH(legacy)) // 存量路由整体挂到 Gin 下
```

`gin.WrapH` 直接调用底层 handler、不继续 Gin 的链，适合挂「完整 handler」；`func(http.Handler) http.Handler` 中间件不能硬塞进 `r.Use`。

## 四、一次请求的生命周期

1. `http.Server` accept 连接，标准库解析请求，生成 `*http.Request`。
2. `Engine.ServeHTTP` 从池中取出 `*gin.Context` 并绑定。
3. 路由树按 Method + Path 得到完整 handler 链（全局 + 组 + 业务）。
4. 引擎最外层调用 `c.Next()`，依次执行链上每个 handler。
5. handler 用 `c.JSON`、`c.String` 写响应（状态码延迟到最后才 flush）。
6. 链返回后，各中间件 `c.Next()` 之后的后置逻辑执行。
7. `Context` 重置并放回池中，等待下一次复用。

第 4 步容易误解：引擎已经替你调用 `c.Next()`，中间件不写 `Next()` 也不会阻断后续 handler；`c.Next()` 的真实用途是「先放行，再回来做后置逻辑」。详见[中间件](../03-基本特性/05-中间件.md)。

## 五、性能数据应该怎么看

| 指标 | 为什么重要 | 怎么看 |
| --- | --- | --- |
| P99 / P999 | 平均值掩盖长尾，长尾决定体感 | k6 / wrk 的 p99，对比业务超时 |
| allocs/op | 分配越多 GC 越频繁，抖动越大 | `go test -bench . -benchmem` |
| 内存 / GC 次数 | 容器 OOM 与停顿的根源 | `GODEBUG=gctrace=1`、pprof heap |
| QPS | 容量规划基础，但要和延迟一起看 | 同机器、同版本、同场景才可比 |

三个误区：拿 Hello World 的 QPS 做容量规划（真实接口有 DB/RPC，差一个量级）；跨机器对比；不标版本。

```go
// internal/router/router_bench_test.go
package router

import (
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/gin-gonic/gin"
)

func BenchmarkPing(b *testing.B) {
    gin.SetMode(gin.TestMode)
    r := gin.New()
    r.GET("/ping", func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"message": "pong"}) })
    req := httptest.NewRequest(http.MethodGet, "/ping", nil)

    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        r.ServeHTTP(httptest.NewRecorder(), req)
    }
}
```

## 六、生态：五件套解决工程问题

- `gin-contrib/*`：cors、requestid、sessions、pprof，中间件章节按需引入。
- `swaggo/swag` + `gin-swagger`：注解生成 OpenAPI，见 [Swagger 注解](../06-Gin进阶/02-Swagger注解与OpenAPI文档.md)。
- `google/wire`：编译期依赖注入，见 [Wire 与配置分层](../07-工程化与测试/02-依赖注入Wire与配置分层Viper.md)。
- `otelgin`：OpenTelemetry 埋点，见可观测性章节。

工具链的完整用法见[工程工具链全景](../06-Gin进阶/01-工程工具链全景.md)。

## 七、为什么选「Gin BFF + Kratos 微服务」

| 关注点 | Gin（BFF） | Kratos（内部服务） |
| --- | --- | --- |
| 协议 | HTTP/1.1、HTTP/2、SSE、文件上传 | gRPC + Protobuf |
| 契约 | OpenAPI（swag 注解） | proto 即契约，生成 server/client |
| 分层 | `cmd + internal/{router,handler,middleware,client}` | 官方 DDD：`biz/data/service/server/conf` |
| 适合 | 聚合下游、裁剪字段、面向浏览器 | 领域服务、内部高 QPS 调用 |

代价：网关要做 gRPC 错误到 HTTP 状态码的映射，链路要靠同一个 `trace_id` 串联。收益：浏览器需求变化只在 BFF 改，领域服务不受影响。整体架构见 [Gin + Kratos 的微服务架构](../12-微服务与Kratos/01-Gin加Kratos的微服务架构.md)。

## 完整示例

```text
mkdir gin-check; cd gin-check
go mod init gin-check
go get github.com/gin-gonic/gin@v1.10.0
go list -m github.com/gin-gonic/gin
```

最后一行应输出 `github.com/gin-gonic/gin v1.10.x`，然后写入并运行：

```go
// main.go
package main

import (
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.New()
    r.Use(gin.Logger(), gin.Recovery())
    r.GET("/ping", func(c *gin.Context) { c.JSON(http.StatusOK, gin.H{"message": "pong"}) })

    // 生产还需 SetMode(ReleaseMode)、SetTrustedProxies 与 http.Server 超时，见 [05 篇](./05-Engine详解与启动参数.md)
    log.Fatal(r.Run(":8080"))
}
```

```powershell
go run .
curl.exe -i http://127.0.0.1:8080/ping
```

## 常见坑

1. **把 Gin 当 `net/http` 中间件容器**：`r.Use` 只接受 `gin.HandlerFunc`，标准库 handler 用 `gin.WrapH` 挂到路由上。
2. **跨 goroutine 持有 `*gin.Context`**：Context 会被复用并重置，异步场景必须 `c.Copy()`。
3. **只看 QPS 不看 P99 与 GC**：压测至少记录 p99 与 `allocs/op`。
4. **生产忘记 `ReleaseMode`**：Debug 模式有路由表打印与额外断言开销。
5. **`gin.Default()` 后又加 Logger**：日志重复；需要自定义就用 `gin.New()`。
6. **以为中间件必须写 `c.Next()`**：引擎已调用，`c.Abort()` 才是阻断。
7. **直接 `@latest`**：版本漂移导致 API 与教程不一致，学习阶段锁 v1.10.0。

## 小结

- Gin 是 `net/http` 之上的薄封装，核心资产是 radix 树、池化 Context 与中间件生态。
- 它不解决微服务治理，所以本教程让 Gin 做 BFF、Kratos 做 gRPC 领域服务。
- 性能结论必须带场景、版本与 P99/内存/GC 一起看。
- 生产启动用 `http.Server` + 超时 + 优雅退出，`r.Run()` 只用于本地实验。

## 练习

1. 去掉 `gin.Recovery()` 并构造 panic handler，观察连接被直接断开，再解释 Recovery 的作用。
2. 运行 `go test -bench BenchmarkPing -benchmem`，对比 `c.JSON` 与 `c.String` 的 `allocs/op`。
3. 用 `gin.WrapH` 把 `http.FileServer` 挂到 `/static/*filepath`，验证静态文件与 Gin 路由共存。

## 官方文档

- Gin 官方文档：<https://gin-gonic.com/zh-cn/docs/>
- Gin 仓库：<https://github.com/gin-gonic/gin>
- Gin API 参考：<https://pkg.go.dev/github.com/gin-gonic/gin>
- Gin 中间件示例：<https://gin-gonic.com/zh-cn/docs/examples/using-middleware/>
