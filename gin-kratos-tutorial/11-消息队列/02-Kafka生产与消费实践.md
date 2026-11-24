# 02-Kafka生产与消费实践

> 一句话定位：用 sarama 把订单事件可靠写进 Kafka，并把「先处理、后提交」的消费组循环写对。

## 本篇目标
- 理清 broker / topic / partition / replica / ISR / offset / consumer group 的关系
- 写出 acks=all、可重试、开启幂等与压缩的生产者，分清同步与异步
- 用 `sarama.NewConsumerGroup` + `ConsumeClaim` 写出正确的至少一次消费
- 理解 rebalance 为什么必然带来重复，并用 ctx 做优雅退出
- 消费者放进 Kratos 独立 worker，配置走 Viper、日志走 zap

## 前置知识
- 已读 [01-消息队列选型对比](01-消息队列选型对比.md)
- 已读 [04-可靠投递-幂等-顺序消息](04-可靠投递-幂等-顺序消息.md)
- 本地 Kafka 已启动（见 01 篇 compose），地址 `127.0.0.1:9092`

## 一、核心概念
| 概念 | 含义 | 实践要点 |
| --- | --- | --- |
| Broker | 单个 Kafka 节点 | 生产至少 3 个，避免单点 |
| Topic | 逻辑消息分类 | 按业务域命名：`mall.order.event.v1` |
| Partition | 物理分片，唯一保证顺序的粒度 | 数量决定最大并行消费度 |
| Replica | 分区副本，Leader 读写、Follower 同步 | 生产环境 3 副本 |
| ISR | 与 Leader 保持同步的副本集合 | `acks=all` 表示写满 ISR 才算成功 |
| Offset | 消费者在分区内的进度 | 提交时机决定投递语义 |
| ConsumerGroup | 消费组，组内一个分区只给一个消费者 | 组间互不影响，天然扇出 |

## 二、配置基线
```go
// internal/data/kafka_config.go
func NewKafkaConfig(cfg *conf.Kafka) (*sarama.Config, error) {
    c := sarama.NewConfig()
    c.Version = sarama.V3_6_0_0
    c.ClientID = cfg.ClientID
    c.Producer.RequiredAcks = sarama.WaitForAll
    c.Producer.Retry.Max = 5
    c.Producer.Retry.Backoff = 200 * time.Millisecond
    c.Producer.Return.Successes = true // 同步生产者必需
    c.Producer.Compression = sarama.CompressionSnappy
    c.Producer.Partitioner = sarama.NewHashPartitioner
    c.Producer.Idempotent = true
    c.Net.MaxOpenRequests = 1 // 幂等的硬性要求：在途请求只能为 1
    c.Consumer.Group.Rebalance.GroupStrategies = []sarama.BalanceStrategy{sarama.NewBalanceStrategyRange()}
    c.Consumer.Offsets.Initial = sarama.OffsetOldest
    c.Consumer.Group.Session.Timeout = 10 * time.Second
    c.Consumer.Group.Heartbeat.Interval = 3 * time.Second
    c.Consumer.MaxProcessingTime = 500 * time.Millisecond
    c.Consumer.Offsets.AutoCommit.Interval = time.Second
    return c, c.Validate()
}
```

## 三、生产者：同步与异步
```go
// internal/data/kafka_producer.go
type kafkaPublisher struct {
    p     sarama.SyncProducer
    topic string
    log   *zap.Logger
}

func NewKafkaPublisher(brokers []string, cfg *sarama.Config, topic string, log *zap.Logger) (biz.OrderEventPublisher, error) {
    p, err := sarama.NewSyncProducer(brokers, cfg)
    if err != nil {
        return nil, fmt.Errorf("new sync producer: %w", err)
    }
    return &kafkaPublisher{p: p, topic: topic, log: log}, nil
}

func (k *kafkaPublisher) Publish(ctx context.Context, ev biz.OrderEvent) error {
    body, err := json.Marshal(ev)
    if err != nil {
        return err
    }
    partition, offset, err := k.p.SendMessage(&sarama.ProducerMessage{
        Topic: k.topic,
        Key:   sarama.StringEncoder(ev.OrderID), // 分区键：同订单必落同一分区
        Value: sarama.ByteEncoder(body),
        Headers: []sarama.RecordHeader{
            {Key: []byte("trace_id"), Value: []byte(traceIDFrom(ctx))},
        },
    })
    if err != nil {
        return fmt.Errorf("send order event: %w", err)
    }
    k.log.Info("published", zap.String("order_id", ev.OrderID),
        zap.Int32("partition", partition), zap.Int64("offset", offset))
    return nil
}
```
异步生产者吞吐更高，但 `Successes()` 与 `Errors()` 两个 channel 都必须消费，否则发送侧阻塞：

```go
// internal/data/kafka_async.go
func NewAsyncPublisher(brokers []string, cfg *sarama.Config, log *zap.Logger) (sarama.AsyncProducer, error) {
    cfg.Producer.Return.Successes = true
    p, err := sarama.NewAsyncProducer(brokers, cfg)
    if err != nil {
        return nil, err
    }
    go func() {
        for {
            select {
            case m, ok := <-p.Successes():
                if !ok {
                    return
                }
                log.Debug("published", zap.Int64("offset", m.Offset))
            case e, ok := <-p.Errors():
                if !ok {
                    return
                }
                log.Error("publish failed", zap.String("topic", e.Msg.Topic), zap.Error(e.Err))
            }
        }
    }
    return p, nil
}
```

