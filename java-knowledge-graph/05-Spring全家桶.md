# Spring 全家桶 — 深度拆解

> 优先级：⭐⭐⭐ | 面试频率：🔥🔥🔥 | 掌握程度：深入

---

## 一、知识树

```
Spring 全家桶
│
├── 1. Spring Core ⭐⭐⭐ 🔥🔥🔥
│   ├── IOC（控制反转）
│   │   ├── BeanFactory vs ApplicationContext
│   │   ├── BeanDefinition 注册
│   │   ├── Bean 的创建与依赖注入
│   │   └── 循环依赖解决（三级缓存）
│   ├── AOP（面向切面编程）
│   │   ├── JDK 动态代理（接口）
│   │   ├── CGLIB 代理（类）
│   │   ├── 切面 / 切点 / 通知
│   │   └── 代理失效场景
│   └── Bean 生命周期
│       ├── 实例化 → 属性填充 → 初始化 → 使用 → 销毁
│       ├── BeanPostProcessor 扩展点
│       └── Aware 接口回调
│
├── 2. Spring MVC ⭐⭐⭐ 🔥🔥
│   ├── 请求处理流程（DispatcherServlet）
│   ├── HandlerMapping / HandlerAdapter
│   ├── 拦截器（HandlerInterceptor）
│   ├── 参数绑定与数据校验
│   └── 统一异常处理
│
├── 3. Spring Boot ⭐⭐⭐ 🔥🔥🔥
│   ├── 自动装配原理（@SpringBootApplication）
│   ├── Starter 机制
│   ├── 自定义 Starter
│   ├── 配置文件加载优先级
│   └── 内嵌 Tomcat 原理
│
├── 4. Spring 事务 ⭐⭐⭐ 🔥🔥🔥
│   ├── @Transactional 核心属性
│   ├── 事务传播行为（7 种）
│   ├── 事务隔离级别
│   ├── 事务失效场景
│   └── 声明式事务底层原理（AOP）
│
└── 5. Spring Cloud（了解级）⭐ 🔥
    ├── Nacos（注册中心 + 配置中心）
    ├── OpenFeign（声明式 HTTP 调用）
    ├── Gateway（网关）
    ├── Sentinel（限流熔断）
    └── Seata（分布式事务）
```

---

## 二、核心知识点精讲

### 2.1 IOC — 控制反转

**核心思想：对象不再自己 new 依赖，由容器注入**

```
传统方式：UserService 自己 new UserDao
IOC 方式：UserService 声明依赖（@Autowired），Spring 容器注入 UserDao

好处：解耦 / 可测试 / 可扩展
```

**BeanFactory vs ApplicationContext：**

| 维度 | BeanFactory | ApplicationContext |
|------|-------------|-------------------|
| Bean 创建时机 | 懒加载（getBean 时创建） | 预加载（容器启动时创建） |
| 国际化 | 不支持 | 支持 |
| 事件发布 | 不支持 | 支持 |
| AOP 集成 | 需手动 | 自动 |
| 实际使用 | 几乎不用 | 常用 |

### 2.2 Bean 生命周期

```
完整流程：

1. BeanDefinition 加载和注册
         │
         ▼
2. 实例化（构造函数）
         │
         ▼
3. 属性填充（@Autowired / setter）
         │
         ▼
4. Aware 接口回调
   (BeanNameAware / BeanFactoryAware / ApplicationContextAware)
         │
         ▼
5. BeanPostProcessor.postProcessBeforeInitialization()
         │
         ▼
6. 初始化方法
   (@PostConstruct / InitializingBean / init-method)
         │
         ▼
7. BeanPostProcessor.postProcessAfterInitialization()
   ← AOP 代理在这里生成
         │
         ▼
8. Bean 就绪，可以使用
         │
         ▼
9. 销毁
   (@PreDestroy / DisposableBean / destroy-method)
```

### 2.3 循环依赖与三级缓存

**问题场景：A 依赖 B，B 依赖 A**

```
三级缓存：

singletonObjects（一级）：完全初始化好的 Bean
earlySingletonObjects（二级）：提前暴露的早期 Bean 引用
singletonFactories（三级）：Bean 的工厂对象（用于生成早期引用）

解决流程（A → B → A）：
1. 创建 A，实例化后将 A 的工厂放入三级缓存
2. A 填充属性时发现需要 B
3. 创建 B，B 填充属性时发现需要 A
4. 从三级缓存获取 A 的工厂，生成 A 的早期引用放入二级缓存
5. B 完成初始化，放入一级缓存
6. A 继续完成初始化，放入一级缓存

注意：构造器注入的循环依赖无法解决（实例化阶段就卡住了）
```

### 2.4 AOP — 面向切面编程

**JDK 动态代理 vs CGLIB：**

| 维度 | JDK 动态代理 | CGLIB |
|------|-------------|-------|
| 要求 | 目标类必须实现接口 | 无要求（不能是 final 类） |
| 原理 | 实现 InvocationHandler | 继承目标类生成子类 |
| 性能 | JDK8+ 接近 CGLIB | 略优 |
| Spring 默认 | 有接口用 JDK | 无接口用 CGLIB |

