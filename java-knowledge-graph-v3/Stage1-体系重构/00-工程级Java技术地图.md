# Java 全栈知识图谱 v3 — 工程级技术地图

> 每个 = 原理 + 项目 + 面试 + 源码

---

## 一、Java 技术全景树（工程级）

```
Java 后端技术体系 v3
│
├── M1: Java 基础 [地基层]
│   ├── 1.1 数据类型与String（装箱拆箱/Integer缓存池/String不可变/常量池/StringBuilder）
│   ├── 1.2 面向对象（封装继承多态/重写重载/抽象类接口/equals-hashCode/深浅拷贝）
│   ├── 1.3 泛型与反射（类型擦除/PECS/Class对象/反射性能）
│   ├── 1.4 集合框架（ArrayList扩容/HashMap底层/put全流程/红黑树转换/LinkedHashMap LRU）
│   └── 1.5 异常与注解（Throwable层次/try-with-resources/元注解/Lombok原理）
│
├── M2: JVM [运行基石]
│   ├── 2.1 内存结构（堆/栈/方法区/元空间/PC/直接内存/对象布局/TLAB）
│   ├── 2.2 类加载（加载→验证→准备→解析→初始化/双亲委派/打破场景）
│   ├── 2.3 垃圾回收（可达性分析/GC Roots/四引用/算法/收集器/G1/ZGC）
│   └── 2.4 调优实战（JVM参数/工具链/Arthas/OOM排查/CPU100%/FullGC）
│
├── M3: 并发编程 [高并发核心]
│   ├── 3.1 线程基础（Thread/Runnable/Callable/6种状态/sleep-wait-join）
│   ├── 3.2 JMM与volatile（主内存工作内存/happens-before/内存屏障/DCL）
│   ├── 3.3 synchronized与CAS（Monitor/monitorenter/CAS/ABA/原子类/LongAdder）
│   ├── 3.4 AQS与Lock（state+CLH/公平非公平/Condition/ReadWriteLock）
│   ├── 3.5 线程池（7参数/执行流程/拒绝策略/参数设定/关闭/异常处理）
│   ├── 3.6 并发容器（ConcurrentHashMap JDK7vs8/CopyOnWrite/BlockingQueue）
│   ├── 3.7 ThreadLocal（ThreadLocalMap/内存泄漏/使用场景/TTL）
│   └── 3.8 并发工具（CountDownLatch/CyclicBarrier/Semaphore/Phaser）
│
├── M4: 锁体系 [并发保障]
│   ├── 4.1 锁分类与升级（6维分类/Mark Word/偏向→轻量→重量）
│   ├── 4.2 各类锁对比（ReentrantLock vs sync/读写锁/StampedLock/死锁）
│   └── 4.3 分布式锁（Redis SETNX+Lua/Redisson看门狗/RedLock/ZK临时节点）
│
├── M5: IO/NIO与网络 [传输层]
│   ├── 5.1 IO模型（BIO/NIO/AIO/Buffer-Channel-Selector）
│   ├── 5.2 多路复用（select/poll/epoll/Reactor模式）
│   └── 5.3 Netty（EventLoop/Pipeline/ByteBuf/零拷贝）
│
├── M6: Spring 全家桶 [框架层]
│   ├── 6.1 IOC与Bean（容器原理/BeanFactory vs AC/依赖注入/生命周期）
│   ├── 6.2 AOP（切面/代理/JDK vs CGLIB/代理失效6场景）
│   ├── 6.3 SpringMVC（DispatcherServlet 9步/拦截器/统一异常处理）
│   ├── 6.4 SpringBoot（@SpringBootApplication/自动装配/自定义Starter）
│   ├── 6.5 事务（7种传播/失效6场景/声明式原理/长事务）
│   └── 6.6 循环依赖（三级缓存/为什么三级/构造器注入为什么不行）
│
├── M7: MySQL [持久化核心]
│   ├── 7.1 存储引擎与索引（InnoDB vs MyISAM/B+树/聚簇非聚簇/回表/覆盖/最左/下推）
│   ├── 7.2 事务与MVCC（ACID实现/4种隔离/隐藏字段+Undo链+ReadView）
│   ├── 7.3 锁与调优（Record/Gap/NextKey/EXPLAIN/慢查询/深分页/Redo-Undo-Binlog）
│   └── 7.4 高可用（主从复制/分库分表/读写分离）
│
├── M8: Redis [缓存层]
│   ├── 8.1 数据类型（5大+3补充/SDS/ziplist/skiplist/渐进式rehash）
│   ├── 8.2 持久化与集群（RDB/AOF/混合/主从/Sentinel/Cluster 16384槽）
│   └── 8.3 缓存问题（穿透/击穿/雪崩/一致性/过期策略/淘汰/热点Key/大Key）
│
├── M9: MyBatis [ORM层]
│   ├── 9.1 核心流程（完整执行链/MapperProxy/# vs $/resultMap/N+1）
│   └── 9.2 动态SQL与缓存（7标签/批量操作/一级二级缓存/插件机制）
│
├── M10: 消息队列 [异步层]
│   ├── 10.1 MQ基础（三大作用/选型对比/RabbitMQ vs Kafka vs RocketMQ）
│   ├── 10.2 Kafka架构（Broker/Topic/Partition/ConsumerGroup/零拷贝/6大性能）
│   └── 10.3 Kafka可靠性（三端保证/acks/幂等/事务/顺序/积压/Rebalance）
│
├── M11: 微服务 [服务治理]
│   ├── 11.1 注册中心（Nacos/Eureka/ZK/CP vs AP）
│   ├── 11.2 配置中心（Nacos Config/Apollo）
│   ├── 11.3 网关（Gateway/路由/过滤器/鉴权）
│   ├── 11.4 服务调用（OpenFeign/Dubbo/负载均衡策略）
│   └── 11.5 链路追踪（Sleuth/Zipkin/SkyWalking）
│
├── M12: 分布式系统 [架构层]
│   ├── 12.1 理论基础（CAP/BASE/一致性哈希/雪花算法）
│   ├── 12.2 分布式事务（2PC/TCC/Saga/消息最终一致/Seata四种模式）
│   ├── 12.3 限流熔断降级（4种限流算法/Sentinel/熔断状态机/幂等）
│   └── 12.4 系统设计（答题框架/秒杀/短链/Feed/IM）
│
└── M13: 网络基础 [通信层]
    ├── 13.1 TCP/IP（三次握手/四次挥手/拥塞控制/流量控制）
    ├── 13.2 HTTP（1.0/1.1/2.0/3.0/HTTPS/TLS）
    └── 13.3 从URL到页面（DNS/TCP/HTTP/渲染）
```

