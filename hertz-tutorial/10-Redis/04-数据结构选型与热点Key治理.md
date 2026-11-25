# 04-数据结构选型与热点Key治理
> 一句话定位：选对 Redis 数据类型只是及格线，能识别并治理大 Key、热 Key 才决定线上是否稳定。
## 本篇目标
- 掌握 String/Hash/List/Set/ZSet/Bitmap/HyperLogLog/Stream 在 mall-lite 中的真实用法
- 能判定大 Key 并给出拆分方案，避免 `DEL` 阻塞主线程
- 掌握热 Key 的探测手段与四种治理手法
- 用 `SCAN` 替代 `KEYS *`，用 Pipeline 降低 RTT
- 理解 `maxmemory` 与淘汰策略，建立 Key 命名与多环境隔离规范
## 前置知识
- 已读 [02-缓存模式与三大经典问题](02-缓存模式与三大经典问题.md)
- 了解 Redis 单线程命令执行模型
- 会用 `redis-cli` 与 `MEMORY USAGE`
## 一、选型总表
| 类型 | 底层结构 | mall-lite 场景 | 典型命令 |
| --- | --- | --- | --- |
| String | SDS | 商品详情 JSON、计数器、分布式锁 | `SET`/`GET`/`INCR` |
| Hash | listpack / hashtable | 购物车、对象部分字段更新 | `HSET`/`HINCRBY`/`HGETALL` |
| List | quicklist | 简单队列、时间线 | `LPUSH`/`BRPOP`/`LRANGE` |
| Set | intset / hashtable | 商品标签、去重、共同关注 | `SADD`/`SISMEMBER`/`SINTER` |
| ZSet | listpack / skiplist | 销量榜、延迟队列 | `ZADD`/`ZINCRBY`/`ZREVRANGE` |
| Bitmap | String 位图 | 签到、活跃标记 | `SETBIT`/`BITCOUNT` |
| HyperLogLog | 稀疏/稠密寄存器 | UV 统计 | `PFADD`/`PFCOUNT` |
| Stream | rax | 轻量事件流、消费组 | `XADD`/`XREADGROUP` |
## 二、String：计数器与详情缓存
```go
func IncrProductViews(ctx context.Context, rdb redis.UniversalClient, id int64) (int64, error) {
	key := fmt.Sprintf("mall:prod:product:views:%d", id)
	pipe := rdb.Pipeline()
	incr := pipe.Incr(ctx, key)
	pipe.Expire(ctx, key, 48*time.Hour)
	if _, err := pipe.Exec(ctx); err != nil {
		return 0, err
	}
	return incr.Val(), nil
}
```
计数器常见错误是只 `INCR` 不设过期，日积月累产生海量永不过期的 key。用 Pipeline 把 `INCR` 与 `EXPIRE` 一起发（不要求原子，重复设置 TTL 无害）；需要严格原子用 Lua。
## 三、Hash：购物车与局部更新
```go
func AddToCart(ctx context.Context, rdb redis.UniversalClient, uid, skuID string, n int64) error {
	key := "mall:order:cart:" + uid
	if err := rdb.HIncrBy(ctx, key, skuID, n).Err(); err != nil {
		return err
	}
	return rdb.Expire(ctx, key, 7*24*time.Hour).Err()
}

func CartItems(ctx context.Context, rdb redis.UniversalClient, uid string) (map[string]string, error) {
	return rdb.HGetAll(ctx, "mall:order:cart:"+uid).Result()
}
```
Hash 的优点是可以只更新某个字段（如商品数量），不必读改整个 JSON。但要注意：
- 单 Hash 字段数控制在 1000 以内，超过就按 uid 分片或改为「每商品一个 key」
- `HGETALL` 大 Hash 会一次性返回全部字段，阻塞 Redis；用 `HSCAN` 分批
## 四、List：简易队列，但别当可靠队列
```go
// 生产：左侧入队
rdb.LPush(ctx, "mall:notify:sms:queue", payload)
// 消费：阻塞右取，最多等 2 秒
res, err := rdb.BRPop(ctx, 2*time.Second, "mall:notify:sms:queue").Result()
```
`BRPOP` 取出即删除，进程崩溃会丢消息，也没有 ACK 与重试。**业务消息请用 Kafka/RocketMQ**（第 11 章），List 只适合日志缓冲、非关键任务等可丢场景。若一定要用，需配合 `BRPOPLPUSH` 的备份队列模式实现「取出未确认」。
## 五、Set 与 ZSet：去重、标签、排行榜
```go
// 商品标签
rdb.SAdd(ctx, "mall:prod:product:tags:1001", "数码", "热销")
// 是否包含某标签
ok, _ := rdb.SIsMember(ctx, "mall:prod:product:tags:1001", "热销").Result()
// 两个商品的共同标签
common, _ := rdb.SInter(ctx, "mall:prod:product:tags:1001", "mall:prod:product:tags:1002").Result()
```
```go
// 销量榜：分数累加，按分数倒序取 TOP 10
rdb.ZIncrBy(ctx, "mall:prod:rank:sales:2026-11", 1, "1001")
top, _ := rdb.ZRevRangeWithScores(ctx, "mall:prod:rank:sales:2026-11", 0, 9).Result()
```
ZSet 同时适合实现**延迟队列**：score 存到期时间戳，消费者轮询 `ZRANGEBYSCORE key 0 now LIMIT 0 100`，取出后 `ZREM`。注意 `ZREM` 与业务处理之间仍可能丢任务，需要兜底扫表。
## 六、Bitmap 与 HyperLogLog：省内存的统计
```go
// 用户签到：offset 用「一年中的第几天」，1 表示签到
day := time.Now().YearDay()
key := fmt.Sprintf("mall:user:sign:%d:%d", uid, time.Now().Year())
rdb.SetBit(ctx, key, int64(day), 1)
count, _ := rdb.BitCount(ctx, key, nil).Result() // 今年签到天数
```
Bitmap 的 offset 上限是 2^32-1，按用户+年分 key，单个 key 一年只占约 46 字节（365 位）。
```go
// 商品详情页 UV：允许约 0.81% 误差，1 万 UV 只用几 KB
rdb.PFAdd(ctx, "mall:prod:uv:1001:20261123", uid)
uv, _ := rdb.PFCount(ctx, "mall:prod:uv:1001:20261123").Result()
```
HyperLogLog 不能取回明细、不能精确计数，适合展示型指标；涉及钱的统计一律用 DB。
## 七、Stream：轻量事件流
```go
// 生产事件，限制长度避免无限增长
rdb.XAdd(ctx, &redis.XAddArgs{
	Stream: "mall:prod:stock-events",
	MaxLen: 10000,
	Approx: true,
	Values: map[string]any{"sku_id": 1001, "delta": -1},
})
// 消费组读取
msgs, _ := rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
	Group:    "stock-consumer",
	Consumer: "pod-1",
	Streams:  []string{"mall:prod:stock-events", ">"},
	Count:    10,
	Block:    time.Second,
}).Result()
```
Stream 支持消费组与 ACK，能力接近轻量 MQ；但持久化依赖 Redis，容量与堆积能力远不如 Kafka。**订单事件仍走 Kafka，Stream 只用于低价值、短生命周期的内部事件**。
## 八、大 Key：判定、危害、拆分
判定参考（不是绝对标准，结合业务）：
| 类型 | 大 Key 阈值 |
| --- | --- |
| String | value > 10KB |
| Hash/Set/ZSet | 元素数 > 5000，或总内存 > 1MB |
| List | 长度 > 5000 |
```bash
redis-cli --bigkeys
redis-cli MEMORY USAGE mall:order:cart:10001 SAMPLES 0
```
```powershell
redis-cli --bigkeys
redis-cli MEMORY USAGE mall:order:cart:10001 SAMPLES 0
```
危害：单命令阻塞主线程（`HGETALL`/`DEL` 大集合）、网络拥塞、Cluster 下数据倾斜、主从同步延迟。
拆分手法：
- 大 Hash 按字段哈希分片：`mall:order:cart:10001:0` ~ `:9`
- 大 List 按时间分桶：`mall:notify:queue:20261123`
- 删除大 Key 用 `UNLINK`（后台释放）而不是 `DEL`
## 九、热 Key：探测与治理
探测手段：
1. `redis-cli --hotkeys`（需 `maxmemory-policy` 为 LFU，`redis-cli CONFIG SET maxmemory-policy allkeys-lfu`）
2. `redis-cli MONITOR` 抽样统计（仅短时使用，开销大）
3. 客户端埋点：在 repository 层统计 key 的 QPS，TOP N 上报
治理四板斧：
| 手段 | 做法 | 适用 |
| --- | --- | --- |
| 本地缓存 | 进程内 LRU/ristretto 缓存热点值，几秒过期 | 读多写少、容忍短暂不一致 |
| Key 打散 | `product:detail:1001:0..9`，随机后缀读，写入时写 10 份 | 极端热点、读 QPS 百万级 |
| 多副本读 | 主从分离，热点读走从库 | 读多写少 |
| 限流降级 | 热点接口单独限流，返回兜底内容 | 大促洪峰 |
打散会带来一致性与存储成本，先用本地缓存 + singleflight，不够再打散。
## 十、禁用 keys *，改用 scan
```go
func ScanProductKeys(ctx context.Context, rdb redis.UniversalClient, pattern string) ([]string, error) {
	var (
		cursor uint64
		out    []string
	)
	for {
		keys, next, err := rdb.Scan(ctx, cursor, pattern, 200).Result()
		if err != nil {
			return nil, err
		}
		out = append(out, keys...)
		cursor = next
		if cursor == 0 {
			return out, nil
		}
	}
}
```
`SCAN` 只保证「遍历期间存在的元素至少返回一次」，可能重复，业务侧要能容忍；`HSCAN`/`SSCAN`/`ZSCAN` 同理。Cluster 下需要遍历每个节点分别 SCAN。
## 十一、Pipeline 减少 RTT
```go
pipe := rdb.Pipeline()
for _, sku := range skus {
	pipe.HGet(ctx, "mall:prod:product:stock:"+sku, "available")
}
cmds, err := pipe.Exec(ctx)
```
一次网络往返处理 N 个命令，适合批量校验库存、批量回填缓存。注意单次 Pipeline 命令数控制在 500~1000，过大同样会阻塞 Redis 且占用客户端内存。
## 十二、内存淘汰与 maxmemory
```bash
redis-cli CONFIG SET maxmemory 8gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG GET maxmemory-policy
```
```powershell
redis-cli CONFIG SET maxmemory 8gb
redis-cli CONFIG SET maxmemory-policy allkeys-lru
redis-cli CONFIG GET maxmemory-policy
```
| 策略 | 行为 | 适用 |
| --- | --- | --- |
| `noeviction` | 达到上限后写入报错 | 当数据库用（不推荐） |
| `allkeys-lru` | 所有 key 中淘汰最久未使用 | 纯缓存，最常用 |
| `allkeys-lfu` | 按访问频率淘汰 | 热点明显、需要 `--hotkeys` |
| `volatile-lru` | 只淘汰设了 TTL 的 key | 缓存与持久数据混存 |
| `volatile-ttl` | 优先淘汰快过期的 key | 短期缓存 |
务必设置 `maxmemory` 小于机器内存（留 20%~30% 给 fork 与碎片），否则触发 OOM Killer 或 swap，延迟飙升。
## 十三、Key 命名规范与多环境隔离
```text
{env}:{domain}:{object}:{id}[:{sub}]
mall:prod:product:detail:1001
mall:prod:order:cart:10001
mall:prod:stock:sku:1001
mall:test:product:detail:1001
```
约定：
- 分隔符统一 `:`，全小写，禁止空格与中文
- 必须带环境前缀，测试与生产共用实例时也不会串数据
- Cluster 下需要一起操作的 key 用相同 hash tag：`{sku:1001}:stock` 与 `{sku:1001}:reserved` 落在同一 slot
- value 用 JSON 时加版本字段，便于后续 schema 演进
## 常见坑
1. **用 `KEYS *` 做清理**：百万 key 时阻塞数秒，直接把接口 P99 打穿。
2. **`DEL` 大 Key**：同步释放内存阻塞主线程，应改用 `UNLINK`。
3. **把 List 当可靠队列**：`BRPOP` 取出即删，宕机丢消息、无重试无死信。
4. **计数器不设 TTL**：每日 UV、浏览量 key 无限增长，最终触发淘汰把真正的缓存挤掉。
5. **HyperLogLog 当精确统计**：误差 0.81%，对账时金额对不上。
6. **大 Hash 无脑 `HGETALL`**：单次返回几十 MB，网卡与 Redis 一起卡死。
7. **Cluster 下多 key 不带 hash tag**：`SINTER`、`MGET` 直接报 `CROSSSLOT`。
8. **热 Key 上同时设过期**：过期瞬间全部回源，热 Key 直接变成雪崩中心。
## 小结
- 选型看访问模式：整存整取用 String，字段更新用 Hash，排序用 ZSet，统计用 Bitmap/HLL
- List 不是可靠队列，业务消息交给 Kafka/RocketMQ
- 大 Key 用 `--bigkeys` 找、用分片拆、用 `UNLINK` 删
- 热 Key 先本地缓存 + singleflight，再考虑打散与多副本
- `SCAN` 代替 `KEYS`，Pipeline 控批量，`maxmemory` 必须设
## 练习
1. 为商品实现「浏览量 + 日榜 + UV」三件套，分别用 String、ZSet、HyperLogLog，并给定 TTL。
2. 构造一个 10 万字段的 Hash，用 `MEMORY USAGE` 与 `--bigkeys` 观察，然后按字段哈希拆成 10 个 key。
3. 用 `redis-cli --hotkeys`（先切 LFU）压测某商品详情，找出热 Key 并给出治理方案。
4. 写一个 `ScanAndDelete(pattern)`，要求用 `SCAN` + `UNLINK`，并在 Cluster 模式下遍历所有节点。
## 官方文档
- Redis 数据类型总览：<https://redis.io/docs/latest/develop/data-types/>
- SCAN 命令：<https://redis.io/docs/latest/commands/scan/>
- 内存优化与淘汰：<https://redis.io/docs/latest/develop/reference/eviction/>
- Redis 大 Key 与热 Key 治理：<https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/>
