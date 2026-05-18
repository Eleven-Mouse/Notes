# RabbitMQ 板块（可靠性 + 高可用 + 实战）

> 面试定位：不仅会用 RabbitMQ，还能讲清可靠性、顺序性、重复消费、积压处理与集群高可用取舍。

---

## 一、先讲结论（面试开场 20 秒）

1. RabbitMQ 适合业务解耦、异步削峰、最终一致性场景，核心是可靠投递与消费治理。
2. 真正“不丢消息”是链路级能力：生产者确认 + Broker 持久化 + 消费者手动 ACK + 失败补偿。
3. 生产瓶颈通常不在“会不会发消息”，而在“消费速度、重试策略、幂等设计、高可用架构”。

---

## 二、核心架构与工作流

### 2.1 基本模型

```text
Producer -> Exchange -> Queue -> Consumer
```

- Producer：发送消息。
- Exchange：按路由规则分发消息。
- Queue：缓存待消费消息。
- Consumer：拉取并处理消息，返回 ACK/NACK。

### 2.2 四种 Exchange（高频）

1. `direct`：精确匹配 routing key。
2. `topic`：通配匹配（`*` 单词、`#` 多词）。
3. `fanout`：广播，不看 routing key。
4. `headers`：按 header 匹配，生产使用较少。

面试建议：先说业务最常用 `direct/topic`，再补 `fanout` 广播通知场景。

### 2.3 虚拟主机 vhost

- 类似逻辑隔离空间，做到多环境/多租户隔离。
- 常见规划：`dev/test/prod` 分 vhost，权限按 vhost 控制。

---

## 三、可靠性三件套（必须答完整）

消息从生产到消费的完整链路：

```text
Producer -> [Broker: Exchange -> Queue] -> Consumer
   ①                 ②                         ③
```

每一段都可能丢消息，三件套分别保障一段。

### 3.1 生产者确认（Publisher Confirm）

问题：生产者发出后不知道 Broker 收到没。

推荐方案：Confirm 模式（不推荐事务模式，吞吐差）。

```yaml
spring:
  rabbitmq:
    publisher-confirm-type: correlated
    publisher-returns: true
    template:
      mandatory: true
```

```java
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (!ack) {
        // 记录失败并补偿重发（重试次数要有限制）
    }
});

rabbitTemplate.setReturnsCallback(returned -> {
    // 到了 Broker 但不可路由，通常是交换机/路由键配置问题
});
```

### 3.2 消息持久化（Queue durable + Message persistent）

1. 队列 durable：Broker 重启后队列仍存在。
2. 消息 persistent：消息写入持久化路径。

```java
QueueBuilder.durable("order.create.queue").build();
```

注意：持久化不等于绝对不丢，极端宕机场景仍可能损失少量未刷盘数据；关键链路建议 Quorum Queue。

### 3.3 消费者手动 ACK

默认自动 ACK 风险：业务没处理完就删消息。

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual
```

```java
@RabbitListener(queues = "order.create.queue")
public void onMessage(Message message, Channel channel,
                      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    try {
        // 1. 幂等校验 2. 业务处理
        channel.basicAck(tag, false);
    } catch (Exception e) {
        channel.basicNack(tag, false, false); // 推荐进入死信，由重试系统接管
    }
}
```

---

## 四、幂等、重复消费与顺序性

### 4.1 为什么会重复消费

1. 消费成功但 ACK 丢失。
2. 消费超时被重投。
3. NACK/requeue 导致再次投递。

### 4.2 幂等落地（面试最爱追问）

1. 业务唯一键判重（如订单号、流水号）。
2. 消费记录表（messageId + 状态 + 时间）。
3. Redis `SETNX` + 过期时间做轻量判重。

### 4.3 顺序消息怎么做

RabbitMQ 默认不能保证全局顺序，只能做“局部顺序”：

1. 同一业务 key（如 userId）路由到同一队列。
2. 单队列单消费者（或同 key 串行处理）。
3. 牺牲并行度换顺序性。

---

## 五、死信队列与延迟队列

### 5.1 死信触发条件

1. 消息被拒绝且 `requeue=false`。
2. 消息 TTL 过期。
3. 队列达到最大长度。

### 5.2 死信队列配置

```java
QueueBuilder.durable("order.normal.queue")
    .withArgument("x-dead-letter-exchange", "order.dlx.exchange")
    .withArgument("x-dead-letter-routing-key", "order.dlx.key")
    .build();
