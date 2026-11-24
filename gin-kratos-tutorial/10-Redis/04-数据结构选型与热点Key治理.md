# 04-数据结构选型与热点 Key 治理

> 一句话定位：选对类型只是及格线；能识别并治理大 Key、热 Key，才决定线上是否稳定。

## 本篇目标

- 掌握 String/Hash/List/Set/ZSet/Bitmap/HyperLogLog/Stream 在 mall-lite 中的真实用法
- 按访问模式选型，并能估算内存、反推 `maxmemory`
- 能判定大 Key 并拆分，用 `UNLINK` 代替 `DEL`
- 掌握热 Key 的探测与治理，用 `SCAN` 替代 `KEYS *`，规范 Key 与序列化

## 前置知识

- 已读 [02-缓存模式与三大经典问题](02-缓存模式与三大经典问题.md)
- 了解 Redis 单线程命令执行模型与 `MEMORY USAGE`，会用 `redis-cli -c` 连接 Cluster

## 一、选型总表

代码都放在 Kratos data 层的 repo 里，接口定义在 biz，`ctx` 透传。

| 类型 | 底层结构 | mall-lite 场景 | 典型命令 |
| --- | --- | --- | --- |
| String | SDS | 商品详情 JSON、计数器、锁 | `SET`/`GET`/`INCR` |
| Hash | listpack / hashtable | 购物车、对象局部更新 | `HSET`/`HINCRBY`/`HGETALL` |
| List | quicklist | 简单队列、时间线 | `LPUSH`/`BRPOP`/`LRANGE` |
| Set | intset / hashtable | 商品标签、去重、共同关注 | `SADD`/`SISMEMBER`/`SINTER` |
| ZSet | listpack / skiplist | 销量榜、延迟队列 | `ZADD`/`ZINCRBY`/`ZREVRANGE` |
| Bitmap | String 位图 | 签到、活跃标记 | `SETBIT`/`BITCOUNT` |
| HyperLogLog | 稀疏/稠密寄存器 | UV 统计 | `PFADD`/`PFCOUNT` |
| Stream | rax | 轻量事件流、消费组 | `XADD`/`XREADGROUP` |

## 二、内存估算

```text
单 key 开销 ≈ 50~100B（dict entry + redisObject + SDS 头）+ value
总内存 ≈ Σ(数据量 × 单条) × 1.2（碎片） × (1 + 副本开销)
```

| 数据 | 规模 | 单条估算 | 合计 |
| --- | --- | --- | --- |
| 商品详情 JSON | 100 万 | 2KB | 2GB |
| 购物车 Hash | 50 万用户 × 20 字段 | 约 1.5KB | 0.75GB |
| 销量榜 ZSet | 100 万成员 | 约 80B | 80MB |
| 日 UV HyperLogLog | 10 万个 key | 最多 12KB | 1.2GB（必须带 TTL） |

`maxmemory` ≈ 物理内存 70%，并小于容器 limit。HyperLogLog 单个 key 固定最多 12KB，量大时必须按天过期，否则比业务数据更占内存。

## 三、String：计数器与缓存值

```go
func (r *productRepo) IncrViews(ctx context.Context, id int64) (int64, error) {
    key := fmt.Sprintf("mall:prod:views:%d:%s", id, time.Now().Format("20060102"))
    pipe := r.data.rdb.Pipeline()
    incr := pipe.Incr(ctx, key)
    pipe.Expire(ctx, key, 48*time.Hour)
    if _, err := pipe.Exec(ctx); err != nil {
        return 0, err
    }
    return incr.Val(), nil
}
```

计数器最常见的错误是只 `INCR` 不设过期，日积月累产生海量永久 key，最终触发淘汰把真正的缓存挤掉。`INCR` + `EXPIRE` 用 Pipeline 发即可，重复设 TTL 无害；需要严格原子再用 Lua。

## 四、Hash：购物车与局部更新

```go
func (r *cartRepo) AddItem(ctx context.Context, uid string, skuID, n int64) error {
    key := "mall:order:cart:" + uid
    pipe := r.data.rdb.Pipeline()
    pipe.HIncrBy(ctx, key, strconv.FormatInt(skuID, 10), n)
    pipe.Expire(ctx, key, 7*24*time.Hour)
    _, err := pipe.Exec(ctx)
    return err
}
```

Hash 可以只更新某个字段，不必读改写整个 JSON。两条红线：单 Hash 字段数控制在 1000 以内，超过按 uid 分片或改成「每商品一个 key」；禁止对未知大小的 Hash 直接 `HGETALL`，用 `HMGET` 或 `HSCAN`。

## 五、List 与 Stream：都别当可靠 MQ