---

## 二、模块依赖链（系统中的位置）

```
                    ┌─────────────┐
                    │   网络基础   │ ← 一切通信的基础
                    │   (M13)     │
                    └──────┬──────┘
                           │ HTTP/TCP
                    ┌──────▼──────┐
                    │   IO/NIO    │ ← 数据传输方式
                    │   (M5)      │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
       ┌──────▼──────┐    │     ┌──────▼──────┐
       │  Spring MVC  │    │     │    Netty     │
       │  (M6.3)     │    │     │   (M5.3)    │
       └──────┬──────┘    │     └──────┬──────┘
              │            │            │
       ┌──────▼──────┐    │     ┌──────▼──────┐
       │ Spring 全家桶│    │     │  消息队列    │
       │  (M6)       │    │     │  (M10)      │
       └──┬───┬───┬──┘    │     └──────┬──────┘
          │   │   │       │            │
          │   │   │  ┌────▼────┐ ┌────▼───┐
          │   │   │  │ 微服务  │  │ 分布式  │
          │   │   │  │ (M11)  │  │ (M12)  │
          │   │   │  └────────┘  └────────┘
          │   │   │
    ┌─────▼┐ │ ┌─▼─────┐
    │MySQL │ │ │Redis  │
    │(M7)  │ │ │(M8)   │
    └──────┘ │ └───────┘
    ┌────────▼────────┐
    │    MyBatis (M9)  │
    └─────────────────┘

    底层支撑：
    ┌────────┐ ┌────────┐ ┌────────┐
    │Java基础│ │  JVM   │ │并发+锁 │
    │ (M1)   │ │ (M2)   │ │(M3+M4) │
    └────────┘ └────────┘ └────────┘
```

---

## 三、16大模块详解（核心作用 + 系统位置 + 依赖关系）

### M1: Java 基础 — 地基层

