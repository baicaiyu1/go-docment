# 10-网络库Netpoll与标准库
> Hertz 同时内置 Netpoll 与 Go 原生网络库：选对了性能翻倍，选错了 TLS、Windows、大文件处处是坑。

## 本篇目标

- 理解 Hertz 默认网络库的选择规则与 `HERTZ_NO_NETPOLL` 的作用。
- 会用 `server.WithTransport` 与 `client.WithDialer` 在 Netpoll / 标准库之间切换。
- 掌握两种网络库在触发模型、平台支持、TLS、大包场景上的差异。
- 能根据业务场景做出合理选型，并知道如何做本地性能对比。
- 避开网络库切换中的常见兼容性问题。

## 前置知识

- 了解 TCP、epoll、水平触发（LT）与边缘触发（ET）的基本概念。
- 知道 Go 标准库 `net` 的 goroutine-per-connection 模型。
- 已能启动一个基本 Hertz 服务，理解 `server.Option` 是函数式配置。

## 1. Hertz 的两套网络库

Hertz 把网络 I/O 抽象成 `network.Transporter`（服务端）与 `network.Dialer`（客户端），内置两套实现：

| 实现 | 包路径 | 特点 |
| --- | --- | --- |
| Netpoll | `github.com/cloudwego/hertz/pkg/network/netpoll` | CloudWeGo 自研，LT 模型，连接由网络库托管，针对小包高并发优化 |
| 标准库 | `github.com/cloudwego/hertz/pkg/network/standard` | 基于 `net` / `net/http` 底层能力，兼容性最好，支持 TLS |

默认选择规则（以 v0.10.6 源码为准）：

- 服务端在 **linux/darwin 且 amd64/arm64**、未设置 `HERTZ_NO_NETPOLL=true` 时，默认使用 Netpoll。
- 其他平台（包括 Windows）自动回退到标准库。
- 使用 `server.WithTLS` 时，若未显式指定 Transporter，会自动改为标准库，因为 **Netpoll 不支持 TLS**。

## 2. 切换网络库

### 2.1 服务端

```go
import (
	"github.com/cloudwego/hertz/pkg/app/server"
	"github.com/cloudwego/hertz/pkg/network/netpoll"
	"github.com/cloudwego/hertz/pkg/network/standard"
)

// 显式使用 Netpoll（默认行为，适合 Linux 高并发小包）
h1 := server.Default(server.WithTransport(netpoll.NewTransporter))

// 显式使用标准库（TLS、Windows、大包流式等场景）
h2 := server.Default(server.WithTransport(standard.NewTransporter))
```

`WithTransport` 接收的是 `func(*config.Options) network.Transporter`，`netpoll.NewTransporter` 与 `standard.NewTransporter` 正好符合该签名，所以可以直接传入。

### 2.2 客户端

```go
import (
	"github.com/cloudwego/hertz/pkg/app/client"
	"github.com/cloudwego/hertz/pkg/network/netpoll"
	"github.com/cloudwego/hertz/pkg/network/standard"
)

// 默认拨号器受平台影响；TLS 场景会回退到标准库
c1, _ := client.NewClient(client.WithDialer(netpoll.NewDialer()))
c2, _ := client.NewClient(client.WithDialer(standard.NewDialer()))
```

### 2.3 环境变量

设置 `HERTZ_NO_NETPOLL=true` 可以全局禁用 Netpoll：

```bash
# Linux / macOS
HERTZ_NO_NETPOLL=true ./server

# PowerShell
$env:HERTZ_NO_NETPOLL = "true"; .\server.exe
```

源码中 `pkg/route/netpoll.go` 与 `pkg/network/dialer/netpoll.go` 会读取该变量，为 `true` 时退回标准库。这常用于排查"换网络库后问题消失"的兼容性故障。

## 3. 关键差异与适用场景

| 维度 | Netpoll | 标准库（go net） |
| --- | --- | --- |
| 触发模型 | LT（水平触发），网络库负责读写事件 | ET（边缘触发），框架参与读写控制 |
| 平台 | 仅 linux/darwin amd64/arm64 | 全平台，含 Windows |
| TLS | 不支持，必须切标准库 | 支持 |
| 小包高并发 | 调度更优，通常性能更好 | 足够好，但连接数高时 goroutine 成本明显 |
| 超大请求体 | 数据可能被一次性读入内存，内存压力大 | 配合流式处理更稳 |
| 中间件 / 生态兼容 | 依赖 `net.Conn`、`http.Hijacker` 的组件可能不兼容 | 兼容性最好 |
| WebSocket / Hijack | 支持有限，需验证 | 兼容常见标准库方案 |

