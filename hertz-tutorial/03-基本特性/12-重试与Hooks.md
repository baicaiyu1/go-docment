# 12-重试与Hooks
> 客户端重试要克制且幂等，服务端 Hooks 要在正确的生命周期做正确的事——一篇文章讲清两者。

## 本篇目标

- 用 `client.WithRetryConfig` 配置重试次数、退避策略与抖动。
- 用 `SetRetryIfFunc` 自定义重试条件，并理解默认幂等判断。
- 掌握 Hertz 的 `OnRun` / `OnShutdown` / `OnAccept` / `OnConnect` 四类 Hook。
- 能把 Hooks 用到预热、优雅下线、注册中心摘除等真实场景。
- 避开重试放大、非幂等重试、Hook 超时等常见问题。

## 前置知识

- 已掌握 Hertz Client 基本用法（见 03-基本特性/11）。
- 理解幂等性：GET/PUT/DELETE 通常幂等，POST 通常不幂等。
- 了解优雅退出的基本流程（见 03-基本特性/07）。

## 1. 客户端重试：次数与延迟

重试通过 `client.WithRetryConfig` 开启，参数来自 `pkg/app/client/retry`：

```go
import (
	"time"

	"github.com/cloudwego/hertz/pkg/app/client"
	"github.com/cloudwego/hertz/pkg/app/client/retry"
)

c, err := client.NewClient(
	client.WithRetryConfig(
		retry.WithMaxAttemptTimes(3),              // 最大尝试次数，包含第一次调用
		retry.WithInitDelay(10*time.Millisecond),  // 初始延迟
		retry.WithMaxDelay(200*time.Millisecond),  // 任何策略下的延迟上限
		retry.WithMaxJitter(5*time.Millisecond),   // 随机抖动上限
		retry.WithDelayPolicy(
			retry.CombineDelay(
				retry.FixedDelayPolicy,   // 固定延迟
				retry.BackOffDelayPolicy, // 指数退避
				retry.RandomDelayPolicy,  // 随机抖动
			),
		),
	),
)
```

参数语义：

| 配置 | 含义 | 默认值 |
| --- | --- | --- |
| `WithMaxAttemptTimes` | 最大尝试次数（含首次） | 1，即不重试 |
| `WithInitDelay` | 初始延迟 | 1ms |
| `WithMaxDelay` | 延迟上限 | 100ms |
| `WithMaxJitter` | 抖动上限，配合随机策略 | 20ms |
| `WithDelayPolicy` | 延迟计算策略 | 返回 0（立即重试） |

策略函数可以任意组合，`CombineDelay` 会把各策略的结果相加，最终再受 `MaxDelay` 约束。`BackOffDelayPolicy` 基于 `InitDelay` 随尝试次数指数增长，适合下游瞬时抖动。

## 2. 重试条件与幂等性

默认情况下，Hertz 只对满足以下条件的请求重试：

- 请求体不是流式（`req.IsBodyStream()` 为 false，流不可回放）；
- 请求方法是幂等方法：GET / HEAD / PUT / DELETE / OPTIONS / TRACE；
- 或底层错误是 `io.EOF`（典型场景：服务端关闭了空闲 keep-alive 连接）。

自定义条件：

```go
c.SetRetryIfFunc(func(req *protocol.Request, resp *protocol.Response, err error) bool {
	// 仅当传输错误，或下游返回 502/503/504 时重试
	if err != nil {
		return true
	}
	switch resp.StatusCode() {
	case consts.StatusBadGateway, consts.StatusServiceUnavailable, consts.StatusGatewayTimeout:
		return true
	default:
		return false
	}
})
```

**幂等性红线**：不要对创建订单、扣款、发消息这类非幂等 POST 配置自动重试，除非下游支持幂等键（Idempotency-Key）。重试不是"更可靠"，而是"可能重复执行"。

## 3. 服务端 Hooks：四个切入点

Hertz 的 Hook 都挂在 Engine 上，`server.Hertz` 通过嵌入暴露了同名属性：

| Hook | 触发时机 | 执行方式 | 典型用途 |
| --- | --- | --- | --- |
| `OnRun` | 服务启动后、开始监听前 | 顺序执行，返回 error 会终止启动 | 配置校验、缓存预热、注册中心注册 |
| `OnShutdown` | 优雅退出时 | 并发执行，受 `WithExitWaitTime` 限制 | 从注册中心摘除、关闭消费者、刷日志 |
| `WithOnAccept` | 连接建立后 | 每个连接一次 | 连接级限流、打标 |
| `WithOnConnect` | 连接可读写后（标准库下是 TLS 握手后） | 每个连接一次 | 读取首包、连接级鉴权 |

### 3.1 启动与退出 Hook

```go
h := server.Default(server.WithExitWaitTime(10 * time.Second))

// 启动前：加载配置、预热缓存。返回 error 会阻止服务启动
h.OnRun = append(h.OnRun, func(ctx context.Context) error {
	if err := warmUpCache(ctx); err != nil {
		return err
	}
	return registerToConsul(ctx) // 注册成功后再对外提供服务
})

// 退出前：先摘流量，再释放资源
h.OnShutdown = append(h.OnShutdown, func(ctx context.Context) {
	deregisterFromConsul(ctx) // 让调用方尽快停止打流量
	stopKafkaConsumer(ctx)
	flushLogs(ctx)
})
```

