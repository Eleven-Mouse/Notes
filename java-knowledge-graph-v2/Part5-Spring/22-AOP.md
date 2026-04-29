
> **优先级：** ⭐⭐⭐⭐⭐ | **面试频率：** 🔥🔥🔥🔥🔥
> **一句话总结：** AOP 是 Spring 框架的两大核心之一，代理模式和代理失效场景是面试重点。

---

## 一、知识树

```
AOP
├── 核心概念（切面/切点/通知/连接点/织入）
├── 五种通知类型
├── JDK 动态代理
├── CGLIB 代理
├── Spring 代理选择规则
├── 代理失效的 6 种场景
└── AOP 实际应用场景
```

---

## 二、AOP 核心概念

| 概念 | 英文 | 说明 | 类比 |
|-----|------|------|------|
| **切面** | Aspect | 横切关注点的模块化（日志切面、事务切面） | 一个功能模块 |
| **切点** | Pointcut | 定义通知在哪些方法上生效（表达式匹配） | 哪里生效 |
| **通知** | Advice | 切面在切点上执行的动作（before/after 等） | 做什么 |
| **连接点** | JoinPoint | 程序执行的某个点（方法调用、异常抛出） | 具体位置 |
| **织入** | Weaving | 将切面应用到目标对象的过程 | 组装过程 |

```java
@Aspect
@Component
public class LogAspect {

    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    @Before("serviceLayer()")
    public void before(JoinPoint jp) {
        log.info("调用: {}", jp.getSignature());
    }

    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void afterReturning(JoinPoint jp, Object result) {
        log.info("返回: {}", result);
    }
}
```

---

## 三、五种通知类型

| 通知类型 | 注解 | 执行时机 | 能否修改返回值 | 能否捕获异常 |
|---------|------|---------|-------------|------------|
| **前置通知** | @Before | 方法执行前 | 否 | 否 |
| **后置通知** | @AfterReturning | 方法正常返回后 | 是 | 否 |
| **异常通知** | @AfterThrowing | 方法抛出异常时 | 否 | 是 |
| **最终通知** | @After | 方法执行后（无论是否异常） | 否 | 否 |
| **环绕通知** | @Around | 包裹整个方法执行 | 是 | 是 |

**环绕通知最强大，可以完全控制方法执行：**

```java
@Around("serviceLayer()")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    long start = System.currentTimeMillis();

    try {
        Object result = pjp.proceed();  // 执行目标方法
        log.info("耗时: {}ms", System.currentTimeMillis() - start);
        return result;                  // 可以修改返回值
    } catch (Exception e) {
        log.error("异常: {}", e.getMessage());
        throw e;                        // 可以吞掉或转换异常
    }
}
```

**执行顺序（Spring 5.2.7+）：**

```
@Around (前半段)
  → @Before
    → 目标方法
  → @AfterReturning / @AfterThrowing
  → @After
@Around (后半段)
```

---

## 四、JDK 动态代理

### 4.1 原理

基于**接口**的代理，运行时动态生成一个实现了相同接口的代理类。

```java
// 目标接口
public interface UserService {
    User getUser(Long id);
}

// 目标实现
public class UserServiceImpl implements UserService {
    public User getUser(Long id) {
        return userRepo.findById(id);
    }
}

// JDK 动态代理
public class JdkProxy implements InvocationHandler {

    private final Object target;

    public JdkProxy(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 前置增强
        System.out.println("Before: " + method.getName());
        // 执行目标方法
        Object result = method.invoke(target, args);
        // 后置增强
        System.out.println("After: " + method.getName());
        return result;
    }
}

// 创建代理对象
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserServiceImpl.class.getClassLoader(),
    UserServiceImpl.class.getInterfaces(),
    new JdkProxy(new UserServiceImpl())
);
```

**JDK 动态代理的局限：** 只能代理接口方法，不能代理类方法。

---

## 五、CGLIB 代理

