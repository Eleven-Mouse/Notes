# Java 全栈知识图谱 v2 — 目录总览

> 面向 2026 互联网大厂 Java 后端招聘标准
> 共 53 个细化文档 / 8 大模块（含微服务专项与 DDD 分层专项）

---

## 学习路线

```
Phase 1（地基，2-3 周）: Part1 Java基础 → Part2 JVM
Phase 2（核心，3-4 周）: Part3 并发编程 → Part4 锁机制
Phase 3（框架，2-3 周）: Part5 Spring 全家桶
Phase 4（存储，2-3 周）: Part6 MySQL + Redis + MyBatis
Phase 5（进阶，2-3 周）: Part7 中间件 → Part8 架构
```

---

## 完整文档索引

### Part 1：Java 基础（5 个文档）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 01 | [数据类型与String](01-数据类型与String.md) | 8种基本类型/自动装箱拆箱/Integer缓存池/String不可变/常量池/StringBuilder | ⭐⭐⭐ |
| 02 | [面向对象](02-面向对象.md) | 封装继承多态/重写vs重载/抽象类vs接口/equals-hashCode契约/深浅拷贝 | ⭐⭐⭐ |
| 03 | [泛型与反射](03-泛型与反射.md) | 类型擦除/桥方法/PECS原则/通配符/Class对象/反射API/性能开销 | ⭐⭐⭐ |
| 04 | [集合框架](04-集合框架.md) | ArrayList扩容/HashMap底层(JDK7vs8)/put流程/红黑树转换/扩容机制/LinkedHashMap LRU | ⭐⭐⭐ |
| 05 | [异常体系与注解](05-异常体系与注解.md) | Throwable层次/try-catch-finally/try-with-resources/元注解/注解+反射/Lombok原理 | ⭐⭐ |

### Part 2：JVM（4 个文档）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 06 | [内存结构](06-内存结构.md) | 堆/栈/方法区/元空间/程序计数器/直接内存/对象内存布局/TLAB | ⭐⭐⭐ |
| 07 | [类加载机制](07-类加载机制.md) | 加载→验证→准备→解析→初始化/双亲委派/打破场景(SPI/Tomcat/OSGi) | ⭐⭐⭐ |
| 08 | [垃圾回收](08-垃圾回收.md) | 可达性分析/GC Roots/四种引用/GC算法/收集器对比/G1 Region化/ZGC | ⭐⭐⭐ |
| 09 | [调优实战](09-调优实战.md) | JVM参数/jps-jstat-jmap-jstack/Arthas/OOM排查/CPU100%排查/FullGC排查 | ⭐⭐ |

### Part 3：并发编程（8 个文档）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 10 | [线程基础](10-线程基础.md) | Thread/Runnable/Callable/6种状态转换/sleep-wait-join-yield对比/守护线程/中断 | ⭐⭐⭐ |
| 11 | [JMM与volatile](11-JMM与volatile.md) | 主内存vs工作内存/三大特性/happens-before 8条/内存屏障/volatile原理/DCL单例 | ⭐⭐⭐ |
| 12 | [synchronized与CAS](12-synchronized与CAS.md) | 三种用法/Monitor/monitorenter/CAS原理/ABA问题/原子类/LongAdder | ⭐⭐⭐ |
| 13 | [AQS与Lock](13-AQS与Lock.md) | AQS核心(state+CLH队列)/公平非公平源码/lock-unlock流程/Condition/ReadWriteLock | ⭐⭐⭐ |
| 14 | [线程池](14-线程池.md) | 7大参数/执行流程/4种拒绝策略/4种线程池问题/参数设定公式/关闭/异常处理 | ⭐⭐⭐ |
| 15 | [并发容器](15-并发容器.md) | ConcurrentHashMap(JDK7vs8)/put-get-resize/CopyOnWriteArrayList/BlockingQueue/跳表 | ⭐⭐⭐ |
| 16 | [ThreadLocal](16-ThreadLocal.md) | Thread→ThreadLocalMap→Entry/内存泄漏分析/使用场景/InheritableThreadLocal/TTL | ⭐⭐⭐ |
| 17 | [并发工具类](17-并发工具类.md) | CountDownLatch/CyclicBarrier/Semaphore/Exchanger/Phaser/对比 | ⭐⭐ |

### Part 4：锁机制（3 个文档）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 18 | [锁分类与锁升级](18-锁分类与锁升级.md) | 6维度分类/Mark Word 64bit/synchronized锁升级(偏向→轻量→重量)/批量重偏向 | ⭐⭐⭐ |
| 19 | [各类锁对比](19-各类锁对比.md) | ReentrantLock vs synchronized/公平非公平/读写锁/StampedLock/死锁4条件 | ⭐⭐ |
| 20 | [分布式锁](20-分布式锁.md) | Redis SETNX+Lua/Redisson看门狗/RedLock/ZooKeeper临时节点/MySQL锁/三种对比 | ⭐⭐⭐ |

