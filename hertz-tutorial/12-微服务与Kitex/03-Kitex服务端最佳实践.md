# Kitex 服务端最佳实践

> 一句话定位：把 Kitex 服务端从“能跑”调到“敢上生产”：限流、超时、中间件、优雅退出、健康检查与错误码一次配齐。

## 本篇目标

- 用一套完整选项启动生产可用的 Kitex 服务
- 理解 `WithLimit`、`WithReadWriteTimeout`、`WithMuxTransport`、`WithExitWaitTime` 的作用
- 掌握服务端超时与 context 传递，避免 goroutine 泄漏
- 写出 recover、访问日志、耗时统计三个必备中间件
- 设计优雅退出与注册中心注销顺序、健康检查方式
- 理解线程模型，并与 Hertz 侧建立统一错误码映射

## 前置知识

- 已读 [02-Kitex快速入门与IDL](02-Kitex快速入门与IDL.md)
- 已读 [03-基本特性/07-优雅退出](../03-基本特性/07-优雅退出.md)
- 了解 `context` 取消、GORM 的 `WithContext`、K8s 探针

## 一、生产级启动骨架

```go
package main

import (
    "time"

    "github.com/cloudwego/kitex/pkg/klog"
    "github.com/cloudwego/kitex/pkg/limit"
    "github.com/cloudwego/kitex/pkg/rpcinfo"
    "github.com/cloudwego/kitex/pkg/utils"
    "github.com/cloudwego/kitex/server"
    "mall-lite/kitex_gen/user/userservice"
)

func main() {
    addr := utils.NewNetAddr("tcp", ":8888")
    svr := userservice.NewServer(
        new(UserServiceImpl),
        server.WithServiceAddr(addr),
        server.WithServerBasicInfo(&rpcinfo.EndpointBasicInfo{ServiceName: "user-service"}),
        server.WithLimit(&limit.Option{MaxConnections: 2000, MaxQPS: 5000}),
        server.WithReadWriteTimeout(5*time.Second),
        server.WithMuxTransport(),          // 与客户端多路复用配套
        server.WithExitWaitTime(10*time.Second),
        server.WithMiddleware(RecoverMW),   // 必须最外层
        server.WithMiddleware(AccessLogMW),
    )
    if err := svr.Run(); err != nil {
        klog.Fatal(err)
    }
}
```

注意：**当前版本没有 `server.WithTimeout`**。服务端不设“全局请求超时”，真正的截止时间来自客户端 `WithRPCTimeout` 或上游 ctx 的 deadline；`WithReadWriteTimeout` 管的是连接读写。网上旧文章里的 `server.WithTimeout` 照抄会编译失败。

## 二、连接数与 QPS 限制

```go
server.WithLimit(&limit.Option{
    MaxConnections: 2000, // 0 表示不限制
    MaxQPS:         5000, // 超出直接拒绝
})
```

`limit.Option` 还支持 `UpdateControl`，可把限流阈值接到配置中心动态更新：

```go
server.WithLimit(&limit.Option{
    MaxConnections: 2000, MaxQPS: 5000,
    UpdateControl: func(u limit.Updater) {
        go watchLimitConfig(func(opt *limit.Option) { u.UpdateLimit(opt) })
    },
})
```

限流参数的经验值：`MaxConnections ≈ 单实例 goroutine 预算`，`MaxQPS ≈ 压测峰值的 1.2 倍`。设得过低，健康检查、内部批处理也会被拒；设得过高则失去保护意义。触发限流时 Kitex 直接返回错误，客户端应映射为 503 而不是 500。

## 三、超时与 context 传递

一次调用的时间预算必须自上而下收紧：

| 层级 | 预算示例 | 配置位置 |
| --- | --- | --- |
| 网关总超时 | 800ms | Hertz `WithReadTimeout` / 业务 ctx |
| order 调 product | 300ms | `client.WithRPCTimeout(300ms)` |
| product 查 MySQL | 200ms | `db.WithContext(ctx)` + 慢查询治理 |

handler 内所有 IO 都必须带 ctx，否则上游取消后服务端还在空转：

```go
func (s *UserServiceImpl) GetUser(ctx context.Context, req *user.GetUserReq) (*user.User, error) {
    if err := ctx.Err(); err != nil {
        return nil, kerrors.NewBizStatusError(49900, "request canceled")
    }
    var u User
    if err := s.db.WithContext(ctx).First(&u, req.UserId).Error; err != nil {
        return nil, err
    }
    return toThriftUser(&u), nil
}
```

异步任务不能直接复用请求 ctx（请求结束就被取消），用 `context.WithoutCancel` 保留值、去掉取消信号，并单独设超时。

## 四、多路复用与连接池

- `server.WithMuxTransport()`：单条连接上并发处理多个请求，减少连接数与握手开销
- 客户端对应 `client.WithTransportProtocol(transport.TTHeader)` + `client.WithMuxConnection(1)`
- 两端必须同时开启，否则会退回短连接或报协议不匹配

```go
import "github.com/cloudwego/kitex/transport"

cli, _ := userservice.NewClient("user-service",
    client.WithHostPorts("127.0.0.1:8888"),
    client.WithTransportProtocol(transport.TTHeader),
    client.WithMuxConnection(1),
)
```

多路复用适合“请求小、并发高”的内部调用。若单个响应体很大（如导出文件），仍建议独立接口或流式方案，避免一条连接被大包拖慢。

## 五、三个必备中间件

中间件签名固定为 `endpoint.Middleware`，顺序按注册顺序由外到内：

