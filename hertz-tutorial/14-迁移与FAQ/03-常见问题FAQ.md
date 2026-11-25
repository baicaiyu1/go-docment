# 03-常见问题FAQ

> 一句话定位：把安装、路由、性能、数据层、消息、微服务与部署中的高频故障按“现象 → 原因 → 解决”整理成可执行的排查手册。

## 本篇目标

- 覆盖 9 类场景、35 个高频问题，每题固定“现象 → 原因 → 解决”。
- 每个答案尽量给出可直接执行的命令或代码，而不是“理论上应该”。
- 建立“先定位、再修复、后验证”的排查习惯。

## 前置知识

- 已跑通 Hertz 服务，读过 [错误处理](../03-基本特性/06-错误处理.md)、[性能调优与压测](../08-部署与运维/04-性能调优与压测.md)。
- 能看懂日志、`pprof`、`EXPLAIN` 与 `kubectl` 的常见输出。

## 一、安装与依赖

### 1. go mod tidy 超时 / 410 Gone
- **现象**：下载依赖卡住，或报 `410 Gone`、`connection reset`。
- **原因**：默认 `proxy.golang.org` 在部分网络不可达，或被企业代理拦截。
- **解决**：`go env -w GOPROXY=https://goproxy.cn,direct`、`GOSUMDB=sum.golang.google.cn`，私有仓库再加 `GOPRIVATE` 后重试。

### 2. hz 命令找不到
- **现象**：`hz: command not found` 或 `'hz' 不是内部或外部命令`。
- **原因**：`GOBIN` 不在 `PATH`，或改完环境变量没重开终端。
- **解决**：`go env GOBIN GOPATH` 找到二进制目录并加入 `PATH`；新开终端后用 `where.exe hz`（Windows）/ `which hz` 验证；中文用户名可 `go env -w GOBIN=D:\Go\bin`。

### 3. hz 生成代码被覆盖
- **现象**：执行 `hz update` 后手写逻辑消失，接口返回空数据。
- **原因**：修改了 `biz/model`、`biz/router`、`router_gen.go` 等生成物。
- **解决**：`git checkout -- biz/model biz/router router_gen.go` 恢复；业务逻辑只写 `biz/handler` 与 `pkg/`；CI 增加 `hz update && git diff --exit-code`。

### 4. hz 与 Hertz 版本不匹配
- **现象**：生成代码引用不存在的 API，编译失败。
- **原因**：hz、Hertz、thriftgo/protoc 版本错位。
- **解决**：`go list -m github.com/cloudwego/hertz` 与 `hz -v` 对齐基线（Hertz v0.10.6 / hz v0.9.8），团队统一版本后重跑生成。

### 5. 启动端口被占用
- **现象**：启动报 `bind: address already in use`。
- **原因**：旧进程未退出，或 8888/8080 被其他服务占用。
- **解决**：Windows `netstat -ano | findstr :8080` 后 `taskkill /PID <pid> /F`；Linux `ss -lntp | grep 8080`；或改端口。

## 二、Hertz 路由与绑定

### 6. 路由 404
- **现象**：接口确定注册了，访问仍返回 404。
- **原因**：方法不匹配、路径前缀/尾斜杠不一致、`Group` 前缀拼错、注册发生在 `Spin` 之后。
- **解决**：启动时打印 `h.Routes()` 逐条核对；开启 `WithHandleMethodNotAllowed(true)` 区分 404/405；所有路由必须在 `Spin` 前注册。

### 7. 路由注册 panic（冲突）
- **现象**：启动 panic，提示 `path already registered` 或参数冲突。
- **原因**：同一方法同一路径注册两次；同一位置混用 `:id` 与 `:uid`；手写路由与 IDL 生成路由重复。
- **解决**：全局搜索冲突路径；参数名统一；以 IDL 为单一事实来源，生成代码不手改。

### 8. 绑定失败返回 400
- **现象**：合法请求也报 400，错误信息含糊。
- **原因**：字段 tag 拼错、`Content-Type` 缺失、JSON 类型不匹配、`validate` 规则过严。
- **解决**：先打印 `err.Error()` 与原始 Body；确认 `json`/`query`/`path` tag 与请求来源一致；用 `BindJSON` 缩小范围；统一 `AbortWithStatusJSON(400, ...)`。

