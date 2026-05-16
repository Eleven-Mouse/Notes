
> **一句话总结：** DispatcherServlet 处理流程是 SpringMVC 的核心，拦截器和统一异常处理是实际开发必备技能。

---

## 一、知识树

```
SpringMVC
├── DispatcherServlet 处理流程（9 步）
├── HandlerMapping / HandlerAdapter / ViewResolver
├── @RequestBody / @ResponseBody 原理
├── 拦截器 HandlerInterceptor
├── Filter vs Interceptor 区别
├── 统一异常处理
├── 统一响应封装
├── 参数校验
└── RESTful API 设计规范
```

---

## 二、DispatcherServlet 处理流程（完整 9 步）

```
客户端请求
   │
   ▼
① DispatcherServlet 接收请求
   │
   ▼
② HandlerMapping 查找 Handler（根据 URL 映射到 Controller 方法）
   │
   ▼
③ HandlerAdapter 适配 Handler（适配不同类型的 Handler）
   │
   ▼
④ 执行拦截器 preHandle()
   │
   ▼
⑤ 执行 Handler（Controller 方法）
   │
   ▼
⑥ ModelAndView 返回（逻辑视图名 + 模型数据）
   │
   ▼
⑦ ViewResolver 解析视图（逻辑名 → 物理视图 / JSON 响应）
   │
   ▼
⑧ 渲染视图（填充模型数据）或 直接写 JSON
   │
   ▼
⑨ 执行拦截器 afterCompletion()，响应客户端
```

### 核心组件说明

| 组件 | 职责 | 常见实现 |
|-----|------|---------|
| **DispatcherServlet** | 前端控制器，统一调度 | FrameworkServlet |
| **HandlerMapping** | URL → Handler 映射 | RequestMappingHandlerMapping |
| **HandlerAdapter** | 适配执行 Handler | RequestMappingHandlerAdapter |
| **ViewResolver** | 视图名 → 物理视图 | InternalResourceViewResolver |
| **HandlerExceptionResolver** | 异常处理 | DefaultHandlerExceptionResolver |

---

## 三、@RequestBody / @ResponseBody 原理

### 3.1 HttpMessageConverter

```
请求 JSON → HttpMessageConverter → Java 对象（@RequestBody）
Java 对象 → HttpMessageConverter → 响应 JSON（@ResponseBody）
```

**常用 HttpMessageConverter：**

| Converter | 处理类型 | Content-Type |
|-----------|---------|-------------|
| MappingJackson2HttpMessageConverter | JSON | application/json |
| StringHttpMessageConverter | String | text/plain |
| ByteArrayHttpMessageConverter | byte[] | application/octet-stream |
| FormHttpMessageConverter | 表单 | application/x-www-form-urlencoded |

```java
@RestController  // = @Controller + @ResponseBody
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        // @ResponseBody 自动将 User 序列化为 JSON
        return userService.findById(id);
    }

    @PostMapping
    public User createUser(@RequestBody @Valid UserDTO dto) {
        // @RequestBody 自动将请求体 JSON 反序列化为 UserDTO
        return userService.create(dto);
    }
}
```

---

## 四、拦截器 HandlerInterceptor

```java
@Component
public class AuthInterceptor implements HandlerInterceptor {

    // 在 Handler 执行前调用（最常用：权限校验、登录检查）
    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response,
                            Object handler) {
        String token = request.getHeader("Authorization");
        if (token == null || !jwtUtil.validate(token)) {
            response.setStatus(401);
            return false;  // 中断请求
        }
        return true;  // 继续执行
    }

    // 在 Handler 执行后、视图渲染前调用
    @Override
    public void postHandle(HttpServletRequest request,
                          HttpServletResponse response,
                          Object handler, ModelAndView modelAndView) {
        // 可以修改 ModelAndView
    }

    // 在请求完成后调用（无论是否异常，用于清理资源）
    @Override
    public void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler, Exception ex) {
        // 日志记录、资源清理
    }
}
```

**注册拦截器：**

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private AuthInterceptor authInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/login", "/api/register");
    }
}
```

---

## 五、Filter vs Interceptor 区别

| 对比维度 | Filter（过滤器） | Interceptor（拦截器） |
|---------|-----------------|---------------------|
| **规范** | Servlet 规范 | Spring 规范 |
| **作用范围** | 所有请求（包括静态资源） | 只拦截 Handler（Controller） |
| **执行时机** | 在 DispatcherServlet 之前 | 在 DispatcherServlet 之后 |
| **能否获取 Handler 信息** | 不能 | 能 |
| **能否注入 Spring Bean** | 可以（但需要配置） | 天然支持 |
| **典型用途** | 编码、跨域、压缩 | 权限校验、日志、登录检查 |

**执行顺序：**

```
请求 → Filter → DispatcherServlet → Interceptor.preHandle → Handler
                                                    ↓
响应 ← Filter ← DispatcherServlet ← Interceptor.afterCompletion ← Handler
```

---

## 六、统一异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 处理业务异常
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusiness(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }

    // 处理参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidation(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining("; "));
        return Result.fail(400, message);
    }

    // 处理所有未捕获异常
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(500, "系统内部错误");
    }
}
```

