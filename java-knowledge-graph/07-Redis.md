# Redis — 深度拆解

> 优先级：⭐⭐⭐ | 面试频率：🔥🔥🔥 | 掌握程度：深入

---

## 一、知识树

```
Redis
│
├── 1. 数据类型 ⭐⭐⭐ 🔥🔥🔥
│   ├── String（SDS 实现）
│   ├── List（quicklist = ziplist + linkedlist）
│   ├── Hash（ziplist / hashtable）
│   ├── Set（intset / hashtable）
│   ├── ZSet（ziplist / skiplist + hashtable）
│   └── 补充类型：Bitmap / HyperLogLog / GEO / Stream
│
├── 2. 底层数据结构 ⭐⭐ 🔥🔥
│   ├── SDS（Simple Dynamic String）
│   ├── ziplist（压缩列表）
│   ├── quicklist（快速列表）
│   ├── skiplist（跳表）
│   ├── hashtable（哈希表）
│   └── intset（整数集合）
│
├── 3. 持久化 ⭐⭐⭐ 🔥🔥🔥
│   ├── RDB（快照）
│   ├── AOF（追加日志）
│   ├── AOF 重写
│   ├── 混合持久化（RDB + AOF）
│   └── RDB vs AOF 选型
│
├── 4. 缓存问题 ⭐⭐⭐ 🔥🔥🔥
│   ├── 缓存穿透（查询不存在的数据）
│   ├── 缓存击穿（热点 key 过期）
│   ├── 缓存雪崩（大量 key 同时过期）
│   ├── 缓存与数据库一致性
│   └── 热点 key 问题
│
├── 5. 集群 ⭐⭐ 🔥🔥
│   ├── 主从复制
│   ├── 哨兵模式（Sentinel）
│   ├── Cluster 模式
│   └── 数据分片（16384 个槽位）
│
├── 6. 分布式锁 ⭐⭐⭐ 🔥🔥🔥
│   ├── SETNX + 过期时间
│   ├── Redisson 实现
│   ├── 看门狗续期
│   └── RedLock 算法
│
└── 7. 其他高频考点 ⭐⭐ 🔥🔥
    ├── 过期删除策略（惰性 + 定期）
    ├── 内存淘汰策略（8 种）
    ├── Pipeline
    ├── Lua 脚本
    └── 单线程为什么快
```

---

## 二、核心知识点精讲

### 2.1 五大数据类型及应用场景

| 类型 | 底层结构 | 常用命令 | 典型场景 |
|------|---------|---------|---------|
| String | SDS | SET/GET/INCR | 缓存 / 计数器 / 分布式锁 / Session |
| List | quicklist | LPUSH/LPOP/RPUSH | 消息队列 / 最新列表 / 栈/队列 |
| Hash | ziplist/hashtable | HSET/HGET/HMGET | 对象存储（用户信息 / 商品详情） |
| Set | intset/hashtable | SADD/SMEMBERS/SINTER | 标签 / 共同好友 / 去重 |
| ZSet | ziplist/skiplist+hashtable | ZADD/ZRANGE/ZRANK | 排行榜 / 延迟队列 / 范围查询 |

**ZSet 为什么用跳表而不用平衡树？**

```
1. 实现简单（比红黑树简单得多）
2. 范围查询高效（链表直接遍历）
3. 内存可调（通过调整层数平衡时间和空间）
4. 查找/插入/删除：O(logN)
```

### 2.2 持久化 — RDB vs AOF

| 维度 | RDB | AOF |
|------|-----|-----|
| 原理 | 定时全量快照 | 追加写命令日志 |
| 触发方式 | save/bgsave/自动 | always/everysec/no |
| 文件大小 | 小（二进制压缩） | 大（文本命令） |
| 恢复速度 | 快 | 慢 |
| 数据安全 | 可能丢两次快照间的数据 | 最多丢 1 秒数据 |
| 适用场景 | 备份 / 冷备 / 容灾 | 数据安全要求高 |

**混合持久化（JDK4.0+ 默认）：**

```
AOF 重写时：
  前半部分 = RDB 格式（快速加载）
  后半部分 = AOF 格式（增量命令）

好处：兼顾恢复速度和数据安全
```

### 2.3 缓存三大问题

#### 缓存穿透

