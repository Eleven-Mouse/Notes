

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

## 五、RabbitMQ 死信队列（DLQ）

### 5.1 什么是死信队列

死信队列（Dead Letter Queue）是 RabbitMQ 的内置机制：**当消息无法被正常消费时，自动转发到一个专门的队列**，而不是丢弃或无限重试。

消息变成"死信"的三种情况：
1. 消费者调用 `basicNack(requeue=false)` 拒绝消息
2. 消息在队列中存活时间超过 TTL（过期）
3. 队列满了，新消息进不来

### 5.2 为什么需要死信队列

秒杀场景下，消费失败的常见原因：

```
数据库连接池耗尽 → 短暂性故障，重试可能成功
库存不足抛异常 → 持久性故障，重试永远失败
代码 bug（NPE 等）→ 必须修复代码才能恢复
```

**没有死信队列时**：`basicNack(requeue=true)` 把消息放回原队列，消费者立刻又消费，又失败，又放回……形成死循环。后果：
- CPU 空转，日志刷屏
- 原队列被失败消息占满，正常消息排不上
- 无法区分"正在重试"和"彻底失败"

**有死信队列时**：`basicNack(requeue=false)` 消息不再回到原队列，而是路由到死信队列。正常队列不受影响，死信队列可以单独监控、告警、人工排查。

### 5.3 配置方法

核心：**在声明主队列时通过 arguments 指定死信目的地**。

```java
// 1. 主队列：声明时绑定 DLX 参数
@Bean
public Queue seckillQueue() {
    Map<String, Object> args = new HashMap<>();
    args.put("x-dead-letter-exchange", "seckill.dlx.exchange");    // 死信去哪个交换机
    args.put("x-dead-letter-routing-key", "seckill.dlx");          // 用什么 routing key
    return new Queue("seckill.queue", true, false, false, args);
}

// 2. 死信交换机（类型和主交换机无关，可以不同）
@Bean
public DirectExchange dlxExchange() {
    return new DirectExchange("seckill.dlx.exchange", true, false);
}

// 3. 死信队列
@Bean
public Queue dlxQueue() {
    return new Queue("seckill.dlx.queue", true);
}

// 4. 绑定
@Bean
public Binding dlxBinding() {
    return BindingBuilder.bind(dlxQueue()).to(dlxExchange()).with("seckill.dlx");
}
```

### 5.4 消息流转全景

```
Producer → seckill.exchange → seckill.queue → Consumer
                                           ├─ ACK → 消息删除
                                           └─ NACK(requeue=false)
                                                ↓ x-dead-letter-exchange 路由
                                          seckill.dlx.exchange
                                                ↓ routing key
                                          seckill.dlx.queue
                                                ↓ 人工处理 / 自动补偿
```

> **关键点**：死信队列不是额外的代码逻辑，而是 RabbitMQ 的内置行为。你只需要在声明队列时配好参数，拒绝消息时用 `requeue=false`，RabbitMQ 自动帮你转发。

---

## 六、Redis 原子操作在秒杀中的应用

### 6.1 为什么秒杀库存要放 Redis

秒杀的瞬时并发极高（万级 QPS），如果直接打到数据库：
- 每个请求都执行 `UPDATE ... SET stock = stock - 1`，数据库扛不住
- 行锁竞争严重，响应时间飙升

Redis 的核心优势：**单线程 + 内存操作，天然保证原子性，且性能是数据库的 100 倍以上**。

### 6.2 DECR 和 INCR 的原子性

Redis 的 `DECR`/`INCR` 是单条命令，执行过程中不会被其他命令打断：

```
时刻 T1: 线程 A 执行 DECR → Redis 从 100 变成 99（原子完成）
时刻 T2: 线程 B 执行 DECR → Redis 从 99 变成 98（原子完成）
```

不需要加锁，不存在"两个线程同时读到 100，都减成 99"的问题。

### 6.3 Lua 脚本：多步操作的原子化

`DECR` 只能减 1，但秒杀需要"先判断库存 > 0，再减"。两步操作之间如果有并发，可能出问题：

```
线程 A: GET stock → 1（还有库存）
线程 B: GET stock → 1（还有库存）
线程 A: DECR stock → 0（扣减成功）
线程 B: DECR stock → -1（超卖了！）
```

Lua 脚本把多步操作合并为一次原子执行：

```lua
local stock = tonumber(redis.call('get', KEYS[1]))
if stock and stock > 0 then
    redis.call('decr', KEYS[1])
    return 1   -- 成功
end
return 0       -- 库存不足
```

