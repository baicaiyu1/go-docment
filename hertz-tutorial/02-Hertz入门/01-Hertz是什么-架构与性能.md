# Hertz 是什么：架构与性能

> 一句话定位：Hertz 是字节跳动开源的 Go 微服务 HTTP 框架，以自研 Netpoll 网络库和分层可扩展架构为核心，面向高并发、可治理的生产级 HTTP 服务。

## 本篇目标

- 说清 Hertz 的定位、适用场景与不适用场景，避免“为了用框架而用框架”。
- 从网络层、请求对象、生态三个维度对比 Hertz、Gin、FastHTTP、标准库 `net/http`。
- 理解 `server.Hertz`、`route.Engine`、`RequestContext`、`protocol`、`network` 的分层关系。
- 知道性能数据应该看哪里，以及怎样避免被单一 QPS 数字误导。
- 认识 hz、Netpoll、Kitex、Volo、hertz-contrib 组成的 CloudWeGo 生态。

## 前置知识

- 学过 Go 基础语法、结构体、接口、goroutine 与 `context`。
- 读过标准库 `net/http` 的入门示例，知道 Handler、Request、Response 的基本概念。
- 会用 Go Modules：`go mod init`、`go mod tidy`、`go get`。

## 定位与适用场景

Hertz 的官方定位是“Go 微服务 HTTP 框架”。它参考了 fasthttp、Gin、Echo 的设计，但并不是 Gin 的替代品，而是围绕字节内部微服务场景重新做了一整套分层：网络层可切换、协议层可扩展、上下文对象池化、代码生成工具配套。

它适合这些场景：

- 微服务中的 HTTP 入口、BFF、API 网关后端，请求以小包、高 QPS 为主。
- 希望统一接入日志、链路追踪、服务发现、限流熔断等治理能力。
- 需要按部署环境切换网络库（Netpoll / Go 原生网络库），或扩展自定义协议。
- 团队愿意使用 hz + IDL 做工程化代码生成，保持接口定义与实现同步。

这些情况要谨慎：

- 你有一大堆现成的 `net/http` 中间件想直接挂上去。Hertz 不是 `http.Handler` 兼容实现，需要 adaptor 转换（见 `03-基本特性/14`）。
- 开发机是 Windows。Netpoll 目前只支持 Linux / macOS 的 amd64、arm64，Windows 会自动回退到 Go 原生网络库。
- 只需要一个几十行的内部小工具，标准库 `net/http` 更省心。
- 你的核心诉求是 HTTP/3 或完整的 HTTP/2 生态，Hertz 当前原生提供 HTTP/1.1 与 ALPN，能力边界要先确认。

## 与 Gin / FastHTTP / net/http 对比

| 维度 | 标准库 net/http | Gin | FastHTTP | Hertz |
| --- | --- | --- | --- | --- |
| 底层网络 | Go 原生网络库 | Go 原生网络库 | fasthttp | Netpoll（可切 Go 原生） |
| 请求对象 | `*http.Request` | `*gin.Context` 包装 | `*fasthttp.RequestCtx` | `*app.RequestContext` |
| 与 net/http 兼容 | 原生 | 兼容 | 不兼容 | 不直接兼容，需 adaptor |
| 中间件生态 | 标准库生态 | 最丰富 | 较少 | 内置 + hertz-contrib |
| 代码生成 | 无 | 无 | 无 | hz（Thrift / Protobuf） |
| 典型优势 | 稳定、无依赖 | 上手快、生态大 | 低分配、高吞吐 | 高吞吐 + 工程化 + 可扩展 |

几个关键差异：

- Gin 构建在 `net/http` 之上，所以能直接使用大量标准库生态；代价是请求解析和对象分配受标准库约束。
- FastHTTP 使用自己的 HTTP 实现，性能好但 API 与标准库差异大，中间件和工具链需要另起一套；Hertz 在协议层参考了它。
- Hertz 选择“自研协议实现 + 类 Gin 的易用 API + hz 工程化”，这是它区别于前两者的核心。

## 整体架构

```text
                 ┌─────────────────────────────────────────┐
                 │            server.Hertz                 │
                 │  ┌───────────────┐  ┌────────────────┐  │
                 │  │ route.Engine  │  │ signalWaiter   │  │
                 │  └───────┬───────┘  └────────────────┘  │
                 └──────────┼──────────────────────────────┘
                            │
        ┌───────────────────┼────────────────────┐
        │                   │                    │
  ┌─────▼─────┐      ┌──────▼──────┐      ┌──────▼──────┐
  │  Router   │      │ Middleware  │      │  Request    │
  │  Tree     │      │   Chain     │      │  Context    │
  └─────┬─────┘      └──────┬──────┘      └──────┬──────┘
        │                   │                    │
        └───────────────────┼────────────────────┘
                            │
                 ┌──────────▼──────────┐
                 │   protocol (HTTP1)  │
                 └──────────┬──────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
      ┌───────▼────────┐          ┌───────▼────────┐
      │ network/netpoll│          │network/standard│
      └────────────────┘          └────────────────┘
```

对应到源码目录：

- `pkg/app/server`：`Hertz` 类型、启动参数、`Spin` 与信号处理。
- `pkg/route`：`Engine`、路由树、`RouterGroup`、中间件链执行。
- `pkg/app`：`RequestContext`、`HandlerFunc`、静态文件 `FS`。
- `pkg/protocol`：HTTP/1.1 请求与响应的解析、序列化。
- `pkg/network`：`netpoll` 与 `standard` 两套网络实现。
- `pkg/common`：配置、日志 `hlog`、错误 `errors`、工具 `utils`。