### Part 5：Spring 全家桶（6 个文档）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 21 | [IOC与Bean](21-IOC与Bean.md) | IOC原理/BeanFactory vs ApplicationContext/依赖注入方式/@Autowired原理/Bean作用域 | ⭐⭐⭐ |
| 22 | [AOP](22-AOP.md) | 切面切点通知/JDK动态代理/CGLIB/代理失效6种场景/AOP应用场景 | ⭐⭐⭐ |
| 23 | [SpringMVC](23-SpringMVC.md) | DispatcherServlet 9步流程/HttpMessageConverter/拦截器vs过滤器/统一异常处理 | ⭐⭐⭐ |
| 24 | [SpringBoot](24-SpringBoot.md) | @SpringBootApplication拆解/自动装配流程/spring.factories/@Conditional/自定义Starter | ⭐⭐⭐ |
| 25 | [事务](25-事务.md) | 7种传播行为/事务失效6种场景/声明式事务原理/编程式事务/长事务问题 | ⭐⭐⭐ |
| 26 | [循环依赖](26-循环依赖.md) | 三级缓存详解/解决流程/为什么需要三级/构造器注入为什么不行/@Lazy解决 | ⭐⭐ |

### Part 6：存储层（8 个文档）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 27 | [MySQL-存储引擎与索引](27-MySQL-存储引擎与索引.md) | InnoDB vs MyISAM/B+树/聚簇vs非聚簇/回表/覆盖索引/最左匹配/索引下推 | ⭐⭐⭐ |
| 28 | [MySQL-事务与MVCC](28-MySQL-事务与MVCC.md) | ACID实现/4种隔离级别/MVCC(隐藏字段+Undo链+ReadView)/RC vs RR差异 | ⭐⭐⭐ |
| 29 | [MySQL-锁与调优](29-MySQL-锁与调优.md) | 表锁行锁/Record-Gap-NextKey/EXPLAIN详解/慢查询/深分页优化/Redo-Undo-Binlog | ⭐⭐⭐ |
| 30 | [Redis-数据类型](30-Redis-数据类型.md) | 五大类型+补充/SDS/ziplist/quicklist/skiplist/hashtable/intset/渐进式rehash | ⭐⭐⭐ |
| 31 | [Redis-持久化与集群](31-Redis-持久化与集群.md) | RDB/AOF/混合持久化/主从复制/Sentinel哨兵/Cluster 16384槽/Gossip协议 | ⭐⭐ |
| 32 | [Redis-缓存问题](32-Redis-缓存问题.md) | 穿透/击穿/雪崩/CacheAside/延迟双删/MQ一致性/过期策略/8种淘汰/热点Key/大Key | ⭐⭐⭐ |
| 33 | [MyBatis-核心](33-MyBatis-核心.md) | 完整执行流程/MapperProxy/{} vs ${}/resultMap/关联映射/N+1问题/MyBatis-Plus | ⭐⭐ |
| 34 | [MyBatis-动态SQL与缓存](34-MyBatis-动态SQL与缓存.md) | 7个动态SQL标签/批量操作/一级二级缓存对比/插件机制/PageHelper原理 | ⭐⭐ |

### Part 7：中间件（3 个文档）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 35 | [Kafka架构](35-Kafka架构.md) | MQ三大作用/Broker-Topic-Partition-ConsumerGroup/6大性能原因/零拷贝/vs RabbitMQ | ⭐⭐ |
| 36 | [Kafka可靠性与消息问题](36-Kafka可靠性与消息问题.md) | 三端保证/acks=all/幂等生产者/事务/消息顺序/重复消费幂等/消息积压/Rebalance | ⭐⭐ |
| 37 | [IO与NIO](37-IO与NIO.md) | BIO/NIO/AIO对比/Buffer-Channel-Selector/select-poll-epoll/Reactor模式/Netty基础 | ⭐⭐ |

### Part 8：架构（15 个文档 + 融合）