Redis 执行 Lua 脚本时是**单线程阻塞**的，整个脚本作为一个不可分割的原子操作，不会被其他命令插入。

### 6.4 库存回滚：消费失败时的 INCR

秒杀流程是"Redis 先扣 → MQ → 消费者再扣 DB"，两步之间存在时间差。如果消费者处理失败（DB 库存不足、代码异常），Redis 的库存已经被扣了，不会自动恢复。

解决方案：消费失败时调用 `INCR` 回补。

```java
// 消费者 catch 块
catch (Exception e) {
    cacheService.increment("seckill:stock:" + goodsId);  // 回补
    channel.basicNack(tag, false, false);                 // 进死信队列
}
```

> **注意**：`INCR` 是简单的 +1，不需要判断边界。因为只有之前 `DECR` 成功（库存 > 0）才会发 MQ，所以回补一定是在一个已扣减的值上加回去，不会溢出。

---

## 七、动态秒杀路径防刷

### 7.1 问题：固定接口路径 = 裸奔

`POST /seckill?userId=1&goodsId=1` 路径固定，攻击者可以：
- 提前写好脚本，活动开始时瞬间批量调用
- 不需要任何前端交互，直接 HTTP 请求
- 一个脚本每秒打几千次

### 7.2 解决思路：动态 Token 路径

核心思想：**秒杀接口不直接暴露，而是需要一个一次性的随机 Token 才能访问**。

```
正常用户流程：
  1. 前端调 GET /seckill/path?userId=1&goodsId=1
  2. 后端生成 UUID，存入 Redis（TTL=60s），返回给前端
  3. 前端用这个 UUID 调 POST /seckill?userId=1&goodsId=1&path=UUID
  4. 后端验证：Redis 中是否存在这个 UUID → 验证通过后立即删除（一次性）

机器人视角：
  必须先调一次 GET 拿 UUID
  拿到后必须在 60s 内用掉
  每个 UUID 只能用一次
  增加了一次网络往返 + 路径不可预测 → 攻击成本大幅提升
```

### 7.3 实现要点

```java
// 1. 生成路径（GET 接口）
@GetMapping("/seckill/path")
public Result getSeckillPath(Long userId, Long goodsId) {
    String uuid = UUID.randomUUID().toString().replace("-", "");
    // 存入 Redis，60s 过期
    cacheService.setCache("seckill:path:" + userId + ":" + goodsId, uuid, 60, TimeUnit.SECONDS);
    return Result.success(uuid);
}

// 2. 验证路径（POST 接口，第一步）
Object cachedPath = cacheService.getCache("seckill:path:" + userId + ":" + goodsId);
if (cachedPath == null || !cachedPath.equals(path)) {
    return Result.error("无效的秒杀路径");    // null = 过期或没申请，不匹配 = 伪造
}
cacheService.deleteCache("seckill:path:" + userId + ":" + goodsId);  // 一次性，用完即删
```

### 7.4 防刷体系层次

秒杀项目中用到了三层防刷，层层递进：

| 层次 | 手段 | 防什么 |
|------|------|--------|
| 第一层 | 动态路径（UUID 一次性） | 防机器人直接脚本调用 |
| 第二层 | SETNX 限流（60s TTL） | 防同一用户重复下单 |
| 第三层 | Redis 预扣库存 | 防超卖，库存为 0 直接拒绝 |

> **面试要点**：动态路径不是万能的，它防的是"直接脚本调用"。真正的高级攻击（模拟浏览器、分布式 IP）还需要配合验证码、IP 限流等手段。

---

## 八、MQ 消费失败的补偿策略

### 8.1 问题：消费者处理失败了怎么办

MQ 的设计目标是"异步削峰"，但异步意味着生产者已经返回"排队中"了，消费者失败时生产者不知道。

秒杀场景下失败的影响：
- Redis 库存被扣了但没下单成功 → 库存虚耗
- 用户轮询结果永远是"排队中" → 体验差
- 消息丢了 → 订单丢失

### 8.2 补偿三件套

消费者 catch 块中按顺序完成三件事：

```java
catch (Exception e) {
    // ① 写失败标记 → 前端轮询能立刻返回"失败"
    cacheService.setCache("seckill:user:" + userId + ":" + goodsId, "-1", 60, SECONDS);

    // ② 回补 Redis 库存 → 恢复库存数字
    cacheService.increment("seckill:stock:" + goodsId);

    // ③ 清除限流标记 → 允许用户重新参与
    cacheService.deleteCache("seckill:limit:" + userId + ":" + goodsId);

    // ④ 消息进死信队列 → 不丢失，可后续人工排查
    channel.basicNack(tag, false, false);
}
```

