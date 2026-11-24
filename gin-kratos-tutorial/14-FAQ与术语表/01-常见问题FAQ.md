# 常见问题 FAQ

> 一句话定位：把 Gin + Kratos 项目从选型到上线最常见的 30 个疑问，按「结论 → 原因/代价 → 正确做法」一次答完。

## 本篇目标

- 覆盖 11 个主题、30 个高频问题，每题都有结论、原因与可执行做法。
- 统一口径：Go 1.25、Gin v1.10.x、Kratos v2.8.x、gRPC + Protobuf（以官方最新稳定版为准）。
- 让「为什么这么选」与「出问题怎么查」都能在一页内找到答案。

## 前置知识

- 读过 [Gin 加 Kratos 的微服务架构](../12-微服务与Kratos/01-Gin加Kratos的微服务架构.md)，清楚 BFF 与 gRPC 服务的分工。
- 能看懂 `go test`、`pprof`、`EXPLAIN` 与 `kubectl describe` 的常见输出。

## 一、选型与架构
### Q1. 为什么用 Gin + Kratos 两套框架，而不是一套打天下？
- **结论**：Gin 负责南北向 HTTP（BFF、鉴权、聚合、SSE），Kratos 负责东西向 gRPC 微服务。
- **原因/代价**：HTTP 中间件生态在 Gin 最成熟，治理能力（注册发现、熔断、链路、DDD 分层）在 Kratos 开箱即用；代价是日志、错误、配置必须统一封装，否则排障口径分裂。
- **正确做法**：日志统一在 `pkg/logger`（Kratos 用 zap 适配器），错误统一在 `pkg/errs`，配置统一走 Viper，见 [配置管理 Viper](../05-治理与扩展/03-配置管理Viper.md)。
### Q2. 内部服务之间能不能直接用 Gin 互调 REST？
- **结论**：不能，内部一律 gRPC + Protobuf；HTTP 只存在于「客户端 → APISIX → Gin 网关」这一段。
- **原因/代价**：proto 是强类型契约，字段变更在编译期暴露；JSON 每次都要序列化字符串、接口靠文档约定。代价是要维护 proto 工作流，调试依赖 grpcurl。
- **正确做法**：契约放 `api/user/v1/user.proto`，用 `buf generate` 生成 `mall-lite/api/user/v1`，服务端与客户端都用生成代码。
### Q3. BFF 里能不能写业务规则？
- **结论**：BFF 只做协议转换、鉴权、聚合、字段裁剪与限流，领域规则必须下沉到 Kratos 服务。
- **原因/代价**：规则写进 BFF 后，App、小程序、后台各写一遍，改一次要全端回归；代价是聚合多一次内存组装，可并行发起多个 gRPC 调用抵消。
- **正确做法**：`app/gateway/internal/handler` 只编排 client；库存扣减、订单状态机放在 `order-service/internal/biz`。

## 二、Gin 使用
### Q4. 路由明明注册了却返回 404 或 405？
- **结论**：先打印路由表核对，八成是方法、前缀或注册时机不对。
- **原因/代价**：Gin 默认对「路径存在但方法不匹配」返回 404，只有开启 `HandleMethodNotAllowed` 才返回 405；`Use` 在路由注册之后调用会直接 panic。
- **正确做法**：启动时遍历 `r.Routes()` 打印 method + path；设置 `r.HandleMethodNotAllowed = true`；所有 `r.Use` 与分组注册放在路由之前。
### Q5. 中间件不写 `c.Next()`，后面的 handler 还会执行吗？
- **结论**：会。Gin 引擎最外层已经调用 `c.Next()`，中间件返回后仍会执行后续 handler。
- **原因/代价**：所以 `c.Next()` 的真实作用是「先执行后续 handler，再回来做后置逻辑」；想拦截必须用 `c.Abort()`，靠不写 `Next` 拦不住任何请求。
- **正确做法**：鉴权失败时 `c.AbortWithStatusJSON(401, ...)`；需要统计耗时就在 `c.Next()` 之后读 `c.Writer.Status()`。
### Q6. 合法请求也被 `ShouldBindJSON` 判成 400？
- **结论**：先把 `err.Error()` 打出来，绝大多数是 tag、Content-Type 或校验规则的问题。
- **原因/代价**：`json`、`form`、`uri`、`binding` tag 混用时绑定来源不匹配，字段类型不一致（字符串传数字）也会失败；解析失败与校验失败共用同一个 error。
- **正确做法**：只调用一次 `ShouldBindJSON`，用 `validator.ValidationErrors` 做中文翻译并回填字段名，见 [参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)。
### Q7. 在 goroutine 里直接用 `*gin.Context` 会数据错乱？
- **结论**：必须 `c.Copy()`，并且只使用副本。
- **原因/代价**：`gin.Context` 是池化复用的，请求结束后会被重置给下一个请求，跨 goroutine 持有原指针会读到别人的参数并触发 data race。
- **正确做法**：

