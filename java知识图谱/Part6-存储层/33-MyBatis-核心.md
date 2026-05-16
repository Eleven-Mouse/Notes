
> MyBatis 是 Java 持久层框架的事实标准，核心流程和 #{} vs ${} 是面试高频题。

---

## 知识树

```
MyBatis 核心
├── 完整执行流程
├── 核心组件
│   ├── SqlSessionFactory
│   ├── SqlSession
│   ├── Executor
│   ├── StatementHandler
│   ├── ParameterHandler
│   └── ResultSetHandler
├── Mapper 代理机制
│   ├── namespace + id 关联
│   └── MapperProxy 动态代理
├── #{} vs ${}
├── resultMap vs resultType
├── 关联映射
│   ├── association（一对一）
│   └── collection（一对多）
├── N+1 问题
├── MyBatis vs JPA/Hibernate
└── MyBatis-Plus
```

---

## 一、MyBatis 执行完整流程

```
1. 读取 mybatis-config.xml → XMLConfigBuilder 解析
2. 构建 Configuration 对象（全局配置）
3. 创建 SqlSessionFactory（重量级对象，单例）
4. SqlSessionFactory.openSession() → 创建 SqlSession
5. SqlSession.getMapper(UserMapper.class)
   → MapperProxy（JDK 动态代理）
6. 调用 Mapper 方法 → MapperProxy.invoke()
7. MapperMethod.execute()
   → SqlSession.selectOne / selectList / insert / update / delete
8. Executor（执行器）
   → 一级缓存查询 / 二级缓存查询
   → 创建 StatementHandler
9. StatementHandler.prepare() → 创建 Statement / PreparedStatement
10. ParameterHandler.setParameters() → 设置参数（#{} 替换为 ?）
11. StatementHandler.query() → 执行 SQL
12. ResultSetHandler.handleResultSets() → 结果映射
13. 返回结果
```

---

## 二、核心组件

| 组件 | 职责 | 生命周期 |
|------|------|---------|
| SqlSessionFactory | 创建 SqlSession，全局唯一 | 应用级别（单例） |
| SqlSession | 执行 SQL，获取 Mapper | 方法/请求级别 |
| Executor | SQL 执行器，管理缓存和事务 | SqlSession 级别 |
| StatementHandler | 操作 JDBC Statement | 一次 SQL 操作 |
| ParameterHandler | 参数设置 | 一次 SQL 操作 |
| ResultSetHandler | 结果集映射 | 一次 SQL 操作 |

**Executor 三种类型**：

| 类型 | 说明 |
|------|------|
| SimpleExecutor | 每次 SQL 创建新 Statement（默认） |
| ReuseExecutor | 复用 Statement（缓存 SQL） |
| BatchExecutor | 批量操作 |

---

## 三、Mapper 接口和 XML 的关联方式

```java
// Mapper 接口
public interface UserMapper {
    User selectById(Long id);
}
```

```xml
<!-- UserMapper.xml -->
<mapper namespace="com.example.mapper.UserMapper">
    <select id="selectById" resultType="User">
        SELECT * FROM user WHERE id = #{id}
    </select>
</mapper>
```

**关联规则**：
- XML 的 `namespace` = Mapper 接口的全限定名
- XML 的 `id` = 接口方法名
- 参数类型和返回类型必须匹配

### MapperProxy 动态代理原理

```java
// MyBatis 内部实现（简化）
public class MapperProxy<T> implements InvocationHandler {
    private SqlSession sqlSession;
    private Class<T> mapperInterface;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) {
        // 不是接口默认方法
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        }
        // 封装为 MapperMethod 执行
        MapperMethod mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
        return mapperMethod.execute(sqlSession, args);
    }
}

// 使用 JDK 动态代理创建 Mapper 实例
UserMapper mapper = (UserMapper) Proxy.newProxyInstance(
    classLoader,
    new Class[]{UserMapper.class},
    new MapperProxy<>(sqlSession, UserMapper.class)
);
```

**关键点**：Mapper 接口不需要实现类，MyBatis 通过 JDK 动态代理在运行时生成代理对象，将方法调用转发给 SqlSession 执行对应的 SQL。

---

## 四、#{} vs ${} 详细对比

| 对比项 | #{} | ${} |
|--------|-----|-----|
| 处理方式 | **预编译**（PreparedStatement 参数占位符 ?） | **字符串拼接**（直接替换到 SQL 中） |
| SQL 注入 | ✅ 安全（参数转义） | ❌ 不安全（直接拼接） |
| 性能 | 数据库可缓存执行计划 | 每次都是新 SQL |
| 使用场景 | 参数值传递 | 动态表名/列名/ORDER BY |