### 5.1 原理

基于**继承**的代理，运行时动态生成目标类的子类，重写方法实现增强。

```java
// CGLIB 代理
public class CglibProxy implements MethodInterceptor {

    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                           MethodProxy proxy) throws Throwable {
        // 前置增强
        System.out.println("Before: " + method.getName());
        // 执行目标方法（调用父类方法）
        Object result = proxy.invokeSuper(obj, args);
        // 后置增强
        System.out.println("After: " + method.getName());
        return result;
    }
}

// 创建代理对象
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(UserServiceImpl.class);  // 继承目标类
enhancer.setCallback(new CglibProxy());
UserServiceImpl proxy = (UserServiceImpl) enhancer.create();
```

**CGLIB 的局限：** 不能代理 final 类和 final 方法（不能继承/重写）。

---

## 六、JDK 动态代理 vs CGLIB

| 对比维度 | JDK 动态代理 | CGLIB 代理 |
|---------|------------|-----------|
| **代理方式** | 实现接口 | 继承目标类 |
| **要求** | 目标类必须实现接口 | 目标类不能是 final |
| **性能（创建）** | 快 | 慢（生成子类） |
| **性能（调用）** | 略慢 | 略快 |
| **代理对象类型** | 与目标类同接口 | 目标类的子类 |

### Spring 代理选择规则

```
目标类实现了接口？
  ├── 是 → 默认使用 JDK 动态代理
  │       （但 Spring Boot 2.x 默认 proxy-target-class=true，即用 CGLIB）
  └── 否 → 使用 CGLIB 代理

配置 spring.aop.proxy-target-class=true → 强制使用 CGLIB
```

---

## 七、代理失效的 6 种场景（高频考点）

| 场景 | 原因 | 解决方案 |
|-----|------|---------|
| **1. 同类内部方法调用** | 调用的是 this 而非代理对象 | 注入自身 / AopContext.currentProxy() |
| **2. 方法是 private** | 不可被重写/代理 | 改为 public/protected |
| **3. 方法是 static** | 静态方法属于类，不参与代理 | 改为实例方法 |
| **4. 方法是 final** | 不可被重写（CGLIB）/ 不可被代理 | 去掉 final |
| **5. 内部 Bean 未被 Spring 管理** | 不是 Spring Bean，不会被代理 | 注册为 Spring Bean |
| **6. 异常被吞掉** | 事务通知捕获不到异常，不回滚 | 正确抛出异常 |

### 7.1 同类内部方法调用（最常见的坑）

```java
@Service
public class UserService {

    public void methodA() {
        // 这里调用的是 this.methodB()，不是代理对象！
        // 所以 methodB 上的 @Transactional 不会生效
        this.methodB();
    }

    @Transactional
    public void methodB() {
        // 事务不生效！
    }
}

// 解决方案1：注入自身
@Service
public class UserService {
    @Autowired
    @Lazy
    private UserService self;

    public void methodA() {
        self.methodB();  // 通过代理对象调用
    }
}

// 解决方案2：使用 AopContext
@Service
public class UserService {
    public void methodA() {
        ((UserService) AopContext.currentProxy()).methodB();
    }
}
// 需要配置：@EnableAspectJAutoProxy(exposeProxy = true)
```

---

## 八、AOP 实际应用场景

| 场景 | 说明 |
|-----|------|
| **日志记录** | 统一记录方法入参、返回值、耗时 |
| **权限校验** | 自定义注解 + 切面，校验用户权限 |
| **事务管理** | @Transactional 底层就是 AOP |
| **性能监控** | 统计方法执行时间 |
| **限流** | 自定义注解 + 切面 + RateLimiter |
| **缓存** | @Cacheable 底层就是 AOP |
| **分布式锁** | 自定义注解 + 切面 + Redis 锁 |

