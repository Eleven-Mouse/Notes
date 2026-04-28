# Stage 3: 项目映射 — 博客 + AI 系统

> 将每个技术点映射到真实项目场景
> 项目背景：一个支持AI辅助写作的博客平台

---

## 项目架构总览

```
┌─────────────────────────────────────────────────────────────┐
│                    博客 + AI 系统架构                         │
│                                                             │
│  用户 → Nginx → Spring Cloud Gateway                       │
│              ↓ (鉴权/限流/路由)                              │
│         ┌──────────────────────────┐                        │
│         │     博客微服务集群        │                        │
│         │  ┌──────┐  ┌──────┐     │                        │
│         │  │用户服务│ │文章服务│     │                        │
│         │  └──────┘  └──────┘     │                        │
│         │  ┌──────┐  ┌──────┐     │                        │
│         │  │评论服务│ │搜索服务│     │                        │
│         │  └──────┘  └──────┘     │                        │
│         └────────┬─────────────────┘                        │
│                  │                                          │
│         ┌────────▼─────────────────┐                        │
│         │     AI 微服务             │                        │
│         │  ┌──────────────┐       │                        │
│         │  │AI写作辅助     │       │                        │
│         │  │对话管理       │       │                        │
│         │  │内容审核       │       │                        │
│         │  └──────────────┘       │                        │
│         └────────┬─────────────────┘                        │
│                  │                                          │
│    ┌─────┐  ┌────┴────┐  ┌───────┐  ┌──────┐              │
│    │MySQL│  │ Redis   │  │Kafka  │  │ ES   │              │
│    └─────┘  └─────────┘  └───────┘  └──────┘              │
│                                                             │
│    注册中心: Nacos    配置中心: Nacos Config                 │
│    链路追踪: SkyWalking   监控: Prometheus + Grafana        │
└─────────────────────────────────────────────────────────────┘
```

---

## 技术点 → 项目映射表

### 1. Spring MVC（Controller层）⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 用户请求"查看文章详情" |
| **技术点** | DispatcherServlet 9步处理流程 |
| **落地方式** | ArticleController.getArticle(id) |
| **涉及知识** | 请求参数绑定、返回值处理、拦截器、统一异常 |
| **问题场景** | 接口响应慢、参数校验不通过 |
| **面试怎么讲** | "在我们的博客系统中，Controller层统一用RestControllerAdvice做异常处理，通过HandlerInterceptor做鉴权和日志记录。一个请求从进来先经过Filter（编码/跨域），再经过拦截器（鉴权/日志），最后到Controller，通过HttpMessageConverter完成JSON序列化。" |

```java
// 实际代码结构
@RestController
@RequestMapping("/api/articles")
public class ArticleController {

    @GetMapping("/{id}")
    public Result<ArticleVO> getArticle(@PathVariable Long id) {
        return Result.success(articleService.getArticleDetail(id));
    }

    @PostMapping
    public Result<Long> createArticle(@RequestBody @Valid ArticleDTO dto) {
        Long id = articleService.createArticle(dto);
        return Result.success(id);
    }
}

// 统一异常处理
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(BusinessException.class)
    public Result<?> handleBusiness(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }
}
```

### 2. Spring 事务（Service层）⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 发布文章（涉及article表+tag关联表+用户统计表） |
| **技术点** | @Transactional传播行为 + 失效场景 |
| **落地方式** | ArticleService.publishArticle() |
| **涉及知识** | 事务传播、回滚策略、长事务优化 |
| **问题场景** | 发布失败但部分数据已入库、事务超时 |
| **面试怎么讲** | "文章发布涉及多表操作，用REQUIRED传播行为保证原子性。特别注意rollbackFor=Exception.class避免检查异常不回滚。MQ消息发送放在事务提交后（TransactionSynchronization.afterCommit），避免事务回滚但消息已发送的不一致问题。长事务通过拆分读写和异步化来优化。" |

### 3. MyBatis（DAO层）⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 文章的分页查询、动态条件搜索 |
| **技术点** | 动态SQL、分页插件、# vs $ |
| **落地方式** | ArticleMapper.xml |
| **涉及知识** | MyBatis执行流程、一级/二级缓存、N+1问题 |
| **问题场景** | SQL注入（用了${}）、N+1查询、缓存脏读 |
| **面试怎么讲** | "DAO层用MyBatis-Plus简化CRUD，复杂查询手写XML。动态SQL处理多条件搜索（where+if标签）。分页用PageHelper插件，底层通过拦截器改写SQL加limit。严格使用#{}防止SQL注入。关联查询用resultMap避免N+1问题。" |