### 9. validate 校验不生效
- **现象**：`validate:"required"` 形同虚设。
- **原因**：Hertz 默认校验器使用 `vd` 标签，不执行 go-playground 的 `validate`。
- **解决**：用 `server.WithCustomValidatorFunc` 注入 `validator.New(validator.WithRequiredStructEnabled())`，再调用 `BindAndValidate`，详见 [参数绑定与校验](../03-基本特性/03-参数绑定与校验.md)。

### 10. Body 读两次为空
- **现象**：先 `c.Body()` 再 `Bind`，绑定结果全为零值。
- **原因**：请求体是流，只能消费一次；读取后游标已到末尾。
- **解决**：只选一种方式；需要同时拿原始 Body 与结构体时，先 `c.Body()` 再 `json.Unmarshal`，不要与 `Bind` 混用。

### 11. 文件上传 413
- **现象**：上传稍大文件返回 `413 Request Entity Too Large`。
- **原因**：`WithMaxRequestBodySize` 默认 4MB，且统计整个 multipart 请求体。
- **解决**：`server.WithMaxRequestBodySize(20<<20)`；业务层再限制单文件大小、数量与后缀；大文件改用对象存储直传。

### 12. 跨域不生效
- **现象**：浏览器报 CORS 错误，OPTIONS 预检 404/403。
- **原因**：未注册 CORS 中间件；`AllowOrigins` 与前端 Origin 不一致；`AllowAllOrigins` 与 `AllowCredentials` 同时为真。
- **解决**：`h.Use(cors.New(cors.Config{...}))` 放在路由前并放行 `OPTIONS`；凭证模式必须写具体 Origin，不能用 `*`。

## 三、中间件与错误处理

### 13. panic 未捕获
- **现象**：一次 panic 导致连接中断，甚至进程退出。
- **原因**：用了 `server.New` 但没注册 Recovery，或 Recovery 不在最外层。
- **解决**：`h.Use(recovery.Recovery(...))` 放在 `Use` 第一位；`server.Default` 已内置，不要重复嵌套。

### 14. 中间件不生效或顺序错
- **现象**：鉴权没执行、日志顺序颠倒、状态码取不到。
- **原因**：`Use` 在路由注册之后；漏写 `c.Next(ctx)`；在 `Next` 前读状态码。
- **解决**：`Use` 必须在路由前；后置逻辑写在 `Next` 之后；要中断用 `Abort`，不要靠“不写 Next”。

### 15. c.Error 后响应重复
- **现象**：响应体出现两段 JSON，或状态码与业务码不一致。
- **原因**：`c.Error` 后继续写响应；中间件又写了一次；先写 Body 再 `c.Error`。
- **解决**：约定 `c.Error(err); return`，由错误中间件统一 `AbortWithStatusJSON`；不要在 handler 与中间件各写一次。

## 四、性能与压测

### 16. 内存持续上涨
- **现象**：RSS 缓慢上涨，GC 后不回落，最终 OOM。
- **原因**：请求对象跨 handler 持有、goroutine 泄漏、读大 Body 未流式、客户端每次新建、本地缓存无上限。
- **解决**：用 `go tool pprof` 看 heap/goroutine；不跨请求持有 Context；全局复用 Client；大 Body 流式；本地缓存设 TTL 与容量。

### 17. P99 抖动
- **现象**：平均延迟正常，P99 周期性飙升。
- **原因**：GC 停顿、连接池排队、下游慢调用、锁竞争、日志同步刷盘。
- **解决**：看 pprof、GC 指标与连接池等待；设置 DB/Redis/RPC 超时；日志异步；对慢下游做隔离与熔断。

### 18. QPS 上不去
- **现象**：压测 QPS 远低于预期，CPU 却不高。
- **原因**：压测机瓶颈、连接数不足、Windows 自动回退标准库、客户端 `MaxConnsPerHost` 太小。
- **解决**：压测机与服务机分离；Linux 上确认 netpoll 生效；调大连接池；用 `hey/wrk` 逐步加压定位拐点。

## 五、MySQL 与 GORM

