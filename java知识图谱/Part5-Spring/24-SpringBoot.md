
> **一句话总结：** SpringBoot 的核心是自动装配，理解 spring.factories + @Conditional + Starter 机制就掌握了面试主动权。

---

## 一、知识树

```
SpringBoot
├── @SpringBootApplication 注解拆解
├── 自动装配流程
│   ├── spring.factories
│   ├── @Conditional 系列注解
│   └── Bean 注册
├── Starter 机制
├── 自定义 Starter
├── 配置文件加载优先级
├── @ConfigurationProperties
├── 内嵌 Tomcat 原理
└── Spring Boot 启动流程
```

---

## 二、@SpringBootApplication 注解拆解

```java
@SpringBootApplication
    ├── @SpringBootConfiguration     → 标识这是配置类（等价 @Configuration）
    ├── @EnableAutoConfiguration     → 开启自动配置（核心）
    │   ├── @AutoConfigurationPackage → 扫描主类所在包及子包
    │   └── @Import(AutoConfigurationImportSelector.class)
    │       → 加载 META-INF/spring.factories 中的自动配置类
    └── @ComponentScan               → 组件扫描（扫描 @Component/@Service/@Repository 等）
```

---

## 三、自动装配完整流程

```
启动 main()
   │
   ▼
SpringApplication.run()
   │
   ▼
创建 ApplicationContext
   │
   ▼
@EnableAutoConfiguration
   │
   ▼
AutoConfigurationImportSelector.selectImports()
   │
   ▼
加载 META-INF/spring.factories
（或 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports）
   │
   ▼
获取所有 AutoConfiguration 类（如 DataSourceAutoConfiguration）
   │
   ▼
过滤（去重 + 排除 @SpringBootApplication(exclude)）
   │
   ▼
@Conditional 条件判断
   │
   ├── @ConditionalOnClass        → classpath 中存在某个类？
   ├── @ConditionalOnMissingBean  → 容器中不存在某个 Bean？
   ├── @ConditionalOnProperty     → 配置文件中某个属性匹配？
   └── @ConditionalOnWebApplication → 是 Web 应用？
   │
   ▼
满足条件的 AutoConfiguration → 注册 Bean
```

**自动配置类示例：** `@AutoConfiguration + @ConditionalOnClass(DataSource.class) + @ConditionalOnMissingBean` → classpath 有依赖且用户未自定义时自动创建 DataSource Bean。

---

## 四、@Conditional 系列注解

| 注解 | 条件 | 典型用途 |
|-----|------|---------|
| @ConditionalOnClass | classpath 中存在指定类 | 只有所需依赖存在时才配置 |
| @ConditionalOnMissingClass | classpath 中不存在指定类 | 没有某个依赖时的备选配置 |
| @ConditionalOnBean | 容器中存在指定 Bean | 依赖其他 Bean 时 |
| @ConditionalOnMissingBean | 容器中不存在指定 Bean | 允许用户自定义覆盖默认配置 |
| @ConditionalOnProperty | 配置属性匹配指定值 | 根据配置开关决定是否生效 |
| @ConditionalOnWebApplication | 是 Web 应用 | Web 相关配置 |
| @ConditionalOnNotWebApplication | 不是 Web 应用 | 非 Web 场景配置 |
| @ConditionalOnExpression | SpEL 表达式 | 复杂条件 |

---

## 五、Starter 机制原理

**Starter 本质：** 一组依赖 + 自动配置类的打包。

```
spring-boot-starter-data-redis
├── 依赖（Jedis/Lettuce、Spring Data Redis）
└── 自动配置类（RedisAutoConfiguration）
    └── 在 spring.factories 中注册
```

**常用 Starter 对应表：**

| Starter | 功能 |
|---------|------|
| spring-boot-starter-web | Web 开发（SpringMVC + Tomcat） |
| spring-boot-starter-data-jpa | JPA 数据库访问 |
| spring-boot-starter-data-redis | Redis |
| spring-boot-starter-security | 安全认证 |
| spring-boot-starter-amqp | RabbitMQ |
| spring-boot-starter-actuator | 监控端点 |

