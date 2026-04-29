
> 优先级：⭐⭐ | 面试频率：🔥🔥 | 掌握程度：熟悉


---

  

## 一、知识树

  

```

MyBatis

│

├── 1. 执行流程 ⭐⭐⭐ 🔥🔥🔥

│   ├── SqlSessionFactory 构建

│   ├── SqlSession 获取

│   ├── Mapper 代理对象生成

│   ├── SQL 解析与执行

│   └── 结果集映射

│

├── 2. 核心组件 ⭐⭐ 🔥🔥

│   ├── mybatis-config.xml

│   ├── Mapper XML / Mapper 接口

│   ├── SqlSessionFactory

│   ├── SqlSession

│   └── Executor（Simple / Reuse / Batch）

│

├── 3. 动态 SQL ⭐⭐⭐ 🔥🔥

│   ├── if / choose / when / otherwise

│   ├── where / set / trim

│   ├── foreach

│   ├── sql / include

│   └── OGNL 表达式

│

├── 4. 缓存机制 ⭐⭐⭐ 🔥🔥🔥

│   ├── 一级缓存（SqlSession 级别，默认开启）

│   └── 二级缓存（Mapper 级别，需手动开启）

│

├── 5. 结果映射 ⭐⭐ 🔥

│   ├── resultMap

│   ├── 一对一（association）

│   ├── 一对多（collection）

│   └── 自动映射 / 驼峰转换

│

└── 6. 常见面试题 ⭐⭐ 🔥🔥

    ├── #{} vs ${} 的区别

    ├── 分页实现

    ├── 批量操作

    └── MyBatis vs JPA/Hibernate

```

  

---

  

## 二、核心知识点精讲

  

### 2.1 执行流程

  

```

1. 读取 mybatis-config.xml / application.yml 配置

         │

         ▼

2. SqlSessionFactoryBuilder.build() → 创建 SqlSessionFactory

         │

         ▼

3. SqlSessionFactory.openSession() → 创建 SqlSession

         │

         ▼

4. SqlSession.getMapper(UserMapper.class)

   → MapperProxy（JDK 动态代理）

         │

         ▼

5. 调用 Mapper 方法 → MapperProxy.invoke()

   → MapperMethod.execute()

         │

         ▼

6. Executor.query() / update()

   ├── 查询二级缓存 → 命中则直接返回

   ├── 查询一级缓存 → 命中则直接返回

   └── 都未命中 → 访问数据库

         │

         ▼

7. StatementHandler → ParameterHandler → 执行 SQL

         │

         ▼

8. ResultSetHandler → 结果集映射 → 返回 Java 对象

```

  

### 2.2 动态 SQL

  

**常用标签：**

  

```xml

<!-- if：条件判断 -->

<select id="findUser" resultType="User">

    SELECT * FROM user

    WHERE status = 1

    <if test="name != null">

        AND name = #{name}

    </if>

    <if test="age != null">

        AND age = #{age}

    </if>

</select>

  

<!-- where：自动处理 AND/OR 前缀 -->

<select id="findUser" resultType="User">

    SELECT * FROM user

    <where>

        <if test="name != null">

            AND name = #{name}

        </if>

        <if test="age != null">

            AND age = #{age}

        </if>

    </where>

</select>

  

<!-- foreach：批量操作 -->

<select id="findByIds" resultType="User">

    SELECT * FROM user WHERE id IN

    <foreach collection="ids" item="id" open="(" separator="," close=")">

        #{id}

    </foreach>

</select>

  

<!-- set：更新时自动去掉多余逗号 -->

<update id="updateUser">

    UPDATE user

    <set>

        <if test="name != null">name = #{name},</if>

        <if test="age != null">age = #{age},</if>

    </set>

    WHERE id = #{id}

</update>

  

<!-- choose/when/otherwise：类似 switch-case -->

<select id="findUser" resultType="User">

    SELECT * FROM user

    <where>

        <choose>

            <when test="id != null">id = #{id}</when>

            <when test="name != null">name = #{name}</when>

            <otherwise>status = 1</otherwise>

        </choose>

    </where>

</select>

```

  