### 19. GORM 连接池耗尽
- **现象**：请求卡住，日志报 `too many connections` 或等连接超时。
- **原因**：`SetMaxOpenConns` 超过 MySQL `max_connections`，或事务/`Rows` 未释放连接。
- **解决**：`SetMaxOpenConns(50)`、`SetMaxIdleConns(20)`、`SetConnMaxLifetime(30*time.Minute)`；确保 `Rows` 关闭；用 `SHOW PROCESSLIST` 与连接池指标定位。

### 20. 软删除与唯一索引冲突
- **现象**：删除后重新插入相同 email/SKU 报唯一键冲突。
- **原因**：`gorm.DeletedAt` 只改标记，唯一索引仍包含已删除行。
- **解决**：新增删除标记列（未删除为 0，删除时写入唯一值如时间戳/ID）并加入唯一索引；或删除时改写唯一字段；或物理删除归档；不要直接去掉软删除。

### 21. GORM N+1 查询
- **现象**：列表接口 SQL 数量随行数增长，P99 很差。
- **原因**：循环里逐个查关联，未用 `Preload` / `Joins`。
- **解决**：`db.Preload("Orders").Find(&users)`；只取需要列；用 `EXPLAIN` 确认走索引；必要时手写 JOIN。

### 22. 事务未回滚
- **现象**：部分写入成功、部分失败，数据不一致。
- **原因**：`Transaction` 回调返回 nil、在事务外用了 `db`、panic 未触发回滚。
- **解决**：`db.Transaction(func(tx *gorm.DB) error { ... return err })`；所有操作使用 `tx`；用 `errors.Is` 判断并返回错误，不要吞错。

## 六、Redis

### 23. Redis 热 key
- **现象**：单个 key QPS 极高，Redis 单核打满，延迟抖动。
- **原因**：热点商品/配置集中在一个 key，流量倾斜。
- **解决**：本地缓存 + 短 TTL；key 加随机后缀拆成多副本；读写分离；限流降级；用监控或 `redis-cli --hotkeys`（需 LFU 策略）定位。

### 24. Redis 大 key
- **现象**：`GET/SET` 慢、删除卡顿、网络带宽突增。
- **原因**：value 达 MB 级，或 Hash/List/Set 元素过多。
- **解决**：拆分 key；只存必要字段；用 `HSCAN/SSCAN` 分批处理；删除用 `UNLINK`；上线前用 `redis-cli --bigkeys` 巡检。

### 25. 缓存与 DB 不一致
- **现象**：更新后读到旧值，或缓存长期脏数据。
- **原因**：先删缓存再写库、并发读写交错、更新失败未补偿、TTL 过长。
- **解决**：Cache-Aside 先更新 DB 再删缓存；延迟双删；订阅 binlog 异步失效；设置合理 TTL 与重试。

## 七、Kafka

### 26. 消费积压
- **现象**：`lag` 持续增长，下游数据延迟。
- **原因**：消费者实例少于分区数、单条处理慢、下游阻塞、频繁 rebalance。
- **解决**：`kafka-consumer-groups.sh --bootstrap-server <broker> --describe --group <g>` 看 lag；消费者扩到分区数；批量与异步处理；避免 `max.poll.interval.ms` 超时。

### 27. 消息重复消费
- **现象**：同一订单被扣两次库存。
- **原因**：至少一次语义、提交 offset 前崩溃、失败重试。
- **解决**：业务幂等（唯一键、状态机、去重表）；消费成功后再提交 offset；用业务 ID 或 `(topic, partition, offset)` 去重。

### 28. 消息顺序错乱
- **现象**：同一订单的创建/支付事件被乱序处理。
- **原因**：消息散落到多个分区；消费者并发处理同一 key。
- **解决**：用业务主键做 partition key；同一 key 单线程消费；需要全局顺序时只能单分区，代价是吞吐。

## 八、Kitex 与注册中心

### 29. Kitex 客户端连接不复用
- **现象**：QPS 升高后端口耗尽、握手耗时高。
- **原因**：每次请求 `client.NewClient`，或连接池配置过小。
- **解决**：Client 全局单例；复用生成的 client；配置连接池与服务发现；关闭时统一 `Close`。

