
> Kafka 是互联网公司最常用的消息中间件，几乎是中高级 Java 面试必考项。

---

## 一、知识树

```
Kafka 架构
├── 为什么需要 MQ
│   ├── 解耦
│   ├── 异步
│   └── 削峰
├── MQ 带来的问题
│   ├── 系统复杂度增加
│   ├── 一致性问题
│   └── 可用性降低
├── 核心概念
│   ├── Broker / Topic / Partition
│   ├── Offset / Consumer Group
│   └── Replica / ISR
├── Kafka 为什么快
│   ├── 顺序写磁盘
│   ├── 零拷贝
│   ├── PageCache
│   ├── 分区并行
│   ├── 批量处理
│   └── 压缩
└── Kafka vs RabbitMQ
```

---

## 二、为什么需要消息队列（MQ）

### 1. 解耦

```
场景：订单系统需要通知库存、物流、积分、短信等多个系统

无 MQ：订单系统直接调用各系统 HTTP 接口
  订单 → 库存（新增）
  订单 → 物流（新增）
  订单 → 积分（新增）
  问题：每新增一个下游系统，订单系统都要改代码

有 MQ：订单系统只发一条消息
  订单 → MQ → 库存 / 物流 / 积分 / 短信 ...
  优势：下游系统增减不影响上游
```

### 2. 异步

```
场景：用户注册后需要发送邮件、短信、初始化数据

同步：注册(50ms) + 邮件(100ms) + 短信(50ms) = 200ms 响应时间
异步：注册(50ms) + 写MQ(5ms) = 55ms 响应时间，其余异步处理
```

### 3. 削峰

```
场景：秒杀活动，瞬时 QPS 从 1000 暴涨到 100000

无 MQ：请求直接打到 DB → DB 瞬间崩溃
有 MQ：请求写入 MQ（百万级吞吐），消费者按自身能力匀速消费
```

---

## 三、MQ 带来的问题

| 问题 | 说明 |
|------|------|
| 系统复杂度增加 | 引入 MQ 后需要考虑消息丢失、重复消费、顺序消费、积压等问题 |
| 一致性问题 | 下游系统处理失败时，数据不一致 |
| 可用性降低 | MQ 本身成为单点，MQ 挂了整个链路中断 |
| 运维成本增加 | 需要维护 MQ 集群的监控、扩容、故障恢复 |

---

## 四、Kafka 核心概念

### 架构图

```
Producer Group                Broker Cluster                  Consumer Group
┌─────────┐    ┌──────────────────────────────────┐    ┌─────────────┐
│Producer1│───▶│ Broker1    Broker2    Broker3     │───▶│ Consumer1   │
│Producer2│───▶│ ┌───────┐ ┌───────┐ ┌───────┐   │───▶│ Consumer2   │
│Producer3│    │ │P0 L   │ │P1 L   │ │P2 L   │   │    │ Consumer3   │
└─────────┘    │ │P1 F   │ │P2 F   │ │P0 F   │   │    └─────────────┘
               │ │P2 F   │ │P0 F   │ │P1 F   │   │
               │ └───────┘ └───────┘ └───────┘   │
               │       Topic: order-topic         │
               └──────────────────────────────────┘

L = Leader Replica   F = Follower Replica
```

### 核心概念一览

| 概念 | 说明 |
|------|------|
| **Broker** | Kafka 服务节点，多个 Broker 组成集群 |
| **Topic** | 消息的逻辑分类，类似于数据库的表 |
| **Partition** | Topic 的物理分片，是并行度的基本单位 |
| **Offset** | 消息在 Partition 内的唯一偏移量（单调递增） |
| **Consumer Group** | 消费者组，组内消费者共同消费一个 Topic 的所有 Partition |
| **Replica** | 分区副本，分为 Leader 和 Follower |
| **ISR** | In-Sync Replicas，与 Leader 保持同步的副本集合 |

### Consumer Group 规则