```go
// app/gateway/internal/handler/notify.go
cCp := c.Copy()
go func() { _ = pushService.Send(cCp.Request.Context(), cCp.GetString("user_id")) }()
```

## 三、Kratos 与 gRPC
### Q8. Kratos 服务启动正常，注册中心却查不到实例？
- **结论**：先确认 `kratos.Registrar(reg)` 传了、注册中心地址对、服务名一致。
- **原因/代价**：漏传 Registrar 时进程照样跑，只是没人知道它；TTL 设得过短会频繁摘除，网络不通则注册静默失败（只在日志里报错）。
- **正确做法**：用 etcdctl 或 Consul UI 核对 key；`etcdreg.RegisterTTL(15*time.Second)` 不要低于 5s；服务名必须与 `discovery:///user-service` 一致。
### Q9. gRPC 客户端每次请求都新建连接，QPS 一高就端口耗尽？
- **结论**：连接必须在启动时创建一次，注入 handler 复用。
- **原因/代价**：每请求 `Dial` 都要服务发现、TCP 握手、HTTP/2 设置帧，延迟翻倍并快速耗尽本地端口；连接本身是并发安全的。
- **正确做法**：

```go
// app/gateway/internal/client/user.go
conn, err := grpc.DialInsecure(ctx,
    grpc.WithEndpoint("discovery:///user-service"), grpc.WithDiscovery(reg))
if err != nil {
    return nil, err
}
return v1.NewUserServiceClient(conn), nil
```

### Q10. Kratos 的业务错误怎么变成正确的 HTTP 状态码？
- **结论**：服务内用 `errors.BadRequest(...)` 构造，网关用 `errors.FromError` 统一映射。
- **原因/代价**：gRPC 只有十几个状态码，业务码必须靠 `reason` 承载；直接把 `err.Error()` 返回客户端会泄露下游服务名与堆栈。
- **正确做法**：服务端 `errors.BadRequest("USER_NOT_FOUND", "user not found")`；网关按 `errors.FromError(err).Code` 映射 400/401/403/429/500 与 `400xx/401xx/...` 业务码。

## 四、Context 与并发
### Q11. 能不能用 `context.Background()` 发起下游调用？
- **结论**：除 `main`、初始化、后台任务外，一律禁止。
- **原因/代价**：它会切断取消信号与超时预算，客户端断开后数据库和 RPC 仍在跑；链路追踪的父子关系也会断掉。
- **正确做法**：handler 里用 `c.Request.Context()` 传给下游；长循环用 `select { case <-ctx.Done(): return ctx.Err() }` 及时退出。
### Q12. 服务跑一段时间后内存涨、goroutine 数只增不减？
- **结论**：按「泄漏」查：无限起 goroutine、channel 无人关闭、本地缓存无上限。
- **原因/代价**：goroutine 持有栈和引用，泄漏会把堆一起拖住；map 并发读写还会直接 `fatal error: concurrent map writes`。
- **正确做法**：goroutine 必须有退出条件，批量任务用 `errgroup.WithContext`；共享 map 用 `sync.RWMutex` 或 `sync.Map`；缓存放 Redis 或带容量淘汰的本地库。

