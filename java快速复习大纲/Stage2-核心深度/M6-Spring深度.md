# Stage 2-3: Spring 全家桶 深度增强

> 优先级 ⭐⭐⭐⭐⭐ | 面试频率 🔥🔥🔥🔥🔥

---
## 知识点 1：IOC 与 Bean 生命周期 ⭐🔥

### 【是什么】
IOC（控制反转）将对象的创建和管理权从代码中转移到Spring容器，通过DI（依赖注入）完成对象间的协作。

### 【原理】

#### Bean 生命周期完整流程

```
┌─────────────────────────────────────────────────────────────┐
│                  Spring Bean 完整生命周期                     │
│                                                             │
│  1. 实例化（Instantiation）                                  │
│     → 反射调用构造方法 new()                                 │
│                                                             │
│  2. 属性赋值（Populate）                                     │
│     → @Autowired / @Value 注入依赖                           │
│                                                             │
│  3. Aware接口回调                                            │
│     → BeanNameAware.setBeanName()                           │
│     → BeanFactoryAware.setBeanFactory()                     │
│     → ApplicationContextAware.setApplicationContext()       │
│                                                             │
│  4. BeanPostProcessor前置处理                                │
│     → postProcessBeforeInitialization()                     │
│     → @PostConstruct注解处理                                 │
│     → InitDestroyAnnotationBeanPostProcessor                │
│                                                             │
│  5. 初始化（Initialization）                                 │
│     → afterPropertiesSet()（InitializingBean）              │
│     → custom init-method                                    │
│                                                             │
│  6. BeanPostProcessor后置处理                                │
│     → postProcessAfterInitialization()                      │
│     → AOP代理在此步生成（AbstractAutoProxyCreator）          │
│                                                             │
│  ★ Bean ready for use ★                                    │
│                                                             │
│  7. 销毁（Destruction）                                     │
│     → @PreDestroy                                           │
│     → destroy()（DisposableBean）                           │
│     → custom destroy-method                                 │
└─────────────────────────────────────────────────────────────┘
```

#### BeanFactory vs ApplicationContext

```
BeanFactory：
  - 懒加载（首次getBean时创建）
  - 最基础的容器
  - 不支持AOP/事件/国际化

ApplicationContext：
  - 预加载（启动时创建所有单例Bean）
  - BeanFactory的超集
  - 支持AOP、事件发布、国际化、资源加载
  - 生产中99%用ApplicationContext
```

### 【怎么用 — 项目落地】

**场景：博客系统中Bean的设计**

```java
// Controller → Service → DAO 三层 Bean 依赖
@RestController
public class ArticleController {
    @Autowired
    private ArticleService articleService;  // 注入Service
}

@Service
public class ArticleService {
    @Autowired
    private ArticleMapper articleMapper;    // 注入DAO

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;  // 注入Redis

    @Autowired
    @Qualifier("aiExecutor")                // 指定Bean名称
    private ThreadPoolExecutor aiExecutor;  // 注入线程池
}

// 注意：@Autowired注入的是代理对象（如果有AOP）
// 所以同类中方法调用不会走AOP（代理失效场景）
```

### 【问题场景】

| 问题 | 原因 | 解决 |
|------|------|------|
| 循环依赖 | A依赖B，B依赖A | 三级缓存解决（setter注入） |
| 代理失效 | 同类方法调用 | 注入自身代理/AopContext |
| Bean不存在 | 条件不满足/@Conditional | 检查自动装配条件 |
| 多Bean冲突 | 同类型多个实现 | @Qualifier/@Primary |

### 【面试怎么问】

**Q：说一下Spring Bean的生命周期？**

> "四阶段：实例化→属性赋值→初始化→销毁。实例化通过反射调用构造方法。属性赋值处理@Autowired/@Value注入。初始化阶段依次处理Aware回调→BeanPostProcessor前置→@PostConstruct→afterPropertiesSet→init-method→BeanPostProcessor后置（AOP代理在此生成）。销毁时依次@PreDestroy→destroy→destroy-method。其中BeanPostProcessor是最重要的扩展点，AOP、@Async、@Transactional都通过它实现。"

**追问：BeanPostProcessor和BeanFactoryPostProcessor区别？**

> "BeanFactoryPostProcessor在Bean实例化之前执行，修改BeanDefinition（如PropertyPlaceholderConfigurer处理${}）。BeanPostProcessor在Bean实例化之后执行，包装或替换Bean实例（如AOP生成代理对象）。前者改'图纸'，后者改'成品'。"

---

## 知识点 2：AOP 原理 ⭐🔥

### 【是什么】
AOP（面向切面编程）将横切关注点（日志、事务、权限）从业务代码中分离出来，通过动态代理实现。

### 【原理】