### 30. Nacos 服务发现失败
- **现象**：Kitex 报 `no instance found`，或 Hertz 注册后调不到。
- **原因**：namespace/group/serviceName 不一致；9848 gRPC 端口未放通；实例非临时或心跳失败；本地缓存过期。
- **解决**：`curl "http://127.0.0.1:8848/nacos/v1/ns/instance/list?serviceName=xxx&groupName=DEFAULT_GROUP"` 核对；检查 8848/9848、网络与鉴权；统一 namespace/group；重启消费者刷新缓存。

### 31. Kitex 超时重试导致重复请求
- **现象**：下游收到两次扣库存请求。
- **原因**：超时后 Kitex 自动重试，而接口非幂等。
- **解决**：非幂等接口关闭重试或只重试连接错误；业务层用请求 ID 幂等；合理设置 `WithRPCTimeout` 与重试次数。

### 32. 跨服务 Trace 断链
- **现象**：网关有 TraceID，下游日志却是新的链路。
- **原因**：只在一端接入 OpenTelemetry；HTTP/RPC 调用未注入 `traceparent`；日志未带 TraceID。
- **解决**：Hertz 与 Kitex 都接 OTel 中间件；客户端透传 metadata/header；日志统一从 context 取 TraceID。

## 九、部署与 K8s

### 33. K8s 优雅退出丢请求
- **现象**：滚动发布时出现 502 或 `connection reset`。
- **原因**：Pod 收到 SIGTERM 后立即退出；Service 摘除有延迟；`terminationGracePeriodSeconds` 太短。
- **解决**：启动时加 `server.WithExitWaitTime(10*time.Second)` 并用 `h.Spin()`；加 `preStop: sleep 10`；`terminationGracePeriodSeconds: 30`；先让就绪探针失败再退出。

### 34. 探针失败导致反复重启
- **现象**：Pod `CrashLoopBackOff` 或一直 NotReady。
- **原因**：探针路径/端口写错；启动慢于 `initialDelaySeconds`；依赖未就绪。
- **解决**：`/ping` 做 liveness，`/ready` 检查 DB/Redis；合理设置 `initialDelaySeconds`、`periodSeconds`、`failureThreshold`；不要把外部依赖失败当 liveness。

### 35. 容器 OOMKilled
- **现象**：Pod 重启，`kubectl describe` 显示 `OOMKilled`。
- **原因**：Go 堆 + 大 Body + 连接池超出 limit；未设 `GOMEMLIMIT`。
- **解决**：`GOMEMLIMIT` 设为 limit 的 70%~80%；限制上传大小与并发；排查 goroutine/缓存泄漏；调整 requests/limits。

## 常见坑

1. 只看现象就改配置，不先看日志、指标与 Trace。
2. 同时改多个变量，无法判断哪个修复生效。
3. 没有最小复现，靠猜。
4. 修复后不做压测与回归。
5. 把“重启好了”当成根因已解决。

## 小结

- 大部分问题可归为四类：配置/版本、请求生命周期、连接池、下游依赖。
- 排查顺序固定为：日志与 Trace → 指标 → 最小复现 → 单点修复 → 回归。
- 每个答案尽量给出可直接执行的命令，避免“理论上应该”。

## 练习

1. 从本篇挑 3 个你遇到过的问题，按“现象/原因/解决”写成团队 runbook。
2. 为“路由 404”和“GORM 连接池耗尽”各写一条监控告警规则。
3. 用 `hey` 复现 P99 抖动，并用 pprof 找出一个具体原因。

## 相关章节

- [错误处理](../03-基本特性/06-错误处理.md)
- [性能调优与压测](../08-部署与运维/04-性能调优与压测.md)
- [连接池与数据库性能调优](../09-MySQL与ORM/05-连接池与数据库性能调优.md)
- [缓存模式与三大经典问题](../10-Redis/02-缓存模式与三大经典问题.md)
- [积压、死信、延迟消息与监控](../11-消息队列/05-积压-死信-延迟消息与监控.md)
- [服务注册与发现 Nacos 实践](../12-微服务与Kitex/05-服务注册与发现Nacos实践.md)
- [Kubernetes 部署与健康检查](../08-部署与运维/03-Kubernetes部署与健康检查.md)