注意 `OnShutdown` 是**并发**执行的，多个 Hook 之间没有先后顺序。如果存在依赖（必须"先摘注册、再关连接"），请合并到一个 Hook 内按顺序写，或使用 `errgroup`/`sync.WaitGroup` 自行编排。

### 3.2 连接级 Hook

```go
h := server.New(
	server.WithHostPorts(":8080"),
	server.WithOnAccept(func(conn net.Conn) context.Context {
		// 此时连接刚建立，还不能读数据
		return context.Background()
	}),
	server.WithOnConnect(func(ctx context.Context, conn network.Conn) context.Context {
		// 此时可 Peek 数据，可用于协议识别或连接级限流
		if b, err := conn.Peek(1); err == nil && b[0] == 0x16 {
			// 0x16 是 TLS handshake 记录类型
		}
		return ctx
	}),
)
```

两种网络库下语义有差异：Netpoll 的 `OnConnect` 在连接加入 epoll 之后，标准库的 `OnConnect` 在 TLS 握手之后。跨网络库复用时不要依赖具体时序。

## 4. Engine 生命周期与优雅下线

一次完整的下线流程应当是这样：

1. 收到 SIGTERM（K8s 删除 Pod 时默认发送）；
2. 调用 `h.Shutdown(ctx)`，Hertz 开始优雅退出；
3. 并发执行 `OnShutdown` Hook，最多等待 `WithExitWaitTime`（默认 5s）；
4. 等待在途请求处理完，连接返回 `Connection: close`；
5. 进程退出。

K8s 场景建议配合 `preStop` 睡眠几秒，让 Service Endpoints 先摘除，再触发退出：

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "5"]
```

这样能避免"注册中心已摘除但仍有连接打进来"的窗口。`Close()` 不会触发 `OnShutdown`，只有 `Shutdown()` 才会走优雅退出流程。

## 5. 真实用途清单

- **预热**：`OnRun` 中加载热点数据、建立下游连接池、校验配置，失败则拒绝启动。
- **注册与摘除**：先注册后监听（`OnRun`），先摘除后关闭（`OnShutdown`）。
- **消息消费**：`OnShutdown` 中停止拉取、提交位点，避免重复消费。
- **可观测性**：`OnShutdown` 中 flush 日志、指标、trace，防止最后一批数据丢失。
- **连接治理**：`OnAccept` 做 IP 黑名单或连接数限制，`OnConnect` 做协议嗅探。

## 常见坑

1. **`MaxAttemptTimes` 当成"重试次数"**：它包含首次调用，配 3 表示最多请求 3 次，即重试 2 次。
2. **对非幂等 POST 开重试**：网络超时不代表下游没执行，可能重复下单；必须使用幂等键或禁止重试。
3. **流式请求体重试**：body 是流不可回放，默认重试条件会直接跳过；需要重试时先把 body 缓存为字节。
4. **重试放大故障**：下游已经过载时重试会雪上加霜，应配合熔断、限流和重试预算。
5. **`OnShutdown` 里假设执行顺序**：Hook 是并发执行的，有依赖就必须合并或自行同步。
6. **Hook 超时**：默认只等 5 秒，超时后进程直接退出，未完成的清理会丢失；重活要异步化或调大 `WithExitWaitTime`。
7. **用 `Close()` 期待触发 Hook**：`Close` 是立即关闭，不执行 `OnShutdown`；优雅退出务必用 `Shutdown`。
8. **`OnRun` 返回 error 仍继续启动**：实际上会终止启动，但错误信息可能只出现在日志里，部署时要检查启动探针。

## 小结

- 重试由 `WithRetryConfig` 控制，次数含首次，延迟由策略组合计算并受上限约束。
- 默认只重试幂等请求和非流式 body；自定义条件用 `SetRetryIfFunc`，但要守住幂等底线。
- `OnRun` 顺序执行于监听前，`OnShutdown` 并发执行于退出时，`OnAccept`/`OnConnect` 是连接级 Hook。
- 生产下线流程 = 摘流量 → 停消费 → 刷数据 → 等在途请求结束。
- 重试与 Hook 都是"放大器"：配置得当提升可靠性，配置不当放大故障。

## 练习

1. 启动一个返回 503 的 mock 下游，配置 `MaxAttemptTimes=3` 与指数退避，用日志确认实际请求了 3 次。
2. 用 `SetRetryIfFunc` 实现"仅 5xx 重试"，并验证 4xx 不会重试。
3. 对同一个 POST 接口开启重试，观察下游收到几次请求，思考幂等键该怎么设计。
4. 在 `OnRun` 中模拟预热失败，确认服务不会开始监听。
5. 写两个 `OnShutdown` Hook，一个睡眠 10 秒，设置 `WithExitWaitTime(2s)`，观察超时日志与进程行为。

## 官方文档

- 重试：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/retry/>
- Hooks：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/hooks/>
- 优雅退出：<https://www.cloudwego.io/zh/docs/hertz/tutorials/basic-feature/graceful-shutdown/>
