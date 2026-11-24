# 03-RocketMQ与RabbitMQ实践

> 一句话定位：RocketMQ 用事务 / 顺序 / 延迟解决交易一致性，RabbitMQ 用 Exchange 解决复杂路由；两者都不能替代消费端幂等。

## 本篇目标
- 掌握 RocketMQ 5.x 的 NameServer / Broker / Topic / Tag / Queue / ConsumerGroup
- 用 `rocketmq-client-go/v2` 写出普通、Tag 过滤、顺序、事务、延迟消息
- 掌握 RabbitMQ 的四种 Exchange、publisher confirm、手动 ack、QoS 与死信
- 用 `amqp091-go` 写出可靠的生产与消费循环
- 能说清两者与 Kafka 的取舍，以及常见踩坑

## 前置知识
- 已读 [01-消息队列选型对比](01-消息队列选型对比.md) 与 [02-Kafka生产与消费实践](02-Kafka生产与消费实践.md)
- 已读 [04-可靠投递-幂等-顺序消息](04-可靠投递-幂等-顺序消息.md)
- 本地 Docker 可用，端口 `9876`、`5672`、`15672` 未被占用

## 一、RocketMQ 5.x 架构
| 概念 | 作用 |
| --- | --- |
| NameServer | 无状态路由中心，客户端从这里拿 Topic 的队列分布 |
| Broker | 存储与转发，主从部署，支持同步 / 异步刷盘 |
| Topic | 逻辑分类，如 `mall.order.event.v1` |
| Tag | Topic 内的二级过滤，在 Broker 侧生效，省网络 |
| Queue | Topic 的物理分片，顺序消息以队列为单位 |
| ConsumerGroup | 集群模式组内分摊队列；广播模式每人一份 |

## 二、普通消息与 Tag 过滤
```go
// internal/data/rocketmq_producer.go
func NewOrderProducer(nameSrv []string) (rocketmq.Producer, error) {
    p, err := rocketmq.NewProducer(
        producer.WithNameServer(nameSrv),
        producer.WithGroupName("mall-order-producer"),
        producer.WithRetry(2),
    )
    if err != nil {
        return nil, err
    }
    return p, p.Start()
}

func PublishPaid(ctx context.Context, p rocketmq.Producer, ev biz.OrderEvent) error {
    body, _ := json.Marshal(ev)
    msg := primitive.NewMessage("mall.order.event.v1", body)
    msg.WithTag("OrderPaid")
    msg.WithKeys([]string{ev.EventID}) // 业务唯一键，便于查询与去重
    res, err := p.SendSync(ctx, msg)
    if err != nil {
        return fmt.Errorf("send: %w", err)
    }
    if res.Status != primitive.SendOK {
        return fmt.Errorf("status %v", res.Status)
    }
    return nil
}
```
```go
// internal/worker/rocketmq_consumer.go
func NewOrderConsumer(nameSrv []string, h func(context.Context, *primitive.MessageExt) error) (rocketmq.PushConsumer, error) {
    c, err := rocketmq.NewPushConsumer(
        consumer.WithNameServer(nameSrv),
        consumer.WithGroupName("mall-stock-consumer"),
        consumer.WithConsumerModel(consumer.Clustering),
    )
    if err != nil {
        return nil, err
    }
    err = c.Subscribe("mall.order.event.v1", consumer.MessageSelector{
        Type:       consumer.TAG,
        Expression: "OrderPaid || OrderCancelled",
    }, func(ctx context.Context, msgs ...*primitive.MessageExt) (consumer.ConsumeResult, error) {
        for _, m := range msgs {
            if err := h(ctx, m); err != nil {
                return consumer.ConsumeRetryLater, err // 交给 Broker 重试
            }
        }
        return consumer.ConsumeSuccess, nil
    })
    if err != nil {
        return nil, err
    }
    return c, c.Start()
}
```

## 三、顺序消息
```go
msg := primitive.NewMessage("mall.order.event.v1", body)
msg.WithShardingKey(orderID) // 同订单进同一队列
_, err := p.SendSync(ctx, msg)
// 消费端：创建时加 consumer.WithConsumerOrder(true)，同队列单线程串行
```
队列级串行：一条消息卡住就拖住整个队列，必须给下游设超时并限制重试。

## 四、事务消息与延迟消息
事务消息保证「本地事务与发消息」的原子性：先投半消息，本地事务成功则提交、失败回滚，Broker 长期未确认会回查。

```go
type orderTxListener struct{ repo biz.OrderRepo }

func (l *orderTxListener) ExecuteLocalTransaction(msg *primitive.Message) primitive.LocalTransactionState {
    var ev biz.OrderEvent
    if err := json.Unmarshal(msg.Body, &ev); err != nil || l.repo.CreateOrder(traceCtx(), ev) != nil {
        return primitive.RollbackMessageState
    }
    return primitive.CommitMessageState
}

func (l *orderTxListener) CheckLocalTransaction(msg *primitive.MessageExt) primitive.LocalTransactionState {
    var ev biz.OrderEvent
    _ = json.Unmarshal(msg.Body, &ev)
    if l.repo.OrderExists(traceCtx(), ev.OrderID) {
        return primitive.CommitMessageState
    }
    return primitive.RollbackMessageState
}

tp, _ := rocketmq.NewTransactionProducer(&orderTxListener{repo: repo},
    producer.WithNameServer(nameSrv), producer.WithGroupName("mall-order-tx"))
_ = tp.Start()
defer tp.Shutdown()
_, err := tp.SendMessageInTransaction(ctx, primitive.NewMessage("mall.order.event.v1", body))
```
```go
msg := primitive.NewMessage("mall.order.delay.v1", body)
msg.WithDelayTimeLevel(16) // 兼容档位 18 级，1s ~ 2h
_, err := p.SendSync(ctx, msg)
```
5.x 也支持任意时间的定时消息（Broker 时间轮），Go 客户端可用 `msg.WithProperty("__STARTDELIVERTIME", strconv.FormatInt(ts, 10))` 传毫秒时间戳，以 Broker 配置为准；无论哪种，消费时都要二次校验订单状态，避免「已支付却被取消」。

