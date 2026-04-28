
> 优先级：⭐⭐⭐⭐ | 面试频率：🔥🔥🔥🔥
> 动态 SQL 是 MyBatis 的灵魂特性，缓存机制是理解 MyBatis 性能的关键。

---

## 知识树

```
MyBatis 动态 SQL 与缓存
├── 动态 SQL 标签
│   ├── if
│   ├── choose-when-otherwise
│   ├── where
│   ├── set
│   ├── trim
│   ├── foreach
│   └── sql-include
├── 批量操作
│   ├── 批量插入
│   ├── 批量更新
│   └── 批量删除
├── OGNL 表达式
├── 缓存机制
│   ├── 一级缓存（SqlSession）
│   ├── 二级缓存（Mapper）
│   └── 一级 vs 二级对比
├── 插件机制
│   ├── Interceptor
│   └── 四大可拦截对象
└── PageHelper 分页原理
```

---

## 一、动态 SQL 标签详解

### 1. if 标签

```xml
<select id="selectByCondition" resultType="User">
    SELECT * FROM user
    WHERE status = 1
    <if test="name != null and name != ''">
        AND name LIKE CONCAT('%', #{name}, '%')
    </if>
    <if test="age != null">
        AND age = #{age}
    </if>
</select>
```

### 2. choose-when-otherwise 标签

```xml
<!-- 类似 switch-case，只匹配第一个满足条件的 -->
<select id="selectByPriority" resultType="User">
    SELECT * FROM user WHERE status = 1
    <choose>
        <when test="id != null">
            AND id = #{id}
        </when>
        <when test="name != null">
            AND name = #{name}
        </when>
        <otherwise>
            AND age > 18
        </otherwise>
    </choose>
</select>
```

### 3. where 标签

```xml
<!-- 自动处理 AND/OR 前缀，无条件时不生成 WHERE -->
<select id="selectDynamic" resultType="User">
    SELECT * FROM user
    <where>
        <if test="name != null">
            AND name = #{name}    <!-- 自动去掉开头的 AND -->
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
</select>
<!-- 所有条件为空：SELECT * FROM user -->
<!-- name 不为空：SELECT * FROM user WHERE name = ? -->
```

### 4. set 标签

```xml
<!-- 动态更新，只更新非空字段，自动处理末尾逗号 -->
<update id="updateSelective">
    UPDATE user
    <set>
        <if test="name != null">name = #{name},</if>
        <if test="age != null">age = #{age},</if>
        <if test="email != null">email = #{email},</if>
    </set>
    WHERE id = #{id}
</update>
```

### 5. trim 标签

```xml
<!-- trim 是 where/set 的通用版本 -->
<!-- 等价于 where -->
<trim prefix="WHERE" prefixOverrides="AND |OR ">
    <if test="name != null">AND name = #{name}</if>
</trim>

<!-- 等价于 set -->
<trim prefix="SET" suffixOverrides=",">
    <if test="name != null">name = #{name},</if>
</trim>
```

| 属性 | 说明 |
|------|------|
| prefix | 内容非空时添加的前缀 |
| suffix | 内容非空时添加的后缀 |
| prefixOverrides | 去掉内容开头指定的关键字 |
| suffixOverrides | 去掉内容末尾指定的关键字 |

### 6. foreach 标签

```xml
<select id="selectByIds" resultType="User">
    SELECT * FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
<!-- 生成：SELECT * FROM user WHERE id IN (1, 2, 3) -->
```

**foreach 参数详解**：

| 参数 | 说明 | 示例 |
|------|------|------|
| collection | 集合参数名 | ids / list / array |
| item | 当前元素变量名 | id |
| index | 索引变量名 | idx（List=索引，Map=key） |
| open | 开始符号 | ( |
| separator | 分隔符 | , |
| close | 结束符号 | ) |

**collection 的值**：
- `@Param("ids")` → 用 ids
- List 参数无注解 → list
- 数组参数无注解 → array

### 7. sql-include 标签

```xml
<!-- 定义可复用的 SQL 片段 -->
<sql id="baseColumns">
    id, name, age, email, create_time
</sql>

<select id="selectAll" resultType="User">
    SELECT <include refid="baseColumns"/> FROM user
</select>

<select id="selectById" resultType="User">
    SELECT <include refid="baseColumns"/> FROM user WHERE id = #{id}
</select>
```

---

## 二、批量操作

### 批量插入

