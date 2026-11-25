# 02-Kafka生产与消费实践
> 一句话定位：用 segmentio/kafka-go 把订单事件可靠地投进 Kafka，再把「先处理后提交」的消费循环写对。
## 本篇目标
- 理清 Topic/Partition/Replica/ConsumerGroup/Offset/ISR 的关系
- 写出 acks=all、可重试、带压缩的生产者，并知道幂等生产该用什么库
- 掌握手动提交 offset 的正确时机，理解 rebalance 带来的重复消费
- 设计分区数与消费并发，保证同一订单的事件有序
- 在 Hertz/Kitex 中用「本地消息表 + 异步投递」避免请求内阻塞
## 前置知识
- 已读 [01-消息队列选型对比](01-消息队列选型对比.md)
- 已读 [11-消息队列/04](04-可靠投递-幂等-顺序消息.md) 之后再回看本文效果更好
- 本地 Kafka：`docker run -d --name kafka -p 9092:9092 apache/kafka:3.9.0`
## 一、核心概念
| 概念 | 含义 | 实践要点 |
| --- | --- | --- |
| Topic | 逻辑消息分类 | 按业务域命名：`mall.order.event.v1` |
| Partition | Topic 的物理分片，唯一保证顺序的粒度 | 数量决定最大并行消费度 |
| Replica | 分区副本，Leader 读写、Follower 同步 | 生产至少 3 副本 |
| ISR | 与 Leader 保持同步的副本集合 | `acks=all` 写入所有 ISR 才算成功 |
| ConsumerGroup | 消费组，组内每个分区只分配给一个消费者 | 不同组互不影响，天然扇出 |
| Offset | 消费者在各分区的消费进度 | 提交时机决定「至少一次」是否成立 |
| Rebalance | 分区在组内重新分配 | 期间消费暂停，可能重复消费 |
## 二、生产者：可靠性与吞吐的平衡
```bash
go get github.com/segmentio/kafka-go
```
```powershell
go get github.com/segmentio/kafka-go
```
```go
type Producer struct {
	w *kafka.Writer
}

func NewProducer(brokers []string) *Producer {
	return &Producer{w: &kafka.Writer{
		Addr:         kafka.TCP(brokers...),
		Topic:        "mall.order.event.v1",
		Balancer:     &kafka.Hash{},          // 同 key 进同一分区
		RequiredAcks: kafka.RequireAll,       // 等价 acks=all
		MaxAttempts:  5,                      // 可重试错误自动重试
		BatchSize:    100,                    // 批量攒够 100 条发送
		BatchTimeout: 10 * time.Millisecond,  // 最多等 10ms
		Compression:  kafka.Snappy,           // 网络与磁盘双省
		Async:        false,                  // 同步等待结果，便于确认
	}}
}

func (p *Producer) SendOrderEvent(ctx context.Context, orderID string, body []byte) error {
	return p.w.WriteMessages(ctx, kafka.Message{
		Key:   []byte(orderID), // key 决定分区，必须稳定
		Value: body,
		Headers: []kafka.Header{
			{Key: "trace_id", Value: []byte(traceIDFrom(ctx))},
		},
	})
}
```
`segmentio/kafka-go` 没有暴露 `enable.idempotence` 配置项。需要幂等生产者时用 sarama：
```go
cfg := sarama.NewConfig()
cfg.Version = sarama.V3_6_0_0
cfg.Producer.RequiredAcks = sarama.WaitForAll
cfg.Producer.Idempotent = true      // 开启幂等，避免重试导致重复
cfg.Net.MaxOpenRequests = 1         // 幂等要求：在途请求只能为 1
cfg.Producer.Retry.Max = 5
cfg.Producer.Compression = sarama.CompressionSnappy
cfg.Producer.Partitioner = sarama.NewHashPartitioner
producer, err := sarama.NewSyncProducer(brokers, cfg)
```
参数取舍：
| 目标 | 手段 | 代价 |
| --- | --- | --- |
| 不丢 | `acks=all` + 幂等 + 足够副本 | 延迟上升 |
| 高吞吐 | 批量、压缩、异步 | 延迟上升、失败难感知 |
| 低延迟 | 批量小、`BatchTimeout` 短 | 吞吐下降 |
## 三、分区、顺序与 key
- 同一 key 经 Hash 分配器固定进同一分区，分区内有序 → 同订单事件有序
- 分区数决定消费并行度上限：12 分区最多 12 个消费者并行（同组）
- 分区数只能增加不能减少，增加会打乱已有 key 的路由，需评估
- key 为空时按轮询分布，同订单事件可能落到不同分区，顺序不保
推荐：订单事件用 `order_id` 做 key；用户行为日志可不设 key 换取均匀分布。
## 四、消费者组与 offset 提交
```go
func Consume(ctx context.Context, brokers []string) error {
	r := kafka.NewReader(kafka.ReaderConfig{
		Brokers:        brokers,
		GroupID:        "mall-stock-consumer",
		Topic:          "mall.order.event.v1",
		MinBytes:       1,
		MaxBytes:       10 << 20,
		MaxWait:        time.Second,
		CommitInterval: 0,                  // 0 = 关闭自动提交
		StartOffset:    kafka.FirstOffset,  // 新组从最早开始
		GroupBalancers: []kafka.GroupBalancer{
			kafka.RangeGroupBalancer{}, kafka.RoundRobinGroupBalancer{},
		},
	})
	defer r.Close()

	for {
		m, err := r.FetchMessage(ctx) // 拉取但不提交
		if err != nil {
			if errors.Is(err, context.Canceled) {
				return nil
			}
			return err
		}
		if err := handleOrderEvent(ctx, m); err != nil {
			log.Printf("handle failed, will retry: %v", err)
			continue // 不提交，下次重新拉取该 offset
		}
		if err := r.CommitMessages(ctx, m); err != nil {
			return err
		}
	}
}
```
提交时机的三种写法：
| 写法 | 结果 | 风险 |
| --- | --- | --- |
| 自动提交（`CommitInterval` 非 0 / `ReadMessage`） | 简单 | 业务未处理完就已提交，宕机丢消息 |
| 先处理再提交（本文） | 至少一次 | 处理成功、提交失败会重复消费 |
| 先提交后处理 | 至多一次 | 处理失败直接丢消息，不推荐 |
结论：**先处理后提交 + 消费端幂等**，这是生产默认。
## 五、rebalance 与并发消费
rebalance 触发条件：消费者加入/退出、订阅 Topic 变化、分区数变化、心跳超时。期间分区暂停消费，未提交的 offset 会被新消费者重新消费，因此重复不可避免。应对：
- 单条处理时间控制在 `MaxPollInterval` 内，避免被踢出组
- 长任务丢到本地队列或线程池，拉取线程只负责分发
- 消费逻辑幂等，重复执行无副作用
高并发消费模型（保持同 key 有序）：
```go
// 拉取串行，按 key 哈希分发到固定 worker，保证同 key 不被并发处理
workers := make([]chan kafka.Message, 8)
for i := range workers {
	workers[i] = make(chan kafka.Message, 256)
	go func(ch <-chan kafka.Message) {
		for m := range ch {
			if err := handleOrderEvent(context.Background(), m); err != nil {
				log.Printf("async handle: %v", err)
			}
		}
	}(workers[i])
}
idx := int(fnv32(key) % uint32(len(workers)))
workers[idx] <- m
```
注意这种方式下 offset 提交要等对应 worker 处理完成，通常改为「按分区顺序处理」或引入位点管理，简单项目建议一个分区一个 goroutine 串行消费。
## 六、与 Hertz/Kitex 集成：异步投递 + 本地消息表
不要在 HTTP/RPC 请求里同步等 Kafka 返回：Broker 抖动会直接拉高接口 P99，且「DB 提交成功、发消息失败」无法回滚。
```sql
CREATE TABLE outbox_event (
  id            BIGINT AUTO_INCREMENT PRIMARY KEY,
  event_id      CHAR(36)     NOT NULL UNIQUE,
  topic         VARCHAR(64)  NOT NULL,
  msg_key       VARCHAR(64)  NOT NULL,
  payload       JSON         NOT NULL,
  status        TINYINT      NOT NULL DEFAULT 0, -- 0 待投递 1 已投递
  retry_count   INT          NOT NULL DEFAULT 0,
  next_retry_at DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  created_at    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  KEY idx_status_next (status, next_retry_at)
);
```
```go
// 与业务写在同一个本地事务里：订单落库成功，事件一定入库
func CreateOrder(ctx context.Context, db *gorm.DB, o *Order, ev *OutboxEvent) error {
	return db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
		if err := tx.Create(o).Error; err != nil {
			return err
		}
		return tx.Create(ev).Error
	})
}
```
独立的 dispatcher 协程每 100ms 扫表投递：
```go
func (d *Dispatcher) run(ctx context.Context) {
	ticker := time.NewTicker(100 * time.Millisecond)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			if err := d.dispatch(ctx); err != nil {
				log.Printf("outbox dispatch: %v", err)
			}
		}
	}
}

func (d *Dispatcher) dispatch(ctx context.Context) error {
	var events []OutboxEvent
	if err := d.db.WithContext(ctx).
		Clauses(clause.Locking{Strength: "UPDATE", Options: "SKIP LOCKED"}).
		Where("status = 0 AND next_retry_at <= NOW()").
		Order("id").Limit(100).Find(&events).Error; err != nil {
		return err
	}
	for _, e := range events {
		if err := d.producer.WriteMessages(ctx, kafka.Message{
			Key: []byte(e.MsgKey), Value: []byte(e.Payload),
		}); err != nil {
			_ = d.db.Model(&OutboxEvent{}).Where("id = ?", e.ID).
				Updates(map[string]any{
					"retry_count":   gorm.Expr("retry_count + 1"),
					"next_retry_at": time.Now().Add(backoff(e.RetryCount)),
				}).Error
			continue
		}
		_ = d.db.Model(&OutboxEvent{}).Where("id = ?", e.ID).Update("status", 1).Error
	}
	return nil
}
```
投递语义是「至少一次」：Kafka 已写入但状态更新失败时会重复投递，消费端必须幂等（见 04 篇）。
## 常见坑
1. **用 `ReadMessage` 自动提交**：offset 在处理前就提交，进程崩溃直接丢消息。
2. **先提交后处理**：处理失败消息也没了，只能人工补数据。
3. **key 为空还指望有序**：同订单事件被轮询到不同分区，消费顺序全乱。
4. **分区数设 1**：消费只能单线程，堆积后无法通过加实例解决。
5. **消费者实例数大于分区数**：多出来的实例空闲，扩容无效还浪费资源。
6. **请求线程里同步发 Kafka**：Broker 抖动直接传导为接口超时；用 Outbox 异步投递。
7. **消费处理超过 `MaxPollInterval`**：被踢出组触发 rebalance，消息反复重投。
8. **忽略 `WriteMessages` 的错误**：批量发送部分失败时，必须看返回错误并落重试。
## 小结
- Kafka 的顺序只在分区内成立，key 决定分区，必须用业务主键
- 生产端 `acks=all` + 重试 + 压缩；需要幂等生产者用 sarama
- 消费端「先处理后提交」+ 幂等，才等于可用的至少一次
- rebalance 必然带来重复，无法消除，只能靠幂等吸收
- 请求链路用「本地消息表 + dispatcher」，不要在 HTTP/RPC 里同步等 MQ
## 练习
1. 用 kafka-go 写一个生产者，发送 10 万条订单事件，对比 `BatchSize=1` 与 `100`、`Snappy` 与不压缩的耗时。
2. 用同一 `order_id` 作为 key 发送 1000 条事件，验证它们落在同一分区且消费有序。
3. 在消费处理中 `sleep(2s)` 并 kill 进程，观察重启后重复消费的条数，给消费逻辑加幂等后再测一次。
4. 为 mall-lite 的 `order-service` 实现 Outbox 表与 dispatcher，验证订单落库失败时不会产生消息。
## 官方文档
- Kafka 官方文档：<https://kafka.apache.org/documentation/>
- kafka-go：<https://github.com/segmentio/kafka-go>
- sarama：<https://github.com/IBM/sarama>
- Kafka 消费者配置：<https://kafka.apache.org/documentation/#consumerconfigs>