**执行顺序很重要**：
1. 先写 `-1` → 用户最关心的是"我到底成功没"，这个要最先响应
2. 再回补库存 → 恢复系统状态
3. 再清限流 → 给用户重新参与的机会
4. 最后 NACK → 消息安全进入 DLQ

### 8.3 DB 扣库存的返回值检查

`reduceStock` 的 SQL 使用了 `WHERE stock_count > 0` 条件：

```sql
UPDATE t_seckill_goods SET stock_count = stock_count - 1
WHERE id = ? AND stock_count > 0
```

当库存为 0 时，这条 SQL 影响 0 行。如果代码不检查返回值，会继续创建订单，造成超卖：

```java
// 错误写法：忽略返回值
seckillGoodsRepository.reduceStock(goodsId);  // 返回 0 也继续
seckillOrderRepository.save(order);            // 超卖订单被创建

// 正确写法：检查返回值
int affected = seckillGoodsRepository.reduceStock(goodsId);
if (affected == 0) {
    throw new RuntimeException("库存不足");  // 抛异常 → 进 catch 补偿流程
}
```

> **经验总结**：任何 UPDATE/DELETE 操作都应该检查返回的影响行数，它是数据库给你最直接的反馈。

---

## 九、MQ 消息可靠性三件套

### 9.1 三个环节，三道防线

消息从生产到消费的完整链路中，每个环节都可能丢消息：

```
Producer → [Broker: Exchange → Queue] → Consumer
   ①           ②                          ③
```

| 环节 | 可能丢消息的场景 | 防线 |
|------|-----------------|------|
| ① Producer → Broker | 网络抖动、Broker 宕机 | Publisher Confirm |
| ② Broker 内部 | Broker 重启，内存中的消息全丢 | 消息持久化（durable queue + persistent message） |
| ③ Broker → Consumer | Consumer 处理到一半挂了 | 手动 ACK |

### 9.2 Publisher Confirm（生产者确认）

**原理**：消息到达 Broker 后，Broker 异步回调 ACK/NACK 告诉生产者结果。

```yaml
# application.yml
spring:
  rabbitmq:
    publisher-confirm-type: correlated   # 开启 confirm
    publisher-returns: true              # 消息不可路由时触发 return 回调
```

```java
// RabbitTemplate 回调
rabbitTemplate.setConfirmCallback((correlationData, ack, cause) -> {
    if (!ack) {
        log.error("消息确认失败: {}", cause);
        // 重发 or 记录到数据库做补偿
    }
});
```

### 9.3 消息持久化

两步缺一不可：
- **队列持久化**：`new Queue("seckill.queue", true)` — `durable=true`
- **消息持久化**：Spring AMQP 默认 `deliveryMode=2`（persistent），不需要额外配置

### 9.4 手动 ACK

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        acknowledge-mode: manual
```

```java
// 消费成功
channel.basicAck(deliveryTag, false);

// 消费失败，重新入队
channel.basicNack(deliveryTag, false, true);

// 消费失败，不重新入队（进死信队列）
channel.basicNack(deliveryTag, false, false);
```

> **面试总结**：三道防线全配上，才能做到消息**至少投递一次（at-least-once）**。配合消费者的幂等性检查，实现"Exactly-Once"的效果。

---

## 十、秒杀系统的 Redis Key 设计总结

完成 Phase 5-6 后，系统中所有 Redis Key 的全景：

| Key 模式 | 操作 | 生命周期 | 用途 |
|----------|------|----------|------|
| `seckill:stock:{goodsId}` | 启动时 SET，秒杀时 DECR，失败时 INCR | 活动期间持久 | 库存预扣减 |
| `seckill:limit:{userId}:{goodsId}` | SETNX，60s TTL | 每次秒杀 60s | 防重复下单 |
| `seckill:path:{userId}:{goodsId}` | SET，60s TTL，验证后 DELETE | 一次性 | 动态路径防刷 |
| `seckill:user:{userId}:{goodsId}` | 消费成功写 orderId，失败写 "-1 | 60s | 结果轮询 |
| `seckill:goods:list` | 缓存 60s | 商品列表缓存 |
| `seckill:goods:{id}` | 缓存 60s | 商品详情缓存 |

> **设计原则**：每个 Key 的命名遵循 `业务:模块:维度` 的层级结构，方便排查和监控。TTL 一定要设置，避免内存泄漏。