```xml
<!-- 方式1：VALUES 多行 -->
<insert id="batchInsert">
    INSERT INTO user (name, age) VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.name}, #{user.age})
    </foreach>
</insert>
<!-- 生成：INSERT INTO user (name, age) VALUES ('张三',25),('李四',30) -->

<!-- 方式2：BatchExecutor -->
<!-- SqlSession session = factory.openSession(ExecutorType.BATCH) -->
```

### 批量更新

```xml
<!-- MySQL 的 CASE WHEN 方式 -->
<update id="batchUpdate">
    UPDATE user SET age = CASE id
    <foreach collection="list" item="user">
        WHEN #{user.id} THEN #{user.age}
    </foreach>
    END
    WHERE id IN
    <foreach collection="list" item="user" open="(" separator="," close=")">
        #{user.id}
    </foreach>
</update>
```

### 批量删除

```xml
<delete id="batchDelete">
    DELETE FROM user WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</delete>
```

---

## 三、OGNL 表达式常用写法

| 表达式 | 含义 |
|--------|------|
| `name != null` | name 不为 null |
| `name != null and name != ''` | name 非空非空串 |
| `list != null and list.size() > 0` | 集合非空 |
| `type == 1` | 等于判断 |
| `name == 'admin' or role == 'super'` | 或条件 |
| `!(age < 18)` | 取反 |

---

## 四、一级缓存（SqlSession 级别）

### 特点

- 默认开启，无法关闭
- 作用域：同一个 SqlSession
- 数据结构：HashMap（key = statementId + SQL + 参数）

### 缓存命中条件

```
同一个 SqlSession + 同一个 Statement（namespace.id）+ 相同 SQL + 相同参数
→ 命中一级缓存
```

### 失效条件

| 场景 | 说明 |
|------|------|
| SqlSession 关闭 | 缓存清空 |
| 执行 INSERT/UPDATE/DELETE | 缓存清空 |
| 手动 clearCache() | 缓存清空 |
| 不同 SqlSession | 不同缓存 |

### 在 Spring 中的作用

```java
// Spring 中每次请求创建新的 SqlSession → 一级缓存基本无效
// 同一个方法内的多次相同查询才能命中
@Transactional
public void method() {
    userMapper.selectById(1);  // 查DB
    userMapper.selectById(1);  // 命中一级缓存
}
```

---

## 五、二级缓存（Mapper 级别）

### 开启方式

```xml
<!-- mybatis-config.xml -->
<setting name="cacheEnabled" value="true"/>

<!-- Mapper XML -->
<cache eviction="LRU" flushInterval="60000" size="1024" readOnly="true"/>
```

### 特点

- 作用域：同一个 Mapper namespace
- 需要**手动开启**
- 实体类必须实现 **Serializable** 接口
- 多个 SqlSession 共享

### 工作流程

```
SqlSession 查询 → 先查二级缓存 → 再查一级缓存 → 最后查数据库
                ← 二级缓存命中   ← 一级缓存命中  ← 查DB

SqlSession 关闭时 → 一级缓存数据写入二级缓存
```

### 脏数据问题

```java
// 不同 namespace 的操作不会清对方缓存
// UserMapper 查了 user 表（缓存了）
// OrderMapper 更新了 user 表（不会清 UserMapper 的二级缓存）
// UserMapper 再次查 → 命中旧缓存 → 脏数据！
```

---

## 六、一级 vs 二级缓存对比

| 对比项 | 一级缓存 | 二级缓存 |
|--------|---------|---------|
| 作用域 | SqlSession | Mapper namespace |
| 默认状态 | **默认开启** | 需手动开启 |
| 共享范围 | 单个会话 | 多个会话 |
| 失效时机 | 会话关闭/DML | namespace 下任意 DML |
| 数据安全 | 安全（会话隔离） | 可能有脏数据 |
| 实体类要求 | 无 | 实现 Serializable |

### 为什么生产中不用二级缓存？

1. **脏数据**：多表关联时，一个 namespace 的更新不会清另一个 namespace 的缓存
2. **颗粒度太粗**：整个 namespace 的缓存一起失效
3. **不适合分布式**：多台机器各自的缓存不一致
4. **替代方案**：用 Redis 做分布式缓存

---

## 七、MyBatis 插件机制

### 四大可拦截对象

| 对象 | 方法 | 拦截场景 |
|------|------|---------|
| Executor | update/query/commit/rollback | SQL 执行全过程 |
| StatementHandler | prepare/parameterize/query/update | Statement 操作 |
| ParameterHandler | setParameters/getParameterObject | 参数处理 |
| ResultSetHandler | handleResultSets/handleOutputParameters | 结果映射 |

