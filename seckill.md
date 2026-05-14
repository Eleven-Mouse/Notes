# MyBatis-Plus 与 DDD 架构知识体系

---

## 一、MyBatis-Plus：从手写 SQL 到自动生成

### 1.1 痛点：纯 MyBatis 时代的体力活

纯 MyBatis 中，每张表都要经历两步重复劳动——先写 Mapper 接口方法，再写对应的 XML SQL：

```java
// 第一步：定义接口方法
public interface UserMapper {
    User selectById(Long id);
    List<User> selectAll();
    int insert(User user);
    int update(User user);
    int deleteById(Long id);
}
```

```xml
<!-- 第二步：每条 SQL 都要手写 -->
<select id="selectById" resultType="User">
    SELECT id, nickname, password, salt, phone, create_time
    FROM t_user
    WHERE id = #{id}
</select>

<insert id="insert">
    INSERT INTO t_user (nickname, password, salt, phone, create_time)
    VALUES (#{nickname}, #{password}, #{salt}, #{phone}, #{createTime})
</insert>
```

一张表 5 个方法 + 5 段 XML，十张表就是 50 段几乎一样的 SQL。问题很明显：**格式雷同、容易出错、维护成本高**。

### 1.2 BaseMapper 的核心原理

MyBatis-Plus 的解法：**你只告诉它"操作哪个实体类"，它自动帮你拼 SQL 并注册到 MyBatis 中。**

整个流程拆解如下：

```
你写的代码                        MyBatis-Plus 启动时做的事
─────────────                    ──────────────────────────
UserMapper                       1. 扫描到这个接口
  extends BaseMapper<User>       2. 拿到泛型 User
        ↓                        3. 反射读取 User 类信息：
        泛型是关键！                 - 类名 User → 表名 t_user
                                   - 属性 id → 列 id
                                   - 属性 createTime → 列 create_time
                                 4. 在内存中拼出 SQL 模板
                                 5. 注册到 MyBatis，等价于你自己写了 XML
```

**通俗理解**：纯 MyBatis 是去餐厅自己写菜单、厨师按单做菜；MyBatis-Plus 是你只说"我要吃 User 这道菜"，厨师已经把做法背下来了，查、增、改、删四种口味自动端上来。

### 1.3 泛型替换机制

`BaseMapper<T>` 中所有方法的泛型 `T` 会被替换成你指定的实体类：

```java
// BaseMapper 内部长这样（简化版）：
public interface BaseMapper<T> {
    int insert(T entity);                          // 插入一条
    int deleteById(Serializable id);               // 按 ID 删除
    int updateById(@Param("et") T entity);         // 按 ID 更新
    T selectById(Serializable id);                 // 按 ID 查询
    List<T> selectList(@Param("ew") Wrapper<T> qw);// 条件查询列表
    long selectCount(@Param("ew") Wrapper<T> qw);  // 条件查询总数
}
```

当你写 `UserMapper extends BaseMapper<User>`，`T` 全部变成 `User`，你的 `UserMapper` 就自动拥有了以上所有方法，一行 SQL 不用写。

### 1.4 表名与列名的推断规则

核心靠两个约定实现自动映射：

| 映射关系 | 规则 | 示例 |
|----------|------|------|
| 类名 → 表名 | 驼峰转下划线 + 可选前缀 | `User` → `t_user`，`SeckillOrder` → `t_seckill_order` |
| 属性名 → 列名 | 驼峰转下划线 | `createTime` → `create_time`，`goodsName` → `goods_name` |

`application.yml` 中这行配置控制驼峰转换：
```yaml
mybatis-plus:
  configuration:
    map-underscore-to-camel-case: true
```

如果表名或列名不遵循约定，用注解覆盖：

```java
@TableName("t_user")              // 显式指定表名
public class User {

    @TableId(type = IdType.AUTO)   // 指定主键策略为自增
    private Long id;

    private String nickname;        // 自动映射到 nickname 列
}
```

### 1.5 BaseMapper 的能力边界

| 能做 | 做不了 |
|------|--------|
| 单表的增删改查 | 多表联查（JOIN） |
| 简单条件查询（WHERE） | 复杂业务逻辑（如"库存 > 0 才扣减"） |
| 分页、排序 | 统计分组（GROUP BY + 聚合函数） |

超出能力的部分，仍需手写 SQL 或使用 `@Select` 注解。例如秒杀项目中的扣减库存：

```java
// SeckillGoodsMapper 需要额外定义：库存 > 0 时才扣减
@Update("UPDATE seckill_goods SET stock_count = stock_count - 1 " +
        "WHERE goods_id = #{goodsId} AND stock_count > 0")
int reduceStock(@Param("goodsId") Long goodsId);
```

