# 01-go-redis接入与连接池
> 一句话定位：把 go-redis v9 接进 mall-lite，参数调对、错误分清、生命周期管好，而不是只会 `Set/Get`。
## 本篇目标
- 用 `redis.NewClient` / `redis.NewClusterClient` 正确初始化客户端
- 逐个理解 `Options` 中连接池、超时、重试参数的真实含义
- 分清 `redis.Nil` 与网络错误，写出可降级的读缓存代码
- 用 Pipeline / TxPipeline / Lua 脚本减少 RTT 并保证原子性
- 在 Hertz 与 Kitex 中注入 Redis，并做到健康检查与优雅关闭
- 看懂 `PoolStats()`，建立连接池监控指标
## 前置知识
- 已读 [09-MySQL与ORM/05](../09-MySQL与ORM/05-连接池与数据库性能调优.md)，理解连接池的意义
- 本地有 Redis 7（`docker run -d -p 6379:6379 redis:7`）
## 一、依赖与最小可用客户端
```bash
go get github.com/redis/go-redis/v9@latest
```
```powershell
go get github.com/redis/go-redis/v9@latest
```
```go
package cache

import (
	"context"
	"time"

	"github.com/redis/go-redis/v9"
)

func NewClient(addr, password string) *redis.Client {
	return redis.NewClient(&redis.Options{
		Addr:     addr,      // "127.0.0.1:6379"
		Password: password,  // 无密码留空
		DB:       0,         // 单机多库；Cluster 不支持 DB
	})
}

func Ping(ctx context.Context, rdb *redis.Client) error {
	cctx, cancel := context.WithTimeout(ctx, time.Second)
	defer cancel()
	return rdb.Ping(cctx).Err()
}
```
`NewClient` 不会立即建连，第一次命令才拨号；所以生产启动时应显式 `Ping` 一次，把配置错误暴露在启动期而不是第一笔请求。
## 二、Options 参数逐个讲
```go
rdb := redis.NewClient(&redis.Options{
	Addr:            "127.0.0.1:6379",
	Password:        cfg.Password,
	DB:              0,
	PoolSize:        50,                     // 每个节点的最大连接数
	MinIdleConns:    10,                     // 预热空闲连接，避免冷启动抖动
	PoolTimeout:     2 * time.Second,        // 连接池取不到连接时最多等多久
	DialTimeout:     1 * time.Second,        // 建连超时
	ReadTimeout:     500 * time.Millisecond, // 单条命令读超时
	WriteTimeout:    500 * time.Millisecond, // 单条命令写超时
	MaxRetries:      2,                      // 失败重试次数（-1 关闭，0 走默认 3 次）
})
```
| 参数 | 调大的后果 | 调小的后果 | 建议 |
| --- | --- | --- | --- |
| PoolSize | Redis 连接数暴涨、内存上升 | 高并发下排队，PoolTimeout 增多 | QPS×平均 RT(s)×1.2，单实例 30~100 |
| MinIdleConns | 空闲占资源 | 突发流量要现拨号 | 设为 PoolSize 的 20% |
| PoolTimeout | 故障时请求堆积更久 | 略抖动就报错 | 100~500ms，配合业务超时 |
| Read/WriteTimeout | 慢查询拖死调用方 | 误杀正常命令 | 200~500ms，必须小于上游超时 |
| MaxRetries | 故障放大、非幂等命令重复执行 | 抖动直接失败 | 1~2，且业务侧再做幂等 |
> 连接池是「共享」的：`PoolSize` 是单进程对该 Redis 节点的总连接上限，不是每 goroutine 一个。
## 三、单机、哨兵、Cluster 三种初始化
```go
// 单机
rdb := redis.NewClient(&redis.Options{Addr: cfg.Addr, Password: cfg.Password})
// 哨兵：Failover 自动切换主节点
srdb := redis.NewFailoverClient(&redis.FailoverOptions{
	MasterName:    "mymaster",
	SentinelAddrs: []string{"10.0.0.11:26379", "10.0.0.12:26379"},
	Password:      cfg.Password,
	PoolSize:      50,
})
// 集群：按 key 的 slot 路由
crdb := redis.NewClusterClient(&redis.ClusterOptions{
	Addrs:    []string{"10.0.1.1:6379", "10.0.1.2:6379", "10.0.1.3:6379"},
	Password: cfg.Password,
	PoolSize: 30,
})
```
`*redis.Client` 与 `*redis.ClusterClient` 都实现了 `redis.UniversalClient` 接口。业务层字段声明为 `rdb redis.UniversalClient`，本地单机与线上集群就能共用一份代码。
注意：`ClusterClient` 不支持 `SELECT DB`，多 key 命令要求 key 落在同一 slot（用 `{tag}` 哈希标签），后文 05 篇详述。
## 四、redis.Nil：未命中不是错误
```go
func (r *ProductRepo) Detail(ctx context.Context, id int64) (*Product, error) {
	key := fmt.Sprintf("mall:prod:product:detail:%d", id)
	raw, err := r.rdb.Get(ctx, key).Bytes()
	switch {
	case err == nil:
		return decodeProduct(raw)
	case errors.Is(err, redis.Nil):
		// 缓存未命中：正常业务流程，回源 MySQL
		return r.loadFromDB(ctx, id)
	default:
		// Redis 故障或超时：降级直查 DB，并打点告警
		r.metrics.CacheError.Inc()
		return r.loadFromDB(ctx, id)
	}
}
```
三种「没有值」要分清：
- key 不存在：`redis.Nil`，属于正常未命中，绝不能当报警
- 连接失败 / 超时：真实错误，应降级并计入错误率
- 类型错误（`WRONGTYPE`）：代码 bug，返回 `*redis.Error` 前缀包含 `WRONGTYPE`
错误判断统一用 `errors.Is(err, redis.Nil)`，不要用 `err.Error() == "redis: nil"` 字符串比较。
## 五、Pipeline 与 TxPipeline
一次商品详情页往往要取商品、库存、价格、促销多个 key，串行 RTT 叠加会让接口变慢：
```go
func (r *ProductRepo) BatchGet(ctx context.Context, ids []int64) ([]*Product, error) {
	pipe := r.rdb.Pipeline()
	cmds := make([]*redis.StringCmd, 0, len(ids))
	for _, id := range ids {
		cmds = append(cmds, pipe.Get(ctx, fmt.Sprintf("mall:prod:product:detail:%d", id)))
	}
	if _, err := pipe.Exec(ctx); err != nil && !errors.Is(err, redis.Nil) {
		return nil, err
	}
	out := make([]*Product, 0, len(ids))
	for _, cmd := range cmds {
		if err := cmd.Err(); err != nil {
			continue // 单个 key 缺失不影响其他 key
		}
		p, err := decodeProduct([]byte(cmd.Val()))
		if err != nil {
			return nil, err
		}
		out = append(out, p)
	}
	return out, nil
}
```
| 方式 | 命令原子性 | 典型用途 |
| --- | --- | --- |
| `Pipeline()` | 不保证，批量发送 | 批量读缓存、批量预热 |
| `TxPipeline()` | MULTI/EXEC 包裹 | 多条写需要原子、且无读依赖 |
| Lua `Script` | 单命令原子 | 读后写、条件扣减、限流 |
```go
pipe := rdb.TxPipeline()
pipe.Set(ctx, "mall:prod:product:detail:1001", body, 30*time.Minute)
pipe.Expire(ctx, "mall:prod:product:stock:1001", 30*time.Minute)
if _, err := pipe.Exec(ctx); err != nil {
	return err
}
```
`TxPipeline` 中命令的返回值要等 `Exec` 后才可用，且不支持在事务内根据读结果决定写什么；有读依赖时必须用 Lua。
## 六、Lua 脚本：用 NewScript 而不是 Eval
```go
var unlockScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
	return redis.call("DEL", KEYS[1])
end
return 0
`)

