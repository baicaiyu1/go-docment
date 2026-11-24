# Kratos 客户端与 Gin 集成

> 一句话定位：在 Gin BFF 里安全调用 Kratos gRPC——启动建一次连接、`discovery:///` 走注册中心、超时预算递减、只对幂等方法重试、熔断可控、错误稳定映射。

## 本篇目标

- 用 `grpc.DialInsecure` + `discovery:///` 建单例客户端，启动时注入 Gin handler
- 接入 etcd / Consul 服务发现，实例扩缩容自动生效，并说清负载均衡怎么选
- 预算递减、只对幂等方法重试、Sentinel 熔断、错误稳定映射成 HTTP 码

## 前置知识

- [03-Kratos服务端最佳实践](03-Kratos服务端最佳实践.md)
- [03-基本特性/06-错误处理](../03-基本特性/06-错误处理.md)

## 一、启动建连，单例注入

每次请求建连都会重建连接、服务发现订阅与熔断统计，QPS 一高就端口耗尽、订阅风暴。正确做法是**启动时建连、注入 handler、退出时统一关闭**；也不要包级 `var` + `init()` 隐式建连，否则单测会被拖去连注册中心。

```go
// app/gateway/internal/client/client.go
package client

import (
    "context"
    "time"

    etcdreg "github.com/go-kratos/kratos/contrib/registry/etcd/v2"
    "github.com/go-kratos/kratos/v2/middleware/recovery"
    "github.com/go-kratos/kratos/v2/middleware/tracing"
    kratosgrpc "github.com/go-kratos/kratos/v2/transport/grpc"
    clientv3 "go.etcd.io/etcd/client/v3"
    ggrpc "google.golang.org/grpc"
    orderv1 "mall-lite/api/order/v1"
    productv1 "mall-lite/api/product/v1"
    userv1 "mall-lite/api/user/v1"
    "mall-lite/app/gateway/internal/conf"
)

type Clients struct {
    User    userv1.UserServiceClient
    Product productv1.ProductServiceClient
    Order   orderv1.OrderServiceClient
}

func New(cfg *conf.Config) (*Clients, func(), error) {
    cli, err := clientv3.New(clientv3.Config{Endpoints: cfg.Registry.Etcd.Endpoints, DialTimeout: 3 * time.Second})
    if err != nil {
        return nil, func() {}, err
    }
    // 注册器只返回 *Registry（同时实现 Registrar 与 Discovery），没有 error
    dis := etcdreg.New(cli, etcdreg.RegisterTTL(15*time.Second))
    dial := func(name string) *kratosgrpc.ClientConn {
        conn, e := kratosgrpc.DialInsecure(context.Background(),
            kratosgrpc.WithEndpoint("discovery:///"+name), // 服务名，不是 IP
            kratosgrpc.WithDiscovery(dis),
            kratosgrpc.WithTimeout(800*time.Millisecond),
            kratosgrpc.WithMiddleware(recovery.Recovery(), tracing.Client(), SentinelClient()),
            kratosgrpc.WithOptions(ggrpc.WithDefaultServiceConfig(retryPolicy)),
        )
        if e != nil {
            panic(e) // 启动期失败直接退出，交给 K8s 重启
        }
        return conn
    }
    uc, pc, oc := dial("user-service"), dial("product-service"), dial("order-service")
    cleanup := func() { _ = uc.Close(); _ = pc.Close(); _ = oc.Close(); _ = cli.Close() }
    return &Clients{User: userv1.NewUserServiceClient(uc), Product: productv1.NewProductServiceClient(pc), Order: orderv1.NewOrderServiceClient(oc)}, cleanup, nil
}
```

## 二、负载均衡与超时预算

Kratos 默认 `wrr`（加权轮询，权重取自实例 `Metadata`），内置 `p2c`（随机两个比负载，绕开慢节点）与 `random`；进程级切换只要 `selector.SetGlobalSelector(p2c.NewBuilder())`，import `github.com/go-kratos/kratos/v2/selector/p2c`。