```xml
<!-- 动态条件搜索 -->
<select id="searchArticles" resultMap="articleResultMap">
    SELECT a.*, u.nickname FROM article a
    LEFT JOIN user u ON a.user_id = u.id
    <where>
        <if test="title != null">
            AND a.title LIKE CONCAT('%', #{title}, '%')
        </if>
        <if test="categoryId != null">
            AND a.category_id = #{categoryId}
        </if>
        <if test="status != null">
            AND a.status = #{status}
        </if>
    </where>
    ORDER BY a.created_at DESC
</select>
```

### 4. MySQL（DB层）⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 文章表设计、索引优化、慢查询排查 |
| **技术点** | B+树索引、MVCC、EXPLAIN |
| **落地方式** | article表索引设计 + SQL调优 |
| **涉及知识** | 聚簇/非聚簇索引、覆盖索引、最左匹配 |
| **问题场景** | 慢查询、死锁、大表DDL |
| **面试怎么讲** | "文章表以id为主键（聚簇索引），对user_id+status建联合索引（查某用户的已发布文章），对category_id+created_at建联合索引（按分类查最新）。通过EXPLAIN发现过全表扫描的SQL，用覆盖索引优化。评论表用分页查询，深分页用延迟关联优化。" |

```sql
-- 深分页优化（延迟关联）
-- 慢的写法：
SELECT * FROM article WHERE status = 1 ORDER BY created_at DESC LIMIT 100000, 20;

-- 优化：先通过覆盖索引拿到id，再回表
SELECT a.* FROM article a
INNER JOIN (
    SELECT id FROM article WHERE status = 1
    ORDER BY created_at DESC LIMIT 100000, 20
) b ON a.id = b.id;
```

### 5. Redis（缓存层）⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 文章详情缓存、热门文章排行、分布式锁 |
| **技术点** | 缓存三大问题、数据类型选型、一致性 |
| **落地方式** | Cache Aside模式 + 延迟双删 |
| **涉及知识** | String/Hash/ZSet、穿透/击穿/雪崩、过期策略 |
| **问题场景** | 缓存击穿（热点文章过期）、一致性延迟 |
| **面试怎么讲** | "文章详情用Redis Hash缓存全字段，TTL=1小时+随机偏移防雪崩。热门文章用ZSet存储（score=浏览量），ZREVRANGE取Top N。缓存击穿用Redisson分布式锁（只允许一个线程重建缓存）。一致性用延迟双删：删缓存→更新DB→延迟500ms再删缓存。极端情况用Canal监听Binlog异步删缓存。" |

### 6. Kafka（异步层）⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 文章发布后通知搜索服务建索引、通知AI服务生成摘要 |
| **技术点** | 消息可靠性、幂等消费、消息顺序 |
| **落地方式** | Spring Kafka + 消费者组 |
| **涉及知识** | 三端保证、acks=all、幂等生产者、offset管理 |
| **问题场景** | 消息丢失、重复消费、消息积压 |
| **面试怎么讲** | "文章发布后通过Kafka通知下游服务。生产者acks=all保证消息不丢失，开启幂等生产者防重复。消费者通过唯一标识（articleId+操作类型）实现幂等，防止重复建索引。消息积压时增加partition和消费者实例数量。Consumer Rebalance导致重复消费，通过业务幂等兜底。" |

```java
// 生产者：文章发布事件
@Transactional(rollbackFor = Exception.class)
public void publishArticle(ArticleDTO dto) {
    Article article = saveArticle(dto);

    // 事务提交后发消息（保证一致性）
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                kafkaTemplate.send("article-events",
                    String.valueOf(article.getId()),
                    JSON.toJSONString(new ArticleEvent("PUBLISHED", article))
                );
            }
        }
    );
}

// 消费者：搜索服务建索引
@KafkaListener(topics = "article-events", groupId = "search-service")
public void handleArticleEvent(ConsumerRecord<String, String> record) {
    ArticleEvent event = JSON.parseObject(record.value(), ArticleEvent.class);
    // 幂等：如果已索引则跳过
    if (searchService.isIndexed(event.getArticleId())) {
        return;
    }
    searchService.indexArticle(event.getArticleId());
}
```