## 五、RabbitMQ：Exchange 与路由
| Exchange 类型 | 路由规则 | 典型用途 |
| --- | --- | --- |
| direct | routing key 完全匹配 | 按事件类型精确分发 |
| topic | `*` 匹配一个词，`#` 匹配多个词 | `order.*.created` 灵活订阅 |
| fanout | 忽略 routing key，广播所有队列 | 配置变更、缓存刷新 |
| headers | 按消息头匹配 | 复杂条件路由（少用） |

```go
// internal/data/rabbitmq.go
func Setup(ch *amqp.Channel) (amqp.Queue, error) {
    if err := ch.ExchangeDeclare("mall.order.exchange", amqp.ExchangeTopic, true, false, false, false, nil); err != nil {
        return amqp.Queue{}, err
    }
    _ = ch.ExchangeDeclare("mall.order.dlx", amqp.ExchangeTopic, true, false, false, false, nil)
    q, err := ch.QueueDeclare("mall.notify.paid", true, false, false, false, amqp.Table{
        "x-dead-letter-exchange":    "mall.order.dlx",
        "x-dead-letter-routing-key": "order.dead",
        "x-message-ttl":             int32(600000), // 10 分钟未消费进死信
    })
    if err != nil {
        return amqp.Queue{}, err
    }
    if err := ch.QueueBind(q.Name, "order.paid.*", "mall.order.exchange", false, nil); err != nil {
        return amqp.Queue{}, err
    }
    return q, ch.Qos(20, 0, false) // prefetch=20
}

func Publish(ctx context.Context, ch *amqp.Channel, eventID string, body []byte) error {
    confirm := ch.NotifyPublish(make(chan amqp.Confirmation, 1))
    if err := ch.PublishWithContext(ctx, "mall.order.exchange", "order.paid.created", false, false,
        amqp.Publishing{DeliveryMode: amqp.Persistent, MessageId: eventID, Body: body}); err != nil {
        return err
    }
    select {
    case c := <-confirm:
        if !c.Ack {
            return errors.New("broker nacked message")
        }
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func Consume(ch *amqp.Channel, q amqp.Queue, handle func([]byte) error, log *zap.Logger) error {
    msgs, err := ch.Consume(q.Name, "notify-worker", false, false, false, false, nil)
    if err != nil {
        return err
    }
    for d := range msgs {
        if err := handle(d.Body); err != nil {
            log.Error("notify failed", zap.String("msg_id", d.MessageId), zap.Error(err))
            _ = d.Nack(false, false) // 不进 requeue，走死信
            continue
        }
        _ = d.Ack(false)
    }
    return nil
}
```
启动时先 `ch.Confirm(false)` 开启 confirm 模式，Publish 里的 `NotifyPublish` 才有意义。

## 六、与 Kafka 的取舍
RocketMQ 胜在事务、延迟与队列级顺序，RabbitMQ 胜在 Exchange 路由与低延迟分发，Kafka 胜在吞吐与回溯；完整决策树见 [01-消息队列选型对比](01-消息队列选型对比.md) 第六节。

## 完整示例：本地启动
```yaml
# deploy/docker/docker-compose.mq.yml
services:
  rabbitmq:
    image: rabbitmq:3.13-management
    ports: ["5672:5672", "15672:15672"]
```

## 常见坑
1. **顺序消费却并发处理**：`WithConsumerOrder(false)` 或多线程处理同一队列，顺序保证失效。
2. **事务消息不实现回查**：Broker 回查失败，半消息长期悬挂。
3. **延迟消息当精确闹钟**：档位有误差，期间订单可能已支付，必须校验状态。
4. **RabbitMQ 自动 ack**：消费者崩溃时消息已确认，直接丢失。
5. **队列 durable 但消息非 persistent**：Broker 重启后队列在、消息没了。

## 小结
- RocketMQ 的强项是事务、顺序、延迟三类业务消息，代价是队列级串行与档位精度。
- RabbitMQ 的强项是 Exchange 路由、死信与低延迟分发，弱项是堆积能力。
- 可靠消费的统一公式：持久化 + confirm + 手动 ack + prefetch + 死信 + 幂等。

## 练习
1. 用 RocketMQ 发一条延迟消息，验证到期可消费，并说明如何避免误取消已支付订单。
2. 用顺序生产者和消费者处理同一 `order_id` 的 100 条状态变更，验证顺序。

## 官方文档
- RocketMQ 官方文档：<https://rocketmq.apache.org/zh/docs/>
- rocketmq-client-go：<https://github.com/apache/rocketmq-client-go>
- RabbitMQ 官方文档：<https://www.rabbitmq.com/docs>
- amqp091-go：<https://github.com/rabbitmq/amqp091-go>
- RabbitMQ 死信：<https://www.rabbitmq.com/docs/dlx>
