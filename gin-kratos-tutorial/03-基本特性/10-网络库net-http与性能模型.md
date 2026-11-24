# 网络库 net/http 与性能模型

> 一句话定位：Gin 没有自己的网络层，它就是一个 `http.Handler`；搞懂 Go netpoller 的边缘触发、连接复用与 `http.Server` 超时，才能把性能问题定位到正确的层。

## 本篇目标

- 说清 Gin 与 `net/http` 的层次关系，知道一个请求在运行时依次经过哪些层。
- 理解 netpoller 的 ET 触发模型，以及 goroutine-per-connection 的真实成本。
- 会配置 keep-alive、客户端连接池与 `http.Server` 的全部关键超时，明确 `MaxConnsPerIP` 并不存在。
- 拿到一份可落地的调优清单：GOMAXPROCS、GC、对象池、避免大 JSON。
- 会用 benchmark 与 pprof 验证优化效果，而不是凭感觉。

## 前置知识

- [标准库 net-http 入门](../01-Go语言基础/09-标准库net-http入门.md)
- [Engine 详解与启动参数](../02-Gin入门/05-Engine详解与启动参数.md)
- 了解 TCP 三次握手、`TIME_WAIT` 与 epoll 的基本概念。

## 一、Gin 只是 net/http 之上的一层

`*gin.Engine` 实现了 `ServeHTTP(w http.ResponseWriter, r *http.Request)`，可以直接当 `http.Server.Handler`：

```go
// main.go
r := gin.New()
r.Use(gin.Logger(), gin.Recovery())
r.GET("/ping", func(c *gin.Context) { c.String(http.StatusOK, "pong") })

srv := &http.Server{Addr: ":8080", Handler: r} // Engine 就是 http.Handler
log.Fatal(srv.ListenAndServe())
```

一个请求的完整路径：

```text
网卡 → 内核 epoll → runtime 网络轮询器唤醒 goroutine
     → net/http 的 conn.serve → ServerHandler.ServeHTTP
     → gin.Engine.ServeHTTP → 路由查找 → 中间件链 → 你的 handler
```

性能问题只有三类战场：系统与网络、`net/http` 配置、业务代码。Gin 只决定第三类中的路由查找与 Context 复用，换 Web 框架很少能救活一个被慢 SQL 拖死的服务。

## 二、netpoller 与 ET 模型

Linux 上 Go 用 epoll，注册方式为**边缘触发（ET）**：内核只通知「状态变化」，所以必须一次读到 `EAGAIN` 为止。goroutine 发起 `Read` 时若无数据会被 park 并登记到网络轮询器，数据到达后唤醒，重新放回 P 的本地队列。

三个工程结论：

- 「goroutine per connection」不等于「线程 per connection」，1 万长连接对应 1 万 goroutine 是常态，成本主要是栈（初始 2KB）与调度。
- 慢客户端会长期占住一个 goroutine 和 fd，必须靠 `ReadHeaderTimeout` 等超时回收。
- `GOMAXPROCS` 决定同时执行 Go 代码的 M 数量。Go 1.25 起默认感知容器 CPU 限额，老版本在 K8s 中必须显式设置，否则按宿主机核数调度，造成大量上下文切换。

用 `ConnState` 观察连接生命周期，验证 keep-alive 是否生效：

```go
// observe.go：统计并发连接
var active int64

srv := &http.Server{
    Addr:    ":8080",
    Handler: r,
    ConnState: func(c net.Conn, s http.ConnState) {
        switch s {
        case http.StateNew:
            log.Printf("conns=%d", atomic.AddInt64(&active, 1))
        case http.StateClosed, http.StateHijacked:
            log.Printf("conns=%d", atomic.AddInt64(&active, -1))
        }
    },
}
```

## 三、连接复用、keep-alive 与 TIME_WAIT

| 现象 | 根因 | 处理 |
| --- | --- | --- |
| 每个请求都三次握手 | 客户端没复用连接 | 复用同一个 `http.Client`，配好 Transport 池 |
| `TIME_WAIT` 堆积 | 短连接 + 本端主动关闭 | 服务端 `IdleTimeout` 大于上游 keep-alive |
| 偶发 `EOF` | 服务端 `IdleTimeout` 比客户端池空闲时间短 | 服务端空闲超时设为客户端的 1.5 倍以上 |
| 高并发下大量新连接 | `MaxIdleConnsPerHost` 默认只有 2 | 按并发量显式调大 |

服务端默认开启 keep-alive，`srv.SetKeepAlivesEnabled(false)` 可关闭（只建议调试用）。客户端侧才是坑：`MaxIdleConnsPerHost` 必须显式调大，其余超时与拨号配置见 [11-客户端Client](11-客户端Client.md)。

```go
tr := &http.Transport{MaxIdleConns: 200, MaxIdleConnsPerHost: 100, IdleConnTimeout: 90 * time.Second}
cli := &http.Client{Transport: tr, Timeout: 5 * time.Second}
```

## 四、http.Server 的超时与容量参数

可直接抄的配置见文末「完整示例」，字段含义如下表；最该配的是 `ReadHeaderTimeout`（防 Slowloris），SSE 路由必须把 `WriteTimeout` 置 0。

| 参数 | 保护的对象 | 说明 |
| --- | --- | --- |
| `ReadHeaderTimeout` | 慢速发送请求头的攻击 | 最该配的一项 |
| `ReadTimeout` | 整个请求读取阶段 | 大文件上传要么调大，要么流式 |
| `WriteTimeout` | 从读完请求到写完响应 | SSE / 长轮询必须置 0 |
| `IdleTimeout` | 空闲 keep-alive 连接 | 不设时回退到 `ReadTimeout` |
| `MaxHeaderBytes` | 请求头内存 | 默认 1MB，别调到 100MB |