func Unlock(ctx context.Context, rdb redis.UniversalClient, key, token string) (bool, error) {
	n, err := unlockScript.Run(ctx, rdb, []string{key}, token).Int64()
	if err != nil {
		return false, err
	}
	return n == 1, nil
}
```
`NewScript` 会自动用 `EVALSHA`，遇到 `NOSCRIPT` 自动回退 `EVAL`，不要自己拼 `Eval` 字符串。约定：
- 所有 key 通过 `KEYS` 传，所有参数通过 `ARGV` 传，禁止字符串拼接
- 脚本保持短小，禁止写循环处理大集合（会阻塞单线程）
- Cluster 下所有 `KEYS` 必须同 slot
## 七、context 超时与取消
```go
cctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
defer cancel()
return rdb.Get(cctx, key).Result()
```
- Hertz 的 `ctx` 会在客户端断开时取消，Redis 命令应透传它，避免做无用功
- 不要把 `context.Background()` 用在请求链路上，否则超时控制全部失效
- 超时值应小于上游（网关）超时，形成「下游先超时，上游还有余量降级」的层次
## 八、注入 Hertz / Kitex 与优雅关闭
```go
// Hertz 侧：中间件里做健康检查（仅 /ready 这类探针使用，别每个请求都 PING）
func RedisReady(rdb *redis.Client) app.HandlerFunc {
	return func(ctx context.Context, c *app.RequestContext) {
		cctx, cancel := context.WithTimeout(ctx, 200*time.Millisecond)
		defer cancel()
		if err := rdb.Ping(cctx).Err(); err != nil {
			c.AbortWithStatusJSON(503, utils.H{"code": 50301, "msg": "redis unavailable"})
			return
		}
		c.JSON(200, utils.H{"status": "ok"})
	}
}