#### JDK动态代理 vs CGLIB
```
┌──────────────┬─────────────────────┬────────────────────┐
│ 维度          │ JDK动态代理          │ CGLIB              │
├──────────────┼─────────────────────┼────────────────────┤
│ 原理         │ 基于接口             │ 基于继承（子类）    │
│ 要求         │ 目标类实现接口        │ 目标类不能是final   │
│ 性能         │ JDK8+比CGLIB快       │ 生成类稍慢，调用快   │
│ Spring默认   │ 有接口用JDK          │ 无接口用CGLIB       │
│ Boot默认     │ 全部CGLIB            │ 全部CGLIB          │
│ 方法限制     │ 只代理接口方法        │ 代理所有public方法  │
└──────────────┴─────────────────────┴────────────────────┘
```

#### 代理失效6种场景（面试必问）

```
场景1：同类方法调用（this.method()不经过代理）
  @Service
  public class ArticleService {
      public void publish() {
          this.log();  // 不走AOP！this是原始对象不是代理
      }
      @Transactional
      public void log() { ... }
  }
  解决：注入自身 / AopContext.currentProxy()

场景2：方法非public（private/protected不代理）

场景3：方法是final/static（无法重写/无法被覆盖）

场景4：Bean未被Spring管理（new出来的不是Spring Bean）

场景5：异步调用（@Async方法在同类调用失效）

场景6：异常被catch（@Transactional内部catch了异常不会回滚）
```

### 【怎么用 — 项目落地】

**场景：博客系统的日志切面**

```java
@Aspect
@Component
public class LogAspect {

    @Around("@annotation(com.blog.annotation.Log)")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        String method = pjp.getSignature().getName();
        Object[] args = pjp.getArgs();
        long start = System.currentTimeMillis();

        try {
            Object result = pjp.proceed();  // 执行目标方法
            long cost = System.currentTimeMillis() - start;
            log.info("method={}, args={}, cost={}ms, result={}", method, args, cost, result);
            return result;
        } catch (Exception e) {
            log.error("method={}, args={}, error={}", method, args, e.getMessage());
            throw e;
        }
    }
}
```

### 【面试怎么问】

**Q：Spring AOP的原理？JDK代理和CGLIB区别？**

> "AOP基于动态代理。如果目标类实现了接口，Spring默认用JDK动态代理（实现相同接口，InvocationHandler拦截）。如果没有接口，用CGLIB（生成子类，MethodInterceptor拦截）。SpringBoot2.x默认全部用CGLIB。代理对象在BeanPostProcessor的后置处理阶段生成。"

**追问：为什么同类方法调用AOP会失效？**

> "因为AOP基于代理对象，而this指向的是原始对象（目标Bean本身）。调用this.method()时直接调用原始对象的方法，不经过代理对象。解决方法有四种：1.注入自身（@Autowired注入自己的代理）2.AopContext.currentProxy()获取当前代理 3.将方法拆到另一个Service 4.通过ApplicationContext.getBean()获取代理。"

---

## 知识点 3：Spring 事务 ⭐🔥

### 【是什么】
Spring 事务是对数据库事务的抽象和管理，通过AOP实现声明式事务控制。

### 【原理】

#### 7种传播行为
```
┌──────────────────┬───────────────────────────────────────────┐
│ 传播行为          │ 说明                                       │
├──────────────────┼───────────────────────────────────────────┤
│ REQUIRED（默认）  │ 有事务加入，无事务新建                      │
│ SUPPORTS         │ 有事务加入，无事务非事务执行                 │
│ MANDATORY        │ 必须在事务中，否则抛异常                    │
│ REQUIRES_NEW     │ 总是新建事务，挂起当前事务                   │
│ NOT_SUPPORTED    │ 非事务执行，挂起当前事务                    │
│ NEVER            │ 非事务执行，存在事务则抛异常                 │
│ NESTED           │ 嵌套事务（Savepoint），外层回滚内层也回滚    │
└──────────────────┴───────────────────────────────────────────┘
```

#### 事务失效6种场景（面试必考）
```
失效1：方法非public（Spring AOP只代理public方法）

失效2：同类方法调用（this.xxx()不经过代理，见AOP代理失效）

失效3：异常被方法内部catch（Spring事务看不到异常就不会回滚）
  @Transactional
  public void update() {
      try {
          articleMapper.update(article);
      } catch (Exception e) {
          log.error(e.getMessage());  // catch了，事务不知道异常
          // 解决：throw new RuntimeException(e) 或手动回滚
      }
  }

失效4：rollbackFor未配置（默认只回滚RuntimeException和Error）
  @Transactional(rollbackFor = Exception.class)  // 必须加！

失效5：数据库引擎不支持（MyISAM不支持事务，必须用InnoDB）

失效6：事务方法所在类未被Spring管理（没有@Component/@Service）
```