```go
func RecoverMW(next endpoint.Endpoint) endpoint.Endpoint {
    return func(ctx context.Context, req, resp interface{}) (err error) {
        defer func() {
            if r := recover(); r != nil {
                klog.CtxErrorf(ctx, "panic recovered: %v", r)
                err = kerrors.NewBizStatusError(50000, "internal error")
            }
        }()
        return next(ctx, req, resp)
    }
}

func AccessLogMW(next endpoint.Endpoint) endpoint.Endpoint {
    return func(ctx context.Context, req, resp interface{}) error {
        start := time.Now()
        err := next(ctx, req, resp)
        method := ""
        if ri := rpcinfo.GetRPCInfo(ctx); ri != nil && ri.To() != nil {
            method = ri.To().Method()
        }
        klog.CtxInfof(ctx, "method=%s cost_ms=%d err=%v",
            method, time.Since(start).Milliseconds(), err)
        return err
    }
}
```

recover 必须是最外层，且**必须给 `resp` 留合法返回值**或返回明确错误；只 recover 不返回错误，客户端会收到空包并报反序列化失败，比 panic 更难查。

## 六、优雅退出与注册中心注销

退出顺序决定了发布期间是否掉请求：

```text
SIGTERM → 从注册中心摘除 → 等待发现刷新（≥2 个刷新周期）→ 停止接收新请求
        → 等待在途请求（WithExitWaitTime）→ 关闭 DB/Redis/Kafka → 进程退出
```

```go
server.RegisterShutdownHook(func() {
    if err := r.Deregister(info); err != nil { // 先摘流量
        klog.Errorf("deregister failed: %v", err)
    }
    time.Sleep(5 * time.Second)               // 等客户端刷新实例列表
    closeDB()
})
```

`WithExitWaitTime` 要小于 K8s 的 `terminationGracePeriodSeconds`，并给 preStop 留出时间：

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
terminationGracePeriodSeconds: 30
```

## 七、健康检查

Kitex 默认不暴露 HTTP `/healthz`。三种做法：

| 场景 | 方式 | 说明 |
| --- | --- | --- |
| Thrift over TCP | K8s `tcpSocket` 探针 | 只验证端口可连，不验证依赖 |
| gRPC / Protobuf | gRPC health probe | 需要实现健康检查服务 |
| 需要业务级检查 | 同进程起管理端口 | Hertz 监听 8081，提供 `/healthz`、`/metrics`、`/debug/pprof` |

## 八、线程模型与 GOMAXPROCS

Kitex 默认是 goroutine-per-connection + Netpoll 事件驱动，handler 在独立 goroutine 中执行，**不要**在 handler 里长时间阻塞（如同步等大批量任务）。CPU 密集逻辑放协程池或异步队列。

Go 1.25 已能感知容器 CPU limit，通常**不需要**手动 `runtime.GOMAXPROCS`。只有在物理机固定核数绑定、或明确观测到线程切换过多时才调整，并配合压测验证。用管理端口的 pprof 观察 goroutine 数与阻塞 profile，比拍脑袋调参靠谱。

## 九、服务端限流、熔断与错误码映射

- 服务端：`WithLimit` 做入口保护，`WithQPSLimiter` 可接自定义限流器
- 熔断：Kitex 的熔断主要在**客户端**侧（见 [04-Kitex客户端与Hertz集成](04-Kitex客户端与Hertz集成.md)），服务端负责快速失败与降级
- 错误码：handler 一律返回 `kerrors.NewBizStatusError(code, msg)`，不要返回裸 `errors.New`

| Kitex 错误 | Hertz 映射 | 说明 |
| --- | --- | --- |
| `kerrors.ErrRPCTimeout` | 504 | 下游超时 |
| `kerrors.ErrCircuitBreak` | 503 | 被熔断 |
| `kerrors.ErrNoDestAddress` / `ErrNoInstance` | 503 | 无可用实例 |
| `BizStatusError(400xx)` | 400 | 参数/业务校验失败 |
| `BizStatusError(429xx)` | 429 | 限流 |
| 其他 | 500 | 未分类内部错误 |

映射函数统一放 `pkg/errs`，网关只调用一次，避免每个 handler 各写一套。

## 常见坑

1. **照抄 `server.WithTimeout`**：当前版本不存在该选项，编译直接失败。
2. **`WithExitWaitTime` 大于 `terminationGracePeriodSeconds`**：进程被 SIGKILL，在途请求与日志全部丢失。
3. **recover 后不返回错误**：客户端收到空响应，报反序列化错误，根因被掩盖。
4. **注册地址写 `127.0.0.1`**：本机自测正常，跨节点调用全部失败。
5. **`MaxQPS` 设得比健康检查还低**：探活被限流，实例被误摘除。
6. **中间件顺序颠倒**：recover 写在最内层，panic 时日志中间件直接崩掉。

## 小结

- 生产启动 = 地址 + 服务名 + 限流 + 读写超时 + 多路复用 + 退出等待 + 中间件
- 当前版本没有 `server.WithTimeout`，超时预算靠客户端 deadline 与 ctx 传递
- recover、访问日志、耗时统计是服务端最低配的三个中间件
- 优雅退出先摘流量再停服务，等待时间要小于 K8s 宽限期

## 练习

1. 给 user-service 加上 `WithLimit`，用压测工具验证超过 QPS 时的错误码。
2. 故意在 handler 里 panic，观察有无 recover 中间件的差异。
3. 把 `WithExitWaitTime` 设为 60s、K8s 宽限期 30s，复现被 SIGKILL 的现象。

## 官方文档

- Kitex 服务端配置：<https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/>
- 超时与重试：<https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/timeout/>
- 中间件：<https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/middleware/>
- Kitex server 包：<https://pkg.go.dev/github.com/cloudwego/kitex/server>