一次请求的大致生命周期：

1. 网络库 accept 连接并读取字节。
2. `protocol/http1` 解析出 `protocol.Request`。
3. `Engine.ServeHTTP` 从对象池取出 `RequestContext`，绑定请求与响应。
4. 路由树匹配出 handler 链，按顺序执行中间件与业务 handler。
5. handler 通过 `c.JSON`、`c.String` 等写入 `protocol.Response`。
6. 引擎序列化响应，交回网络库写出。
7. `RequestContext` 重置后放回池中，等待下一次请求。

## 性能数据应该怎么看

Hertz 官方维护了独立的压测仓库 [hertz-benchmark](https://github.com/cloudwego/hertz-benchmark)，长期对比 Hertz、Gin、Echo、FastHTTP 等框架。看数据时至少注意四点：

- **场景**：小包、大包、Keep-Alive、TLS 的结论完全不同。Hertz 在小包场景更有优势；大包（如 > 1MB 上传）更推荐 Go 原生网络库加流式处理。
- **网络库**：Netpoll 是 LT 触发，Go 原生是 ET 触发，调度策略不同。Windows 上跑不出 Netpoll 的数据。
- **机器**：CPU 核数、内核版本、网卡、压测工具都会显著影响结果，不能把某张图里的数字直接搬进容量规划。
- **版本**：框架版本、Go 版本、编译参数都会变化，压测报告要标注完整环境。

正确做法是：用官方 benchmark 建立预期，再在自己的机器和业务接口上做一轮基准测试，最后以 P99 延迟、内存占用、GC 次数综合判断。

## 生态

| 项目 | 作用 |
| --- | --- |
| Hertz | HTTP 框架本体 |
| hz | 命令行代码生成工具，支持 Thrift / Protobuf IDL |
| Netpoll | 自研高性能网络库，Hertz 默认集成 |
| hertz-contrib | 社区扩展仓库：CORS、JWT、RequestID、日志、Tracer 等 |
| hertz-examples | 官方示例集合 |
| Kitex | CloudWeGo 的 Go RPC 框架，常与 Hertz 组成前后端微服务 |
| Volo | CloudWeGo 的 Rust RPC / HTTP 框架 |

## 版本与仓库

- 教程基线：Hertz **v0.10.6**、hz **v0.9.8**、Go **1.25**。
- 开源协议：Apache-2.0。
- 源码仓库：<https://github.com/cloudwego/hertz>
- 扩展仓库：<https://github.com/hertz-contrib>
- 示例仓库：<https://github.com/cloudwego/hertz-examples>

## 动手确认环境

先确认 Go 版本与模块缓存可用，再拉取与教程一致的 Hertz 版本。

Windows PowerShell：

```powershell
go version
go env GOPATH GOMODCACHE
mkdir hertz-check
cd hertz-check
go mod init hertz-check
go get github.com/cloudwego/hertz@v0.10.6
go list -m github.com/cloudwego/hertz
```

Linux / macOS bash：

```bash
go version
go env GOPATH GOMODCACHE
mkdir -p hertz-check
cd hertz-check
go mod init hertz-check
go get github.com/cloudwego/hertz@v0.10.6
go list -m github.com/cloudwego/hertz
```

最后一行输出应为 `github.com/cloudwego/hertz v0.10.6`。如果下载缓慢，先配置国内代理，见 `00-开始之前/03-Go-Module与国内代理.md`。

## 常见坑

1. **把 Hertz 当成 net/http 兼容框架**：`http.HandlerFunc` 不能直接传给 `h.GET`，签名是 `func(context.Context, *app.RequestContext)`。需要复用时走 adaptor。
2. **在 Windows 上照抄 Netpoll 的性能结论**：Windows 会自动使用 Go 原生网络库，压测数据和 Linux 不可比。
3. **只看 QPS 不看 P99 与内存**：高吞吐可能伴随长尾延迟或大包内存膨胀，尤其是 Netpoll 的 LT 模式。
4. **以为 `server.New()` 自带 Recovery**：只有 `server.Default()` 默认注册了 `recovery.Recovery()`；用 `New` 要自己加。
5. **版本漂移**：网上大量示例基于旧版本或 `@latest`，API 可能与本教程不一致。学习阶段请锁定 v0.10.6。

## 小结

- Hertz 的核心竞争力是“高性能网络层 + 分层可扩展 + hz 工程化”，不是简单复刻 Gin。
- 它最适合微服务 HTTP 场景；需要 net/http 生态时要用 adaptor，Windows 上要接受网络库回退。
- 性能结论必须结合场景、网络库、机器和版本看，官方 benchmark 只是起点。
- 后续章节会沿着“跑通 → 路由/Context/绑定/中间件 → 生产化”逐步展开。

## 练习

1. 用自己的话画出 Hertz 的请求生命周期，并标注每一步对应的源码包。
2. 打开 hertz-benchmark，找一组小包场景数据，记录测试环境、QPS、P99，说明它能否代表你的业务。
3. 列出你当前项目最看重的三个能力（如生态、吞吐、代码生成），据此判断是否应该选 Hertz。

## 官方文档

- Hertz 概览：<https://www.cloudwego.io/zh/docs/hertz/overview/>
- 快速开始：<https://www.cloudwego.io/zh/docs/hertz/getting-started/>
- 网络库：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/network-lib/>
- 性能测试仓库：<https://github.com/cloudwego/hertz-benchmark>
- GitHub 仓库：<https://github.com/cloudwego/hertz>