### 2.3 缓存机制

  

#### 一级缓存（SqlSession 级别）

  

```

作用域：同一个 SqlSession 内

默认开启：不可关闭

缓存 key：statementId + SQL + 参数 + 分页参数

  

失效场景：

1. SqlSession 关闭（不同请求 = 不同 SqlSession）

2. 执行了 INSERT/UPDATE/DELETE（清空缓存）

3. 手动调用 clearCache()

  

验证：

  同一个 SqlSession 中连续两次相同查询 → 只发一条 SQL

```

  

#### 二级缓存（Mapper 级别）

  

```

作用域：整个 Mapper（跨 SqlSession 共享）

需要手动开启：

  1. mybatis-config.xml 中 <setting name="cacheEnabled" value="true"/>

  2. Mapper XML 中添加 <cache/>

  3. 实体类实现 Serializable 接口

  

生效时机：SqlSession 关闭或提交后才写入二级缓存

  

注意：

  - 多表关联查询时可能出现脏读（关联表的数据变化了但缓存没更新）

  - 生产中一般不用二级缓存，用 Redis 替代

```

  

**一级 vs 二级缓存：**

  
| 维度 | 一级缓存 | 二级缓存 |
|------|---------|---------|
| 作用域 | SqlSession | Mapper namespace |
| 默认状态 | 开启 | 关闭 |
| 跨 Session | 不共享 | 共享 |
| 失效条件 | DML / close / clearCache | 关联表更新 |
| 生产使用 | 基本无用（Spring 中每次请求新建 SqlSession） | 很少用（用 Redis 替代） |

  

### 2.4 #{} vs ${}

  
| 维度 | #{} | ${} |
|------|-----|-----|
| 处理方式 | PreparedStatement 参数绑定（? 占位） | 字符串拼接 |
| SQL 注入 | 安全 | 不安全 |
| 性能 | 预编译，可复用执行计划 | 每次生成新 SQL |
| 使用场景 | 参数值传递 | 动态表名 / 列名 / ORDER BY |

  

```xml

<!-- #{} → PreparedStatement -->

SELECT * FROM user WHERE name = #{name}

<!-- 解析为: SELECT * FROM user WHERE name = ? -->

  

<!-- ${} → 字符串拼接 -->

SELECT * FROM ${tableName} ORDER BY ${orderColumn}

<!-- 解析为: SELECT * FROM user ORDER BY create_time -->

```

  

### 2.5 MyBatis vs JPA/Hibernate

  
| 维度 | MyBatis | JPA/Hibernate |
|------|---------|---------------|
| SQL 控制 | 手写 SQL，灵活度高 | 自动生成，不够灵活 |
| 学习成本 | 低 | 高（JPQL / HQL / 实体映射） |
| 复杂查询 | 灵活（动态 SQL） | 困难 |
| 缓存 | 简单（一二级缓存） | 完善（一级 + 二级 + 查询缓存） |
| 跨数据库 | 差（SQL 可能不通用） | 好（方言自动适配） |
| 国内使用率 | 极高 | 较低 |
| 国际使用率 | 较低 | 较高 |

  

---

  

## 三、高频面试题（8 题）

  

### Q1：MyBatis 的执行流程？

```

配置加载 → SqlSessionFactory → SqlSession → MapperProxy（动态代理）

→ MapperMethod → Executor → StatementHandler → ParameterHandler

→ 执行 SQL → ResultSetHandler → 结果映射 → 返回

  

简化版：配置 → 会话 → 代理 → 执行 → 映射 → 返回

```

  

### Q2：#{} 和 ${} 的区别？

```

#{}：预编译参数绑定（?），防 SQL 注入，传值用

${}：字符串拼接，有注入风险，传表名/列名用

  

开发中 99% 用 #{}，只有动态表名/ORDER BY 才用 ${}

用 ${} 时必须做白名单校验

```

  

### Q3：MyBatis 的一级缓存和二级缓存？