```
核心规则：一个 Partition 只能被同一个 Consumer Group 中的一个 Consumer 消费

Topic: order-topic (3个Partition)
Consumer Group A:
  Consumer1 → Partition0
  Consumer2 → Partition1
  Consumer3 → Partition2

如果 Consumer 数量 > Partition 数量：多余的 Consumer 空闲
如果 Consumer 数量 < Partition 数量：部分 Consumer 消费多个 Partition
```

---

## 五、Kafka 为什么这么快（6 大原因）

### 1. 顺序写磁盘

```
传统认知：磁盘慢 → 实际上磁盘的顺序写速度接近内存的随机写

磁盘随机写：约 100 KB/s
磁盘顺序写：约 600 MB/s（接近内存随机写 800 MB/s）

Kafka 的消息追加到日志文件末尾（Append-Only），永远顺序写
```

### 2. 零拷贝（Zero Copy）

```
传统 IO（4 次拷贝 + 4 次上下文切换）：
  磁盘 → 内核缓冲区(PageCache) → 用户缓冲区 → Socket缓冲区 → 网卡
  用户态 ←→ 内核态 切换 4 次

sendfile 零拷贝（2 次拷贝 + 2 次上下文切换）：
  磁盘 → 内核缓冲区(PageCache) → 网卡（直接 DMA 传输）
  无需将数据拷贝到用户空间

Java 层面：FileChannel.transferTo() 底层调用 sendfile()
```

### 3. PageCache

```
Kafka 大量使用操作系统的 PageCache：
- 写入时先写 PageCache，由 OS 异步刷盘
- 读取时先查 PageCache，命中则直接返回
- 利用 OS 的 LRU 算法管理缓存页
- JVM GC 不影响 PageCache
```

### 4. 分区并行

```
一个 Topic 拆分为多个 Partition：
- 不同 Partition 分布在不同 Broker 上
- 生产者可并行写入不同 Partition
- 消费者组内多个消费者并行消费不同 Partition
- 线性扩展：N 个 Partition 理论上吞吐量提升 N 倍
```

### 5. 批量处理

```java
// 生产者批量发送配置
props.put("batch.size", 16384);       // 一批最大 16KB
props.put("linger.ms", 5);            // 最多等待 5ms
props.put("buffer.memory", 33554432); // 缓冲区 32MB
```

### 6. 消息压缩

```
支持 gzip / snappy / lz4 / zstd 压缩算法
- 减少网络传输量
- 减少磁盘存储量
- 生产者压缩，消费者解压，Broker 透传
```

---

## 六、Kafka vs RabbitMQ 详细对比

| 维度 | Kafka | RabbitMQ |
|------|-------|----------|
| **定位** | 分布式消息流平台 | 传统消息队列 |
| **吞吐量** | 百万级 QPS | 万级 QPS |
| **延迟** | ms 级 | us 级 |
| **消息模型** | 发布-订阅（Topic/Partition） | Exchange-Queue 绑定 |
| **持久化** | Append-Only 日志文件 | 内存为主，可选持久化 |
| **消息回溯** | 支持（通过 Offset 随机读取） | 不支持（消费即删除） |
| **消息顺序** | 分区内严格有序 | 单 Queue 有序 |
| **事务** | 支持跨 Partition 事务 | 支持（但不推荐高吞吐场景） |
| **适用场景** | 大数据/日志/事件流/消息 | 业务消息/任务队列/RPC |
| **社区生态** | 大数据生态（Spark/Flink 集成） | AMQP 标准协议 |
| **选型建议** | 高吞吐/大数据/日志收集 | 复杂路由/低延迟/中小规模 |

**选型建议：**
- 日志收集、大数据流处理、事件驱动架构 → **Kafka**
- 业务消息异步处理、复杂路由、延迟敏感 → **RabbitMQ**
- 如果团队已有其中之一，非极端场景不必引入第二个

---

## 七、面试题