### 7. 并发编程（线程池+锁）⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | AI调用是耗时操作，需要线程池隔离 + 并发控制 |
| **技术点** | ThreadPoolExecutor + Semaphore + 分布式锁 |
| **落地方式** | AI调用独立线程池 + 限流 |
| **涉及知识** | 线程池参数、拒绝策略、CAS、AQS |
| **问题场景** | AI接口慢导致线程池打满、OOM |
| **面试怎么讲** | "AI调用是慢操作（5-30秒），用独立线程池隔离，核心线程4个，队列500。超过队列用CallerRunsPolicy让调用者线程执行，起到降速保护。同时用Redis + Lua实现分布式限流，每用户每分钟最多10次AI调用。用Semaphore控制同时最多4个AI请求，避免资源耗尽。" |

### 8. JVM 调优 ⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 线上AI服务偶尔OOM |
| **技术点** | 堆分析、GC调优、Arthas诊断 |
| **落地方式** | JVM参数配置 + 监控告警 |
| **涉及知识** | 内存区域、GC算法、工具链 |
| **问题场景** | OOM、Full GC频繁、CPU 100% |
| **面试怎么讲** | "AI服务的对话上下文较大，出现过一次OOM。通过-XX:+HeapDumpOnOutOfMemoryError导出dump，MAT分析发现是本地缓存（Caffeine）未设上限，AI上下文对象堆积。加了maximumSize限制后解决。GC用G1，-Xmx4g，MaxGCPauseMillis=200ms，线上观察Young GC约50ms，Mixed GC约150ms。" |

### 9. 微服务组件 ⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 博客服务调用AI服务 |
| **技术点** | 注册中心、网关、Feign、负载均衡 |
| **落地方式** | Nacos + Gateway + OpenFeign |
| **涉及知识** | CAP、服务发现、路由规则、限流 |
| **问题场景** | 服务不可用、雪崩效应 |
| **面试怎么讲** | "微服务架构，Nacos做注册中心（AP模式，支持临时实例心跳检测）。Gateway做网关，统一鉴权和限流。服务间用OpenFeign调用，开启Sentinel做熔断降级。AI服务不可用时熔断，返回'AI服务繁忙'的降级响应。负载均衡用RoundRobin策略。" |

### 10. 分布式事务 ⭐

| 维度 | 映射 |
|------|------|
| **项目场景** | 文章发布需同步更新ES索引 |
| **技术点** | 最终一致性 vs 强一致性 |
| **落地方式** | Kafka消息 + 本地消息表 |
| **涉及知识** | 2PC/TCC/Saga/消息一致性 |
| **问题场景** | ES索引和DB数据不一致 |
| **面试怎么讲** | "文章发布和ES建索引不需要强一致，用Kafka消息实现最终一致性。如果要求更强的一致性（如积分扣减），用Seata AT模式。但Seata有性能开销，大多数场景用消息最终一致性就够了。关键点：消息发送和业务操作在同一个事务中（本地消息表模式），保证消息不丢失。" |

---

## 项目中常见的坑与解决方案

| 坑 | 现象 | 技术点 | 解决 |
|----|------|--------|------|
| 热门文章缓存击穿 | 突发大量请求打DB | Redis缓存 | Redisson分布式锁重建 |
| AI调用超时拖垮线程池 | 接口响应变慢 | 线程池 | 独立线程池隔离 + 超时控制 |
| 事务中发MQ | 事务回滚但消息已发 | Spring事务 | afterCommit中发消息 |
| 树形评论递归StackOverflow | 深度嵌套评论崩溃 | JVM栈 | 改为迭代+层级限制 |
| MySQL深分页慢 | 翻到第5000页要3秒 | MySQL索引 | 延迟关联优化 |
| 同类方法事务失效 | 更新操作不回滚 | Spring AOP | 拆到不同Service |
| ThreadLocal泄漏 | 线程池复用导致用户信息错乱 | ThreadLocal | afterCompletion中remove |
| Kafka消费重复 | Rebalance导致重复建索引 | MQ可靠性 | 业务幂等（唯一键去重） |
