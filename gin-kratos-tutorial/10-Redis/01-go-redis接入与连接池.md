# 01-go-redis 接入与连接池

> 一句话定位：go-redis v9 的稳定性由 `Options` 里的池与超时参数决定；接入的第一步是在 Kratos data 层建唯一客户端、定清错误语义，而不是先写 `Get/Set`。

## 本篇目标

- 用 Viper 配置创建唯一的 `redis.UniversalClient`，并完成启动期健康检查
- 逐个理解连接池与超时参数，按 QPS 与 RT 算出 `PoolSize`
- 用 Pipeline / TxPipeline / Lua 降 RTT、保原子，用 zap Hook 记录慢命令

## 前置知识

- 已读 [09-MySQL与ORM/05-连接池与数据库性能调优](../09-MySQL与ORM/05-连接池与数据库性能调优.md)
- 已读 [05-治理与扩展/03-配置管理Viper](../05-治理与扩展/03-配置管理Viper.md)，本地有 Redis 7

## 一、依赖与配置

```bash
go get github.com/redis/go-redis/v9@v9.7.0
```

```powershell
go get github.com/redis/go-redis/v9@v9.7.0
```

```yaml
# configs/product-service.yaml，在教程基线 redis 段上补齐
redis:
  addr: "127.0.0.1:6379"
  password: ""
  db: 0
  pool_size: 50
  min_idle_conns: 10
```

其余 `pool_timeout`/`dial_timeout`/`read_timeout`/`write_timeout` 按第三节取值。结构体用 `mapstructure` tag 对应（Viper 的 `Unmarshal` 内置字符串转 `time.Duration` 钩子），密码只从 `MALL_REDIS_PASSWORD` 或 Secret 注入。

## 二、创建客户端

```go
// app/product-service/internal/data/redis.go
func NewRedisClient(cfg config.RedisConfig, logger *zap.Logger) (redis.UniversalClient, error) {
    rdb := redis.NewClient(&redis.Options{
        Addr:            cfg.Addr,
        Password:        cfg.Password,
        DB:              cfg.DB,
        PoolSize:        cfg.PoolSize,
        MinIdleConns:    cfg.MinIdleConns,
        PoolTimeout:     cfg.PoolTimeout,
        DialTimeout:     cfg.DialTimeout,
        ReadTimeout:     cfg.ReadTimeout,
        WriteTimeout:    cfg.WriteTimeout,
        ConnMaxIdleTime: 5 * time.Minute,
        ConnMaxLifetime: 30 * time.Minute,
        MaxRetries:      2,
    })
    rdb.AddHook(newSlowLogHook(logger, 20*time.Millisecond))

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    if err := rdb.Ping(ctx).Err(); err != nil {
        _ = rdb.Close()
        return nil, fmt.Errorf("redis ping: %w", err)
    }
    return rdb, nil
}
```

`NewClient` 不建连，第一条命令才拨号。启动时显式 `Ping`，地址错、密码错、网络不通都会暴露在启动期；失败返回 error，让 Kratos `app.Run()` 退出。

## 三、连接池参数

| 参数 | 含义 | 调大的后果 | 建议 |
| --- | --- | --- | --- |
| `PoolSize` | 单进程到该节点的最大连接数 | Redis 连接数与内存上涨 | `QPS×RT(s)×1.2`，单实例 30~100 |
| `MinIdleConns` | 常驻空闲连接 | 空闲占资源 | `PoolSize` 的 20% |
| `PoolTimeout` | 取不到连接最多等多久 | 故障时请求堆积 | 300~500ms |
| `DialTimeout` | 建连超时 | 故障感知慢 | 1s |
| `Read/WriteTimeout` | 单条命令读写超时 | 慢命令拖死调用方 | 200~500ms，小于上游超时 |
| `ConnMaxIdleTime` | 空闲多久关闭 | 连接长期占用 | 5 分钟 |
| `ConnMaxLifetime` | 连接最长寿命 | 不感知实例变更 | 30 分钟 |
| `MaxRetries` | 失败重试次数 | 故障放大、非幂等命令重复 | 1~2 |