### 【怎么用 — 项目落地】

**场景：博客系统的文章发布（涉及多表操作）**

```java
@Service
public class ArticleService {

    @Transactional(rollbackFor = Exception.class)
    public void publishArticle(ArticleDTO dto) {
        // 1. 保存文章（article表）
        Article article = new Article();
        article.setTitle(dto.getTitle());
        article.setContent(dto.getContent());
        articleMapper.insert(article);   // 操作1

        // 2. 保存标签关联（article_tag表）
        tagService.bindTags(article.getId(), dto.getTagIds());  // 操作2

        // 3. 更新用户文章计数（user_stats表）
        userStatsMapper.incrementArticleCount(dto.getUserId());  // 操作3

        // 4. 发送文章发布事件（不走事务，异步）
        // 注意：MQ发送不能放在事务方法里，应放在事务提交后
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronization() {
                @Override
                public void afterCommit() {
                    kafkaTemplate.send("article-published", article);
                }
            }
        );
    }
}
```

### 【面试怎么问】

**Q：Spring事务的传播行为？REQUIRED和REQUIRES_NEW区别？**

> "REQUIRED是默认传播行为，有事务加入没有就新建。REQUIRES_NEW总是新建事务并挂起当前事务。区别：REQUIRED是共用同一个事务，任一回滚全部回滚。REQUIRES_NEW是独立事务，内层回滚不影响外层，外层回滚也不影响已提交的内层。实际场景：日志记录用REQUIRES_NEW，即使业务回滚日志也要保留。"

**追问：Spring事务什么时候会失效？**

> "六种：1.方法非public 2.同类调用（this不经过代理）3.异常被内部catch 4.rollbackFor没配对（检查异常默认不回滚）5.MyISAM引擎 6.没被Spring管理。最容易被忽视的是rollbackFor，必须配成Exception.class，否则IOException等检查异常不会回滚。"

---

## 知识点 4：循环依赖与三级缓存 ⭐🔥

### 【是什么】
Spring 通过三级缓存解决单例Bean的 setter 注入循环依赖问题。

### 【原理】

#### 三级缓存
```java
// Spring DefaultSingletonBeanRegistry
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>();       // 一级：完整Bean
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>();   // 二级：早期引用
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>();      // 三级：ObjectFactory
```

#### 解决流程（A依赖B，B依赖A）
```
1. 创建A → 实例化A（半成品，未注入属性）
2. 将A的ObjectFactory放入三级缓存
   singletonFactories.put("a", () -> getEarlyBeanReference(a))
3. A注入B → 发现B未创建
4. 创建B → 实例化B（半成品）
5. 将B的ObjectFactory放入三级缓存
6. B注入A → 从三级缓存获取A的ObjectFactory
   → 调用getObject()获取A的早期引用（可能是代理对象）
   → A升级到二级缓存，移除三级缓存
7. B初始化完成 → B进入一级缓存
8. A继续注入B（从一级缓存拿B）
9. A初始化完成 → A进入一级缓存

为什么需要三级缓存？
  因为A可能需要AOP代理，而AOP代理在postProcessAfterInitialization中生成
  如果没有三级缓存，A被B引用时就直接返回半成品（无代理）
  三级缓存的ObjectFactory.get()会在早期引用时就创建代理
  保证了B拿到的A是代理对象
```

#### 构造器注入为什么不能解决？
```
因为构造器注入在"实例化"阶段就需要依赖
此时Bean还没创建出来（连半成品都没有），无法放入缓存
所以构造器注入的循环依赖无法自动解决

解决：@Lazy延迟注入
public A(@Lazy B b) {
    this.b = b;  // 注入的是B的代理对象，首次使用时才真正创建
}
```

### 【面试怎么问】

**Q：Spring怎么解决循环依赖？为什么需要三级缓存？**

> "Spring通过三级缓存解决setter注入的循环依赖。一级存完整Bean，二级存早期引用，三级存ObjectFactory。创建A时先实例化半成品，将ObjectFactory放入三级缓存。A注入B时触发B创建，B又需要A，从三级缓存调用ObjectFactory.get()获取A的早期引用，A升级到二级缓存。B完成后进入一级缓存，A继续注入B完成后也进入一级缓存。需要三级缓存是因为要支持AOP：ObjectFactory.get()会判断是否需要代理，保证早期引用也是代理对象。两级缓存无法区分'是否需要代理'这个时机。"

**追问：构造器注入的循环依赖能解决吗？**

> "不能。因为构造器注入在实例化阶段就需要依赖，此时连半成品都没创建出来，无法放入缓存。解决方法是用@Lazy注解在构造参数上，Spring会注入一个代理对象，延迟到首次使用时才真正创建。或者改用setter注入/字段注入。"