`http.Server` **没有** `MaxConns`、`MaxConnsPerIP` 这类字段。限制单 IP 并发要在网关（APISIX `limit-conn`）做，或包一层 `net.Listener` 计数、用中间件信号量——不要去找不存在的参数。`MaxConnsPerHost` 只存在于客户端的 `Transport`。

## 五、调优清单与验证

1. **GOMAXPROCS**：与 CPU limit 一致；CPU 密集型可略大，IO 型不要盲目调大。
2. **GC**：设 `GOMEMLIMIT`（容器内存的 80%）减少 OOM；`GOGC` 调大降频率但增内存，必须配合压测。
3. **对象池**：高频小对象用 `sync.Pool`；**不要池化 `*gin.Context`**，框架已池化，重复池化会踩 `c.Copy()` 语义。
4. **避免大 JSON**：响应超 1MB 就考虑分页、字段裁剪、protobuf；序列化开销常大于网络。
5. **验证**：benchmark 量化框架开销 → `hey`/`wrk` 真实压测 → pprof 找热点。

```go
// bench_test.go：只测框架开销
func BenchmarkPing(b *testing.B) {
    r := gin.New()
    r.GET("/ping", func(c *gin.Context) { c.String(http.StatusOK, "pong") })
    w := httptest.NewRecorder()
    req := httptest.NewRequest(http.MethodGet, "/ping", nil)
    b.ReportAllocs()
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        w.Body.Reset()
        r.ServeHTTP(w, req)
    }
}
```

```bash
# bash 与 PowerShell 通用
go test -run '^$' -bench BenchmarkPing -benchmem -cpuprofile cpu.out -memprofile mem.out
go tool pprof -top cpu.out
hey -n 100000 -c 200 http://127.0.0.1:8080/ping
```

## 六、与 FastHTTP 类框架的取舍

FastHTTP 通过复用请求/响应对象、放弃部分 `net/http` 生态换取更低分配率，代价是生态与迁移成本。判断标准：业务以 DB / 下游 RPC 为主时 Gin 完全够用，瓶颈在 IO 等待；纯转发、每秒几十万极小包再考虑 fasthttp 系；不要为跑分换框架，先把连接池、超时、SQL、序列化治一遍。

## 完整示例

```go
// cmd/gateway/main.go
package main

import (
    "errors"
    "log"
    "net/http"
    "time"

    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.New()
    r.Use(gin.Logger(), gin.Recovery())
    r.GET("/ping", func(c *gin.Context) { c.String(http.StatusOK, "pong") })

    srv := &http.Server{
        Addr: ":8080", Handler: r,
        ReadHeaderTimeout: 3 * time.Second,
        ReadTimeout:       10 * time.Second,
        WriteTimeout:      15 * time.Second,
        IdleTimeout:       90 * time.Second,
        MaxHeaderBytes:    1 << 20,
    }
    // 优雅退出见 07-优雅退出：signal.NotifyContext + srv.Shutdown(ctx)
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        log.Fatalf("listen: %v", err)
    }
}
```

## 常见坑

1. **以为换框架能解决慢**：多数 P99 问题来自 DB、下游、锁竞争，先 pprof 再谈框架。
2. **`WriteTimeout=15s` 又做 SSE**：连接到点被切断，客户端一直重连；SSE 路由必须置 0（见 [SSE 与流式响应](15-SSE与流式响应.md)）。
3. **`MaxIdleConnsPerHost` 用默认值 2**：连接池瞬间打满，表现大量 `TIME_WAIT` 与握手延迟。
4. **服务端 `IdleTimeout` 小于客户端池空闲时间**：复用到已被关闭的连接，报 `EOF`。
5. **容器里不设 GOMAXPROCS**：按宿主机核数调度，CPU 被 cgroup 限流后线程空转。
6. **想找 `MaxConnsPerIP`**：`http.Server` 没有这个字段，并发限制前移到网关或自定义 Listener。
7. **在 `sync.Pool` 里放 `*gin.Context`**：破坏 `c.Copy()` 生命周期，出现串数据。
8. **benchmark 不 `Reset` 响应体**：`Recorder` 的 Body 无限增长，测出的是分配而非框架性能。

## 小结

- Gin 是 `http.Handler`，网络与调度能力全部来自 `net/http` 与 runtime 网络轮询器（Linux 上 ET）。
- 连接问题优先查 keep-alive 与连接池参数，`MaxIdleConnsPerHost` 默认 2 是高频事故点。
- `http.Server` 只提供超时与 `MaxHeaderBytes`，没有 `MaxConnsPerIP`；调优顺序是「系统 → 配置 → 业务」，用 benchmark、pprof、`hey` 三层验证。

## 练习

1. 用 `ConnState` 统计连接数，连续发 10 个共用 Client 的请求，确认只建 1 条连接；再把 `MaxIdleConnsPerHost` 设为 1、并发 20 观察变化。
2. 给 `/ping` 写 benchmark，比较裸 `gin.New()` 与加 5 个空中间件后的 `ns/op`、`allocs/op`。
3. 写一个 10 秒才返回的接口，分别设 `WriteTimeout=15s` 与 `0`，用 `curl -i` 观察差异。

## 官方文档

- `net/http#Server`：<https://pkg.go.dev/net/http#Server>
- `net/http#Transport`：<https://pkg.go.dev/net/http#Transport>
- netpoller 源码：<https://github.com/golang/go/blob/master/src/runtime/netpoll.go>
- pprof：<https://pkg.go.dev/net/http/pprof>
- Gin 官方文档：<https://gin-gonic.com/zh-cn/docs/>