默认 `PoolSize = 10 × GOMAXPROCS`，容器里按 CPU limit 算只是兜底值，仍要按 QPS 复核；重试对 `GET` 安全，对 `INCR` 会让计数翻倍，写命令幂等由业务保证。

## 四、错误语义与 context 超时

```go
// app/product-service/internal/data/product.go
func (r *productRepo) Detail(ctx context.Context, id int64) (*biz.Product, error) {
    key := fmt.Sprintf("mall:prod:product:detail:%d", id)
    raw, err := r.data.rdb.Get(ctx, key).Bytes()
    switch {
    case err == nil:
        return decodeProduct(raw)
    case errors.Is(err, redis.Nil):
        return r.loadFromDB(ctx, id) // 未命中：正常流程，不进错误率
    default:
        r.metrics.cacheError.WithLabelValues("get").Inc()
        return r.loadFromDB(ctx, id) // 故障降级直查 DB
    }
}
```

- key 不存在 → `redis.Nil`，业务未命中，绝不能当告警
- 连接失败 / 超时 / 重定向失败 → 真实故障，降级并计入错误率
- `WRONGTYPE` → 代码 bug，返回 `*redis.Error`，要修复而不是掩盖

判断统一用 `errors.Is(err, redis.Nil)`。命令要继承上游 `ctx`，需要更短预算时用 `context.WithTimeout(ctx, 200*time.Millisecond)`；禁止在请求链路用 `context.Background()`，否则超时与取消全部失效。

## 五、Pipeline、TxPipeline、Lua

| 方式 | 原子性 | 典型用途 |
| --- | --- | --- |
| `Pipeline()` | 不保证，仅批量发送 | 批量读、批量预热 |
| `TxPipeline()` | MULTI/EXEC 包裹 | 多条写需原子且无读依赖 |
| `redis.NewScript` | 单命令原子 | 读后写、条件扣减、限流 |

```go
func (r *productRepo) batchGet(ctx context.Context, keys []string) ([]string, error) {
    pipe := r.data.rdb.Pipeline()
    cmds := make([]*redis.StringCmd, 0, len(keys))
    for _, k := range keys {
        cmds = append(cmds, pipe.Get(ctx, k))
    }
    _, _ = pipe.Exec(ctx) // 部分 key 缺失以 redis.Nil 体现在单条 cmd 上
    out := make([]string, 0, len(cmds))
    for _, cmd := range cmds {
        if cmd.Err() == nil {
            out = append(out, cmd.Val())
        }
    }
    return out, nil
}
```

`TxPipeline` 的结果要等 `Exec` 后才可读，不能在事务内根据读结果决定写什么；有读依赖就用 Lua：

```go
var stockScript = redis.NewScript(`
local stock = redis.call('GET', KEYS[1])
if not stock then return -1 end
if tonumber(stock) < tonumber(ARGV[1]) then return 0 end
return redis.call('DECRBY', KEYS[1], ARGV[1])
`)

func deductStock(ctx context.Context, rdb redis.UniversalClient, skuID string, n int) (int64, error) {
    return stockScript.Run(ctx, rdb, []string{"mall:prod:stock:sku:" + skuID}, n).Int64()
}
```

`NewScript` 自动走 `EVALSHA`，遇到 `NOSCRIPT` 回退 `EVAL`。key 只走 `KEYS`、参数只走 `ARGV`；脚本短小，禁止遍历大集合；Cluster 下所有 `KEYS` 必须同 slot。

## 六、Kratos data 层封装与慢命令监控

接口 `ProductCache`（`GetDetail(ctx, id)` / `SetDetail(ctx, p, ttl)`）定义在 biz，`productCache` 在 data 实现，所有方法首参都是 `ctx`；biz 不 import go-redis，data 也不向上层泄露 `*redis.Client`，禁止在 data 里用 `context.Background()`。