| 维度 | 说明 |
|------|------|
| **核心作用** | 所有代码的根基，决定了代码质量和理解深度 |
| **系统位置** | 最底层，一切框架和中间件的构建基础 |
| **依赖** | 无前置依赖，自身是所有其他模块的前置 |
| **被依赖** | JVM（对象模型）、并发（线程模型）、Spring（注解/反射） |
| **面试权重** | HashMap底层 + ConcurrentHashMap = 100% 必考 |
| **来源融合** | 小林（图解集合）+ JavaGuide（面试覆盖）+ javabetter（基础进阶） |

### M2: JVM — 运行基石

| 维度 | 说明 |
|------|------|
| **核心作用** | 理解 Java 程序"怎么跑的"，排障调优的终极武器 |
| **系统位置** | Java 代码与操作系统之间的桥梁 |
| **依赖** | M1（对象模型、类结构） |
| **被依赖** | M3（内存可见性/JMM）、M6（Spring Bean生命周期） |
| **面试权重** | 内存结构 + GC + 类加载 = 100% 必考 |
| **来源融合** | 小林（图解GC）+ 《深入理解JVM》+ JavaGuide + bugstack（实战调优） |

### M3: 并发编程 — 高并发核心

| 维度 | 说明 |
|------|------|
| **核心作用** | 解决多线程下的数据安全与协作问题 |
| **系统位置** | 贯穿全栈：线程池（Tomcat）→ 锁（库存）→ 异步（MQ） |
| **依赖** | M2（JMM/volatile/内存屏障）、M4（锁机制） |
| **被依赖** | M4（AQS基础）、M8（Redis分布式锁）、M10（MQ消费并发） |
| **面试权重** | synchronized + AQS + 线程池 = 100% 必考 |
| **来源融合** | 《Java并发编程实战》+ 小林（图解AQS）+ JavaGuide + source-code-hunter（源码） |

### M4: 锁体系 — 并发保障

| 维度 | 说明 |
|------|------|
| **核心作用** | 从单机锁到分布式锁，保证数据一致性 |
| **系统位置** | Service层（synchronized）→ 分布式（Redis/ZK锁） |
| **依赖** | M3（CAS/AQS基础）、M8（Redis命令） |
| **被依赖** | M12（分布式事务锁）、M7（MySQL行锁） |
| **面试权重** | 锁升级 + 分布式锁 = 高频 |
| **来源融合** | 小林（锁升级图解）+ JavaGuide + bugstack（源码分析） |

### M5: IO/NIO与网络 — 传输层

| 维度 | 说明 |
|------|------|
| **核心作用** | 理解数据如何在网络和磁盘间高效传输 |
| **系统位置** | Tomcat（NIO）→ Netty（RPC）→ Kafka（零拷贝） |
| **依赖** | M13（TCP/IP基础） |
| **被依赖** | M6（SpringMVC底层）、M10（Kafka零拷贝）、M11（RPC通信） |
| **面试权重** | BIO/NIO/AIO + epoll = 中高频 |
| **来源融合** | 小林（图解IO模型）+ source-code-hunter（Netty源码） |

### M6: Spring 全家桶 — 框架层

| 维度 | 说明 |
|------|------|
| **核心作用** | 企业级开发的骨架，IoC容器 + AOP切面 + 事务管理 |
| **系统位置** | 承上启下：接收请求 → 业务编排 → 持久化 |
| **依赖** | M1（反射/注解）、M2（类加载）、M3（并发Bean创建） |
| **被依赖** | M9（MyBatis整合）、M11（Spring Cloud基础） |
| **面试权重** | IOC + AOP + Bean生命周期 + 事务 = 100% 必考 |
| **来源融合** | bugstack（源码流程）+ JavaGuide + source-code-hunter（Spring源码） |

### M7: MySQL — 持久化核心

| 维度 | 说明 |
|------|------|
| **核心作用** | 数据最终落地的保障，事务 + 索引 + 锁三大核心 |
| **系统位置** | 数据存储层，系统最下游 |
| **依赖** | M3（事务隔离/锁）、M5（磁盘IO） |
| **被依赖** | M12（分布式事务）、M8（缓存一致性） |
| **面试权重** | 索引 + 事务 + MVCC + 锁 = 100% 必考 |
| **来源融合** | 小林（图解B+树/MVCC）+ JavaGuide + PDai（体系化） |

### M8: Redis — 缓存层

