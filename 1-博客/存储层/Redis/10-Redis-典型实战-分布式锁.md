---
publishTime: 2026-05-20T15:48:44+08:00
---

>Redis 分布式锁的本质，是用低成本互斥解决多实例并发冲突，但必须配合幂等、超时、原子解锁和一致性兜底。

 ## 先定边界
  Redis 分布式锁解决的是“多进程/多机器并发访问同一资源”的互斥问题，不是强一致事务系统。
  在 2026 年互联网实际架构里，它通常作为“并发冲突削峰器”，最终正确性仍要靠数据库约束、幂等、版本校验来兜底。

  ## 为什么它重要（基于真实场景）

  1. 真实公司都把 Redis 放在核心链路。Grab 在 2019-02-15 的事故复盘里公开过：Redis 集群节点故障期间，核心预订服务短时成
     功率掉到约 1/21，说明并发控制和容灾是业务生死线。
  2. Airbnb 在 2025-10-09 工程文中写到，其 KV 服务早期 QoS 依赖 Redis 分布式计数限流，目标是防止单调用方拖垮整个平台。
  3. Redis 官方锁文档明确指出：主从异步复制下，故障切换可能出现“双持锁”，所以锁实现必须是“正确实现”，不能只写 SETNX +
     DEL。
----
  一、最小实现方案（你必须先吃透）

1. 获取锁
```
     SET lock:{bizKey} {token} NX PX {ttlMs}
```
  - NX：只在不存在时设置
  - PX：锁自动过期，防死锁
  - token：每次加锁唯一（UUID）

  2. 释放锁

  - Redis 8.4+：DELEX key IFEQ token
  - 老版本：Lua 原子脚本（先比对 token，再 DEL）

  3. Java 最小代码（Jedis 思路）

```
  String token = UUID.randomUUID().toString();
  String ok = jedis.set(lockKey, token, SetParams.setParams().nx().px(ttlMs));
  if (!"OK".equals(ok)) return false;

  try {
      // critical section
  } finally {
      String lua = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                   "return redis.call('del', KEYS[1]) else return 0 end";
      jedis.eval(lua, Collections.singletonList(lockKey), Collections.singletonList(token));
  }
```
  4. 最小实现的 4 个红线

  - 不要 SETNX 和 EXPIRE 分两步（原子性）
  - 不要裸 DEL
  - 不要不设 TTL
  - 不要把 ttl 设成拍脑袋常量（要按业务耗时分布设）

-----

  二、中级实现方案

  1. 用 Redisson，而不是手撸全部细节
```
   tryLock(waitTime, leaseTime, unit)
   默认 watchdog 约 30s（可配 lockWatchdogTimeout），用于自动续租
```
  2. 参数策略

  - waitTime：等锁上限，避免线程无限堆积
  - leaseTime：若任务可估时，显式设置；若不可估时，用 watchdog 但要有“最大执行时长”保护
  - 重试使用抖动退避，防止锁风暴

  3. 配套能力

  - 幂等键（requestId/orderId）
  - 监控：获取成功率、等待 P95/P99、超时率、续租次数、异常解锁
  - 限流与熔断：锁竞争高时主动降级

-----

  三、高级实现方案

  1. 多节点 Redlock（N=5 常见）

  - 只在多数节点加锁成功且耗时 < TTL 才算成功
  - 失败后要尽快释放已加到的部分锁
  - 优点：容错更强；代价：实现与运维复杂度高

  2. Fencing Token（非常关键）

  - 每次成功加锁后，再拿一个递增 token（如 INCR fence:{resource}）
  - 下游（DB/队列消费者）只接受“更大 token”的写入
  - 这样即使旧锁“复活”或网络延迟，旧请求也会被拒绝

  3. 主从复制安全增强

  - 锁写入后可结合 WAIT，提升副本确认概率（是“提升概率”，不是绝对强一致）
  - 支付/账务最终仍建议用 DB 唯一约束+版本号/状态机兜底

  4. 什么时候别用 Redis 锁

  - 需要线性一致协调（如强主从切换协调、全局唯一主控）
  - 优先考虑 etcd/ZooKeeper/Consul 这类一致性协调系统

------

  四、落地指南

  1. 选场景

  - 先从“可重试、可降级”的资源互斥入手：缓存重建、定时任务防重、库存预占
  - 不要一上来就改资金主链路

  2. 锁粒度设计

  - lock:{resourceType}:{resourceId}
  - 拒绝全局大锁

  3. TTL 设计

  - 初版可用：ttl = max(3 * p99_critical_section, floor)
  - 再用真实监控回调校正
  - 超时失败：业务返回“稍后重试”或进入补偿队列

  5. 验证清单
  - 故障演练：Redis 主从切换、网络抖动、JVM STW
  - 回归验证：幂等是否生效、是否出现双写/误删

  6. 上线节奏

  - 灰度 5% -> 20% -> 全量
  - 每阶段观察 24h：错误率、超时率、平均等待、业务成功率

-------

  你可以按这个学习顺序

  1. 官方正确实现（SET NX PX + 安全解锁）
  2. Redisson RLock 参数与 watchdog
  3. 锁失效/双持锁故障模型
  4. fencing token + DB 约束组合拳
  5. 压测和故障