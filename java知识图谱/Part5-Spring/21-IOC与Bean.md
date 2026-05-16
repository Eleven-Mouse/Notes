
> **一句话总结：** IOC 是 Spring 的灵魂，Bean 的生命周期和依赖注入是面试必考的核心。

---

## 一、知识树

```
IOC 与 Bean
├── IOC 核心思想（控制反转 / 依赖注入）
├── BeanFactory vs ApplicationContext
├── BeanDefinition 注册流程
├── 依赖注入方式（构造器/setter/字段）
├── @Autowired 注入原理
├── @Autowired vs @Resource
├── @Component vs @Bean
└── Bean 作用域
```

---

## 二、IOC 核心思想

### 2.1 控制反转（Inversion of Control）

**传统方式（主动获取依赖）：**

```java
public class OrderService {
    private OrderRepository orderRepo = new OrderRepositoryImpl(); // 主动创建
    private UserService userRepo = new UserServiceImpl();          // 强耦合
}
```

**IOC 方式（被动接收依赖）：**

```java
public class OrderService {
    @Autowired
    private OrderRepository orderRepo; // Spring 注入，不关心实现
}
```

**控制反转的含义：** 对象的创建和依赖关系的管理从代码中**转移**到了 Spring 容器，对象本身变为被动接收。

### 2.2 依赖注入（Dependency Injection）

IOC 是思想，DI 是实现方式。Spring 通过以下三种方式实现 DI：

| 注入方式 | 示例 | 推荐度 |
|---------|------|-------|
| **构造器注入** | `public Service(Repo repo)` | 推荐（强制依赖、不可变） |
| **Setter 注入** | `public void setRepo(Repo repo)` | 可选（可选依赖） |
| **字段注入** | `@Autowired private Repo repo` | 不推荐（隐藏依赖、不可测试） |

```java
// 推荐：构造器注入（Spring 官方推荐）
@Service
public class OrderService {
    private final OrderRepository orderRepo;

    public OrderService(OrderRepository orderRepo) {
        this.orderRepo = orderRepo;
    }
}
```

---

## 三、BeanFactory vs ApplicationContext

| 对比维度 | BeanFactory | ApplicationContext |
|---------|------------|-------------------|
| **Bean 创建时机** | 懒加载（调用时创建） | 预加载（启动时创建） |
| **国际化** | 不支持 | 支持 MessageSource |
| **事件发布** | 不支持 | 支持 ApplicationEvent |
| **AOP 集成** | 需手动处理 | 自动集成 |
| **资源访问** | 不支持 | 支持 ResourceLoader |
| **BeanPostProcessor** | 需手动注册 | 自动注册 |
| **使用场景** | 资源受限环境 | 绝大多数场景 |

**ApplicationContext 继承体系：**

```
ApplicationContext
├── MessageSource         → 国际化
├── ResourcePatternResolver → 资源访问
├── ApplicationEventPublisher → 事件发布
└── EnvironmentCapable    → 环境配置
```

---

## 四、BeanDefinition 注册流程

```
配置文件/注解扫描
       │
       ▼
BeanDefinitionReader（解析配置）
       │
       ▼
BeanDefinition（Bean 的元数据描述）
       │
       ▼
BeanDefinitionRegistry（注册到容器）
       │
       ▼
BeanFactory（存储 BeanDefinition）
       │
       ▼
getBean() 时实例化
```

**BeanDefinition 包含的信息：**
- Bean 的 Class 全限定名
- 作用域（scope）
- 是否懒加载
- 依赖关系
- 初始化/销毁方法
- 构造函数参数、属性值

---

## 五、@Autowired 注入原理

### 5.1 注入流程

```
@Autowired 注入流程：
1. 按类型（byType）从容器中查找匹配的 Bean
2. 找到 0 个 → 报错 NoSuchBeanDefinitionException（required=true 时）
3. 找到 1 个 → 直接注入
4. 找到多个 → 按名称（byName）筛选
   │
   ├── 字段名/参数名与某个 Bean 名称匹配 → 注入该 Bean
   ├── 不匹配 → 检查 @Qualifier 指定的名称
   └── 都不匹配 → 报错 NoUniqueBeanDefinitionException
```

```java
// 多个实现时用 @Qualifier 指定
@Autowired
@Qualifier("mysqlOrderRepo")
private OrderRepository orderRepo;

// 允许不注入（非必须依赖）
@Autowired(required = false)
private Optional<CacheService> cacheService;
```

---

## 六、@Autowired vs @Resource 区别

| 对比维度 | @Autowired | @Resource |
|---------|-----------|-----------|
| **来源** | Spring 框架 | JSR-250（Java 标准） |
| **匹配方式** | 先 byType，后 byName | 先 byName，后 byType |
| **指定名称** | @Qualifier("name") | @Resource(name="name") |
| **required 属性** | 支持（required=false） | 不支持 |
| **适用位置** | 构造器/setter/字段 | setter/字段（不支持构造器） |

```java
// @Resource 先按名称 "orderRepo" 查找
@Resource
private OrderRepository orderRepo; // 先找 name="orderRepo" 的 Bean

// @Autowired 先按类型 OrderRepository 查找
@Autowired
private OrderRepository orderRepo; // 先找类型为 OrderRepository 的 Bean
```

---

## 七、@Component vs @Bean 区别

| 对比维度 | @Component | @Bean |
|---------|-----------|-------|
| **使用位置** | 类上 | @Configuration 类的方法上 |
| **Bean 来源** | Spring 自动扫描 | 手动编码创建 |
| **适用场景** | 自己写的类 | 第三方库的类（无法加注解） |
| **命名** | 类名首字母小写 | 方法名即为 Bean 名称 |
| **自定义创建逻辑** | 不支持 | 支持（方法内可写任意逻辑） |