预算逐层收紧：网关 3s → Gin→order 800ms → Gin→user/product 300ms → order→product 300ms。`WithTimeout` 只是默认值，不是保险丝；ctx 自带更早 deadline 时以更早者为准，剩余不足 100ms 直接返回 504。

```go
// pkg/timeout/budget.go
func Shrink(ctx context.Context, d time.Duration) (context.Context, context.CancelFunc) {
    if dl, ok := ctx.Deadline(); ok && time.Until(dl)-20*time.Millisecond < d {
        d = time.Until(dl) - 20*time.Millisecond // 预留 20ms 写回响应，避免上游已断开
    }
    return context.WithTimeout(ctx, d)
}
```

## 三、只对幂等方法重试，熔断用 Sentinel

`GetUser`、`GetProduct` 可重试；下单、扣库存不配，除非带幂等键。重试次数要小、退避要短，否则故障时三倍流量打向下游。

```go
const retryPolicy = `{"methodConfig":[{"name":[{"service":"user.v1.UserService","method":"GetUser"}],"retryPolicy":{"MaxAttempts":2,"InitialBackoff":"0.05s","MaxBackoff":"0.1s","RetryableStatusCodes":["UNAVAILABLE"]}}]}`
```

熔断用 `circuitbreaker.LoadRules` 配 `ErrorRatio`，资源名与 03 的中间件一致；`SentinelClient()` 按 `blockErr.BlockType()` 区分：熔断返回 `503 CIRCUIT_OPEN`，限流返回 `429 RATE_LIMITED`，打开后快速失败，不再排队。

```go
circuitbreaker.LoadRules([]*circuitbreaker.Rule{{
    Resource: "grpc:/user.v1.UserService/GetUser", Strategy: circuitbreaker.ErrorRatio,
    RetryTimeoutMs: 3000, MinRequestAmount: 20, StatIntervalMs: 10000, Threshold: 0.5}})
```

## 四、gRPC 错误 → HTTP 统一响应

`*errors.Error` 带 `Code`、`Reason`、`Message`，网关只映射一次；禁止把堆栈、下游服务名、SQL 返回给客户端，5xx 一律固定文案。

```go
// pkg/errs/grpc.go
var httpStatus = map[int32]int{400: 400, 401: 401, 403: 403, 404: 404, 429: 429, 503: 503, 504: 504}
var bizCode = map[int32]int{400: 40001, 401: 40101, 403: 40301, 404: 40401, 429: 42901, 503: 50301, 504: 50401}

func FromGRPC(err error, requestID string) (int, gin.H) {
    se := kerrors.FromError(err)
    status, code := httpStatus[se.Code], bizCode[se.Code]
    if status == 0 { status, code = http.StatusInternalServerError, 50000 }
    return status, gin.H{"code": code, "message": "request failed", "request_id": requestID}
}
```

## 五、Gin handler：单个 RPC 与并发聚合

单个 RPC 只做参数校验、预算、透传 metadata，业务判断全在服务端。

```go
// app/gateway/internal/handler/user.go
func (h *UserHandler) Get(c *gin.Context) {
    id, err := strconv.ParseInt(c.Param("id"), 10, 64)
    if err != nil || id <= 0 { c.JSON(http.StatusBadRequest, gin.H{"code": 40001, "message": "invalid user id", "request_id": requestID(c)}); return }
    ctx, cancel := timeout.Shrink(c.Request.Context(), 300*time.Millisecond)
    defer cancel()
    ctx = metadata.NewClientContext(ctx, metadata.Metadata{ // 键前缀固定 x-md-global-
        "x-md-global-request-id": requestID(c), "x-md-global-user-id": c.GetString("user_id")})
    u, err := h.cli.User.GetUser(ctx, &userv1.GetUserRequest{UserId: id})
    if err != nil { fail(c, err); return }
    c.JSON(http.StatusOK, gin.H{"code": 0, "message": "ok", "data": u, "request_id": requestID(c)})
}
```

多个 RPC 用 errgroup 并发：订单是主数据必须先成功，商品与买家信息才并发。