```go
// app/product-service/internal/data/hook.go
type slowLogHook struct {
    logger    *zap.Logger
    threshold time.Duration
}

func (h *slowLogHook) DialHook(next redis.DialHook) redis.DialHook { return next }
func (h *slowLogHook) ProcessPipelineHook(n redis.ProcessPipelineHook) redis.ProcessPipelineHook { return n }

func (h *slowLogHook) ProcessHook(next redis.ProcessHook) redis.ProcessHook {
    return func(ctx context.Context, cmd redis.Cmder) error {
        start := time.Now()
        err := next(ctx, cmd)
        if d := time.Since(start); d > h.threshold {
            h.logger.Warn("redis slow command",
                zap.String("cmd", cmd.Name()), zap.Duration("latency", d), zap.Error(err))
        }
        return err
    }
}
```

`rdb.PoolStats()` 返回 `Hits/Misses/Timeouts/TotalConns/IdleConns/StaleConns`，每 10 秒用 zap 打一条或转成 Prometheus 指标；`Timeouts > 0` 就是明确告警（看板见 [08-部署与运维/05-可观测运维与告警](../08-部署与运维/05-可观测运维与告警.md)）。

## 完整示例

`app/product-service/internal/data/data.go`，全服务只建一个客户端，由 Wire 注入：

```go
func NewData(cfg config.RedisConfig, db *gorm.DB, log *zap.Logger) (*Data, func(), error) {
    rdb, err := NewRedisClient(cfg, log)
    if err != nil {
        return nil, nil, err
    }
    cleanup := func() { _ = rdb.Close() } // 先停流量，再由 Kratos app.Stop 关连接
    return &Data{rdb: rdb, db: db, log: log}, cleanup, nil
}

var ProviderSet = wire.NewSet(NewData, NewProductRepo, NewProductCache)
```

单机用 `redis.NewClient`，哨兵用 `redis.NewFailoverClient`，Cluster 用 `redis.NewClusterClient`；三者都实现 `redis.UniversalClient`，业务只依赖接口（细节见 [05-高可用部署与运维](05-高可用部署与运维.md)）。

## 常见坑

1. **把 `redis.Nil` 当异常上报**：未命中把错误率打到 100%，监控彻底失真。
2. **超时大于上游超时**：Redis 卡住时等不到降级，网关先超时，协程被拖满。
3. **`PoolSize` 拍脑袋设 10**：2000 QPS、5ms RT 理论并发是 10，突发与 Pipeline 至少留 2 倍余量。
4. **Cluster 上随手 `MGET`/事务**：报 `CROSSSLOT`，设计 key 时就要用 `{tag}`。
5. **Lua 里 `redis.call('KEYS', '*')`**：阻塞主线程，生产直接雪崩。
6. **每次请求 `redis.NewClient`**：连接泄漏 + 池失效，客户端必须进程内单例注入。

## 小结

- 真正决定稳定性的是 `PoolSize`/`PoolTimeout`/`ReadTimeout`，不是 `Addr`
- `redis.Nil` 是业务信号，其他错误才是故障信号，处理路径必须分开
- 批量读用 Pipeline，读改写用 Lua，Cluster 先想 slot
- 客户端启动期 Ping、请求期透传 ctx、关闭期 Close，连接池与慢命令必须进监控

## 练习

1. 实现 `BatchGet(ids []int64)`，100 个 id 只产生 1 次 RTT，并正确处理部分 key 缺失。
2. 用 `redis.NewScript` 写「读库存，够则扣减并返回剩余」，并发 1000 次验证不超卖。
3. 用 `PoolStats` 每 10 秒输出一条 zap 日志，压测时观察 `Timeouts` 并调整 `PoolSize`。

## 官方文档

- go-redis 指南：<https://redis.uptrace.dev/guide/>
- go-redis Options 源码：<https://github.com/redis/go-redis/blob/master/options.go>
- Redis 命令参考：<https://redis.io/docs/latest/commands/>
- 客户端连接与多路复用：<https://redis.io/docs/latest/develop/clients/pools-and-multiplexing/>