| # | 文档 | 核心内容 | 优先级 |
|---|------|---------|--------|
| 38 | [微服务组件](38-微服务组件.md) | 注册中心对比/Nacos/Eureka/ZK/配置中心/OpenFeign/Dubbo/Gateway/负载均衡策略 | ⭐ |
| 39 | [限流熔断降级](39-限流熔断降级.md) | 4种限流算法图解/Sentinel/熔断状态机/降级策略/组件对比/接口幂等5种方案 | ⭐⭐ |
| 40 | [分布式理论与事务](40-分布式理论与事务.md) | CAP/BASE/一致性哈希/雪花算法/2PC/TCC/Saga/消息最终一致性/Seata四种模式 | ⭐⭐ |
| 41 | [系统设计](41-系统设计.md) | 答题框架/秒杀系统(含Lua脚本)/短链系统(Base62)/Feed/IM/搜索建议 | ⭐ |
| 43 | [Nacos注册与配置中心](Part8-架构/43-Nacos注册与配置中心.md) | 注册发现流程/namespace-group-dataId/动态刷新/AP与CP取舍/高可用 | ⭐⭐⭐ |
| 44 | [OpenFeign声明式服务调用](Part8-架构/44-OpenFeign声明式服务调用.md) | 动态代理机制/超时重试/负载均衡/降级策略/调用链治理 | ⭐⭐⭐ |
| 45 | [SpringCloudGateway网关](Part8-架构/45-SpringCloudGateway网关.md) | 路由与过滤器/统一鉴权/网关限流/灰度路由/网关稳定性 | ⭐⭐⭐ |
| 46 | [Seata分布式事务](Part8-架构/46-Seata分布式事务.md) | TC/TM/RM/AT模式原理/undo_log/冲突与性能取舍/适用边界 | ⭐⭐⭐ |
| 47 | [Sentinel限流熔断降级](Part8-架构/47-Sentinel限流熔断降级.md) | 流控规则/熔断规则/热点参数/Feign与Gateway协同/规则治理 | ⭐⭐⭐ |
| 48 | [DDD四层与启动层总览](Part8-架构/48-DDD四层与启动层总览.md) | application/domain/adapter/infrastructure/start 全局边界 | ⭐⭐⭐ |
| 49 | [DDD-Application层](Part8-架构/49-DDD-Application层.md) | 用例编排/事务边界/命令查询模型/与Controller职责分离 | ⭐⭐⭐ |
| 50 | [DDD-Domain层](Part8-架构/50-DDD-Domain层.md) | 实体值对象聚合/领域服务/不变量/反贫血模型 | ⭐⭐⭐ |
| 51 | [DDD-Adapter层](Part8-架构/51-DDD-Adapter层.md) | 入站出站适配/协议转换/异常映射/DTO装配 | ⭐⭐ |
| 52 | [DDD-Infrastructure层](Part8-架构/52-DDD-Infrastructure层.md) | 仓储实现/中间件集成/技术可替换性/可观测治理 | ⭐⭐ |
| 53 | [DDD-Start启动层](Part8-架构/53-DDD-Start启动层.md) | 启动装配/配置管理/模块装配/运行时拓扑 | ⭐⭐ |
| 42 | [知识图谱融合](42-知识图谱融合.md) | 完整技术地图/调用链分析/面试路线图/频率热力图/答题模板/项目补强/文件索引 | ⭐⭐⭐ |

---

## 面试频率热力图

```
面试必考（100%）:
  ✅ 01-数据类型与String（Integer缓存/String不可变）
  ✅ 04-集合框架（HashMap 底层/put流程/扩容）
  ✅ 06-内存结构（堆栈方法区）
  ✅ 08-垃圾回收（GC算法/G1）
  ✅ 11-JMM与volatile（三大特性/happens-before）
  ✅ 14-线程池（7参数/执行流程）
  ✅ 18-锁分类与锁升级（synchronized锁升级）
  ✅ 21-IOC与Bean（依赖注入）
  ✅ 25-事务（传播行为/失效场景）
  ✅ 27-MySQL索引（B+树/最左匹配）
  ✅ 28-MySQL事务与MVCC
  ✅ 30-Redis数据类型
  ✅ 32-Redis缓存问题（穿透/击穿/雪崩）

面试高频（70%）:
  🔶 03-泛型与反射（类型擦除）
  🔶 07-类加载机制（双亲委派）
  🔶 12-synchronized与CAS
  🔶 13-AQS与Lock
  🔶 15-并发容器（ConcurrentHashMap）
  🔶 16-ThreadLocal（内存泄漏）
  🔶 20-分布式锁
  🔶 22-AOP（代理失效）
  🔶 24-SpringBoot（自动装配）
  🔶 26-循环依赖（三级缓存）
  🔶 29-MySQL锁与调优

面试常问（50%）:
  🔸 02-面向对象
  🔸 09-调优实战
  🔸 17-并发工具类
  🔸 19-各类锁对比
  🔸 31-Redis持久化与集群
  🔸 35-Kafka架构
  🔸 36-Kafka可靠性
  🔸 40-分布式理论与事务

面试加分（30%）:
  ▪️ 05-异常体系与注解
  ▪️ 23-SpringMVC
  ▪️ 33-MyBatis核心
  ▪️ 34-MyBatis动态SQL与缓存
  ▪️ 37-IO与NIO
  ▪️ 38-微服务组件
  ▪️ 39-限流熔断降级
  ▪️ 41-系统设计
```
