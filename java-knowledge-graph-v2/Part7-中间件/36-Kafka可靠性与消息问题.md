
> 消息可靠性是 Kafka 面试的核心重灾区，务必掌握三端保证和幂等方案。

---

## 一、知识树

```
Kafka 可靠性与消息问题
├── 消息不丢失（三端保证）
│   ├── 生产者端（acks / retries / 幂等）
│   ├── Broker 端（副本 / ISR / unclean 选举）
│   └── 消费者端（手动提交 Offset）
├── Exactly-Once 语义
│   ├── 幂等生产者（PID + SequenceNumber）
│   └── 事务（跨 Partition 原子写入）
├── 消息顺序保证
│   └── 分区内有序 / Key 路由
├── 重复消费与幂等
│   └── 4 种实现方案
├── 消息积压处理
├── Rebalance 机制
└── 面试题
```

---

## 二、消息不丢失的三端保证

### 1. 生产者端

```java
// 可靠发送配置
props.put("acks", "all");                    // 等待所有 ISR 副本确认
props.put("retries", Integer.MAX_VALUE);     // 无限重试
props.put("enable.idempotence", "true");     // 开启幂等
props.put("max.in.flight.requests.per.connection", "5"); // 幂等时可为5
```

**acks 参数详解：**

| acks 值 | 行为 | 可靠性 | 吞吐量 |
|---------|------|--------|--------|
| 0 | 发送即忘，不等响应 | 最低 | 最高 |
| 1 | 等 Leader 写入成功 | 中等 | 中等 |
| all（-1） | 等所有 ISR 副本写入成功 | 最高 | 最低 |

### 2. Broker 端

```properties
# 副本配置
replication.factor=3              # 每个 Partition 3 个副本
min.insync.replicas=2             # ISR 中至少 2 个副本
unclean.leader.election.enable=false  # 禁止非 ISR 副本成为 Leader
```

**关键点：**
- `replication.factor >= 3`：保证有冗余副本
- `min.insync.replicas = 2`：配合 `acks=all`，至少 2 个副本写入成功才算成功
- `unclean.leader.election.enable = false`：防止数据丢失（但可能牺牲可用性）

```
场景：replication.factor=3, min.insync.replicas=2
  Leader 写入 → ISR 中 2 个写入成功 → 返回 ACK
  如果 ISR 中只剩 1 个 → 拒绝写入（宁可不可用也不丢数据）
```

### 3. 消费者端

```java
// 手动提交 Offset
props.put("enable.auto.commit", "false");

// 消费逻辑
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        // 1. 先处理业务
        processMessage(record);
        // 2. 业务处理成功后再提交 Offset
        consumer.commitSync();
    }
}
```

**关键原则：先处理业务，再提交 Offset**

---

## 三、恰好一次语义（Exactly-Once）

### Kafka 的三种消息语义

| 语义 | 说明 | 场景 |
|------|------|------|
| At Most Once | 最多一次，可能丢失 | acks=0 |
| At Least Once | 至少一次，可能重复 | acks=all + 手动提交（先提交再处理可能重复） |
| Exactly Once | 恰好一次 | 幂等生产者 + 事务 |

### 1. 幂等生产者原理

```
Producer 初始化时分配唯一 PID（Producer ID）
每条消息携带 Sequence Number（每个 <PID, Partition> 递增）

Broker 端：
  维护每个 <PID, Partition> 的最新 Sequence Number
  如果收到 SeqN <= 已提交的 SeqN → 判定为重复消息，丢弃
```

```
Producer                    Broker
  PID=1, Partition=0
  SeqN=0 ──────────────▶  写入成功，记录 SeqN=0
  SeqN=1 ──────────────▶  写入成功，记录 SeqN=1
  SeqN=1 (重试) ───────▶  SeqN=1 <= 已有 SeqN=1 → 丢弃（幂等）
  SeqN=2 ──────────────▶  写入成功，记录 SeqN=2
```

**限制：** 幂等只保证单 Partition 单会话内的幂等，跨 Partition 或重启后不保证。

### 2. Kafka 事务