---

## 七、统一响应封装（ResponseBodyAdvice）

```java
@Data
public class Result<T> {
    private int code; private String message; private T data;
    public static <T> Result<T> ok(T data) { /* 200 + data */ }
    public static <T> Result<T> fail(int code, String msg) { /* code + msg */ }
}

// ResponseBodyAdvice 自动封装所有 Controller 返回值
@RestControllerAdvice(basePackages = "com.example.controller")
public class ResponseAdvice implements ResponseBodyAdvice<Object> {
    public boolean supports(MethodParameter rt, Class<?> ct) {
        return !rt.getParameterType().equals(Result.class); // 避免重复封装
    }
    public Object beforeBodyWrite(Object body, ...) {
        if (body instanceof Result) return body;
        return Result.ok(body);
    }
}
```

---

## 八、参数校验（@Valid + 自定义注解）

```java
public class UserDTO {
    @NotBlank @Size(min = 2, max = 20)
    private String username;
    @Email private String email;
    @Min(0) @Max(150) private Integer age;
}
// Controller: @RequestBody @Valid UserDTO dto → 校验失败抛 MethodArgumentNotValidException

// 自定义校验注解：@Constraint(validatedBy = PhoneValidator.class)
```

---

## 九、RESTful API 设计规范

| 操作 | HTTP 方法 | URL | 说明 |
|-----|----------|-----|------|
| 查询列表 | GET | /api/users | 获取用户列表 |
| 查询详情 | GET | /api/users/{id} | 获取单个用户 |
| 创建 | POST | /api/users | 创建用户 |
| 全量更新 | PUT | /api/users/{id} | 更新用户（全字段） |
| 部分更新 | PATCH | /api/users/{id} | 更新用户（部分字段） |
| 删除 | DELETE | /api/users/{id} | 删除用户 |

**命名规范：**
- URL 使用名词复数（users 而非 user）
- 使用连字符而非下划线（user-profile 而非 user_profile）
- URL 中不使用动词（getUser → GET /users/{id}）
- 版本控制（/api/v1/users）

---

## 十、常见误区

| 误区 | 正解 |
|-----|------|
| Filter 和 Interceptor 是同一层 | Filter 在 Servlet 层，Interceptor 在 Spring 层 |
| @RestController 包含 @Component | 是的，它包含 @Controller + @ResponseBody |
| 拦截器能拦截所有请求 | 只能拦截到 HandlerMapping 映射到的请求 |
| 统一异常处理能捕获 Filter 中的异常 | 不能，Filter 在 DispatcherServlet 之前 |
| @Valid 必须配合 BindingResult | 不配 BindingResult 校验失败会直接抛异常（推荐） |

---

## 十一、面试题

### Q1：描述 DispatcherServlet 的处理流程？
**答：** 请求到达 DispatcherServlet → HandlerMapping 查找 Handler → HandlerAdapter 适配 → 执行拦截器 preHandle → 执行 Controller → 返回 ModelAndView → ViewResolver 解析视图 → 渲染响应 → 拦截器 afterCompletion。

### Q2：Filter 和 Interceptor 的区别？
**答：** Filter 是 Servlet 规范，在 DispatcherServlet 之前执行，能拦截所有请求包括静态资源；Interceptor 是 Spring 规范，在 Handler 执行前后执行，只能拦截 Controller 方法，能获取 Handler 信息和注入 Spring Bean。

### Q3：@RequestBody 的原理是什么？
**答：** Spring MVC 通过 HttpMessageConverter 将请求体中的 JSON 反序列化为 Java 对象。根据请求的 Content-Type 选择对应的 Converter（如 application/json 使用 MappingJackson2HttpMessageConverter），通过反射将 JSON 字段映射到对象属性。

### Q4：统一异常处理是怎么实现的？
**答：** 使用 @RestControllerAdvice + @ExceptionHandler，本质是 AOP。Spring MVC 在 Handler 执行抛出异常后，HandlerExceptionResolver 会查找匹配的 @ExceptionHandler 方法来处理异常，将异常转换为统一格式的响应。

### Q5：拦截器的三个方法分别在什么时候执行？
**答：** preHandle 在 Handler 执行前，用于权限校验，返回 false 中断请求；postHandle 在 Handler 执行后视图渲染前，可以修改 ModelAndView；afterCompletion 在请求完成后（视图渲染后），用于资源清理和日志记录，无论是否异常都会执行。

### Q6：如何设计一个 RESTful API？
**答：** 资源用名词复数作 URL，用 HTTP 方法表示操作（GET 查询/POST 创建/PUT 更新/DELETE 删除），使用合适的状态码，版本控制（/api/v1/），URL 中不用动词，响应使用统一格式。

---

## 十二、关联知识

- [21-IOC与Bean](21-IOC与Bean.md) — Controller 本身就是 Spring Bean
- [22-AOP](22-AOP.md) — 统一异常处理底层是 AOP
- [24-SpringBoot](24-SpringBoot.md) — Spring Boot 自动配置 MVC 组件
- [Part7-中间件/32-Tomcat.md](../Part7-中间件/32-Tomcat.md) — DispatcherServlet 与 Servlet 容器
