# RabbitMQ 可靠性保障三件套

消息从生产到消费的完整链路：

```
Producer → [Broker: Exchange → Queue] → Consumer
   ①           ②                          ③
```

每一步都可能丢消息，三件套分别保障一段。

---

## 一、生产者确认（Publisher Confirm）

### 问题

生产者发消息到 Broker，默认是 **fire-and-forget**——发了就完了，不知道 Broker 收没收到。网络抖动、Broker 宕机，消息就丢了，生产者还不知道。

### 两种解决机制

#### 1. Transaction 模式（不推荐）

```java
channel.txSelect();    // 开启事务
channel.basicPublish(...);
channel.txCommit();    // 提交
// 失败则 channel.txRollback();
```

每次发消息都要同步等 Broker 应答，吞吐量直接暴跌。

#### 2. Confirm 模式（推荐）

```java
// 开启 confirm 模式
channel.confirmSelect();

// 异步监听确认
channel.addConfirmListener(new ConfirmListener() {
    @Override
    public void handleAck(long deliveryTag, boolean multiple) {
        // Broker 确认收到，消息安全了
    }

    @Override
    public void handleNack(long deliveryTag, boolean multiple) {
        // Broker 拒绝了，需要重发
    }
});

channel.basicPublish(...);
```

原理：Broker 收到消息后异步回调 ACK/NACK，不阻塞生产者，性能好。

### Spring Boot 配置

application.yml：

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated   # 开启 confirm，异步回调
    publisher-returns: true              # 消息不可路由时触发 return 回调
```

Producer 代码：

```java
@Component
public class SeckillMessageProducer {

    private final RabbitTemplate rabbitTemplate;

    public SeckillMessageProducer(RabbitTemplate rabbitTemplate) {
        this.rabbitTemplate = rabbitTemplate;

        // 消息到达 Broker 的确认回调
        rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
            if (ack) {
                log.info("消息已确认: {}", correlationData);
            } else {
                log.error("消息确认失败: {}, 原因: {}", correlationData, cause);
                // 重发 or 记录到数据库做补偿
            }
        });

        // 消息无法路由到任何队列时的回调
        rabbitTemplate.setReturnsCallback(returned -> {
            log.error("消息无法路由: {}, replyText: {}",
                returned.getMessage(), returned.getReplyText());
        });
    }
}
```

---

## 二、消息持久化（durable queue + persistent message）

### 问题

消息存在内存里，Broker 重启就全没了。持久化 = 写到磁盘。

### 两步缺一不可

- 队列不持久化 → Broker 重启队列都没了，消息无处可存
- 消息不持久化 → 队列还在但消息没了

### 1. 队列持久化

```java
// durable = true
channel.queueDeclare("seckill_queue", true, false, false, null);
//                                     ↑
//                                  durable
```

### 2. 消息持久化

```java
AMQP.BasicProperties props = new AMQP.BasicProperties.Builder()
    .deliveryMode(2)  // 2 = persistent
    .build();

channel.basicPublish("", "seckill_queue", props, message.getBytes());
//                              ↑ deliveryMode=2 ↑
```

### Spring Boot 配置

```java
@Configuration
public class RabbitMQConfig {

    @Bean
    public Queue seckillQueue() {
        return QueueBuilder.durable("seckill.queue")
                // 死信队列配置（处理失败的消息）
                .withArgument("x-dead-letter-exchange", "seckill.dlx.exchange")
                .withArgument("x-dead-letter-routing-key", "seckill.dlx")
                .build();
    }

    @Bean
    public DirectExchange seckillExchange() {
        return ExchangeBuilder.directExchange("seckill.exchange").durable(true).build();
    }
}
```

### 持久化的代价

- 吞吐量下降（每次都要写磁盘）
- 不是 100% 不丢（Broker 先写内存页缓存，还没刷盘就宕机会丢，但概率极低）
- 如果要求绝对不丢 → 用仲裁队列（Quorum Queue，RabbitMQ 3.10+）

---

## 三、消费者手动 ACK

### 问题

默认是**自动 ACK**——消息一投递给消费者就从队列删除。如果消费者处理到一半挂了，消息就永远丢了。

### 手动 ACK 的思路

消费者处理完业务逻辑后，主动告诉 Broker "我处理完了"，Broker 才删消息。处理失败就 NACK，让 Broker 重新投递。

### Spring Boot 配置

application.yml：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual
```

Consumer 代码：

```java
@Component
public class SeckillMessageConsumer {

    @RabbitListener(queues = "seckill.queue")
    public void handleSeckill(Message message, Channel channel,
                              @Header(AmqpHeaders.DELIVERY_TAG) long deliveryTag) throws IOException {

        try {
            String body = new String(message.getBody());
            SeckillMessage seckillMsg = parseMessage(body);

            // === 执行业务逻辑 ===
            seckillService.executeSeckill(seckillMsg);

            // 业务成功，手动确认
            channel.basicAck(deliveryTag, false);
            log.info("秒杀消息处理成功: {}", seckillMsg);

        } catch (Exception e) {
            log.error("秒杀消息处理失败", e);

            // 方式一：重新入队重试
            channel.basicNack(deliveryTag, false, true);

            // 方式二：不重新入队 → 进死信队列（适合重试太多次的情况）
            // channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

### ACK 方式一览

| 方法 | 含义 |
|---|---|
| `basicAck(tag, false)` | 确认这条消息处理成功 |
| `basicNack(tag, false, true)` | 处理失败，重新入队 |
| `basicNack(tag, false, false)` | 处理失败，不重新入队（进死信队列） |
| `basicReject(tag, false)` | 拒绝单条消息，等同于 Nack 不重入队 |

### 关键陷阱：幂等性

手动 ACK + 重试 = 同一条消息可能被消费多次。消费者必须保证幂等：

```java
public void executeSeckill(SeckillMessage msg) {
    // 先查是否已处理过（用唯一键判重）
    SeckillOrder existing = orderMapper.selectByUserIdAndGoodsId(
        msg.getUserId(), msg.getGoodsId());
    if (existing != null) {
        log.info("重复消息，跳过: {}", msg);
        return;  // 幂等：已处理过直接返回，然后 ACK
    }

    // 执行扣库存、创建订单...
}
```

---

## 三件套协作全景图

```
Producer                    Broker                      Consumer
   │                          │                            │
   │──── ① confirm ──────────>│                            │
   │<──── ack/nack ───────────│                            │
   │                          │                            │
   │                      ② 持久化                          │
   │                      (queue + message                  │
   │                       写入磁盘)                        │
   │                          │                            │
   │                          │──── 投递消息 ─────────────>│
   │                          │                            │
   │                          │<─── ③ 手动ACK/NACK ───────│
   │                          │      (处理完才确认)         │
   │                          │                            │
   │                          │  NACK + requeue             │
   │                          │<───────────────────────────│
   │                          │──── 重新投递 ─────────────>│
```

---

## 总结

| 保障手段 | 解决的问题 | 核心配置 |
|---|---|---|
| Publisher Confirm | 消息到 Broker 不丢 | `publisher-confirm-type: correlated` |
| 消息持久化 | Broker 重启不丢 | `durable=true` + `deliveryMode=2` |
| 消费者手动 ACK | 消费者处理不丢 | `acknowledge-mode: manual` |

三道防线全配上，才能真正做到消息**至少投递一次（at-least-once）**。