```java
// 跨 Partition 的原子写入
producer.initTransactions();

try {
    producer.beginTransaction();
    producer.send(record1); // Partition 0
    producer.send(record2); // Partition 1
    producer.send(record3); // Partition 2
    // 提交消费者的 Offset（Consume-Transform-Produce 模式）
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

**事务原理：**
- 引入 Transaction Coordinator（事务协调器）
- 类似 2PC：写 Prepare → 写数据 → 写 Commit
- 消费者设置 `isolation.level=read_committed` 只读取已提交的事务消息

---

## 四、消息顺序保证

```
保证级别：
  ✅ 分区内严格有序（Kafka 天然保证）
  ❌ 分区间无序（不同 Partition 的消息无法保证顺序）
  ❌ 全局有序（除非只用 1 个 Partition）

实现方案：
  将需要保证顺序的消息设置相同的 Key
  → Hash(Key) % PartitionCount → 路由到同一 Partition
```

```java
// 相同 OrderId 的消息会路由到同一个 Partition
producer.send(new ProducerRecord<>("order-topic", orderId, orderEvent));
```

---

## 五、重复消费与幂等方案

### 4 种幂等实现方案

| 方案 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **数据库唯一约束** | 利用唯一索引去重 | 简单可靠 | 依赖数据库 |
| **Redis SETNX** | `SETNX msgId 1` 原子操作 | 高性能 | 需维护 Redis |
| **状态机** | 业务状态只能单向流转 | 语义清晰 | 需设计状态流转 |
| **全局 ID 去重表** | 消费前查去重表 | 通用性强 | 多一次查询 |

### Redis SETNX 示例

```java
public void consume(Message msg) {
    String msgId = msg.getId();
    // 利用 Redis SETNX 原子性判断
    Boolean isFirst = redisTemplate.opsForValue()
        .setIfAbsent("msg:consume:" + msgId, "1", 24, TimeUnit.HOURS);
    if (Boolean.FALSE.equals(isFirst)) {
        log.warn("重复消息，跳过: {}", msgId);
        return;
    }
    // 执行业务逻辑
    processBusiness(msg);
}
```

### 状态机示例

```java
// 订单状态只能单向流转
public enum OrderStatus {
    CREATED(1), PAID(2), SHIPPED(3), COMPLETED(4);
    // 状态只能从小到大，不允许回退
}

public void processPayment(Order order) {
    if (order.getStatus().getCode() >= OrderStatus.PAID.getCode()) {
        log.warn("订单已支付，跳过: {}", order.getId());
        return; // 幂等：重复消费直接跳过
    }
    order.setStatus(OrderStatus.PAID);
    orderDao.update(order);
}
```

---

## 六、消息积压处理

### 紧急处理方案

```
1. 快速扩容消费者数量（不超过 Partition 数量）
2. 如果消费者扩容到上限还不够：
   - 临时新建一个 Topic（Partition 数量为原来的 10 倍）
   - 原消费者只做转发：旧 Topic → 新 Topic
   - 新消费者消费新 Topic（10 倍并行度）
   - 积压消费完后恢复原架构
3. 检查消费端是否有慢查询 / 外部调用超时
```

### 长期预防方案

```
1. 监控 Consumer Lag（消费者落后量）
2. 合理设置 Partition 数量（提前规划容量）
3. 消费端做批量处理，提高单次消费吞吐
4. 设置合理的告警阈值（Lag > N 触发告警）
```

---

## 七、消费者 Rebalance

### 触发条件

| 触发条件 | 说明 |
|---------|------|
| Consumer 加入组 | 新实例启动 |
| Consumer 离开组 | 实例宕机或主动关闭 |
| Consumer 心跳超时 | `session.timeout.ms` 内未收到心跳 |
| 订阅的 Topic 变化 | Partition 数量变化 |
| Consumer 调用 unsubscribe | 取消订阅 |

### Rebalance 的影响

```
1. Rebalance 期间所有消费者停止消费（STW）
2. 如果频繁 Rebalance 会导致消费延迟
3. 大规模消费者组（> 100）的 Rebalance 可能持续数分钟

优化：
  - session.timeout.ms = 25s（心跳超时）
  - heartbeat.interval.ms = 8s（心跳间隔）
  - max.poll.interval.ms = 5min（处理超时）
  - 使用 CooperativeStickyAssignor（增量分配，非全量重分配）