| 维度 | 说明 |
|------|------|
| **核心作用** | 高性能 KV 存储，缓存 + 分布式锁 + 限流 |
| **系统位置** | MySQL 前的加速层，MQ旁的协调层 |
| **依赖** | M5（IO模型/epoll）、M4（分布式锁） |
| **被依赖** | M12（限流/幂等）、M6（Spring Cache） |
| **面试权重** | 数据结构 + 缓存三大问题 + 一致性 = 100% 必考 |
| **来源融合** | 小林（图解数据结构）+ JavaGuide + 《Redis设计与实现》 |

### M9: MyBatis — ORM层

| 维度 | 说明 |
|------|------|
| **核心作用** | Java对象 ↔ SQL的桥梁，半自动ORM |
| **系统位置** | Service → DAO 之间的持久层 |
| **依赖** | M6（Spring整合）、M1（反射/动态代理） |
| **被依赖** | M7（SQL优化/索引配合） |
| **面试权重** | 执行流程 + # vs $ + 缓存 = 中频 |
| **来源融合** | bugstack（源码流程）+ source-code-hunter（MyBatis源码） |

### M10: 消息队列 — 异步层

| 维度 | 说明 |
|------|------|
| **核心作用** | 系统解耦、异步处理、流量削峰 |
| **系统位置** | 服务之间的异步通信管道 |
| **依赖** | M5（零拷贝/顺序写）、M3（消费者并发） |
| **被依赖** | M12（消息最终一致性） |
| **面试权重** | 三端保证 + 消息不丢失 + 幂等 = 高频 |
| **来源融合** | 小林（图解Kafka）+ JavaGuide + PDai（选型对比） |

### M11: 微服务 — 服务治理

| 维度 | 说明 |
|------|------|
| **核心作用** | 大规模系统的拆分和治理方案 |
| **系统位置** | 多个 Spring Boot 应用之间的协调层 |
| **依赖** | M6（Spring Boot基础）、M13（网络通信） |
| **被依赖** | M12（分布式问题在微服务中的体现） |
| **面试权重** | 注册中心 + 网关 + 服务调用 = 高频 |
| **来源融合** | JavaGuide + PDai（架构化）+ interviewguide（面试题） |

### M12: 分布式系统 — 架构层

| 维度 | 说明 |
|------|------|
| **核心作用** | 多节点协同的理论基础和工程实践 |
| **系统位置** | 架构最顶层，统领所有中间件和组件 |
| **依赖** | M3~M11（所有模块的综合应用） |
| **被依赖** | M13（网络分区） |
| **面试权重** | 分布式事务 + CAP + 限流 = 高频 |
| **来源融合** | PDai（体系化）+ JavaGuide + interviewguide（面试题） |

### M13: 网络基础 — 通信层

| 维度 | 说明 |
|------|------|
| **核心作用** | 理解系统间通信的底层机制 |
| **系统位置** | 一切分布式通信的基石 |
| **依赖** | 无前置 |
| **被依赖** | M5（IO模型）、M11（RPC通信）、M12（网络分区） |
| **面试权重** | 三次握手 + HTTP/HTTPS = 中频 |
| **来源融合** | 小林（图解TCP）+ JavaGuide + javabetter |

---

## 四、模块间核心依赖关系矩阵

### 4.1 总览矩阵

```
读法：行模块 → 列模块 = 行模块学习列模块时需要的依赖程度

列头说明：
  M1=基础 M2=JVM M3=并发 M4=锁 M5=IO/NIO M6=Spring
  M7=MySQL M8=Redis M9=MyBatis M10=MQ M11=微服务 M12=分布式 M13=网络

   
		         M1  M2  M3  M4  M5  M6  M7  M8  M9  M10 M11 M12 M13
		M1 基础   -   ★   ★   ·   ·   ★   ·   ·   ★   ·   ·   ·   ·
		M2 JVM    ·   -   ★   ·   ·   ★   ·   ·   ·   ·   ·   ·   ·
		M3 并发   ·   ·   -   ★   ·   ★   ★   ★   ·   ★   ·   ★   ·
		M4 锁     ·   ·   ·   -   ·   ·   ★   ★   ·   ·   ·   ★   ·
		M5 IO     ·   ·   ·   ·   -   ·   ·   ·   ·   ★   ★   ·   ★
		M6 Spring ·   ·   ·   ·   ·   -   ·   ·   ★   ·   ★   ·   ·
		M7 MySQL  ·   ·   ·   ·   ·   ·   -   ★   ·   ·   ·   ★   ·
		M8 Redis  ·   ·   ·   ·   ·   ·   ·   -   ·   ·   ·   ★   ·
		M9 MyBatis ·   ·   ·   ·   ·   ·   ·   ·   -   ·   ·   ·   ·
		M10 MQ    ·   ·   ·   ·   ·   ·   ·   ·   ·   -   ·   ★   ·
		M11 微服务·   ·   ·   ·   ·   ·   ·   ·   ·   ·   -   ★   ·
		M12 分布式·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   -   ·
		M13 网络  ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   ·   -


★ = 强依赖（必须先理解前置模块才能深入）
· = 弱关联（了解即可，不会成为理解障碍）
```