> **核心总结**：BaseMapper 解决的是 80% 的重复 CRUD，剩下 20% 的复杂查询仍需手写。

---

## 二、QueryWrapper：用 Java 代码拼 SQL 条件

### 2.1 是什么

QueryWrapper 是 MyBatis-Plus 提供的条件构造器，**用 Java 链式调用代替手写 WHERE 子句**。

### 2.2 常见用法

```java
// 单条件：WHERE phone = '13800138000'
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("phone", phone);           // eq = equal（等于）
userMapper.selectOne(wrapper);        // selectOne = 只期望返回一条

// 多条件：WHERE user_id = 1 AND goods_id = 2
QueryWrapper<SeckillOrder> wrapper = new QueryWrapper<>();
wrapper.eq("user_id", userId);
wrapper.eq("goods_id", goodsId);
```

更多常用方法（作为速查表）：

| 方法 | 含义 | 对应 SQL |
|------|------|----------|
| `eq("col", val)` | 等于 | `col = val` |
| `ne("col", val)` | 不等于 | `col != val` |
| `gt("col", val)` | 大于 | `col > val` |
| `ge("col", val)` | 大于等于 | `col >= val` |
| `like("col", val)` | 模糊匹配 | `col LIKE '%val%'` |
| `orderByDesc("col")` | 降序 | `ORDER BY col DESC` |

### 2.3 为什么不直接写 SQL

简单查询用 QueryWrapper 更快更安全——它自动做参数化处理，防止 SQL 注入。复杂查询（多表联查、子查询）才需要手写 SQL。

> **开发建议**：能用 QueryWrapper 搞定的就不要写 SQL，减少出错概率。

---

## 三、DDD 四层架构

### 3.1 从三层到四层：为什么要多一层

传统三层架构和 DDD 四层架构的对应关系：

| 传统三层 | DDD 四层 | 职责 |
|----------|----------|------|
| Controller | Interface（接口层） | 接收请求、参数校验、返回响应 |
| Service | Application（应用层） | 编排业务流程、协调领域对象 |
| — | Domain（领域层） | 核心业务规则、实体、仓储接口 |
| DAO/Mapper | Infrastructure（基础设施层） | 数据库访问、外部服务调用 |

**关键区别**：传统架构把业务逻辑塞在 Service 里，DDD 把业务逻辑下沉到 Domain 层的实体中，Service 只做"协调"。

### 3.2 正确的调用链

```
Controller → Service → Repository → Mapper → DB
   接口层       应用层      领域层(接口)   基础设施层   数据库
```

注意：Domain 层定义 Repository **接口**，Infrastructure 层提供 **实现**。这样领域层不依赖具体技术（MyBatis、JPA 都行），实现了依赖倒置。

### 3.3 两种开发顺序

| 方式 | 顺序 | 特点 | 适用场景 |
|------|------|------|----------|
| 自底向上 | domain → infrastructure → application → controller | 先打好地基再盖楼 | 领域模型清晰、需求稳定 |
| 自顶向下 | controller → service → repository → mapper | 先定接口再填实现 | 需求不明确、先出原型 |

两种方式没有绝对优劣，实际开发中经常混用。

### 3.4 项目目录结构（以秒杀项目为例）

```
seckill/                              # 父工程（pom 类型）
├── pom.xml                           # 父 POM，管理公共依赖版本
│
├── seckill-domain/                   # 领域层（不依赖任何其他模块）
│   └── src/main/java/seckill/domain/
│       ├── entity/                        # 领域实体
│       │   ├── User.java
│       │   ├── Goods.java
│       │   ├── SeckillGoods.java
│       │   └── SeckillOrder.java
│       └── repository/                    # 仓储接口（只定义接口，不实现）
│           ├── UserRepository.java
│           ├── GoodsRepository.java
│           ├── SeckillGoodsRepository.java
│           └── SeckillOrderRepository.java
│
├── seckill-infrastructure/           # 基础设施层（依赖 domain 层）
│   └── src/main/java/seckill/
│       ├── mapper/                         # MyBatis-Plus Mapper
│       │   ├── UserMapper.java
│       │   ├── GoodsMapper.java
│       │   ├── SeckillGoodsMapper.java
│       │   └── SeckillOrderMapper.java
│       └── repository/                     # 仓储接口的实现
│           ├── UserRepositoryImpl.java
│           ├── GoodsRepositoryImpl.java
│           ├── SeckillGoodsRepositoryImpl.java
│           └── SeckillOrderRepositoryImpl.java
│
├── seckill-application/              # 应用层（依赖 domain 层）
│   └── src/main/java/seckill/service/
│       ├── GoodsApplicationService.java
│       ├── SeckillApplicationService.java
│       ├── OrderApplicationService.java
│       └── vo/                             # 视图对象
│           ├── GoodsVO.java
│           ├── SeckillGoodsVO.java
│           └── OrderVO.java
│
├── seckill-interface/                # 接口层（依赖 application 层）
│   └── src/main/java/seckill/
│       ├── controller/                     # REST Controller
│       │   ├── GoodsController.java
│       │   ├── SeckillController.java
│       │   └── OrderController.java
│       └── dto/                            # 数据传输对象
│           ├── request/
│           │   ├── LoginRequest.java
│           │   └── SeckillRequest.java
│           └── response/
│               └── Result.java
│
└── seckill-starter/                  # 启动模块（聚合所有模块）
    └── src/main/
        ├── java/seckill/SeckillApplication.java
        └── resources/
            ├── application.yml
            └── db/schema.sql
```