```
问题：查询一个数据库和缓存中都不存在的数据 → 每次请求都打到数据库

解决方案：
1. 缓存空值（SET null_key "" EX 60）
   - 简单但有内存浪费，可能被恶意攻击构造大量不同 key
2. 布隆过滤器（Bitmap 实现）
   - 所有可能的数据哈希到位图，查询前先过滤
   - 有误判率（可能漏过），但不会误拦
3. 参数校验（拦截非法请求）
```

#### 缓存击穿

```
问题：某个热点 key 过期瞬间，大量并发请求打到数据库

解决方案：
1. 互斥锁（SETNX）
   SETNX lock_key 1 EX 5
   - 获取锁的线程查数据库并回填缓存
   - 未获取锁的线程短暂数据后重试
2. 逻辑过期（不设 TTL，数据中加逻辑过期时间字段）
   - 发现逻辑过期 → 异步线程更新缓存 → 返回旧数据
   - 牺牲强一致性换取高可用
3. 热点 key 永不过期 + 异步刷新
```

#### 缓存雪崩

```
问题：大量 key 在同一时刻过期 / Redis 宕机 → 所有请求打到数据库

解决方案：
1. 过期时间加随机值（避免同时过期）
   SET key value EX base_time + random(0, 300)
2. 多级缓存（Redis + 本地缓存 Caffeine）
3. 熔断降级（Hystrix / Sentinel）
4. Redis 集群高可用（主从 + 哨兵 / Cluster）
```

### 2.4 缓存与数据库一致性

```
经典问题：先更新数据库还是先更新缓存？

标准方案：Cache Aside Pattern（旁路缓存模式）
  读：先读缓存 → 缓存未命中 → 读数据库 → 写入缓存
  写：先更新数据库 → 再删除缓存

为什么是删除缓存而不是更新缓存？
  - 并发场景下更新缓存可能导致数据不一致
  - 删除缓存更简单，下次读时自然回填

延迟双删：
  更新数据库 → 删除缓存 → 延迟一段时间 → 再删除缓存
  目的：防止第二次删除前有旧数据被写回缓存

终极方案：
  更新数据库 → 删除缓存 → 发送消息到 MQ → 消费者确保缓存被删除
```

### 2.5 内存淘汰策略

| 策略 | 含义 |
|------|------|
| noeviction（默认） | 不淘汰，内存满了直接报错 |
| allkeys-lru | 所有 key 中淘汰最近最少使用的 |
| allkeys-lfu | 所有 key 中淘汰使用频率最低的 |
| allkeys-random | 所有 key 中随机淘汰 |
| volatile-lru | 设置了过期时间的 key 中淘汰 LRU |
| volatile-lfu | 设置了过期时间的 key 中淘汰 LFU |
| volatile-random | 设置了过期时间的 key 中随机淘汰 |
| volatile-ttl | 设置了过期时间的 key 中淘汰 TTL 最短的 |

**选型建议：**
- 确定 key 都会被访问 → allkeys-lru
- 有冷热数据区分 → allkeys-lfu（Redis 4.0+）
- 部分数据可以丢弃 → volatile-ttl

### 2.6 Redis 单线程为什么快？

```
1. 纯内存操作（纳秒级）
2. IO 多路复用（epoll，单线程处理大量连接）
3. 避免上下文切换（单线程无锁竞争）
4. 高效的数据结构（SDS / ziplist / skiplist）

注意：Redis 6.0 引入多线程处理网络 IO（读写），命令执行仍是单线程
```

---

## 三、高频面试题（10 题）

### Q1：Redis 支持哪些数据类型？分别什么场景？
```
String：缓存 / 计数器 / 分布式锁
List：消息队列 / 时间线
Hash：对象存储（比 String 存储 JSON 更节省内存）
Set：去重 / 交集并集（共同好友）
ZSet：排行榜 / 延迟队列

补充：Bitmap（签到统计）/ HyperLogLog（UV 统计）/ GEO（位置服务）
```

### Q2：Redis 的持久化方式？怎么选？
```
RDB：定时快照，恢复快，可能丢数据
AOF：追加日志，数据安全，恢复慢

生产建议：
- 都开：AOF 保证数据安全，RDB 用于快速恢复和备份
- 用混合持久化（AOF 重写时前半部分用 RDB 格式）
```

### Q3：缓存穿透 / 击穿 / 雪崩的区别和解决方案？
```
穿透：查不存在的数据 → 布隆过滤器 / 缓存空值
击穿：热点 key 过期 → 互斥锁 / 逻辑过期
雪崩：大量 key 同时过期 → 过期时间加随机值 / 多级缓存 / 熔断
```