```java
// 自定义注解 + AOP 实现限流
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int permits() default 10;
    int seconds() default 1;
}

@Aspect
@Component
public class RateLimitAspect {

    @Around("@annotation(rateLimit)")
    public Object around(ProceedingJoinPoint pjp, RateLimit rateLimit) throws Throwable {
        // 限流逻辑
        if (!rateLimiter.tryAcquire(rateLimit.permits(), rateLimit.seconds())) {
            throw new RateLimitException("请求过于频繁");
        }
        return pjp.proceed();
    }
}
```

---

## 九、常见误区

| 误区 | 正解 |
|-----|------|
| AOP 只能用于日志 | 事务、缓存、权限、限流等都是 AOP 的应用 |
| Spring 默认用 JDK 动态代理 | Spring Boot 2.x 默认用 CGLIB |
| @Around 可以替代所有通知 | 技术上可以，但语义不清晰，应按需选择 |
| private 方法也能被 AOP 增强 | 不能，private 方法不可被重写/代理 |
| final 方法不影响 AOP | final 方法不可被重写，CGLIB 无法代理 |

---

## 十、面试题

### Q1：Spring AOP 的实现原理是什么？
**答：** 基于动态代理实现。有接口时默认使用 JDK 动态代理（Proxy + InvocationHandler），无接口时使用 CGLIB（生成子类 + MethodInterceptor）。Spring Boot 2.x 默认使用 CGLIB。

### Q2：JDK 动态代理和 CGLIB 的区别？
**答：** JDK 基于 interface，生成实现接口的代理类；CGLIB 基于 inheritance，生成目标类的子类。JDK 要求目标类必须实现接口，CGLIB 要求目标类不能是 final。CGLIB 创建慢但调用快。

### Q3：什么是代理失效？哪些场景会失效？
**答：** 代理失效是指 AOP 增强不生效。常见场景：同类内部调用（this 而非代理对象）、private/static/final 方法、目标对象不是 Spring Bean。最常见的是同类内部调用，解决方法是注入自身或使用 AopContext.currentProxy()。

### Q4：@Around 和 @Before 有什么区别？
**答：** @Before 只能在方法前执行逻辑，不能控制方法是否执行。@Around 包裹整个方法，可以决定是否执行目标方法（proceed()）、修改参数、修改返回值、捕获异常。@Around 功能最强大但应谨慎使用。

### Q5：Spring AOP 和 AspectJ 的区别？
**答：** Spring AOP 是运行时代理（只对 Spring Bean 生效），AspectJ 是编译时/加载时织入（对所有类生效）。Spring AOP 只支持方法级别的切点，AspectJ 支持字段、构造器等更细粒度的切点。

### Q6：事务注解 @Transactional 在同类方法调用时为什么失效？
**答：** 因为同类内部调用使用的是 this 而非代理对象，绕过了 AOP 代理层，所以事务增强不生效。解决方法是通过注入自身或 AopContext.currentProxy() 获取代理对象来调用。

### Q7：CGLIB 为什么不能代理 final 方法？
**答：** CGLIB 通过生成目标类的子类来创建代理，子类需要重写父类方法来添加增强逻辑。final 方法不能被重写，因此无法代理。

### Q8：如何选择通知类型？
**答：** 只需前置逻辑用 @Before；需要返回值用 @AfterReturning；需要异常处理用 @AfterThrowing；需要完全控制（含修改返回值/捕获异常）用 @Around；需要类似 finally 的逻辑用 @After。优先使用语义明确的通知。

---

## 十一、关联知识

- [21-IOC与Bean](./21-IOC与Bean.md) — AOP 依赖 IOC 容器管理 Bean
- [25-事务](./25-事务.md) — @Transactional 底层是 AOP 实现
- [24-SpringBoot](./24-SpringBoot.md) — Spring Boot 的自动配置依赖 AOP
- [Part1-Java基础/03-动态代理.md](../Part1-Java基础/03-动态代理.md) — JDK 动态代理和 CGLIB 的底层原理
