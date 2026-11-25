# Kitex 客户端与 Hertz 集成

> 一句话定位：在 Hertz 里安全地调用 Kitex 客户端——单例复用、超时预算、服务发现、重试熔断与错误码转换，一个都不能少。

## 本篇目标

- 掌握 `client.NewClient` 的常用选项与连接复用方式
- 区分直连、Nacos 解析两种连接方式的使用场景
- 设计“网关剩余时间 → RPC 超时”的预算传递
- 在 Hertz handler 中调用单个与多个 Kitex 服务
- 把 RPC 错误稳定映射成 HTTP 状态码与业务码
- 避开每请求建客户端、超时倒挂、非幂等重试等坑

## 前置知识

- 已读 [03-Kitex服务端最佳实践](03-Kitex服务端最佳实践.md)
- 已读 [03-基本特性/11-客户端Client](../03-基本特性/11-客户端Client.md)
- 了解 Hertz 的 `app.RequestContext`、`BindAndValidate` 与错误处理

## 一、客户端必须单例

Kitex 客户端内部持有连接池、服务发现缓存、熔断统计与 goroutine。**每个请求 `NewClient` 等于每次重建连接与订阅**，QPS 一高就会出现端口耗尽、Nacos 订阅风暴和内存上涨。

```go
import (
    "time"

    "github.com/cloudwego/kitex/client"
    "github.com/cloudwego/kitex/transport"
    "mall-lite/kitex_gen/user/userservice"
)

func NewUserClient(resolver client.Option) (userservice.Client, error) {
    return userservice.NewClient("user-service",
        resolver,
        client.WithRPCTimeout(300*time.Millisecond),
        client.WithConnectTimeout(100*time.Millisecond),
        client.WithTransportProtocol(transport.TTHeader),
        client.WithMuxConnection(1),
    )
}
```

在启动时构造一次，注入到 handler：

```go
type App struct {
    UserCli    userservice.Client
    ProductCli productservice.Client
    OrderCli   orderservice.Client
}
```

不要用包级 `var` + `init()` 隐式建连——单测会被拖去连注册中心，失败原因也不可见。

## 二、直连与服务发现

| 方式 | 写法 | 适用 |
| --- | --- | --- |
| 直连 | `client.WithHostPorts("127.0.0.1:8888")` | 本地开发、单机调试 |
| Nacos | `client.WithResolver(resolver.NewNacosResolver(namingCli))` | 测试/生产 |

直连地址**不能写死在代码里**，至少来自配置或环境变量。生产环境用注册中心，具体接入见 [05-服务注册与发现Nacos实践](05-服务注册与发现Nacos实践.md)。

```go
import (
    "github.com/kitex-contrib/registry-nacos/resolver"
)

r := resolver.NewNacosResolver(namingCli, resolver.WithGroup("mall-lite"))
cli, err := userservice.NewClient("user-service", client.WithResolver(r))
```

## 三、超时预算：跟着网关剩余时间走

一个请求从进入网关到返回，总预算是固定的。下游超时必须**小于**上游剩余时间，否则上游已经超时，下游还在占资源。

```go
func (h *OrderHandler) Create(ctx context.Context, c *app.RequestContext) {
    deadline, ok := ctx.Deadline()
    if !ok {
        c.String(consts.StatusInternalServerError, "no deadline")
        return
    }
    remain := time.Until(deadline)
    if remain < 100*time.Millisecond {
        c.JSON(consts.StatusGatewayTimeout, utils.H{"code": 50401, "msg": "request budget exhausted"})
        return
    }
    // 预留 50ms 给序列化与响应写回
    callCtx, cancel := context.WithTimeout(ctx, remain-50*time.Millisecond)
    defer cancel()

    order, err := h.OrderCli.CreateOrder(callCtx, &order.CreateOrderReq{ /* ... */ })
    // ...
}
```

`WithRPCTimeout` 是默认值，ctx 自带更早的 deadline 时以更早者为准；`WithConnectTimeout` 只影响建连，不要设成秒级——建连慢说明网络或实例有问题，应快速失败。

## 四、连接池与多路复用

- 客户端默认长连接；`WithMuxConnection(1)` 表示复用 1 条多路复用连接（服务端需 `WithMuxTransport()`）
- 连接数不是越多越好：每条连接都有内存与心跳成本，多路复用下单连接即可支撑较高 QPS
- 需要独立连接池时用 `client.WithConnPool`（示例示意，具体实现以官方文档为准），通常只有在隔离大小包流量时才需要

## 五、重试与熔断

```go
import (
    "github.com/cloudwego/kitex/client"
    "github.com/cloudwego/kitex/pkg/retry"
)

cli, _ := orderservice.NewClient("order-service",
    client.WithResolver(r),
    client.WithFailureRetry(retry.NewFailurePolicy()), // 参数选项以官方文档为准
    client.WithCircuitBreaker(cbSuite),                 // 熔断套件，见 kitex/pkg/circuitbreak
)
```

三条纪律（客户端中间件用 `client.WithMiddleware(mw)`，签名同为 `endpoint.Middleware`，只做观测与染色、不做重试）：

1. **只对幂等方法重试**。`GetUser`、`GetProduct` 可以；`CreateOrder`、`DeductStock` 默认不重试，除非带幂等键。
2. 重试次数 ≤ 2，且设置总时长上限，避免“重试把下游打死”。
3. 熔断打开后返回 `kerrors.ErrCircuitBreak`，网关应快速失败并给出可读提示，而不是继续排队。