---

## 六、自定义 Starter 的完整步骤

### 6.1 创建 Starter 项目

```
my-spring-boot-starter/
├── pom.xml
└── src/main/
    ├── java/com/example/starter/
    │   ├── MyService.java                    → 核心服务
    │   ├── MyServiceProperties.java          → 配置属性
    │   └── MyServiceAutoConfiguration.java   → 自动配置类
    └── resources/
        └── META-INF/
            └── spring.factories              → 注册自动配置类
```

### 6.2 关键代码

```java
// 配置属性
@ConfigurationProperties(prefix = "my.service")
public class MyServiceProperties {
    private String name = "default";
    private int timeout = 3000;
}

// 自动配置类
@AutoConfiguration
@ConditionalOnClass(MyService.class)
@EnableConfigurationProperties(MyServiceProperties.class)
public class MyServiceAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public MyService myService(MyServiceProperties props) {
        return new MyService(props);
    }
}
```

### 6.3 注册自动配置

**方式一（spring.factories，兼容旧版）：**

```properties
# META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.starter.MyServiceAutoConfiguration
```

**方式二（Spring Boot 2.7+ 推荐方式）：**

```
# META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
com.example.starter.MyServiceAutoConfiguration
```

### 6.4 使用：引入依赖 + 配置 `my.service.name=my-app`

---

## 七、配置文件加载优先级

**高优先级 → 低优先级（高优先级覆盖低优先级）：**

```
1. 命令行参数              --server.port=8081
2. JNDI 属性
3. Java 系统属性           -Dserver.port=8081
4. OS 环境变量             SERVER_PORT=8081
5. 外部配置文件
   ├── config/application.yml  （jar 包同级 config 目录）
   ├── application.yml          （jar 包同级目录）
   ├── classpath:config/application.yml
   └── classpath:application.yml
6. @PropertySource 引入的配置
7. 默认属性（SpringApplication.setDefaultProperties）
```

**properties vs yml：** 同级目录下 properties 优先级高于 yml。

**多环境：** `spring.profiles.active=dev` 激活对应配置文件（application-dev.yml）。

---

## 八、@ConfigurationProperties 属性绑定

三种使用方式：`@Component + @ConfigurationProperties`、`@Bean + @ConfigurationProperties`、`@EnableConfigurationProperties`。
支持松散绑定（my-name -> myName）、JSR-303 校验（@Validated）、复杂数据类型（List/Map/Duration）。

---

## 九、内嵌 Tomcat 原理

```
Spring Boot 启动
   │
   ▼
SpringApplication.run()
   │
   ▼
判断是否 Web 应用（classpath 是否有 Servlet API）
   │
   ▼
创建 AnnotationConfigServletWebServerApplicationContext
   │
   ▼
refresh() 过程中调用 onRefresh()
   │
   ▼
ServletWebServerApplicationContext.createWebServer()
   │
   ▼
从容器获取 ServletWebServerFactory（TomcatServletWebServerFactory）
   │
   ▼
TomcatServletWebServerFactory.getWebServer()
   │
   ▼
new Tomcat() → 配置 Connector/Engine/Host/Context
   │
   ▼
注册 DispatcherServlet → Tomcat.start()
```

**切换容器：** 排除 `spring-boot-starter-tomcat`，引入 `spring-boot-starter-undertow`（高并发性能更好）或 `spring-boot-starter-jetty`。

---

## 十、Spring Boot 启动流程

```
1. new SpringApplication() → 推断应用类型 + 加载 Initializer/Listener
2. run()
   ├── starting() 事件
   ├── 准备 Environment（配置文件、环境变量）→ environmentPrepared()
   ├── 创建 ApplicationContext
   ├── 准备上下文 → contextPrepared() / contextLoaded()
   ├── refresh()（核心：Bean 定义注册、自动配置生效）
   ├── afterRefresh() → started() 事件
   ├── 执行 CommandLineRunner / ApplicationRunner
   └── ready() 事件
```