### 4.2 每一行强依赖详解

> 格式：`行模块 → 列模块 ★`：具体依赖什么知识点、为什么必须先学

---

#### M1: Java 基础 → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M1 → M2 JVM ★** | 对象模型（类结构/字段/方法）、类文件格式（.class字节码）、基本类型在JVM中的表示 | JVM研究的对象就是Java类和对象。不懂Java对象怎么定义的，就无法理解JVM怎么在内存中存放对象（对象头/实例数据/对齐填充） |
| **M1 → M3 并发 ★** | Thread类/Runnable接口、wait/notify机制、synchronized关键字语法 | 并发编程直接使用Java语言级线程API。不理解Thread的基本用法和状态转换，就无法深入线程池、锁、AQS等高级并发机制 |
| **M1 → M6 Spring ★** | 反射机制（Class对象/Method/Field）、注解（@Component/@Autowired）、动态代理（Proxy类） | Spring IOC本质是反射+工厂模式，AOP本质是动态代理，@Autowired依赖注入靠反射读字段。不理解反射和注解，Spring源码完全看不懂 |
| **M1 → M9 MyBatis ★** | 动态代理（Proxy.newProxyInstance）、反射（Method.invoke）、泛型（TypeReference） | MyBatis的核心就是给Mapper接口生成动态代理（MapperProxy），通过反射调用SQL。不理解代理模式，MyBatis执行流程无从理解 |

---

#### M2: JVM → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M2 → M3 并发 ★** | JMM内存模型（主内存/工作内存）、volatile底层（内存屏障/Lock前缀指令）、happens-before规则、对象头（Mark Word） | Java并发编程的可见性/有序性问题本质是JMM问题。volatile的底层是JVM层面的内存屏障指令。synchronized锁升级依赖对象头Mark Word结构。不懂JMM就不知道为什么需要volatile |
| **M2 → M6 Spring ★** | 类加载机制（双亲委派/ClassLoader）、运行时数据区（堆/方法区/栈）、GC对Bean生命周期的影响 | Spring IOC启动时大量使用反射和类加载。Bean的创建在堆上，BeanDefinition在方法区/元空间。循环依赖和三级缓存涉及对象引用在JVM中的管理。OOM排查需要懂JVM |

---

#### M3: 并发编程 → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M3 → M4 锁 ★** | CAS操作（Unsafe类/compareAndSwap）、AQS框架（state+CLH队列）、ReentrantLock源码 | 锁体系的核心AQS就是并发编程模块的内容。分布式锁的单机基础（synchronized/ReentrantLock）也来自并发编程。锁升级的偏向锁/轻量级锁依赖对synchronized Monitor的理解 |
| **M3 → M6 Spring ★** | ConcurrentHashMap（Bean存储）、线程安全（单例Bean的创建）、ThreadLocal（用户上下文传递） | Spring IOC容器内部用ConcurrentHashMap存储Bean。Bean创建过程涉及线程安全问题（三级缓存的并发访问）。@Async和事件发布使用线程池。ThreadLocal用于RequestContextHolder |
| **M3 → M7 MySQL ★** | 事务隔离级别（和Java锁的类比）、行锁/表锁概念、MVCC的读写锁思想、连接池（Druid/HikariCP底层是线程池） | 数据库事务的隔离级别概念和Java并发中的线程安全思想一脉相承。连接池本质就是线程池的变种。InnoDB的读写锁和Java的ReadWriteLock原理类似 |
| **M3 → M8 Redis ★** | 分布式锁（Redisson基于AQS的Redis实现）、原子操作（Lua脚本和CAS的类比）、单线程模型（和并发的关系）、Pipeline（批量和并发的权衡） | Redis分布式锁是Java并发锁的分布式版本。Redisson的RLock内部用到了Java的AQS框架。Redis单线程避免了锁竞争，理解这个需要并发知识 |
| **M3 → M10 MQ ★** | 消费者并发消费（多线程消费partition）、消息顺序性（和线程安全的类比）、Rebalance（消费者组的线程协调） | Kafka消费者是多线程消费的，消费者数量和partition数量的关系类似线程池。消息的顺序消费需要单线程或加锁。Rebalance过程中的协调类似并发编程中的同步问题 |
| **M3 → M12 分布式 ★** | 分布式锁（从单机锁到分布式锁的演进）、限流算法（令牌桶/漏桶的并发实现）、幂等性（CAS思想在分布式的应用） | 分布式系统中的并发控制是单机并发的扩展。分布式锁是Redisson基于单机锁概念实现的。限流算法需要CAS/原子操作。幂等性本质是"只执行一次"的并发控制 |