## 五、超时、重试与熔断
### Q13. 超时应该怎么设？
- **结论**：逐层递减形成超时预算：网关 3s > gRPC 2s > 数据库/Redis 1s。
- **原因/代价**：下游超时比上游长，上游先返回失败而下游继续算，等于白烧资源；全部设成同一个值则无法判断是哪一层慢。
- **正确做法**：配置里写 `server.http.timeout: 3s`、`server.grpc.timeout: 2s`；每次调用用 `context.WithTimeout` 且 `defer cancel()`。
### Q14. 加了自动重试，下游反而被打挂？
- **结论**：只对幂等接口重试，且限定连接类错误与 5xx，配合指数退避加抖动。
- **原因/代价**：重试会放大流量（重试 3 次等于 4 倍 QPS），超时类失败重试往往雪上加霜；非幂等写接口重试会造成重复扣款。
- **正确做法**：只读接口与带幂等键的写接口可重试 1~2 次；退避上限不超过剩余超时预算；扣库存这类接口靠业务幂等兜底。
### Q15. 限流和熔断该用哪个？Sentinel 规则为什么不生效？
- **结论**：限流保护自己入口，熔断保护调用方不被慢下游拖死，两者都要有。
- **原因/代价**：只限流不熔断，慢下游会把连接池和 goroutine 占满；只熔断不限流，突发流量照样压垮自身。
- **正确做法**：资源名与埋点一致（HTTP `GET:/api/v1/orders`，gRPC `grpc:/user.v1.UserService/GetUser`）；`sentinel.InitDefault()` 后再 `flow.LoadRules`；`Entry` 后必须 `defer e.Exit()`；限流返回 429 + `Retry-After`，熔断返回 503。

## 六、数据库与事务
### Q16. GORM 报 `too many connections` 或请求卡在等连接？
- **结论**：连接池上限要小于 MySQL `max_connections` 除以实例数，并排查连接泄漏。
- **原因/代价**：`SetMaxOpenConns` 设成 100 而 10 个 Pod 就是 1000 条，MySQL 默认 151 直接拒绝；`Rows` 不 `Close`、事务不结束会长期占用连接。
- **正确做法**：`SetMaxOpenConns(50)`、`SetMaxIdleConns(20)`、`SetConnMaxLifetime(30*time.Minute)`（小于中间件空闲超时）；用 `SHOW PROCESSLIST` 与连接池指标核对。
### Q17. 事务里部分写入成功、部分失败？
- **结论**：必须用 `db.Transaction`，闭包里全部使用 `tx`，并在出错时返回 error。
- **原因/代价**：混用外层 `db` 会走另一条连接，既不在事务内，还容易和 `tx` 抢锁造成死锁；返回 nil 就等于主动提交。
- **正确做法**：

```go
// app/order-service/internal/data/order.go
return db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
    if err := tx.Create(&order).Error; err != nil { return err }
    return tx.Model(&Stock{}).Where("sku = ? AND stock >= ?", sku, n).
        UpdateColumn("stock", gorm.Expr("stock - ?", n)).Error
})
```

### Q18. 软删除后重建同一条记录，报唯一键冲突？
- **结论**：唯一索引里带上 `deleted_at` 解决不了，改用「删除标记 + 唯一值」方案。
- **原因/代价**：MySQL 唯一索引把多行 NULL 视为不冲突，但软删除只是把 `deleted_at` 写成时间，旧行仍占用 email/SKU 的值。
- **正确做法**：新增 `deleted_flag`（未删除为 0，删除时写主键或时间戳）参与唯一索引；查询始终过滤 `deleted_at IS NULL`；不要为了省事直接去掉软删除。

## 七、缓存
### Q19. 更新数据库后，缓存和数据库不一致？
- **结论**：Cache-Aside，先更新 DB 再删除缓存，配合 TTL 与异步失效兜底。
- **原因/代价**：先删缓存再写库，中间有请求回填旧值；双写必然有并发窗口，只能缩短而不能消除。
- **正确做法**：写完库删 key（失败要重试）；敏感数据用延迟双删或订阅 binlog 失效；TTL 兜底，不要设成永不过期。
### Q20. 缓存穿透、击穿、雪崩分别怎么防？
- **结论**：穿透防「查不到」，击穿防「热点过期」，雪崩防「同时过期」。
- **原因/代价**：三者都表现为数据库压力骤增，但根因不同，用错手段等于没防。
- **正确做法**：穿透用布隆过滤器或空值缓存（短 TTL）；击穿用 singleflight 或互斥重建；雪崩用随机 TTL、多级缓存与限流降级，见 [缓存模式与三大经典问题](../10-Redis/02-缓存模式与三大经典问题.md)。
### Q21. Redis 分布式锁为什么还是会重复执行？
- **结论**：多半是「没校验持有者就删」或「业务执行超过锁过期时间」。
- **原因/代价**：`SETNX` 后直接 `DEL` 会删掉别人的锁；锁自动过期而业务没跑完，第二个请求就进来了。
- **正确做法**：`SET key val NX PX 10000`，value 用唯一 token，释放时用 Lua 比对 token 再删；长任务加看门狗续期；关键路径再加数据库唯一约束。