func main() {
	cfg := config.MustLoad()
	rdb := cache.NewClient(cfg.Redis.Addr, cfg.Redis.Password)
	if err := cache.Ping(context.Background(), rdb); err != nil {
		log.Fatalf("redis connect: %v", err) // 启动期失败直接退出，交给编排系统重启
	}
	defer rdb.Close()

	repo := repository.NewProductRepo(rdb, db)
	h := server.Default()
	h.GET("/ready", cache.RedisReady(rdb))
	handler.Register(h, service.NewProductService(repo))
	h.Spin() // 内部处理 SIGTERM，返回后执行 defer
}
```
Kitex 侧同理，通过构造函数把 `redis.UniversalClient` 注入 service 实现，不在 handler 里 `redis.NewClient`。优雅关闭顺序：先停止接流量（Hertz `Spin` 返回 / Kitex `svr.Stop`），再 `rdb.Close()`，保证在途请求还能用完连接。
## 九、连接池指标与监控
```go
func ReportPoolStats(rdb *redis.Client) {
	st := rdb.PoolStats()
	log.Printf("redis pool hits=%d misses=%d timeouts=%d total=%d idle=%d stale=%d",
		st.Hits, st.Misses, st.Timeouts, st.TotalConns, st.IdleConns, st.StaleConns)
}
```
| 指标 | 含义 | 异常信号 |
| --- | --- | --- |
| `Hits` | 直接从池中拿到空闲连接 | — |
| `Misses` | 池空需新建连接 | 持续上涨说明 PoolSize 偏小 |
| `Timeouts` | 等连接超时次数 | 大于 0 就应告警 |
| `TotalConns` | 当前总连接数 | 长期等于 PoolSize 说明打满 |

配合 `redis_exporter` 暴露 `redis_up`、`redis_connected_clients`、`redis_keyspace_hits_total` 等指标，在 Grafana 上建立「命中率 + 池耗尽 + 超时率」三张图。
## 常见坑
1. **把 `redis.Nil` 当异常上报**：缓存未命中会瞬间把错误率打到 100%，监控彻底失真。
2. **超时配置大于上游超时**：Redis 卡住时请求永远等不到降级，网关先超时，线程/协程被拖满。
3. **`PoolSize` 拍脑袋设 10**：QPS 2000、RT 5ms 就需要约 10 个连接只是理论值，加上突发与 Pipeline，实际应留 2 倍余量。
4. **在 Cluster 上用 `KEYS`/多 key 事务**：报 `CROSSSLOT Keys in request don't hash to the same slot`，必须在设计 key 时用 `{tag}`。
5. **Lua 里用 `redis.call('KEYS', '*')`**：阻塞 Redis 主线程，生产环境直接引发雪崩。
6. **`MaxRetries` 调大后执行非幂等命令**：`INCR` 重试会多计数，重试只适合只读命令。
## 小结
- `Options` 里真正决定稳定性的是 `PoolSize`/`PoolTimeout`/`ReadTimeout`，不是 `Addr`
- `redis.Nil` 是业务信号，其他错误才是故障信号，两者必须分开处理
- 批量读用 Pipeline，需要原子读改写用 Lua，Cluster 下先想 slot
- Redis 客户端在启动期 `Ping`、在请求期透传 context、在关闭期 `Close`
- 连接池指标必须进监控，否则「偶发变慢」永远定位不到
## 练习
1. 给 mall-lite 的 `product-service` 写一个 `repository/redis.go`，用 `UniversalClient` 支持单机与 Cluster 切换。
2. 实现 `BatchGetProducts(ids []int64)`，要求 100 个 id 只产生 1 次 RTT，并正确处理部分 key 缺失。
3. 用 `redis.NewScript` 写一个「读库存，够则扣减并返回剩余」的脚本，并验证并发 1000 次不超卖。
4. 用 `PoolStats` 每 10 秒打一条日志，压测时观察 `Timeouts` 是否出现，并调整 `PoolSize` 复测。
## 官方文档
- go-redis 官方文档：<https://redis.uptrace.dev/guide/>
- go-redis Options 源码：<https://github.com/redis/go-redis/blob/master/options.go>
- Redis 命令参考：<https://redis.io/docs/latest/commands/>
- Redis 客户端连接管理：<https://redis.io/docs/latest/develop/clients/pools-and-multiplexing/>