```go
_ = r.data.rdb.LPush(ctx, "mall:notify:sms:queue", payload).Err() // 生产
res, err := r.data.rdb.BRPop(ctx, 2*time.Second, "mall:notify:sms:queue").Result()
```

`BRPOP` 取出即删除，进程崩溃就丢消息，没有 ACK、重试与死信。业务消息用 Kafka（见 [11-消息队列/02-Kafka生产与消费实践](../11-消息队列/02-Kafka生产与消费实践.md)），List 只用于非关键任务；必须用时要 `BRPOPLPUSH` 备份队列实现「取出未确认」。Stream 有消费组与 ACK，适合低价值内部事件：

```go
_ = r.data.rdb.XGroupCreateMkStream(ctx, "mall:prod:stock-events", "stock-consumer", "0").Err()
_ = r.data.rdb.XAdd(ctx, &redis.XAddArgs{
    Stream: "mall:prod:stock-events", MaxLen: 10000, Approx: true,
    Values: map[string]any{"sku_id": 1001, "delta": -1},
}).Err()
res, err := r.data.rdb.XReadGroup(ctx, &redis.XReadGroupArgs{
    Group: "stock-consumer", Consumer: "pod-1", Count: 10, Block: 2 * time.Second,
    Streams: []string{"mall:prod:stock-events", ">"},
}).Result()
for _, s := range res {
    for _, m := range s.Messages {
        if handleStockEvent(ctx, m) == nil {
            _ = r.data.rdb.XAck(ctx, s.Stream, "stock-consumer", m.ID).Err()
        }
    }
}
```

Stream 能力接近轻量 MQ，但持久化依赖 Redis，堆积与容量远不如 Kafka，订单事件仍走 Kafka。

## 六、Set 与 ZSet：标签、排行榜、延迟队列

```go
_ = r.data.rdb.SAdd(ctx, "mall:prod:tags:1001", "数码", "热销").Err()
hot, _ := r.data.rdb.SIsMember(ctx, "mall:prod:tags:1001", "热销").Result()
common, _ := r.data.rdb.SInter(ctx, "mall:prod:tags:1001", "mall:prod:tags:1002").Result()
_ = r.data.rdb.ZIncrBy(ctx, "mall:prod:rank:sales:202611", 1, "1001").Err()
top, _ := r.data.rdb.ZRevRangeWithScores(ctx, "mall:prod:rank:sales:202611", 0, 9).Result()
```

ZSet 也适合**延迟队列**：score 存到期时间戳，消费者轮询 `ZRangeByScore(key, 0, now, Limit 0 100)`，取出后 `ZRem`。注意 `ZREM` 与业务处理之间仍可能丢任务，必须有兜底扫表。

## 七、Bitmap 与 HyperLogLog

```go
day := time.Now().YearDay()
key := fmt.Sprintf("mall:user:sign:%d:%d", uid, time.Now().Year())
_ = r.data.rdb.SetBit(ctx, key, int64(day), 1).Err()
count, _ := r.data.rdb.BitCount(ctx, key, nil).Result()

_ = r.data.rdb.PFAdd(ctx, "mall:prod:uv:1001:20261123", uid).Err()
uv, _ := r.data.rdb.PFCount(ctx, "mall:prod:uv:1001:20261123").Result()
```

Bitmap 的 offset 上限是 2^32-1，按年分 key 一年只占约 46 字节。HyperLogLog 有约 0.81% 误差、不能取明细，只适合展示型指标；涉及金额的统计一律走 DB。

## 八、大 Key：判定、发现、拆分

| 类型 | 大 Key 阈值 |
| --- | --- |
| String | value > 10KB |
| Hash/Set/ZSet | 元素数 > 5000 或总内存 > 1MB |
| List | 长度 > 5000 |

```bash
redis-cli --bigkeys
redis-cli MEMORY USAGE mall:order:cart:10001 SAMPLES 0
```

PowerShell 中命令相同。危害：单命令阻塞主线程（`HGETALL`/`DEL` 大集合）、网络拥塞、Cluster 数据倾斜、主从同步延迟。拆分：大 Hash 按字段哈希分片 `mall:order:cart:10001:0..9`；大 List 按时间分桶；大 Set 按成员哈希分片；删除统一用 `UNLINK`（后台释放），不要在代码或运维脚本里用 `DEL`。

## 九、热 Key：探测与治理

探测：`redis-cli --hotkeys`（需 `maxmemory-policy` 为 LFU）；`MONITOR` 短时抽样（开销大）；data 层埋点统计 key 的 QPS 并上报 TOP N。