## 八、消息与最终一致
### Q22. 同一条消息被消费两次，库存扣重了？
- **结论**：消费端必须幂等，不要指望 MQ 提供 exactly-once。
- **原因/代价**：Kafka 默认至少一次语义，提交 offset 前崩溃、重试、rebalance 都会重复投递。
- **正确做法**：用业务唯一键建去重表，或状态机 + 乐观锁（`WHERE status = 'CREATED'`）；消费成功后再提交 offset。
### Q23. 数据库写成功、消息没发出去，怎么办？
- **结论**：用本地消息表（Outbox）+ 定时投递器，保证「落库即一定发出」。
- **原因/代价**：先写库再发消息，中间进程崩溃就丢消息；先发消息再写库，可能发出不存在的订单。
- **正确做法**：业务数据与 outbox 记录在同一事务写入，后台任务扫描未发送记录投递 Kafka 并标记；消费端幂等，见 [分布式事务与最终一致性](../12-微服务与Kratos/08-分布式事务与最终一致性.md)。
### Q24. 同一订单的事件顺序乱了？
- **结论**：用订单号做 partition key，保证同 key 落到同一分区，并按 key 串行消费。
- **原因/代价**：Kafka 只保证分区内有序；消费者多线程处理同一 key 同样会乱序；全局有序只能单分区，吞吐会被锁死。
- **正确做法**：生产端指定 `Key`，消费端每个分区单 goroutine；只对乱序不敏感的业务才并发处理。

## 九、可观测性
### Q25. 网关有 TraceID，下游日志却是新链路？
- **结论**：两端都要接入 OTel，并且客户端要透传 metadata。
- **原因/代价**：只在一端埋点只能看到半条链路；gin 的 context 被替换过而没写回 `c.Request` 时，下游拿不到 `traceparent`。
- **正确做法**：Gin 用 `otelgin.Middleware("gateway")`，Kratos 服务端 `tracing.Server()`、客户端 `tracing.Client()`；换 context 时写回 `c.Request = c.Request.WithContext(ctx)`。
### Q26. 日志里搜不到 request_id / trace_id，指标口径也不一致？
- **结论**：字段名与指标名必须在项目内固定，由中间件统一注入。
- **原因/代价**：各服务自行打印字段，排障时无法用一条命令串起日志、指标、链路。
- **正确做法**：zap 固定字段 `service`、`env`、`request_id`、`trace_id`、`user_id`、`latency_ms`；指标统一 `http_server_requests_total`、`grpc_server_handled_total` 等命名。

## 十、部署与 K8s
### Q27. 滚动发布时出现 502 或 `connection reset`？
- **结论**：退出太快，没等在途请求处理完，Service 也还没摘干净。
- **原因/代价**：Pod 收到 SIGTERM 立即退出会掐断连接；`terminationGracePeriodSeconds` 默认 30s，小于慢请求耗时同样会断。
- **正确做法**：`signal.NotifyContext` + `srv.Shutdown(ctx)` 或 Kratos `app.Stop(ctx)`；`preStop: sleep 10` 让 Endpoint 先摘除；就绪探针先失败再退出。
### Q28. Pod 反复 `CrashLoopBackOff` 或 `OOMKilled`？
- **结论**：liveness 只判断进程是否活着，readiness 才检查依赖；内存要给 Go 留余量。
- **原因/代价**：把数据库检查写进 liveness，DB 抖动就会触发重启风暴；容器 limit 100MiB 而堆上限 128MiB 必然被杀。
- **正确做法**：`/livez` 只返回 200，`/readyz` 检查 MySQL/Redis；设 `GOMEMLIMIT` 为 limit 的 70%~80%；限制上传体积与并发。