**依赖方向**：starter → interface → application → domain ← infrastructure。注意 domain 层不依赖任何其他模块，这是 DDD 的核心原则。

---

## 四、充血模型与贫血模型

### 4.1 什么是贫血模型

Entity 只是一个纯数据容器（字段 + getter/setter），所有业务逻辑都堆在 Service 里：

```java
// Entity：只有数据，没有行为
public class User {
    private Long id;
    private String name;
    private Integer status;
    // getter / setter ...
}

// Service：所有业务逻辑都在这里
public class UserService {
    public void activateUser(User user) {
        user.setStatus(1);                     // 改状态
        user.setUpdatedAt(LocalDateTime.now()); // 改时间
    }

    public boolean canLogin(User user) {
        return user.getStatus() == 1;          // 判断逻辑也在 Service
    }
}
```

**问题**：Entity 不知道自己能干什么、不能干什么，业务规则散落在各个 Service 中，容易出现重复判断、规则不一致。

### 4.2 什么是充血模型

**核心原则：谁拥有数据，谁就拥有操作这些数据的行为。**

```java
// Entity：既有数据，也有业务逻辑
public class User {
    private Long id;
    private String name;
    private Integer status;

    // 自己的状态，自己负责改变（且自带校验）
    public void activate() {
        if (this.status == 1) {
            throw new RuntimeException("用户已激活，不要重复操作");
        }
        this.status = 1;
        this.updatedAt = LocalDateTime.now();
    }

    // 自己的规则，自己来判断
    public boolean canLogin() {
        return this.status == 1;
    }
}
```

Service 变成薄薄的协调层：

```java
public class UserService {
    public void activateUser(Long userId) {
        User user = userRepository.findById(userId);  // 1. 取对象
        user.activate();                                // 2. 调对象自己的方法
        userRepository.save(user);                      // 3. 存回去
    }
}
```

### 4.3 一张图看懂区别

```
贫血模型：
┌──────────┐       ┌──────────────────┐
│  Entity   │      │     Service      │
│ (纯数据)  │ ←───  │ (所有业务逻辑都在这)│
└──────────┘       └──────────────────┘

充血模型：
┌──────────────────┐       ┌────────────┐
│     Entity       │      │   Service  │
│ (数据 + 业务逻辑)  │ ←───  │ (薄协调层)  │
└──────────────────┘       └────────────┘
```

### 4.4 什么时候该把逻辑下沉到 Entity

判断标准：**这段逻辑放到别的 Service 里还能不能复用？**

- 能复用 → 下沉到 Entity（充血）
- 涉及多个 Entity 协作 → 留在 Service（Service 本来就是干协调的）

### 4.5 Service 的标准三步套路

充血模型下，Service 永远只做三件事：

```
1. 从 Repository 拿到对象
2. 调对象自己的方法（业务逻辑在对象身上）
3. 通过 Repository 存回去
```

> **常见误区**：充血模型不是把 CRUD（save/delete/select）放到 Entity 里，而是把**业务规则**（判断、计算、状态变更）放到 Entity 里。数据存取仍然是 Repository 的事。

### 4.6 在秒杀项目中的应用

充血模型和 Repository 层改造的关系：

- Repository（基础设施层）只负责存取，返回有行为的 Entity
- Entity（领域层）包含业务规则，如"秒杀商品库存够不够""用户能不能参与秒杀"
- Service（应用层）做协调：取对象 → 调方法 → 存回去

---

## 五、核心概念速查表

| 概念 | 一句话解释 |
|------|-----------|
| BaseMapper | 通过泛型自动生成单表 CRUD 的 SQL |
| QueryWrapper | 用 Java 代码代替手写 WHERE 条件 |
| DDD 四层架构 | 把业务逻辑从 Service 下沉到 Domain 层 |
| 贫血模型 | Entity 只有数据，业务逻辑全在 Service |
| 充血模型 | Entity 同时拥有数据和业务行为 |
| Repository 接口 | Domain 层定义接口，Infrastructure 层实现，实现依赖倒置 |