```go
// app/gateway/internal/handler/order.go
func (h *OrderHandler) Detail(c *gin.Context) {
    id, _ := strconv.ParseInt(c.Param("id"), 10, 64)
    ctx, cancel := timeout.Shrink(c.Request.Context(), 800*time.Millisecond)
    defer cancel()
    o, err := h.cli.Order.GetOrder(ctx, &orderv1.GetOrderRequest{OrderId: id})
    if err != nil { fail(c, err); return }
    g, gctx := errgroup.WithContext(ctx) // 共享 ctx：一个分支失败即取消其他分支
    var items []*productv1.Product
    buyer := &userv1.User{Nickname: "匿名用户"}
    g.Go(func() (err error) { // 强依赖：失败就让整个聚合失败
        items, err = h.cli.Product.BatchGetProducts(gctx, &productv1.BatchGetProductsRequest{OrderId: id})
        return err
    })
    g.Go(func() error { // 弱依赖：失败降级，不影响详情页
        if u, e := h.cli.User.GetUser(gctx, &userv1.GetUserRequest{UserId: o.GetUserId()}); e == nil {
            buyer = u
        }
        return nil
    })
    if err := g.Wait(); err != nil {
        fail(c, err)
        return
    }
    c.JSON(http.StatusOK, gin.H{"code": 0, "message": "ok", "request_id": requestID(c),
        "data": gin.H{"order": o, "items": items, "buyer": buyer}})
}

func fail(c *gin.Context, err error) {
    status, body := errs.FromGRPC(err, requestID(c))
    c.JSON(status, body)
}
```

每个分支只写自己的局部变量，`g.Wait()` 之后才读，`go test -race` 才不会告警；要在 goroutine 里用 `*gin.Context` 必须先 `c.Copy()`。

## 六、bufconn 测试：不起端口也能测 handler

```go
// app/gateway/internal/handler/user_test.go
func TestGetUserHandler(t *testing.T) {
    lis := bufconn.Listen(1 << 20)
    srv := ggrpc.NewServer()
    userv1.RegisterUserServiceServer(srv, &stubUser{})
    go func() { _ = srv.Serve(lis) }()
    t.Cleanup(srv.Stop)
    dial := func(ctx context.Context, _ string) (net.Conn, error) { return lis.DialContext(ctx) }
    conn, err := ggrpc.NewClient("passthrough:///bufnet", ggrpc.WithContextDialer(dial),
        ggrpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil { t.Fatal(err) }
    t.Cleanup(func() { _ = conn.Close() })
    r := gin.New()
    r.GET("/api/v1/users/:id", (&UserHandler{cli: &client.Clients{User: userv1.NewUserServiceClient(conn)}}).Get)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, httptest.NewRequest(http.MethodGet, "/api/v1/users/42", nil))
    if w.Code != http.StatusOK { t.Fatalf("code=%d body=%s", w.Code, w.Body.String()) }
}
```

## 完整示例

```bash
go build ./app/gateway/... && go run ./app/gateway/cmd
curl.exe -s -H "X-Request-Id: demo-1" http://127.0.0.1:8080/api/v1/users/42
```

## 常见坑

1. **每个请求都建连**：连接、订阅、熔断器全部重建，QPS 一高就端口耗尽。
2. **RPC 超时比网关超时还长**：网关已返回 504，下游还在跑，重试再来一遍。
3. **调用用 `context.Background()`**：超时与 trace 断链，日志里找不到下游。
4. **给下单接口开重试**：用户被重复扣款，只对幂等方法重试；聚合不共享 ctx 还会导致 goroutine 泄漏。

## 小结

- 客户端启动时创建一次，注入 handler，退出时统一 `Close`
- `discovery:///服务名` + etcd/Consul 支持自动扩缩容，地址来自配置
- 超时预算递减，ctx 原样透传；只对幂等方法重试，熔断用 Sentinel，错误在 `pkg/errs` 映射一次

## 练习

1. 把直连客户端改成 etcd 发现，滚动重启 user-service，观察调用不中断。
2. 让商品服务延迟 2s，验证聚合接口在预算内失败且其他分支被取消。

## 官方文档

- Kratos 服务发现：<https://go-kratos.dev/docs/component/registry>
- Kratos gRPC 传输：<https://go-kratos.dev/docs/component/transport/grpc>