### Interceptor 接口

```java
@Intercepts({
    @Signature(type = StatementHandler.class, method = "prepare",
               args = {Connection.class, Integer.class})
})
public class SlowSqlInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = invocation.proceed();  // 执行原方法
        long cost = System.currentTimeMillis() - start;
        if (cost > 1000) {
            // 记录慢SQL
        }
        return result;
    }
}
```

### PageHelper 分页原理

```java
// 使用
PageHelper.startPage(1, 10);
List<User> users = userMapper.selectAll();
// 自动返回前10条，自动查询总数
```

**原理**：
1. PageHelper 是 MyBatis 插件，拦截 Executor 的 query 方法
2. 通过 ThreadLocal 存储分页参数
3. 拦截到 SQL 后，自动添加 LIMIT/OFFSET（或 COUNT 查询）
4. 查询完成后清除 ThreadLocal

---

## 面试题精选

### 1. MyBatis 的动态 SQL 有哪些标签？
> if/choose-when-otherwise/where/set/trim/foreach/sql-include。if 做条件判断，choose 做多选一，where/set 自动处理 SQL 语法，foreach 遍历集合，sql-include 复用 SQL 片段。

### 2. foreach 的 collection 参数怎么写？
> @Param 注解用注解值，List 无注解用 list，数组无注解用 array，Map 用 key。

### 3. 一级缓存和二级缓存的区别？
> 一级缓存 SqlSession 级别，默认开启，会话内有效；二级缓存 Mapper 级别，手动开启，多会话共享。一级缓存在 SqlSession 关闭时数据写入二级缓存。

### 4. 为什么生产环境不用二级缓存？
> 多表关联时有脏数据问题（一个 namespace 更新不会清另一个 namespace 的缓存），分布式环境下缓存不一致。推荐用 Redis 做分布式缓存替代。

### 5. PageHelper 的分页原理？
> 基于 MyBatis 插件机制，拦截 Executor 的 query 方法，通过 ThreadLocal 获取分页参数，自动改写 SQL 添加 LIMIT/OFFSET，并自动执行 COUNT 查询获取总数。

### 6. MyBatis 插件能拦截哪些对象？
> 四大对象：Executor（SQL 执行）、StatementHandler（Statement 操作）、ParameterHandler（参数处理）、ResultSetHandler（结果映射）。通过 @Intercepts + @Signature 注解声明拦截点。

---

## 常见误区

| 误区 | 正解 |
|------|------|
| 一级缓存可以关闭 | 一级缓存始终开启，无法关闭 |
| 二级缓存默认开启 | 需要手动配置 `<cache/>` |
| Spring 中一级缓存很有用 | Spring 每次请求新建 SqlSession，一级缓存基本无效 |
| foreach 只能用于 IN | 还可用于批量 INSERT（VALUES 多行） |
| trim 标签不常用 | where 和 set 都是 trim 的语法糖 |

---

## 实战场景

**场景 1：多条件动态查询**
```xml
<select id="search" resultType="User">
    SELECT <include refid="baseColumns"/> FROM user
    <where>
        <if test="name != null">AND name LIKE CONCAT('%', #{name}, '%')</if>
        <if test="minAge != null">AND age >= #{minAge}</if>
        <if test="maxAge != null">AND age &lt;= #{maxAge}</if>
        <if test="status != null">AND status = #{status}</if>
        <choose>
            <when test="sortField == 'age'">ORDER BY age</when>
            <otherwise>ORDER BY id</otherwise>
        </choose>
    </where>
</select>
```

**场景 2：批量插入 + ON DUPLICATE KEY UPDATE**
```xml
<insert id="batchInsertOrUpdate">
    INSERT INTO user (id, name, age) VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.id}, #{user.name}, #{user.age})
    </foreach>
    ON DUPLICATE KEY UPDATE name = VALUES(name), age = VALUES(age)
</insert>
```

---

## 关联知识

- [33-MyBatis-核心](./33-MyBatis-核心.md) — MyBatis 执行流程、Mapper 代理、#{} vs ${}
- [27-MySQL-存储引擎与索引](./27-MySQL-存储引擎与索引.md) — 动态 SQL 生成的查询要走索引
- [29-MySQL-锁与调优](./29-MySQL-锁与调优.md) — 批量操作的锁和性能优化
- [32-Redis-缓存问题](./32-Redis-缓存问题.md) — MyBatis 缓存与 Redis 缓存的选型