重试与熔断的阈值建议放配置中心动态调整，见 [06-配置中心与动态配置](06-配置中心与动态配置.md)。

## 六、Hertz handler 调用单个 RPC

```go
func (h *UserHandler) Get(ctx context.Context, c *app.RequestContext) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil {
        c.JSON(consts.StatusBadRequest, utils.H{"code": 40001, "msg": "invalid id"})
        return
    }
    u, err := h.UserCli.GetUser(ctx, &user.GetUserReq{UserId: id})
    if err != nil {
        status, body := errs.FromRPC(err)
        c.JSON(status, body)
        return
    }
    c.JSON(consts.StatusOK, utils.H{"user_id": u.UserId, "nickname": u.Nickname})
}
```

关键点：把 Hertz 的 `ctx` 原样传给 Kitex，不要新建 `context.Background()`，否则超时与 trace 全部断掉。

## 七、BFF 聚合多个 RPC

订单详情页需要订单 + 商品 + 用户。串行调用会把延迟相加，应并发且共享同一个预算：

```go
import "golang.org/x/sync/errgroup"

func (h *OrderHandler) Detail(ctx context.Context, c *app.RequestContext) {
    orderID := c.Query("order_id")
    g, gctx := errgroup.WithContext(ctx)

    var o *order.Order
    var items []*product.Product
    var buyer *user.User

    g.Go(func() error {
        var err error
        o, err = h.OrderCli.GetOrder(gctx, &order.GetOrderReq{OrderId: orderID})
        return err
    })
    g.Go(func() error {
        var err error
        items, err = h.ProductCli.BatchGetProducts(gctx, &product.BatchGetReq{OrderId: orderID})
        return err
    })
    g.Go(func() error {
        var err error
        buyer, err = h.UserCli.GetUser(gctx, &user.GetUserReq{UserId: 42})
        return err
    })
    if err := g.Wait(); err != nil {
        status, body := errs.FromRPC(err)
        c.JSON(status, body)
        return
    }
    c.JSON(consts.StatusOK, assemble(o, items, buyer))
}
```

聚合原则：**核心数据强依赖，非核心可降级**。用户头像拿不到不该让整个订单详情失败，可以返回默认头像并记录 warn。

## 八、RPC 错误 → HTTP 映射

```go
// pkg/errs/rpc.go
func FromRPC(err error) (int, utils.H) {
    if biz, ok := kerrors.FromBizStatusError(err); ok {
        return httpStatusOf(biz.BizStatusCode()),
            utils.H{"code": biz.BizStatusCode(), "msg": biz.BizMessage()}
    }
    switch {
    case errors.Is(err, kerrors.ErrRPCTimeout):
        return consts.StatusGatewayTimeout, utils.H{"code": 50401, "msg": "upstream timeout"}
    case errors.Is(err, kerrors.ErrCircuitBreak):
        return consts.StatusServiceUnavailable, utils.H{"code": 50302, "msg": "service busy"}
    case errors.Is(err, kerrors.ErrNoDestAddress), errors.Is(err, kerrors.ErrNoInstance):
        return consts.StatusServiceUnavailable, utils.H{"code": 50301, "msg": "no available instance"}
    default:
        return consts.StatusInternalServerError, utils.H{"code": 50000, "msg": "internal error"}
    }
}
```

业务码用统一区间：`400xx` 参数、`401xx` 鉴权、`429xx` 限流、`5xxxx` 系统。网关对外只暴露业务码和可读消息，不返回堆栈与下游服务名。

## 九、泛化调用简介

泛化调用（Generic Call）让调用方在没有生成代码的情况下，按 IDL 描述发起 RPC，适合网关做动态路由、测试平台、多租户插件。核心是用 `generic` 包构造描述对象，再通过 `client.WithGeneric(g)` 注入；具体构造函数以 `kitex/pkg/generic` 文档为准（示例示意）。

代价是失去编译期类型检查、性能略低、调试更麻烦。**业务服务之间不要用泛化调用替代生成代码**，只把它当作平台能力。

## 常见坑

1. **每个请求 `NewClient`**：连接、订阅、熔断器全部重建，QPS 一高就端口耗尽。
2. **RPC 超时比网关超时还长**：网关已返回 504，下游还在跑，资源被无效占用。
3. **调用时用 `context.Background()`**：超时与 trace 断链，排查时找不到下游。
4. **对下单接口开重试**：用户被重复扣款，事后靠对账才发现。
5. **把注册中心地址写死在代码里**：换环境必须重新编译。
6. **并发聚合不共享 ctx**：任一分支超时无法取消其他分支，goroutine 泄漏。

## 小结

- 客户端在启动时创建一次，通过依赖注入传给 handler
- 开发用直连、生产用注册中心；地址来自配置，不写死
- 超时预算自上而下递减，ctx 原样透传
- 只对幂等方法重试，熔断与限流阈值放配置中心

## 练习

1. 把示例中的直连客户端改成 Nacos 解析，验证实例扩缩容后自动生效。
2. 给网关设置 500ms 超时、RPC 设置 800ms，观察并解释现象后修正。
3. 用 errgroup 聚合三个 RPC，人为让其中一个超时，验证其他分支被取消。

## 官方文档

- Kitex 客户端配置：<https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/>
- 超时与重试：<https://www.cloudwego.io/zh/docs/kitex/tutorials/basic-feature/timeout/>
- 泛化调用：<https://www.cloudwego.io/zh/docs/kitex/tutorials/advanced-feature/generic-call/>
- Kitex client 包：<https://pkg.go.dev/github.com/cloudwego/kitex/client>