**代理失效场景（必考）：**

```java
@Service
public class UserService {
    public void methodA() {
        this.methodB();  // ← 代理失效！methodB 的事务不生效
    }

    @Transactional
    public void methodB() {
        // ...
    }
}

// 原因：this 是原始对象，不是代理对象
// 解决：
// 1. 注入自身：@Autowired UserService self; self.methodB();
// 2. AopContext.currentService() 获取当前代理对象
// 3. 将 methodB 拆到另一个 Service
```

### 2.5 Spring MVC 请求流程

```
客户端发送请求
     │
     ▼
DispatcherServlet（前端控制器）
     │
     ├─→ HandlerMapping → 找到对应的 Handler（Controller 方法）
     │
     ├─→ HandlerAdapter → 执行 Handler
     │     │
     │     ├─→ 参数解析（@RequestParam / @RequestBody）
     │     ├─→ 数据校验（@Valid）
     │     ├─→ 执行 Controller 方法
     │     └─→ 返回 ModelAndView 或 @ResponseBody 直接序列化
     │
     ├─→ ViewResolver（如果是视图渲染）→ 渲染页面
     │
     └─→ 返回响应给客户端
```

### 2.6 Spring Boot 自动装配原理

```
@SpringBootApplication 包含三个注解：
  @SpringBootConfiguration  → 本质是 @Configuration
  @ComponentScan            → 包扫描
  @EnableAutoConfiguration  → 核心自动装配

@EnableAutoConfiguration 流程：
1. @Import(AutoConfigurationImportSelector.class)
2. SpringFactoriesLoader.loadFactoryNames()
3. 读取所有 META-INF/spring.factories 中的配置类
4. 根据 @Conditional 系列注解过滤（按条件装配）
   - @ConditionalOnClass（类路径有某个类才装配）
   - @ConditionalOnMissingBean（容器没有某个 Bean 才装配）
   - @ConditionalOnProperty（配置项满足条件才装配）
5. 符合条件的配置类被加载，创建对应的 Bean
```

**自定义 Starter 的步骤：**

```
1. 创建 xxx-spring-boot-starter-autoconfigure 模块
2. 编写配置类（@Configuration + @ConditionalOnXxx）
3. 编写属性类（@ConfigurationProperties）
4. 在 META-INF/spring.factories 中注册自动配置类
5. 创建 xxx-spring-boot-starter 模块（依赖 autoconfigure）
```

### 2.7 Spring 事务

**7 种传播行为（重点记住前 3 个）：**

| 传播行为 | 含义 | 使用场景 |
|---------|------|---------|
| **REQUIRED**（默认） | 有事务就加入，没有就新建 | 大部分场景 |
| **REQUIRES_NEW** | 总是新建事务，挂起当前事务 | 独立日志记录 |
| **NESTED** | 嵌套事务（保存点），回滚不影响外层 | 子操作可选回滚 |
| SUPPORTS | 有事务就加入，没有就非事务执行 | 查询方法 |
| NOT_SUPPORTED | 非事务执行，挂起当前事务 | 不需要事务的操作 |
| MANDATORY | 必须在事务中，否则抛异常 | 强制要求事务 |
| NEVER | 必须非事务，否则抛异常 | 不允许事务的操作 |

**事务失效的 6 种场景（必背）：**

```
1. 方法不是 public（Spring AOP 默认只代理 public 方法）
2. 自调用（this.method()，不走代理）
3. 异常被 catch 吞掉（Spring 感知不到异常，不会回滚）
4. 异常类型不对（默认只回滚 RuntimeException，checked 异常不回滚）
   → 解决：@Transactional(rollbackFor = Exception.class)
5. 数据库引擎不支持（MyISAM 不支持事务，要用 InnoDB）
6. Bean 没有被 Spring 管理（不是 Spring 容器中的 Bean）
```

---

## 三、架构图

### Spring Boot 应用启动流程

```
main()
  │
  ▼
SpringApplication.run()
  │
  ├─→ 创建 Bootstrap Context
  ├─→ 创建 Application Context
  │     │
  │     ├─→ 加载 BeanDefinition
  │     │     ├── @ComponentScan 扫描
  │     │     └── @EnableAutoConfiguration 自动装配
  │     │
  │     ├─→ BeanFactory 后置处理
  │     │     └── 解析 @Configuration / @Bean
  │     │
  │     ├─→ 注册 BeanPostProcessor
  │     │
  │     ├─→ 实例化所有单例 Bean（解决循环依赖）
  │     │
  │     └─→ 容器就绪
  │
  ├─→ 启动内嵌 Tomcat
  │     └── DispatcherServlet 注册
  │
  └─→ 发布 ApplicationReadyEvent
```

---

## 四、高频面试题（10 题）

### Q1：什么是 IOC？Spring 是怎么实现的？
```
IOC（控制反转）：对象的创建和依赖管理交给容器

实现方式：
1. BeanDefinition：描述 Bean 的元数据
2. BeanFactory：Bean 的容器
3. 反射：通过反射创建 Bean 实例
4. 依赖注入：@Autowired / 构造器 / setter
```