```xml
<!-- #{} -->
<select id="selectById">
    SELECT * FROM user WHERE id = #{id}
</select>
<!-- 生成的SQL：SELECT * FROM user WHERE id = ? -->
<!-- 参数：id=1 → PreparedStatement.setInt(1, 1) -->

<!-- ${} -->
<select id="selectByOrder">
    SELECT * FROM user ORDER BY ${column}
</select>
<!-- 生成的SQL：SELECT * FROM user ORDER BY create_time -->
<!-- 直接替换，不经过预编译 -->

<!-- ${} 的合法场景：动态表名/列名 -->
<select id="selectByTable">
    SELECT * FROM ${tableName} WHERE id = #{id}
</select>
```

**核心区别**：#{} 是参数占位符，${} 是字符串替换。能用 #{} 就用 #{}。

---

## 五、resultMap vs resultType

| 对比项 | resultType | resultMap |
|--------|-----------|-----------|
| 使用场景 | 列名和属性名一致（或驼峰映射） | 列名和属性名不一致、复杂映射 |
| 复杂度 | 简单 | 灵活但复杂 |
| 关联查询 | 不支持 | 支持 association/collection |

```xml
<!-- resultType：简单映射 -->
<select id="selectAll" resultType="User">
    SELECT id, user_name, age FROM user
</select>

<!-- resultMap：自定义映射 -->
<resultMap id="userResultMap" type="User">
    <id column="id" property="id"/>
    <result column="user_name" property="userName"/>
    <result column="age" property="age"/>
</resultMap>

<select id="selectAll" resultMap="userResultMap">
    SELECT id, user_name, age FROM user
</select>
```

---

## 六、关联映射

### association（一对一）

```xml
<!-- 用户有一个身份证 -->
<resultMap id="userWithIdCard" type="User">
    <id column="id" property="id"/>
    <result column="name" property="name"/>
    <association property="idCard" javaType="IdCard">
        <id column="card_id" property="id"/>
        <result column="card_no" property="cardNo"/>
    </association>
</resultMap>
```

### collection（一对多）

```xml
<!-- 用户有多个订单 -->
<resultMap id="userWithOrders" type="User">
    <id column="id" property="id"/>
    <result column="name" property="name"/>
    <collection property="orders" ofType="Order">
        <id column="order_id" property="id"/>
        <result column="order_no" property="orderNo"/>
    </collection>
</resultMap>
```

---

## 七、N+1 查询问题

**问题**：查询 N 个用户，每个用户再查一次关联数据 → 1 + N 次 SQL。

```xml
<!-- 这种配置会产生 N+1 问题 -->
<resultMap id="userWithOrders" type="User">
    <collection property="orders" column="id"
                select="selectOrdersByUserId" fetchType="lazy"/>
</resultMap>
```

### 解决方案

**方案1：联合查询（JOIN）**

```xml
<select id="selectUserWithOrders" resultMap="userWithOrders">
    SELECT u.*, o.id as order_id, o.order_no
    FROM user u
    LEFT JOIN orders o ON u.id = o.user_id
</select>
```

**方案2：延迟加载（lazy loading）**

```xml
<collection property="orders" column="id"
            select="selectOrdersByUserId" fetchType="lazy"/>
<!-- 只有关联属性被访问时才执行子查询 -->
```

---

## 八、MyBatis vs JPA/Hibernate

| 对比项 | MyBatis | JPA/Hibernate |
|--------|---------|---------------|
| SQL 控制 | 手写 SQL，完全控制 | 自动生成，HQL/Criteria |
| 学习成本 | 低 | 高 |
| 灵活性 | 高（复杂 SQL 友好） | 简单 CRUD 快，复杂查询麻烦 |
| 缓存 | 一级+二级缓存 | 一级+二级缓存 + 查询缓存 |
| 数据库迁移 | 差（手写 SQL 绑定数据库） | 好（Dialect 自动适配） |
| 适用场景 | 复杂业务、性能优化 | 快速开发、简单 CRUD |

---

## 九、MyBatis-Plus 常用功能

### 通用 CRUD

```java
public interface UserMapper extends BaseMapper<User> {
    // 自动拥有：insert / deleteById / updateById / selectById / selectList 等
}
```

### 条件构造器

```java
// QueryWrapper
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("age", 25)
       .like("name", "张")
       .between("create_time", startDate, endDate)
       .orderByDesc("id");
List<User> users = userMapper.selectList(wrapper);

// LambdaQueryWrapper（类型安全）
LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(User::getAge, 25)
       .like(User::getName, "张");
```