| 手段 | 做法 | 适用 | 代价 |
| --- | --- | --- | --- |
| 本地缓存 | 进程内 ristretto/bigcache，1~3 秒过期 | 读多写少、可容忍短暂不一致 | 多副本一致性窗口 |
| Key 打散 | `detail:1001:0..9`，读随机后缀，写写 10 份 | 极端热点、百万 QPS | 存储 ×10、写放大 |
| 多副本读 | 热点读走从库 | 读多写少 | 可能读到旧值 |
| 限流降级 | 热点接口单独限流 + 兜底内容 | 大促洪峰 | 部分用户降级 |

先本地缓存 + singleflight，不够再考虑打散；打散会放大存储与写入，别默认上。

## 十、SCAN 与 Pipeline

```go
func (r *productRepo) ScanKeys(ctx context.Context, pattern string) ([]string, error) {
    var cursor uint64
    var out []string
    for {
        keys, next, err := r.data.rdb.Scan(ctx, cursor, pattern, 200).Result()
        if err != nil {
            return nil, err
        }
        out = append(out, keys...)
        if cursor = next; cursor == 0 {
            return out, nil
        }
    }
}
```

`SCAN` 只保证遍历期间存在的元素至少返回一次，可能重复，业务要能容忍；`HSCAN`/`SSCAN`/`ZSCAN` 同理。Cluster 下每个主节点要分别 SCAN（`ForEachMaster`）。批量命令用 `Pipeline()` 一次 RTT 发完，单批控制在 500~1000，避免阻塞 Redis 并占用客户端内存。

## 十一、TTL、淘汰与序列化

| 策略 | 行为 | 适用 |
| --- | --- | --- |
| `noeviction` | 达上限后写入报错 | 当数据库用，不推荐 |
| `allkeys-lru` | 淘汰最久未用 | 纯缓存，最常用 |
| `allkeys-lfu` | 按访问频率淘汰 | 热点明显，可配 `--hotkeys` |
| `volatile-lru` | 只淘汰有 TTL 的 key | 缓存与持久数据混存 |

序列化：小对象用 JSON（可读、跨语言）；大对象或高频读写用 MessagePack/Protobuf（体积小、编解码快）；不要用 `gob`。value 里加版本字段（如带 `json:"v"` 标签的 `V` 字段）便于 schema 演进，读取时先判断版本能否解析。

## 十二、Key 命名与 Cluster hash tag

格式统一为 `{env}:{domain}:{object}:{id}[:{sub}]`，例如 `mall:prod:product:detail:1001`、`mall:prod:cart:uid:10001`。分隔符统一 `:`、全小写、禁止空格与中文，必须带环境前缀。需要一起操作的 key 用相同 hash tag，如 `{sku:1001}:stock` 与 `{sku:1001}:reserved` 必在同一 slot。

## 常见坑

1. **用 `KEYS *` 做清理**：百万 key 时阻塞数秒，P99 直接打穿。
2. **`DEL` 大 Key**：同步释放内存阻塞主线程，改用 `UNLINK`。
3. **把 List 当可靠队列**：取出即删，宕机丢消息、无重试无死信。
4. **计数器不设 TTL**：浏览量、UV key 无限增长，把有效缓存挤掉。
5. **HyperLogLog 当精确统计**：0.81% 误差，对账金额对不上。
6. **大 Hash 无脑 `HGETALL`**：一次返回几十 MB，网卡与 Redis 一起卡死。
7. **Cluster 多 key 不带 hash tag**：`MGET`、`SINTER` 直接 `CROSSSLOT`。
8. **热 Key 同时设短 TTL**：过期瞬间全部回源，热 Key 变雪崩中心。

## 小结

- 选型看访问模式：整存整取 String，字段更新 Hash，排序 ZSet，统计 Bitmap/HLL
- List 不是可靠队列，业务消息交给 Kafka/Stream 按价值分级
- 大 Key 用 `--bigkeys` 找、分片拆、`UNLINK` 删；热 Key 先本地缓存 + singleflight
- `SCAN` 代替 `KEYS`，Pipeline 控批量，`maxmemory` 必须设

## 练习

1. 实现商品「浏览量 + 日榜 + UV」三件套，分别用 String、ZSet、HyperLogLog，并给出 TTL。
2. 构造 10 万字段的 Hash，用 `MEMORY USAGE` 与 `--bigkeys` 观察，再按字段哈希拆成 10 个 key。
3. 切 LFU 后用 `--hotkeys` 压测某商品详情，找出热 Key 并写治理方案。

## 官方文档

- Redis 数据类型：<https://redis.io/docs/latest/develop/data-types/>
- SCAN 命令：<https://redis.io/docs/latest/commands/scan/>
- 内存优化与淘汰：<https://redis.io/docs/latest/develop/reference/eviction/>
- 大 Key 与性能优化：<https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/>