---

#### M4: 锁体系 → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M4 → M7 MySQL ★** | InnoDB行锁（Record/Gap/Next-Key）和Java锁的对比、乐观锁/悲观锁在数据库的实现、死锁检测和预防 | MySQL的行锁机制和Java的synchronized/ReentrantLock在概念上类似。数据库的乐观锁（version字段）对应Java的CAS思想。死锁的四个条件在数据库和Java中完全一致 |
| **M4 → M8 Redis ★** | 分布式锁（SETNX+Lua+Redisson看门狗）、Redis事务（MULTI/EXEC和锁的关系）、RedLock算法 | Redis最常见的分布式锁实现方案。Redisson的看门狗机制涉及锁续约问题。理解单机锁（M4.1/M4.2）是理解分布式锁（M4.3）的前置条件 |
| **M4 → M12 分布式 ★** | 分布式事务中的锁（2PC/TCC的资源锁定）、一致性保证（锁在分布式事务中的角色）、幂等性的锁实现 | 2PC第一阶段需要锁定资源（类似synchronized获取锁）。TCC的Try阶段本质是预留资源（乐观锁）。分布式幂等通常用Redis锁或数据库唯一约束实现 |

---

#### M5: IO/NIO → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M5 → M10 MQ ★** | 零拷贝（sendfile/mmap）→Kafka高性能核心、顺序写（磁盘IO优化）、PageCache（操作系统层面的缓存） | Kafka高性能的6大原因中有3个和IO直接相关：零拷贝、顺序写、PageCache。不理解mmap/sendfile就无法理解Kafka为什么比其他MQ快 |
| **M5 → M11 微服务 ★** | RPC通信底层（Netty/NIO）、序列化/反序列化（网络传输）、HTTP/2（gRPC底层） | 微服务间的通信（Dubbo/gRPC/Feign）底层都是网络IO。Dubbo默认用Netty（NIO框架）。gRPC基于HTTP/2（多路复用）。理解NIO/epoll才能理解RPC为什么高效 |
| **M5 → M13 网络 ★** | epoll/select/poll → TCP连接管理、Socket编程 → 网络通信基础、TLS → HTTPS安全传输 | IO多路复用（epoll）是网络编程的核心技术。NIO的Selector基于epoll实现。理解TCP的三次握手/四次挥手才能理解为什么连接池可以复用连接 |

---

#### M6: Spring → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M6 → M9 MyBatis ★** | Spring整合MyBatis（SqlSessionFactoryBean/MapperScannerConfigurer）、@MapperScan注解、Spring事务管理和MyBatis的结合 | MyBatis在Spring中通过SqlSessionFactoryBean注册BeanDefinition，MapperScannerConfigurer扫描Mapper接口。Spring的@Transactional控制MyBatis的SqlSession事务。不理解Spring IOC就无法理解MyBatis怎么被Spring管理的 |
| **M6 → M11 微服务 ★** | Spring Cloud基于Spring Boot、@EnableDiscoveryClient/@EnableFeignClients注解、Spring Cloud Gateway（WebFlux） | Spring Cloud是完全基于Spring Boot的微服务框架。服务注册（@EnableDiscoveryClient）、远程调用（@EnableFeignClients）、网关（Gateway）都依赖Spring的自动装配和注解驱动 |

---