```java
// @Component：自己写的类
@Component
public class MyService { }

// @Bean：第三方类
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(5))
            .build();
    }
}
```

---

## 八、Bean 的作用域

| 作用域 | 说明 | 创建时机 |
|-------|------|---------|
| **singleton**（默认） | 单例，整个容器只有一个 | 容器启动时 |
| **prototype** | 每次获取创建新实例 | 每次请求时 |
| **request** | 每个 HTTP 请求一个实例 | Web 环境 |
| **session** | 每个 HTTP Session 一个实例 | Web 环境 |
| **application** | 每个 ServletContext 一个 | Web 环境 |

```java
@Component
@Scope("prototype")
public class PrototypeBean { }

// 或使用 @RequestScope / @SessionScope
@Component
@RequestScope
public class RequestScopedBean { }
```

### singleton 中注入 prototype 的问题

```java
// 问题：单例 Bean 中的 prototype Bean 只会注入一次
@Component
public class SingletonBean {
    @Autowired
    private PrototypeBean prototypeBean; // 始终是同一个实例！

    // 解决方案1：使用 @Lookup
    @Lookup
    public PrototypeBean getPrototypeBean() {
        return null; // Spring 会覆盖这个方法
    }

    // 解决方案2：注入 ObjectFactory/Provider
    @Autowired
    private ObjectFactory<PrototypeBean> prototypeBeanFactory;

    public void doSomething() {
        PrototypeBean bean = prototypeBeanFactory.getObject(); // 每次获取新实例
    }
}
```

---

## 九、常见误区

| 误区 | 正解 |
|-----|------|
| @Autowired 是按名称注入 | 先按类型，类型有多个才按名称 |
| @Resource 是按类型注入 | 先按名称，找不到才按类型 |
| @Component 和 @Bean 完全等价 | @Bean 可以自定义创建逻辑，适合第三方类 |
| singleton 是线程安全的 | 单例只是实例唯一，内部状态仍需考虑线程安全 |
| @Autowired(required=false) 就不会报错 | 不报错但注入为 null，使用时可能 NPE |
| Bean 默认懒加载 | ApplicationContext 默认预加载（Eager） |

---

## 十、实战场景

**场景：多数据源配置**

```java
@Configuration
public class DataSourceConfig {

    @Bean("primaryDataSource")
    @Primary
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean("secondaryDataSource")
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
}

// 使用时指定具体的数据源
@Service
public class OrderService {
    @Autowired
    @Qualifier("secondaryDataSource")
    private DataSource dataSource;
}
```

---

## 十一、面试题

### Q1：什么是 IOC？Spring 是怎么实现的？
**答：** IOC（控制反转）是将对象的创建和依赖管理从代码转移到容器的思想。Spring 通过 DI（依赖注入）实现 IOC，支持构造器、Setter、字段三种注入方式。底层通过 BeanDefinition 描述 Bean 元数据，BeanFactory 负责创建和管理。

### Q2：@Autowired 和 @Resource 的区别？
**答：** @Autowired 是 Spring 注解，先按类型匹配再按名称；@Resource 是 JSR-250 标准注解，先按名称匹配再按类型。@Autowired 配合 @Qualifier 指定名称，@Resource 用 name 属性指定。

### Q3：@Component 和 @Bean 的区别？
**答：** @Component 加在类上，通过组件扫描自动注册；@Bean 加在方法上，手动创建并注册 Bean。@Bean 适合注册第三方库的类，因为无法在源码中加 @Component。

### Q4：singleton Bean 注入 prototype Bean 有什么问题？
**答：** singleton Bean 只实例化一次，其中的 prototype Bean 也只注入一次，后续调用始终是同一个实例。解决方案：使用 @Lookup 方法、ObjectFactory 或 Provider 每次获取新的 prototype 实例。

### Q5：BeanFactory 和 ApplicationContext 的区别？
**答：** BeanFactory 是基础容器，懒加载，不自动注册 BeanPostProcessor；ApplicationContext 是增强容器，预加载，自动支持国际化、事件发布、AOP 集成等。实际开发都用 ApplicationContext。

### Q6：@Autowired 注入多个同类型 Bean 怎么办？
**答：** 三种方式：1) @Qualifier 指定名称；2) 字段名与 Bean 名称一致；3) 其中一个加 @Primary 标记为首选。

### Q7：为什么推荐构造器注入？
**答：** 1) 依赖不可变（final 字段）；2) 依赖不为 null（构造时强制传入）；3) 便于单元测试（直接 new 传入 mock）；4) 防止循环依赖（启动时报错而不是运行时）。

### Q8：Bean 的生命周期是怎样的？
**答：** 实例化→属性赋值→BeanNameAware/BeanFactoryAware→BeanPostProcessor.postProcessBeforeInitialization→InitializingBean.afterPropertiesSet→init-method→BeanPostProcessor.postProcessAfterInitialization→使用→DisposableBean.destroy→destroy-method。

---

## 十二、关联知识

- [22-AOP](22-AOP.md) — AOP 依赖 IOC 容器
- [24-SpringBoot](24-SpringBoot.md) — 自动装配基于 Bean 注册
- [25-事务](25-事务.md) — @Transactional 的 Bean 被 AOP 代理
- [26-循环依赖](26-循环依赖.md) — IOC 解决循环依赖的三级缓存
