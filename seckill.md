
 ---
 # MyBatis-Plus 的 BaseMapper 为什么能自动生成 SQL？                                       
     
  先回到没有 MyBatis-Plus 的时代

  纯 MyBatis 时代，你要自己写一切：

 ```
 // Mapper 接口 — 自己定义方法
  public interface UserMapper {
      User selectById(Long id);
      List<User> selectAll();
      int insert(User user);
      int update(User user);
      int deleteById(Long id);
  }

  <!-- 还要写对应的 XML，每一条 SQL 都要手写 -->
  <select id="selectById" resultType="User">
      SELECT id, nickname, password, salt, phone, create_time
      FROM t_user
      WHERE id = #{id}
  </select>

  <insert id="insert">
      INSERT INTO t_user (nickname, password, salt, phone, create_time)
      VALUES (#{nickname}, #{password}, #{salt}, #{phone}, #{createTime})
  </insert>
  <!-- update、delete 也要写... -->
 ```
  痛点： 每张表都要写一遍几乎相同的 SQL，全是体力活。

  MyBatis-Plus 做了什么？

  一句话：它帮你把那些"千篇一律的 SQL"自动生成了。

  原理是这样的：

  你写的代码：

  UserMapper extends BaseMapper<User>
                         ↑
                这个泛型 User 就是关键！

  MyBatis-Plus 启动时做的事：
    1. 扫描到你的 UserMapper
    2. 看到 BaseMapper<User>，知道了泛型是 User 类
    3. 读取 User 类的信息：
       - 类名 User → 推断表名 t_user（驼峰转下划线）
       - 属性 id → 推断列 id
       - 属性 nickname → 推断列 nickname
       - 属性 createTime → 推断列 create_time
    4. 自动在内存里拼出 SQL：
       SELECT id, nickname, password, salt, phone, create_time
       FROM t_user
       WHERE id = ?
    5. 把拼好的 SQL 注册到 MyBatis 里，就像你自己写的一样

  通俗比喻

  没有 MyBatis-Plus：
    你去餐厅吃饭，要自己写菜单（SQL），厨师（MyBatis）按菜单做菜

  有了 MyBatis-Plus：
    你只需要说"我要吃 User 这道菜"（BaseMapper<User>）
    厨师已经把 User 这道菜的做法背下来了
    自动给你做出：查、增、改、删 四种口味

    你不用写菜单了！

  BaseMapper 内部给你提供了哪些方法？

  // BaseMapper<T> 里面大概长这样（简化版）：
  public interface BaseMapper<T> {

      // 插入一条
      int insert(T entity);

      // 根据 ID 删除
      int deleteById(Serializable id);

      // 根据 ID 更新
      int updateById(@Param("et") T entity);

      // 根据 ID 查询
      T selectById(Serializable id);

      // 查询所有
      List<T> selectList(@Param("ew") Wrapper<T> queryWrapper);

      // 查总数
      long selectCount(@Param("ew") Wrapper<T> queryWrapper);
  }

  所以你写 UserMapper extends BaseMapper<User>，你的 UserMapper 就自动拥有了上面所有方法，泛型 T 被替换成了 User。

  它是怎么推断表名和列名的？

  核心靠两个约定：

  约定 1：类名 → 表名
    User        → t_user（自动加前缀 t_，驼峰转下划线）
    SeckillOrder → t_seckill_order

  约定 2：属性名 → 列名
    goodsName → goods_name（驼峰转下划线）
    createTime → create_time

  这就是 application.yml 里这行配置的作用：
    map-underscore-to-camel-case: true

  如果你的表名/列名不遵循约定，可以用注解覆盖：

  @TableName("t_user")           // 指定表名
  public class User {

      @TableId(type = IdType.AUTO)  // 指定主键自增
      private Long id;

      private String nickname;       // 自动映射到 nickname 列
  }

  但是！复杂 SQL 它搞不定

  BaseMapper 能搞定的：单表的增删改查 ✅
  BaseMapper 搞不定的：
    - 多表联查 → 还是要自己写 SQL 或用 @Select 注解
    - 复杂的扣减库存（乐观锁） → 需要自己写
    - 统计、分组 → 需要自己写

  所以在我们的秒杀项目里，SeckillGoodsMapper 就需要额外写一个 reduceStock 方法，因为"库存 > 0 时才扣减"这个逻辑不是普通
  CRUD 能覆盖的。




## ddd四层架构和传统三层架构在开发上的区别
 
  其实有两种开发顺序：
     
  │   方式   │    顺序  │         特点         │

  │ 自底向上 │ domain → infrastructure → application → controller │ 先打好地基，再盖楼   │

  │ 自顶向下 │ controller → service → repository → mapper    │ 先定好接口，再填实现 │
  
  正确的调用链：
    Controller → Service → Repository → Mapper → DB

 ---
  QueryWrapper 是什么？

  简单理解：用 Java 代码拼 SQL 的 WHERE 条件。

  // 等价于 SQL：SELECT * FROM t_user WHERE phone = '13800138000'
  QueryWrapper<User> wrapper = new QueryWrapper<>();
  wrapper.eq("phone", phone);             // eq = equal（等于）
  userMapper.selectOne(wrapper);          // selectOne = 只查一条

  // 多条件：
  // 等价于 SQL：WHERE user_id = 1 AND goods_id = 2
  QueryWrapper<SeckillOrder> wrapper = new QueryWrapper<>();
  wrapper.eq("user_id", userId);
  wrapper.eq("goods_id", goodsId);

  为什么用 QueryWrapper 不直接写 SQL？ 简单查询用 QueryWrapper 更快更安全（自动防 SQL 注入），复杂查询才需要自己写 SQL。