## 四、消费组
```go
// internal/worker/order_consumer.go
type orderHandler struct {
    repo biz.OrderRepo
    log  *zap.Logger
}

func (h *orderHandler) Setup(sarama.ConsumerGroupSession) error   { return nil }
func (h *orderHandler) Cleanup(sarama.ConsumerGroupSession) error { return nil }

func (h *orderHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
    for msg := range claim.Messages() {
        var ev biz.OrderEvent
        if err := json.Unmarshal(msg.Value, &ev); err != nil {
            h.log.Error("bad message, skip", zap.Int64("offset", msg.Offset), zap.Error(err))
            sess.MarkMessage(msg, "") // 毒消息不能卡住分区
            continue
        }
        if err := h.repo.HandleOrderEvent(sess.Context(), ev); err != nil {
            h.log.Error("handle failed", zap.String("order_id", ev.OrderID), zap.Error(err))
            return err // 结束本次 generation，下一次 Consume 重试
        }
        sess.MarkMessage(msg, "") // 处理成功才标记，由后台按周期提交
    }
    return nil
}
```
`MarkMessage` 交给 autocommit 周期落位点，要更强语义就在批量末尾显式 `sess.Commit()`；rebalance 时 `Setup`/`Cleanup` 各执行一次，handler 状态必须可重置。

## 五、消费循环与优雅退出
```go
type OrderConsumer struct {
    cg      sarama.ConsumerGroup
    topics  []string
    handler *orderHandler
    cancel  context.CancelFunc
}

func (w *OrderConsumer) Start(ctx context.Context) error {
    ctx, w.cancel = context.WithCancel(ctx)
    for ctx.Err() == nil {
        if err := w.cg.Consume(ctx, w.topics, w.handler); err != nil &&
            !errors.Is(err, sarama.ErrClosedConsumerGroup) {
            return err
        }
    }
    return nil
}

func (w *OrderConsumer) Stop(ctx context.Context) error {
    w.cancel()
    return w.cg.Close() // 触发 Cleanup 并提交已标记位点
}
```
`Start` / `Stop` 正是 Kratos `transport.Server` 的接口。ctx 由 Kratos 在 SIGTERM 时 cancel，禁止用 `context.Background()` 起消费协程。

## 六、调优与监控
优先调四个参数：`RequiredAcks=WaitForAll`（不丢）、`Flush.Frequency=5~50ms`（攒批）、`MaxProcessingTime` 必须小于 `Session.Timeout`、`AutoCommit.Interval=1s`（崩溃最多重放该窗口）。监控四件套是消费速率、处理耗时 P99、失败率、lag：前三项用 `promauto` 埋 Counter/Histogram，lag 交给 `kafka_exporter`，人工核对用 `kafka-consumer-groups.sh --describe --group order-service`，告警规则见 05 篇。

## 完整示例：order-worker 启动
```go
// cmd/order-worker/main.go
func main() {
    logger := zap.Must(zap.NewProduction())
    cfg := conf.MustLoad("configs/order-service.yaml") // Viper 封装
    kcfg, err := data.NewKafkaConfig(cfg.Kafka)
    if err != nil {
        logger.Fatal("bad kafka config", zap.Error(err))
    }
    cg, err := sarama.NewConsumerGroup(cfg.Kafka.Brokers, cfg.Kafka.Group, kcfg)
    if err != nil {
        logger.Fatal("new consumer group", zap.Error(err))
    }
    app := kratos.New(
        kratos.Name("order-worker"),
        kratos.Logger(zapLog.NewLogger(logger)), // contrib/log/zap/v2
        kratos.Server(&worker.OrderConsumer{Cg: cg, Topics: []string{cfg.Kafka.Topic}}),
    )
    if err := app.Run(); err != nil {
        logger.Fatal("worker exit", zap.Error(err))
    }
}
```

## 常见坑
1. **`NewSyncProducer` 没设 `Return.Successes`**：直接返回配置错误，误以为 broker 挂了。
2. **开幂等但没把 `Net.MaxOpenRequests` 设成 1**：`Validate()` 失败。
3. **key 为空还指望有序**：消息被轮询到不同分区，同订单顺序全乱。
4. **消费者实例数大于分区数**：多出的实例空闲，扩容无效。
5. **处理耗时超过 `MaxProcessingTime`**：被踢出组触发 rebalance，消息反复重投。
6. **返回错误后立刻重试同一批**：热循环打满 CPU；应先退避或转死信。

## 小结
- 顺序只在分区内成立，key 必须用业务主键（如 `order_id`）。
- 生产端 `acks=all` + 幂等 + 压缩；关键事件用同步发送拿到 offset。
- 消费端「先处理、后 `MarkMessage`」+ 幂等，才是可用的至少一次。
- rebalance 必然带来重复，只能靠幂等吸收；用 Kratos 管理生命周期，ctx 取消即退出。

## 练习
1. 发送 10 万条订单事件，对比 `Flush.Frequency` 为 0 与 50ms 的耗时。
2. 用同一 `order_id` 作 key 发 1000 条事件，验证同分区且有序。

## 官方文档
- Kafka 官方文档：<https://kafka.apache.org/documentation/>
- sarama：<https://github.com/IBM/sarama>
- 生产者配置：<https://kafka.apache.org/documentation/#producerconfigs>
- 消费者配置：<https://kafka.apache.org/documentation/#consumerconfigs>
- Kratos 传输层：<https://go-kratos.dev/docs/component/transport>