### Q2：Spring Bean 的生命周期？
```
简化版：实例化 → 属性填充 → Aware回调 → 初始化前 → 初始化 → 初始化后 → 使用 → 销毁

关键扩展点：
- BeanPostProcessor（初始化前后，AOP 代理在此生成）
- @PostConstruct / @PreDestroy（初始化和销毁回调）
```

### Q3：Spring 是怎么解决循环依赖的？
```
三级缓存：
1. singletonObjects：完整 Bean
2. earlySingletonObjects：早期引用
3. singletonFactories：Bean 工厂

流程：A→B→A 时，A 实例化后暴露工厂到三级缓存，B 需要 A 时从三级缓存获取早期引用

局限：只能解决 setter 注入的循环依赖
不能解决：构造器注入的循环依赖 / 原型(prototype)作用域的循环依赖
```

### Q4：Spring AOP 的实现原理？
```
动态代理：
1. 有接口 → JDK 动态代理（InvocationHandler）
2. 无接口 → CGLIB（生成子类）

核心概念：
- 切面（Aspect）：横切关注点
- 切点（Pointcut）：在哪里切入
- 通知（Advice）：切入后做什么（Before/After/Around/AfterReturning/AfterThrowing）
```

### Q5：Spring Boot 的自动装配原理？
```
@SpringBootApplication
  → @EnableAutoConfiguration
    → @Import(AutoConfigurationImportSelector)
      → 读取 META-INF/spring.factories
        → 加载所有自动配置类
          → @ConditionalOnXxx 条件过滤
            → 符合条件的配置生效
```

### Q6：@Transactional 失效的场景？
```
1. 非 public 方法
2. 同一个类中自调用（不走代理）
3. 异常被 catch 吞掉
4. 异常类型不对（默认只回滚 RuntimeException）
5. 数据库不支持事务（MyISAM）
6. 没被 Spring 管理
```

### Q7：BeanFactory 和 ApplicationContext 的区别？
```
ApplicationContext 是 BeanFactory 的子接口，功能更强大：
1. 国际化支持（MessageSource）
2. 事件发布（ApplicationEvent）
3. AOP 自动集成
4. 资源访问（ResourceLoader）
5. Bean 预加载（启动时创建，而非懒加载）
```

### Q8：Spring 中 @Component 和 @Bean 的区别？
```
@Component：标注在类上，通过 @ComponentScan 扫描注册
@Bean：标注在方法上，方法返回值作为 Bean 注册

@Bean 更灵活：
1. 可以注册第三方类（你无法修改源码加 @Component）
2. 可以控制 Bean 的创建逻辑
3. 可以指定初始化/销毁方法
```

### Q9：Spring Boot Starter 的原理？
```
Starter = 自动配置类 + 依赖管理

1. 引入 Starter 依赖
2. spring.factories 中注册了自动配置类
3. 自动配置类根据 @Conditional 条件创建 Bean
4. @ConfigurationProperties 绑定配置文件属性

本质：约定优于配置 + 条件装配
```

### Q10：DispatcherServlet 的处理流程？
```
1. 客户端请求 → DispatcherServlet
2. DispatcherServlet → HandlerMapping 找 Handler
3. HandlerAdapter 执行 Handler（Controller 方法）
4. Controller 返回结果
5. 如果是 @ResponseBody → HttpMessageConverter 序列化返回
6. 如果是视图 → ViewResolver 渲染页面
7. 返回响应
```

---

## 五、常见误区

| 误区 | 正确理解 |
|------|---------|
| @Autowired 是按类型注入 | 默认按类型，配合 @Qualifier 按名称 |
| prototype Bean 的循环依赖能解决 | 不能，三级缓存只对 singleton 生效 |
| @Transactional 加在 private 方法上也能生效 | 不能，Spring AOP 不代理 private 方法 |
| Spring Boot 只是自动配置 | 还包括内嵌容器、Starter、Actuator 等 |
| @ComponentScan 扫描所有包 | 只扫描主启动类所在包及其子包 |

---

## 六、实战应用场景

| 场景 | Spring 特性 |
|------|-----------|
| 全局异常处理 | @RestControllerAdvice + @ExceptionHandler |
| 统一响应封装 | ResponseBodyAdvice |
| 权限校验 | HandlerInterceptor + AOP |
| 日志记录 | AOP @Around 切面 |
| 多环境配置 | application-{profile}.yml + @Profile |
| 配置加密 | jasypt-spring-boot-starter |

---

## 七、关联模块

- ← [01-Java基础.md](01-Java基础.md)：反射 + 注解是 Spring IOC/AOP 的底层
- → [06-MySQL.md](06-MySQL.md)：Spring 事务与 MySQL 事务隔离级别配合
- → [08-MyBatis.md](08-MyBatis.md)：MyBatis 通过 Spring Boot Starter 集成
- → [10-微服务与分布式.md](10-微服务与分布式.md)：Spring Cloud 基于 Spring Boot