经验法则（与官方文档一致）：

- **请求体较大（> 1MB）**：推荐标准库 + 流式处理（如 `server.WithStreamBody(true)`、`SetBodyStream`）。
- **其他场景**：优先 Netpoll，享受更高吞吐与更低延迟。
- **需要 TLS、Windows、标准库生态组件**：直接用标准库，不要硬扛。

## 4. 性能对比怎么做

不要背数字，数字依赖版本、机器、请求大小与并发模型。正确姿势是自己压测：

```go
// go test -bench=BenchmarkPing -benchmem -count=5
func BenchmarkPingNetpoll(b *testing.B) {
	h := server.New(server.WithTransport(netpoll.NewTransporter))
	h.GET("/ping", pingHandler)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		// 用 ut.PerformRequest 只测框架开销，不含真实网络
		ut.PerformRequest(h.Engine, "GET", "/ping", nil)
	}
}
```

真实网络对比则用 `wrk` / `hey`：

```bash
hey -n 200000 -c 500 http://127.0.0.1:8080/ping
```

重点观察 QPS、P99 延迟、RSS 内存与 GC。小包场景 Netpoll 往往更占优；大包或长连接流式场景标准库更稳。

## 5. 运行时按配置选择网络库

企业项目常把网络库做成配置项，避免改代码：

```go
func newServer(useNetpoll bool) *server.Hertz {
	opts := []config.Option{server.WithHostPorts(":8080")}
	if useNetpoll {
		opts = append(opts, server.WithTransport(netpoll.NewTransporter))
	} else {
		opts = append(opts, server.WithTransport(standard.NewTransporter))
	}
	return server.New(opts...)
}
```

注意：**不要同时**把 `WithTLS` 和 `WithTransport(netpoll.NewTransporter)` 配在一起。`WithTLS` 只在"当前没有 Transporter"时才自动改标准库，选项按传入顺序生效，后写的会覆盖先写的。

## 常见坑

1. **TLS + Netpoll 组合**：`WithTLS` 后又被 `WithTransport(netpoll.NewTransporter)` 覆盖，握手失败。正确做法是只写 `WithTLS`，或显式 `WithTransport(standard.NewTransporter)`。
2. **Windows 上以为在用 Netpoll**：Windows 会条件编译自动切标准库，压测结果与 Linux 不可直接比较。
3. **大请求体把内存打爆**：Netpoll 的 LT 模型可能一次性读入大量数据，建议大于 1MB 的请求切标准库并开启流式。
4. **中间件依赖 `http.Hijacker` 或具体 `net.Conn`**：Netpoll 下会拿不到预期类型，表现为 panic 或功能失效；这类组件应切标准库。
5. **`OnAccept` / `OnConnect` 语义不同**：Netpoll 中 `OnAccept` 在加入 epoll 前、`OnConnect` 在加入后；标准库中 `OnAccept` 在 TLS 握手前、`OnConnect` 在 TLS 握手后。写 hook 时不要假设二者等价。
6. **全局 `route.SetTransporter` 与实例配置混用**：全局设置会影响所有实例，容易造成"某次改动污染全部服务"，新代码统一用 `server.WithTransport`。

## 小结

- Hertz 默认在 Linux/macOS 使用 Netpoll，Windows 与 TLS 场景自动或必须使用标准库。
- 服务端用 `WithTransport`，客户端用 `WithDialer` 切换；`HERTZ_NO_NETPOLL=true` 可全局回退。
- 小包高并发选 Netpoll，大包、TLS、Windows、标准库生态选 go net。
- 性能结论要用真实压测得出，不要迷信固定数字。
- 选项顺序会互相覆盖，TLS 与 Netpoll 不能共存。

## 练习

1. 分别在 Linux（或 WSL）与 Windows 上启动同一个 Hertz 服务，打印当前网络库类型或观察日志，验证默认选择差异。
2. 用 `hey` 对小包接口分别压测 Netpoll 与标准库，记录 QPS、P99 与内存占用。
3. 构造一个 5MB 请求体的上传接口，对比两种网络库下的内存曲线，再改用流式读取观察变化。
4. 故意写出 `WithTLS` + `WithTransport(netpoll.NewTransporter)` 的组合，复现错误并解释原因。
5. 用 `HERTZ_NO_NETPOLL=true` 启动服务，确认行为与显式 `WithTransport(standard.NewTransporter)` 一致。

## 官方文档

- 网络库：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/network-lib/>
- Engine 选项：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/engine/>
- 客户端配置：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/client/>