## 十一、性能
### Q29. 压测 QPS 上不去，但 CPU 也不高？
- **结论**：先排除压测机与连接数瓶颈，再看是否被同步日志或连接池卡住。
- **原因/代价**：压测机和被测服务同机抢 CPU、客户端连接池太小、每个请求等日志刷盘，都会让 CPU 空转等 IO。
- **正确做法**：压测机与服务分离；调大 HTTP/gRPC 客户端连接池；日志异步或降采样；用 `hey`/`ghz` 逐步加压找拐点，见 [性能调优与压测](../08-部署与运维/04-性能调优与压测.md)。
### Q30. 内存持续上涨，GC 后不回落怎么定位？
- **结论**：抓 heap 与 goroutine 两份 profile 对比，绝大多数是「无上限缓存 + 跨请求持有对象」。
- **原因/代价**：本地 map 缓存不淘汰、把 `*gin.Context` 塞进全局队列、读大 Body 未限制，都会让老年代只增不减。
- **正确做法**：把 `net/http/pprof` 暴露到独立端口，执行 `go tool pprof -http=:8081 http://127.0.0.1:9101/debug/pprof/heap`；确认引用链后再改代码。

## 完整示例
把上述关键结论落到可运行代码：统一中间件、超时与优雅退出。

```go
// app/gateway/cmd/main.go
func main() {
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()
    r := gin.New()
    r.Use(gin.Recovery(), otelgin.Middleware("gateway"), middleware.RequestID(), middleware.Sentinel())
    r.SetTrustedProxies(nil)
    r.Group("/api/v1").GET("/users/:id", userHandler.Get)
    srv := &http.Server{Addr: ":8080", Handler: r, ReadHeaderTimeout: 2 * time.Second}
    go func() {
        if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
            zap.L().Fatal("gateway start failed", zap.Error(err))
        }
    }()
    <-ctx.Done()
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    _ = srv.Shutdown(shutdownCtx) // 等在途请求结束后再退出
}
```

配套配置见 `configs/gateway.yaml`：`server.http.timeout: 3s`、`server.grpc.timeout: 2s`、`sentinel.enabled: true`，可用 `MALL_SERVER_HTTP_TIMEOUT` 覆盖。

## 常见坑

1. **只改配置不看日志**：重启好不等于根因解决，先看 error 日志与 Trace 再动手。
2. **一次改多个变量**：无法归因，每次只改一个并用压测回归验证。
3. **HTTP 与 gRPC 超时设成同一个值**：失去分层判断能力，容易出现下游比上游活得久。
4. **给非幂等接口加重试**：重复下单、重复扣款，比超时本身更严重。
5. **`c.Abort()` 之前已经写了响应**：状态码与业务码对不上，客户端解析失败。
6. **认为 `gin.Context` 可以跨 goroutine 传递**：忘记 `c.Copy()` 是最常见的隐性 data race。

## 小结

- 选型上：Gin 做 BFF，Kratos 做 gRPC 服务，契约以 proto 为唯一事实来源。
- 稳定性三件套：超时预算、幂等重试、限流 + 熔断，缺一不可；数据侧问题集中在连接池、事务边界、缓存失效与消息幂等。
- 排障顺序固定：日志与 Trace → 指标 → 最小复现 → 单点修复 → 回归。

## 练习

1. 在网关中间件里统计 P99，并给一个 handler 加 `time.Sleep`，观察指标变化。
2. 用 `buf breaking` 制造一次不兼容的 proto 修改，让 CI 把它拦下来。
3. 关掉 MySQL 再启动服务，验证 readiness 失败但 liveness 依然通过。

## 官方文档

- [Gin 官方文档](https://gin-gonic.com/zh-cn/docs/)
- [Kratos 官方文档](https://go-kratos.dev/docs/)
- [gRPC-Go 快速开始](https://grpc.io/docs/languages/go/quickstart/)
- [Sentinel Go](https://sentinelguard.io/zh-cn/docs/golang/quick-start.html)
- [GORM 文档](https://gorm.io/zh_CN/docs/)