```

典型场景：消费失败转死信，交给补偿任务或人工干预。

### 5.3 延迟消息实现

常见两种：

1. TTL + DLX（通用方案，精度一般）。
2. `x-delayed-message` 插件（更直接，精度更好）。

业务例子：30 分钟未支付自动取消订单。

---

## 六、流控、积压与吞吐优化

### 6.1 prefetch（消费者限流）

- 作用：限制单消费者未 ACK 消息数，防止“吃太多撑死”。

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        prefetch: 50
```

### 6.2 消息积压排查思路

1. 看队列深度是否持续上涨。
2. 对比生产 TPS 与消费 TPS。
3. 排查慢消费代码（DB/远程调用/锁竞争）。
4. 扩消费实例、拆分队列、引入批处理。

### 6.3 重试策略建议

1. 不要无限 requeue，容易形成“毒消息风暴”。
2. 重试次数有限制，失败入死信。
3. 死信统一补偿，做到可观测可回放。

---

## 七、高可用：镜像队列 vs Quorum Queue

### 7.1 经典镜像队列（历史方案）

- 通过策略复制到多个节点。
- 已逐步被 Quorum Queue 替代。

### 7.2 Quorum Queue（推荐）

1. 基于 Raft，一致性更强。
2. 故障恢复更稳，适合关键业务队列。
3. 代价：资源开销更高，吞吐通常低于普通队列。

面试回答模板：核心订单流用 Quorum，非核心高吞吐日志流可用普通队列。

---

## 八、与 Kafka 的面试对比（常问）

1. RabbitMQ：低延迟、路由灵活、协议友好，业务消息处理体验好。
2. Kafka：高吞吐、可回放、日志流与大数据链路更强。
3. 选型关键：业务语义优先，不是“谁更快”。

---

## 九、Spring Boot 生产级配置示例

```yaml
spring:
  rabbitmq:
    host: 127.0.0.1
    port: 5672
    username: guest
    password: guest
    virtual-host: /prod
    publisher-confirm-type: correlated
    publisher-returns: true
    template:
      mandatory: true
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 50
        concurrency: 5
        max-concurrency: 20
```

---

## 十、面试高频追问速答

### Q1：怎么保证 RabbitMQ 消息不丢？

答题顺序：
1. 生产端 Confirm + Return。
2. Broker 端 durable + persistent + 高可用队列。
3. 消费端手动 ACK + 幂等。
4. 失败消息死信化与补偿。

### Q2：消息重复了怎么办？

- 先承认“至少一次语义下重复是常态”，再说幂等方案：
业务唯一键、去重表、Redis 判重、状态机防重入。

### Q3：消息积压怎么处理？

- 先止损：扩消费者、临时提升并发。
- 再治本：定位慢点（DB/远程依赖/锁），拆分队列和消费组。

### Q4：延迟队列如何实现？

- TTL + DLX 或延迟插件；核心是保证到期后可稳定触发并可重试。

### Q5：为什么手动 ACK 更安全？

- 因为“业务成功”与“消息确认”绑定，避免自动 ACK 的提前删除风险。

---

## 十一、常见踩坑

1. 只开 Confirm 不做 Return，结果路由失败感知不到。
2. 开了手动 ACK 但 catch 里忘记 ACK/NACK，导致消息堆积。
3. 无限重试不入死信，最终拖垮消费者。
4. 忽略幂等，重投后出现重复扣库存/重复下单。
5. 高并发下 prefetch 太大，单节点内存与处理延迟暴涨。

---

## 十二、总结（面试收尾）

RabbitMQ 真正考察的是“分布式稳定性设计”，不是 API 记忆。  
回答时按“投递可靠 -> 消费可靠 -> 幂等补偿 -> 高可用治理”的顺序展开，面试官会更容易判断你有生产经验。