```

一级缓存：SqlSession 级别，默认开启，Spring 中基本没用

二级缓存：Mapper 级别，需手动开启，生产中一般用 Redis 替代

  

Spring 集成 MyBatis 后，每次请求创建新的 SqlSession

→ 一级缓存在同一请求内有效（一次 HTTP 请求中的多次查询）

```

  

### Q4：MyBatis 是怎么进行分页的？

```

逻辑分页（RowBounds）：查询所有数据后在内存中截取（不推荐）

物理分页：在 SQL 中加 LIMIT（推荐）

  

实现方式：

1. 手动写 LIMIT（最简单）

2. PageHelper 插件（最常用）

   PageHelper.startPage(pageNum, pageSize);

   list = mapper.findAll();  // 自动加 LIMIT

3. MyBatis-Plus 的 IPage

```

  

### Q5：MyBatis 如何实现批量插入？

```

方式 1：foreach 拼接

INSERT INTO user (name, age) VALUES

<foreach collection="list" item="u" separator=",">

    (#{u.name}, #{u.age})

</foreach>

  

方式 2：Batch Executor

SqlSession session = factory.openSession(ExecutorType.BATCH);

// 每 1000 条 flush 一次

  

方式 3：MySQL 的 LOAD DATA INFILE（百万级）

```

  

### Q6：Mapper 接口是怎么和 XML 关联的？

```

关联规则：

1. namespace = Mapper 接口的全限定名

2. SQL 的 id = Mapper 接口的方法名

3. 参数类型 = 方法参数类型

4. 返回类型 = 方法返回类型

  

底层：JDK 动态代理（MapperProxy）

调用 Mapper 方法 → MapperProxy.invoke() → 根据 接口全限定名+方法名 找到对应的 SQL

```

  

### Q7：MyBatis 的插件（拦截器）机制？

```

可拦截的四大对象：

1. Executor：增删改查、事务管理

2. StatementHandler：SQL 执行

3. ParameterHandler：参数处理

4. ResultSetHandler：结果集处理

  

实现：实现 Interceptor 接口 + @Intercepts 注解

  

常见插件：

- PageHelper（分页）

- MyBatis-Plus（通用 CRUD）

- 自定义慢 SQL 监控

```

  

### Q8：resultMap 和 resultType 的区别？

```

resultType：自动映射（字段名和属性名一致时使用，或开启驼峰转换）

resultMap：手动映射（字段名和属性名不一致，或有关联关系时使用）

  

关联映射：

  一对一：<association property="dept" column="dept_id" select="findDept"/>

  一对多：<collection property="orders" column="id" select="findOrders"/>

  

注意：嵌套查询可能导致 N+1 问题（每个主记录都发一次子查询）

解决：使用 JOIN 一次性查出 + resultMap 手动映射

```

  

---

  

## 四、常见误区

  
| 误区 | 正确理解 |
|------|---------|
| MyBatis 的一级缓存很有用 | Spring 中每次请求新建 SqlSession，一级缓存作用有限 |
| 二级缓存应该开启 | 多表关联场景容易出现脏数据，生产中用 Redis |
| #{} 和 ${} 性能一样 | #{} 有预编译优化，但差距很小 |
| MyBatis 不能做复杂映射 | resultMap + association/collection 可以处理 |
| MyBatis 只能写 XML | 也可以用注解（@Select/@Insert），但复杂 SQL 仍建议 XML |

  

---

  

## 五、实战应用场景

  
| 场景 | MyBatis 方案 |
|------|-------------|
| 动态条件查询 | 动态 SQL（if/where/choose） |
| 批量操作 | foreach / Batch Executor |
| 分页 | PageHelper 插件 |
| 多表关联 | resultMap + association/collection |
| 通用 CRUD | MyBatis-Plus（增强工具） |
| 慢 SQL 监控 | 自定义 Interceptor 拦截 StatementHandler |

  

---

  

## 六、关联模块

  

- ← [05-Spring全家桶.md](05-Spring全家桶.md)：MyBatis 通过 Spring Boot Starter 集成

- ← [06-MySQL.md](06-MySQL.md)：MyBatis 的 SQL 调优依赖 MySQL 知识

- → [07-Redis.md](07-Redis.md)：MyBatis 二级缓存的替代方案