---

## 十一、常见误区

| 误区 | 正解 |
|-----|------|
| SpringBoot 是新框架 | 它是 Spring 的脚手架，底层还是 Spring |
| 自动配置不可覆盖 | @ConditionalOnMissingBean 允许用户自定义覆盖 |
| Starter 就是依赖包 | Starter = 依赖 + 自动配置类 + spring.factories 注册 |
| SpringBoot 只能用 yml | properties、yml、JSON、环境变量都支持 |
| @Value 和 @ConfigurationProperties 完全等价 | 后者支持松散绑定、JSR-303 校验、批量绑定 |

---

## 十二、面试题

### Q1：SpringBoot 自动装配的原理是什么？
**答：** @EnableAutoConfiguration 通过 AutoConfigurationImportSelector 加载 META-INF/spring.factories 中注册的所有自动配置类，然后通过 @ConditionalOnClass/@ConditionalOnMissingBean 等条件注解过滤，只有满足条件的配置类才会注册 Bean。用户可通过自定义 Bean 覆盖默认配置（@ConditionalOnMissingBean）。

### Q2：@SpringBootApplication 包含哪些注解？
**答：** 包含 @SpringBootConfiguration（等价 @Configuration）、@EnableAutoConfiguration（开启自动配置）、@ComponentScan（组件扫描）。主类所在包及子包都会被扫描。

### Q3：如何自定义一个 Starter？
**答：** 1) 创建配置属性类（@ConfigurationProperties）；2) 创建核心服务类；3) 创建自动配置类（@AutoConfiguration + @ConditionalOnClass + @ConditionalOnMissingBean）；4) 在 spring.factories 或 AutoConfiguration.imports 中注册自动配置类；5) 打包后引入依赖即可使用。

### Q4：@ConditionalOnMissingBean 的作用是什么？
**答：** 当容器中不存在指定类型的 Bean 时才创建。这是 SpringBoot 的设计哲学：约定大于配置，提供默认实现，同时允许用户自定义覆盖。

### Q5：Spring Boot 配置文件的加载优先级是什么？
**答：** 命令行参数 > 系统属性 > 环境变量 > 外部配置文件（config/ > 根目录 > classpath:config/ > classpath:/）> 默认属性。高优先级覆盖低优先级。

### Q6：Spring Boot 的内嵌 Tomcat 是怎么启动的？
**答：** Spring Boot 判断是 Web 应用后创建 ServletWebServerApplicationContext，在 refresh() 的 onRefresh() 阶段调用 createWebServer()，通过 TomcatServletWebServerFactory 创建并启动内嵌 Tomcat，同时注册 DispatcherServlet。

### Q7：@ConfigurationProperties 和 @Value 有什么区别？
**答：** @ConfigurationProperties 支持松散绑定（my-name → myName）、JSR-303 校验（@Validated）、批量绑定所有属性、复杂数据类型（List/Map/Duration）。@Value 只能逐个绑定，不支持松散绑定和校验，适合少量配置。

### Q8：Spring Boot 2.7+ 注册自动配置的方式有什么变化？
**答：** 推荐使用 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 文件替代 spring.factories。新方式一个类一行，更清晰，且不会和其他 SPI 配置混在一起。

---

## 十三、关联知识

- [21-IOC与Bean](21-IOC与Bean.md) — 自动配置本质是注册 Bean
- [22-AOP](22-AOP.md) — @Conditional 等注解底层用到条件判断
- [23-SpringMVC](23-SpringMVC.md) — Spring Boot 自动配置 MVC 组件
- [Part7-中间件/32-Tomcat.md](../Part7-中间件/32-Tomcat.md) — 内嵌 Tomcat 的工作原理