```

---

## 八、面试题

### Q1：如何保证 Kafka 消息不丢失？🔥🔥🔥🔥🔥

三端保证：
1. **生产者**：`acks=all` + `retries=MAX` + 幂等生产者
2. **Broker**：`replication.factor >= 3` + `min.insync.replicas = 2` + 禁止 unclean 选举
3. **消费者**：关闭自动提交，业务处理完再手动提交 Offset

### Q2：Kafka 的 Exactly-Once 怎么实现？🔥🔥🔥🔥

三种层次：(1) 幂等生产者（PID + SeqN 单分区内去重）；(2) 事务（跨 Partition 原子写入）；(3) Consume-Transform-Produce 模式（消费+处理+生产+提交 Offset 在一个事务中）。

### Q3：Kafka 幂等生产者的原理？🔥🔥🔥🔥

Producer 分配唯一 PID，每条消息携带递增的 Sequence Number。Broker 维护每个 `<PID, Partition>` 的最新 SeqN，收到重复 SeqN 的消息直接丢弃。

### Q4：消息积压了怎么处理？🔥🔥🔥🔥

紧急：扩容消费者 → 临时扩 Partition 转发方案 → 排查消费端瓶颈。长期：监控 Lag、提前规划容量、批量消费。

### Q5：Rebalance 是什么？如何避免频繁 Rebalance？🔥🔥🔥

Consumer Group 内的 Partition 重新分配过程。避免：合理设置心跳参数、使用 CooperativeStickyAssignor、避免消费逻辑耗时超过 `max.poll.interval.ms`。

### Q6：如何保证消息的顺序消费？🔥🔥🔥

Kafka 保证分区内有序。将需要保证顺序的消息用相同的 Key，路由到同一 Partition。如果需要全局有序，只用一个 Partition（牺牲吞吐）。

### Q7：如何处理重复消费？🔥🔥🔥

幂等方案：数据库唯一约束、Redis SETNX、状态机、全局 ID 去重表。业务层面保证消费操作的幂等性是根本解决方案。

### Q8：Kafka 事务的原理？🔥🔥

引入 Transaction Coordinator，类似 2PC。流程：beginTransaction → 写数据到多个 Partition → sendOffsetsToTransaction → commitTransaction。消费者设置 `read_committed` 只读已提交事务的消息。

### Q9：acks=all 就一定不丢消息吗？🔥

不一定。如果 ISR 中只有 Leader 自己（所有 Follower 都掉出 ISR），`acks=all` 等同于 `acks=1`。所以必须配合 `min.insync.replicas >= 2` 使用。

### Q10：消费者手动提交 Offset 有什么注意事项？🔥

注意：(1) 先处理业务再提交，否则宕机会丢消息；(2) 同步提交（commitSync）更可靠但阻塞，异步提交（commitAsync）性能好但可能丢失 Offset；(3) 推荐组合使用：异步 + 关闭时同步。

---

## 九、常见误区

| 误区 | 正确理解 |
|------|----------|
| acks=all 就不会丢数据 | 必须配合 min.insync.replicas >= 2 |
| 幂等生产者能解决所有重复 | 只保证单 Partition 单会话 |
| 消费者越多越好 | 超过 Partition 数量会空闲，且增加 Rebalance 风险 |
| 自动提交 Offset 更方便 | 可能丢消息（提交了但没处理完）或重复消费（处理完但没提交） |
| Rebalance 是正常的 | 频繁 Rebalance 说明配置或消费逻辑有问题 |

---

## 十、实战场景

### 金融支付消息可靠消费

```java
// 金融场景：支付结果通知，要求不丢 + 不重复
public void handlePaymentNotification(PaymentNotification notification) {
    // 1. 幂等检查（Redis SETNX）
    String key = "pay:notify:" + notification.getPayId();
    if (!redis.setIfAbsent(key, "1", 7, DAYS)) {
        return; // 重复通知
    }
    try {
        // 2. 本地事务：更新订单状态
        orderService.updatePaymentStatus(notification);
        // 3. 手动提交 Offset
        ack();
    } catch (Exception e) {
        redis.delete(key); // 失败则清除标记，允许重试
        throw e; // 触发重试
    }
}
```

---

## 十一、关联知识

- [35-Kafka架构](35-Kafka架构.md) - Kafka 基础架构和核心概念
- [40-分布式理论与事务](../Part8-架构/40-分布式理论与事务.md) - 分布式事务、2PC
- [39-限流熔断降级](../Part8-架构/39-限流熔断降级.md) - 接口幂等性
- [41-系统设计](../Part8-架构/41-系统设计.md) - 消息队列在系统设计中的应用
