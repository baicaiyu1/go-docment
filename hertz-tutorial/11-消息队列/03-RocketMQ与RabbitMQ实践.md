# 03-RocketMQ与RabbitMQ实践
> 一句话定位：把 RocketMQ 的事务/顺序/延迟能力与 RabbitMQ 的路由/死信能力，落到 Go 代码与 mall-lite 的业务判断上。
## 本篇目标
- 掌握 RocketMQ 的 NameServer/Broker/Topic/Tag/Queue/ConsumerGroup 与四种消息类型
- 用 `rocketmq-client-go/v2` 写出普通、顺序、事务、延迟消息的生产与消费
- 掌握 RabbitMQ 的 Exchange 类型、ack、持久化、prefetch 与死信交换机
- 用 `amqp091-go` 写出可靠消费循环
- 能根据业务特征在两者之间做出选择
## 前置知识
- 已读 [01-消息队列选型对比](01-消息队列选型对比.md) 与 [02-Kafka生产与消费实践](02-Kafka生产与消费实践.md)
- 了解 MQ 的至少一次投递与幂等
- 本地 Docker 可用
## 一、RocketMQ 架构与概念
| 概念 | 作用 |
| --- | --- |
| NameServer | 无状态路由中心，客户端从这里获取 Topic 的队列分布 |
| Broker | 存储与转发消息，主从部署，支持同步/异步刷盘 |
| Topic | 逻辑分类，如 `mall.order.event.v1` |
| Tag | Topic 下的二级过滤，消费端可在 Broker 侧过滤 |
| Queue | Topic 的物理分片，顺序消息以队列为单位 |
| ConsumerGroup | 消费组，集群模式下组内分摊队列 |
```bash
docker run -d --name rmqnamesrv -p 9876:9876 apache/rocketmq:5.3.1 sh mqnamesrv
docker run -d --name rmqbroker --link rmqnamesrv -p 10911:10911 -p 10909:10909 apache/rocketmq:5.3.1 sh mqbroker -n rmqnamesrv:9876
```
```powershell
docker run -d --name rmqnamesrv -p 9876:9876 apache/rocketmq:5.3.1 sh mqnamesrv
docker run -d --name rmqbroker --link rmqnamesrv -p 10911:10911 -p 10909:10909 apache/rocketmq:5.3.1 sh mqbroker -n rmqnamesrv:9876
```
## 二、普通消息：订单事件投递
```bash
go get github.com/apache/rocketmq-client-go/v2
```
```powershell
go get github.com/apache/rocketmq-client-go/v2
```
```go
p, err := rocketmq.NewProducer(
	rocketmq.WithGroupName("mall-order-producer"),
	rocketmq.WithNameServer([]string{"127.0.0.1:9876"}),
)
if err != nil {
	return err
}
if err := p.Start(); err != nil {
	return err
}
defer p.Shutdown()
res, err := p.SendSync(context.Background(),
	&primitive.Message{Topic: "mall.order.event.v1", Body: body})
if err != nil {
	return err
}
log.Printf("msgID=%s status=%v", res.MsgID, res.Status)
```
消费端：
```go
c, err := rocketmq.NewPushConsumer(
	rocketmq.WithGroupName("mall-stock-consumer"),
	rocketmq.WithNameServer([]string{"127.0.0.1:9876"}),
	rocketmq.WithConsumerModel(rocketmq.Clustering),
)
if err != nil {
	return err
}
err = c.Subscribe("mall.order.event.v1", consumer.MessageSelector{
	Type:       consumer.TAG,
	Expression: "OrderPaid || OrderCancelled",
}, func(ctx context.Context, msgs ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
	for _, m := range msgs {
		if err := handleOrderEvent(ctx, m); err != nil {
			return consumer.ConsumeRetryLater, err // 交给 Broker 重试
		}
	}
	return consumer.ConsumeSuccess, nil
})
if err != nil {
	return err
}
return c.Start()
```
Tag 过滤发生在 Broker 端，能减少网络传输；复杂过滤条件留给消费端，不要用 Tag 表达组合逻辑。
## 三、顺序消息
RocketMQ 的顺序保证以 Queue 为单位：同一业务键通过 selector 固定到同一队列，消费端顺序消费同一队列。
```go
// 生产：把 orderID 作为 sharding key，底层选择器会把同 key 映射到同一队列
res, err := p.SendSyncWithShardingKey(context.Background(),
	&primitive.Message{Topic: "mall.order.event.v1", Body: body}, orderID)
```
消费端使用顺序消费模式，同一队列由单线程串行处理：
```go
c, _ := rocketmq.NewPushConsumer(
	rocketmq.WithGroupName("mall-order-seq-consumer"),
	rocketmq.WithNameServer([]string{"127.0.0.1:9876"}),
	rocketmq.WithConsumerModel(rocketmq.Clustering),
	rocketmq.WithConsumerOrder(true), // 顺序消费
)
```
顺序消费的代价：队列级串行，吞吐受限；一条消息阻塞会拖住整个队列，必须设置处理超时并保证不无限重试。
## 四、事务消息与延迟消息
**事务消息**解决「本地事务与发消息」的原子性：
```go
type OrderTxListener struct{ db *gorm.DB }

func (l *OrderTxListener) ExecuteLocalTransaction(msg *primitive.Message) primitive.LocalTransactionState {
	var ev OrderEvent
	if err := json.Unmarshal(msg.Body, &ev); err != nil {
		return primitive.RollbackMessageState
	}
	if err := l.db.Create(&Order{ID: ev.OrderID, Status: StatusPending}).Error; err != nil {
		return primitive.RollbackMessageState
	}
	return primitive.CommitMessageState
}
// Broker 回查：本地事务到底成功没有
func (l *OrderTxListener) CheckLocalTransaction(msg *primitive.MessageExt) primitive.LocalTransactionState {
	var ev OrderEvent
	_ = json.Unmarshal(msg.Body, &ev)
	var cnt int64
	l.db.Model(&Order{}).Where("id = ?", ev.OrderID).Count(&cnt)
	if cnt > 0 {
		return primitive.CommitMessageState
	}
	return primitive.RollbackMessageState
}
```
```go
tp, err := rocketmq.NewTransactionProducer(&OrderTxListener{db: db},
	rocketmq.WithGroupName("mall-order-tx"),
	rocketmq.WithNameServer([]string{"127.0.0.1:9876"}),
)
if err != nil {
	return err
}
_ = tp.Start()
defer tp.Shutdown()
_, err = tp.SendMessageInTransaction(context.Background(),
	primitive.NewMessage("mall.order.event.v1", body))
```
**延迟消息**用于订单超时取消：
```go
msg := primitive.NewMessage("mall.order.delay.v1", body)
msg.WithDelayTimeLevel(16) // RocketMQ 预置延迟级别（4.x 为 1s~2h 共 18 级）
_, err := p.SendSync(ctx, msg)
```
延迟级别是固定档位，不能精确到「30 分钟零 5 秒」；业务侧仍需在消费时校验订单状态，避免「已支付却被取消」。
## 五、RabbitMQ：Exchange 与路由
| Exchange 类型 | 路由规则 | 典型用途 |
| --- | --- | --- |
| direct | routing key 完全匹配 | 按事件类型分发 |
| topic | `*` 匹配一个词、`#` 匹配多个词 | `order.*.created` 灵活订阅 |
| fanout | 忽略 routing key，广播到所有队列 | 配置变更通知 |
| headers | 按消息头匹配 | 复杂条件路由（少用） |