### Q4：Redis 怎么实现分布式锁？
```
基础版：
  SET lock_key unique_id NX EX 30
  （加锁和设过期时间必须原子操作）

解锁（Lua 脚本保证原子性）：
  if redis.call('get',KEYS[1]) == ARGV[1] then
    return redis.call('del',KEYS[1])
  else return 0 end

进阶版（Redisson）：
  - 看门狗自动续期
  - 可重入锁（hash 结构）
  - RedLock（多节点防止单点故障）
```

### Q5：Redis 集群方案有哪些？
```
主从复制：读写分离，主写从读
哨兵模式：自动故障转移（监控 + 通知 + 自动故障转移）
Cluster：数据分片（16384 个槽位），每个节点负责一部分

Cluster 原理：
  key -> CRC16(key) % 16384 -> 对应节点
  每个主节点可以有从节点（高可用）
```

### Q6：过期删除策略？
```
惰性删除：访问 key 时检查是否过期，过期则删除（CPU 友好）
定期删除：每隔一段时间随机检查一批 key，删除过期的（折中方案）

配合内存淘汰策略：如果过期删除后内存仍不足，触发内存淘汰
```

### Q7：Redis 和 Memcached 的区别？
```
数据类型：Redis 5 种 + 补充类型 / Memcached 只有 String
持久化：Redis 支持 / Memcached 不支持
集群：Redis 原生支持 / Memcached 靠客户端
线程模型：Redis 单线程（6.0 网络 IO 多线程）/ Memcached 多线程
内存管理：Redis 有淘汰策略 / Memcached 用 LRU

结论：新项目基本都用 Redis
```

### Q8：如何保证缓存和数据库的一致性？
```
标准方案（Cache Aside）：
  读：缓存 → 数据库 → 回填缓存
  写：更新数据库 → 删除缓存

延迟双删：更新DB → 删缓存 → 等待 → 再删缓存

强一致性：只能通过分布式事务 / 分布式锁实现（牺牲性能）
最终一致性：MQ 异步删除缓存（推荐）
```

### Q9：跳表是什么？为什么 ZSet 用跳表？
```
跳表：多层有序链表，通过多级索引实现 O(logN) 查找

为什么不用红黑树：
1. 实现简单（红黑树维护复杂）
2. 范围查询高效（链表直接遍历，红黑树需要中序遍历）
3. 内存更灵活（可调整层数）
```

### Q10：热点 key 问题怎么处理？
```
问题：某个 key 被大量请求访问（如热门商品），单个 Redis 节点成为瓶颈

解决方案：
1. 本地缓存（Caffeine）减轻 Redis 压力
2. 热点 key 打散（key 加随机后缀，分散到多个 key）
3. 读写分离（从节点分担读压力）
4. Redis Cluster 分散到不同节点
```

---

## 四、常见误区

| 误区 | 正确理解 |
|------|---------|
| Redis 是单线程所以性能差 | 单线程避免了锁竞争，内存操作极快 |
| AOF 的 always 最安全但推荐用 | 生产用 everysec（性能和安全的折中） |
| 缓存一定要和数据库强一致 | 大多数场景最终一致性即可 |
| SETNX 就够了 | 需要加过期时间 + Lua 解锁 + 续期（用 Redisson） |
| Redis Cluster 和哨兵二选一 | Cluster 包含了高可用，哨兵适合不需要分片的场景 |

---

## 五、实战应用场景

| 场景 | Redis 方案 |
|------|-----------|
| 排行榜 | ZADD + ZREVRANGE |
| 限时活动 | SET key value EX 86400 |
| 分布式锁 | Redisson |
| 好友关系 | SINTER / SUNION |
| 签到统计 | SETBIT / BITCOUNT |
| UV 统计 | PFADD / PFCOUNT（HyperLogLog，0.81% 误差） |
| 消息队列 | Stream（Redis 5.0+）/ List（简单场景） |
| 购物车 | HSET cart:{userId} {productId} {quantity} |

---

## 六、关联模块

- ← [06-MySQL.md](06-MySQL.md)：Redis 缓存 + MySQL 持久化的经典组合
- ← [04-锁机制.md](04-锁机制.md)：Redis 分布式锁是单机锁的延伸
- → [05-Spring全家桶.md](05-Spring全家桶.md)：Spring Boot 通过 spring-boot-starter-data-redis 集成
- → [10-微服务与分布式.md](10-微服务与分布式.md)：Redis 作为分布式缓存和会话存储