#### M7: MySQL → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M7 → M8 Redis ★** | 缓存和数据库的一致性问题（Cache Aside/延迟双删/Canal）、MySQL的Binlog（Canal监听）、读写分离（Redis做读缓存） | Redis作为MySQL前的缓存层，两者的数据一致性是核心问题。Canal通过监听MySQL Binlog来异步更新Redis。读写分离方案中Redis缓存热点查询。不理解MySQL的事务和Binlog就无法设计缓存一致性方案 |

---

#### M8: Redis → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M8 → M12 分布式 ★** | 分布式限流（Redis+Lua实现滑动窗口/令牌桶）、分布式幂等（Redis SETNX去重）、分布式Session（Redis存储用户会话）、分布式ID（Redis INCR） | 分布式系统中的限流/幂等/会话/ID生成都需要Redis。限流用Redis+Lua脚本实现原子计数。幂等用SETNX实现去重。这些是分布式系统的核心基础设施 |

---

#### M10: MQ → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M10 → M12 分布式 ★** | 消息最终一致性（分布式事务方案之一）、事务消息（RocketMQ/Kafka）、事件驱动架构（EDA） | 分布式事务的最终一致性方案依赖MQ：本地事务+消息发送（本地消息表/事务消息）。Saga模式也可以基于MQ实现。理解MQ的可靠性保证才能设计出可靠的分布式事务方案 |

---

#### M11: 微服务 → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M11 → M12 分布式 ★** | 服务治理中的分布式问题（服务雪崩→熔断降级、服务发现→CAP理论、配置管理→一致性）、分布式追踪（链路追踪） | 微服务架构天然引入分布式问题：多个服务实例需要注册发现（CAP）、服务调用失败需要熔断降级（限流算法）、跨服务调用需要链路追踪（TraceId传递）。微服务是分布式问题的具体工程化场景 |

---

#### M13: 网络 → 各模块

| 依赖 | 知识点 | 为什么 |
|------|--------|--------|
| **M13 → M5 IO/NIO ★** | TCP/IP协议栈→Socket编程、三次握手/四次挥手→连接管理、epoll→IO多路复用 | NIO的Channel+Selector本质是对TCP Socket的封装。epoll是Linux内核提供的IO多路复用机制，是NIO高性能的底层支撑。理解TCP连接的建立和释放才能理解连接池的设计 |

---

### 4.3 按学习阶段的依赖链路

```
Phase 1 [地基] 的模块之间：
  M13 网络 ──────────────→ 无前置依赖（最先学）
  M1 基础 ──────────────→ 无前置依赖（最先学）
  M2 JVM  ←── M1（对象模型）
            ←── M13（TCP/IP，理解网络通信的底层）

Phase 2 [核心] 的模块之间：
  M3 并发 ←── M2（JMM/内存屏障/volatile）
  M4 锁   ←── M3（CAS/AQS/ReentrantLock）
  M5 IO   ←── M13（TCP/Socket）

Phase 3 [框架] 的模块之间：
  M6 Spring ←── M1（反射/注解/动态代理）
             ←── M2（类加载机制）
             ←── M3（并发安全/ThreadLocal）
  M9 MyBatis ←── M6（Spring整合IOC）
              ←── M1（动态代理/反射）

Phase 4 [存储] 的模块之间：
  M7 MySQL ←── M3（事务/锁的概念）
             ←── M5（磁盘IO/索引结构）
  M8 Redis ←── M4（分布式锁基础）
             ←── M5（epoll单线程模型）
             ←── M7（缓存和DB一致性）

Phase 5 [进阶] 的模块之间：
  M10 MQ       ←── M5（零拷贝/顺序写）
                ←── M3（消费者并发）
  M11 微服务   ←── M6（Spring Boot基础）
                ←── M5（RPC/Netty通信）
  M12 分布式   ←── M3（并发控制思想）
                ←── M4（锁在分布式的演进）
                ←── M8（限流/幂等基础设施）
                ←── M7（分布式事务/分库分表）
                ←── M10（消息最终一致性）
                ←── M11（微服务中的分布式问题）
```

**学习路径（按依赖链排序）**：

```
Phase 1 [地基，2-3周]:  M1 Java基础 → M13 网络 → M2 JVM
Phase 2 [核心，3-4周]:  M3 并发 → M4 锁 → M5 IO/NIO
Phase 3 [框架，2-3周]:  M6 Spring → M9 MyBatis
Phase 4 [存储，2-3周]:  M7 MySQL → M8 Redis
Phase 5 [进阶，2-3周]:  M10 MQ → M11 微服务 → M12 分布式
```

---