本地启动：`docker run -d --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:4-management`。
```go
conn, err := amqp.Dial("amqp://guest:guest@127.0.0.1:5672/")
if err != nil {
	return err
}
defer conn.Close()
ch, err := conn.Channel()
if err != nil {
	return err
}
defer ch.Close()
// 声明死信交换机（队列与绑定略）
_ = ch.ExchangeDeclare("mall.order.dlx", "topic", true, false, false, false, nil)
// 业务交换机与队列，绑定死信
_ = ch.ExchangeDeclare("mall.order.exchange", "topic", true, false, false, false, nil)
q, err := ch.QueueDeclare("mall.notify.paid", true, false, false, false, amqp.Table{
	"x-dead-letter-exchange":    "mall.order.dlx",
	"x-dead-letter-routing-key": "order.dead",
})
if err != nil {
	return err
}
if err := ch.QueueBind(q.Name, "order.paid.*", "mall.order.exchange", false, nil); err != nil {
	return err
}
if err := ch.Qos(20, 0, false); err != nil { // prefetch=20，防止消费者被压垮
	return err
}
```
生产与消费：
```go
err = ch.PublishWithContext(ctx, "mall.order.exchange", "order.paid.created", false, false,
	amqp.Publishing{
		ContentType:  "application/json",
		DeliveryMode: amqp.Persistent, // 消息持久化
		MessageId:    eventID,
		Timestamp:    time.Now(),
		Body:         body,
	})
```
```go
msgs, err := ch.Consume(q.Name, "notify-worker", false, false, false, false, nil)
if err != nil {
	return err
}
for d := range msgs {
	if err := handleNotify(d.Body); err != nil {
		log.Printf("notify failed: %v", err)
		_ = d.Nack(false, false) // requeue=false，进入死信
		continue
	}
	_ = d.Ack(false) // 处理成功才确认
}
```
| 可靠性开关 | 作用 |
| --- | --- |
| `QueueDeclare` durable=true | 队列元数据持久化 |
| `amqp.Persistent` | 消息写盘 |
| 手动 ack + prefetch | 处理成功才确认，限制在途消息 |
| 死信交换机 | 失败消息集中处理，避免 requeue 死循环 |
| publisher confirm | 确认 Broker 已接收，生产端可靠 |
## 六、如何选择
| 业务特征 | 推荐 |
| --- | --- |
| 订单创建 + 本地事务 + 消息同时成功 | RocketMQ 事务消息 |
| 订单 30 分钟未支付取消 | RocketMQ 延迟级别（或定时扫表） |
| 同一订单状态严格有序 | RocketMQ 顺序消息 / Kafka 同 key 分区 |
| 一条消息按规则路由到多个下游 | RabbitMQ topic/fanout |
务实建议：交易主链路选 RocketMQ；通知、任务、运营后台选 RabbitMQ；已有 Kafka 的团队可用 Outbox 补齐事务能力，避免三套 MQ 并存。
## 常见坑
1. **顺序消息用并发消费**：`WithConsumerOrder(false)` 或多线程处理同一队列，顺序保证失效。
2. **RabbitMQ 自动 ack**：消费者崩溃时消息已确认，直接丢失。
3. **队列 durable 但消息非 persistent**：Broker 重启后队列还在、消息没了。
4. **不设 prefetch**：消费者一次被推几千条，内存暴涨且其他消费者空闲。
5. **事务消息不实现回查**：Broker 回查失败，半消息长期悬挂。
## 小结
- RocketMQ 的强项是事务、顺序、延迟三类业务消息，代价是队列级串行与固定延迟档位
- RabbitMQ 的强项是 Exchange 路由、死信与低延迟任务分发，弱项是堆积能力
- 可靠消费的统一公式：手动 ack + 持久化 + prefetch + 死信 + 幂等
## 练习
1. 用 RocketMQ 发送一条延迟消息，验证 10 秒后可被消费，并说明如何避免「已支付订单被取消」。
2. 用 RocketMQ 顺序生产者与顺序消费者处理同一 orderID 的 100 条状态变更，验证顺序。
3. 用 RabbitMQ 搭建 topic Exchange + 两个队列（`order.paid.*` 与 `order.#`），验证路由结果。
4. 构造一条必然失败的消息，验证它进入死信队列而不是无限 requeue。
## 官方文档
- RocketMQ 官方文档：<https://rocketmq.apache.org/zh/docs/>
- rocketmq-client-go：<https://github.com/apache/rocketmq-client-go>
- RabbitMQ 官方文档：<https://www.rabbitmq.com/docs>
- amqp091-go：<https://github.com/rabbitmq/amqp091-go>