### 分页插件

```java
@Configuration
public class MyBatisPlusConfig {
    @Bean
    public PaginationInterceptor paginationInterceptor() {
        return new PaginationInterceptor();
    }
}

// 使用
Page<User> page = new Page<>(1, 10);  // 第1页，每页10条
IPage<User> result = userMapper.selectPage(page, wrapper);
result.getRecords();   // 数据列表
result.getTotal();      // 总条数
```

---

## 面试题精选

### 1. MyBatis 的执行流程是什么？
> 加载配置 → 构建 Configuration → 创建 SqlSessionFactory → 获取 SqlSession → 动态代理获取 Mapper → Executor 执行 → StatementHandler 操作 JDBC → ParameterHandler 设参 → 执行 SQL → ResultSetHandler 映射结果。

### 2. #{} 和 ${} 的区别？
> #{} 是预编译参数占位符（?），安全防注入；${} 是字符串直接替换，不安全。参数值用 #{}，动态表名/列名用 ${}。

### 3. Mapper 接口是怎么和 XML 关联的？
> XML 的 namespace 等于 Mapper 接口全限定名，SQL 的 id 等于方法名。运行时通过 JDK 动态代理创建 MapperProxy，将方法调用转发给 SqlSession 执行对应 SQL。

### 4. MyBatis 有哪些 Executor？
> SimpleExecutor（默认，每次创建 Statement）、ReuseExecutor（复用 Statement）、BatchExecutor（批量操作）。

### 5. resultMap 和 resultType 的区别？
> resultType 用于简单映射（列名=属性名或驼峰映射），resultMap 用于自定义映射（列名≠属性名、一对一、一对多关联）。

### 6. 什么是 N+1 问题？怎么解决？
> 查 N 条主数据后，每条再查一次关联数据，产生 1+N 次 SQL。解决：JOIN 联合查询一次查出、或使用延迟加载（fetchType=lazy）。

### 7. MyBatis 和 Hibernate 的区别？
> MyBatis 手写 SQL 控制力强、学习成本低、适合复杂业务；Hibernate 自动生成 SQL 开发效率高、数据库迁移方便、适合快速开发和简单 CRUD。

### 8. MyBatis-Plus 有什么优势？
> 通用 CRUD（不用写 XML）、条件构造器（类型安全的 Lambda）、分页插件、代码生成器、乐观锁插件等，极大减少样板代码。

---

## 常见误区

| 误区 | 正解 |
|------|------|
| Mapper 接口需要有实现类 | 不需要，JDK 动态代理生成代理对象 |
| ${} 完全不能用 | 动态表名/列名/ORDER BY 场景必须用 ${} |
| resultType 不能做复杂映射 | 可以配合驼峰映射，但复杂关联需要 resultMap |
| MyBatis 已经过时 | 国内仍是主流，MyBatis-Plus 进一步增强 |
| SqlSession 是线程安全的 | 不是线程安全的，不能跨线程共享 |

---

## 实战场景

**场景 1：复杂条件动态查询**
```java
// 用 MyBatis-Plus LambdaQueryWrapper
LambdaQueryWrapper<Order> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(order.getStatus() != null, Order::getStatus, order.getStatus())
       .ge(startTime != null, Order::getCreateTime, startTime)
       .le(endTime != null, Order::getCreateTime, endTime)
       .orderByDesc(Order::getCreateTime);
```

**场景 2：多表关联 resultMap**
```xml
<resultMap id="orderDetail" type="OrderDTO">
    <id column="order_id" property="orderId"/>
    <result column="order_no" property="orderNo"/>
    <association property="user" javaType="User">
        <id column="user_id" property="id"/>
        <result column="user_name" property="name"/>
    </association>
    <collection property="items" ofType="OrderItem">
        <id column="item_id" property="id"/>
        <result column="product_name" property="productName"/>
    </collection>
</resultMap>
```

---

## 关联知识

- [34-MyBatis-动态SQL与缓存](34-MyBatis-动态SQL与缓存.md) — 动态 SQL 标签和缓存机制
- [27-MySQL-存储引擎与索引](27-MySQL-存储引擎与索引.md) — MyBatis 写的 SQL 的索引优化
- [29-MySQL-锁与调优](29-MySQL-锁与调优.md) — EXPLAIN 分析 MyBatis 生成的 SQL
- [32-Redis-缓存问题](32-Redis-缓存问题.md) — MyBatis 二级缓存 vs Redis 缓存