### Q1：为什么选择 Kafka 而不是其他 MQ？🔥🔥🔥🔥🔥

从吞吐量、持久化、回溯能力、生态四个维度回答。Kafka 适用于高吞吐、大数据场景，RabbitMQ 适用于复杂路由和低延迟场景。

### Q2：Kafka 为什么这么快？🔥🔥🔥🔥🔥

6 大原因：顺序写磁盘、零拷贝（sendfile）、PageCache、分区并行、批量处理、消息压缩。重点讲零拷贝和顺序写。

### Q3：解释 Kafka 的 Consumer Group 机制？🔥🔥🔥🔥

一个 Consumer Group 内，每个 Partition 只能被一个 Consumer 消费。组内 Rebalance 会重新分配 Partition。Consumer 数量不应超过 Partition 数量。

### Q4：Partition 有什么作用？如何确定 Partition 数量？🔥🔥🔥🔥

Partition 是并行度的基本单位，决定了最大并行消费者数量。Partition 数量建议：预期吞吐量 / 单个 Partition 吞吐量，通常建议 3-10 个。

### Q5：什么是 ISR？🔥🔥🔥🔥

ISR（In-Sync Replicas）是与 Leader 保持同步的副本集合。Follower 从 Leader 拉取数据，如果落后太多（超过 `replica.lag.time.max.ms`）会被踢出 ISR。

### Q6：Kafka 的零拷贝原理？🔥🔥🔥

传统 IO 经过 4 次数据拷贝，Kafka 使用 `sendfile()` 系统调用，数据直接从 PageCache 通过 DMA 传输到网卡，跳过用户空间拷贝。

### Q7：MQ 带来了哪些问题？🔥🔥🔥

系统复杂度增加（消息丢失/重复/顺序/积压）、一致性问题（下游失败）、可用性降低（MQ 挂了全完）、运维成本。

### Q8：Kafka 和 RabbitMQ 怎么选？🔥🔥

高吞吐、大数据、日志、流处理选 Kafka；复杂路由、低延迟、中小规模业务消息选 RabbitMQ。

### Q9：Producer 发送消息的流程？🔥

序列化 → 分区器（指定 Partition / 按 Key Hash / 轮询）→ 写入 RecordAccumulator（缓冲区批量打包）→ Sender 线程发送到 Broker → Broker 返回 ACK。

### Q10：如何保证消息的顺序性？🔥🔥

Kafka 只保证分区内有序。将需要保证顺序的消息路由到同一个 Partition（使用相同的 Key）。全局有序需要只用一个 Partition（牺牲吞吐）。

---

## 八、常见误区

| 误区 | 正确理解 |
|------|----------|
| Kafka 保证全局消息有序 | 只保证分区内有序，不保证全局有序 |
| 消费者越多消费越快 | 超过 Partition 数量的消费者会空闲 |
| Kafka 消息消费后立即删除 | 消息按保留策略删除，支持回溯消费 |
| 零拷贝 = 没有拷贝 | 仍有 2 次拷贝，只是比传统 4 次少 |
| Partition 越多越好 | Partition 过多会导致 Leader 选举变慢、文件句柄增多 |

---

## 九、实战场景

### 日志收集架构

```
应用服务器 → Filebeat → Kafka → Logstash → Elasticsearch → Kibana
                                        → Flink（实时计算）
```

### 用户行为追踪

```
前端埋点 → Nginx → Kafka → Flink（实时清洗） → HBase（存储）
                                            → MySQL（报表）
```

---

## 十、关联知识

- [36-Kafka可靠性与消息问题](36-Kafka可靠性与消息问题.md) - Kafka 消息不丢失、幂等、事务
- [39-限流熔断降级](39-限流熔断降级.md) - 消息积压时结合限流策略
- [40-分布式理论与事务](40-分布式理论与事务.md) - 消息最终一致性
- [37-IO与NIO](37-IO与NIO.md) - 零拷贝、epoll 底层原理
