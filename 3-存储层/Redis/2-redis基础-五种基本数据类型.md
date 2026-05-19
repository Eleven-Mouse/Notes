
>这是redis最基础的知识，但是不代表它们就好学


## String 字符串

Redis string字符串存储**字节序列**，包括文本，序列化对象和二进制数组，它常用于缓存但也支持其他功能，例如实现计数器和执行位运算

由于redis 的 key 是字符串类型，当我们使用字符串类型作为 value 时，实际上是将一个字符串 映射到另一个字符串上，string 数据类型在许多场景非常有用，例如缓存 HTML 片段或页面

- 命令使用

| 命令     | 简述                    | 使用                |
| ------ | --------------------- | ----------------- |
| GET    | 获取存储在给定 key 中的 vualue | GET name          |
| SET    | 设置存储在给定 key 中的 value  | SET name value    |
| DEL    | 删除存储在给定 key 中的 value  | DEL name          |
| INCR   | 将 key 存储的值加1(原子性)     | INCR key          |
| DECR   | 将 key 存储的值减1(原子性)     | DECR key          |
| INCRBY | 将 key 存储的值加上amount    | INCR key amount   |
| DECRBY | 将 key 存储的值减去amount    | DECRBY key amount |

-----
- 命令执行
```plaintext
> SET bike:1 Deimos
OK
> GET bike:1
"Deimos"

> MSET bike:1 "Deimos" bike:2 "Ares" bike:3 "Vanth"
OK
> MGET bike:1 bike:2 bike:3
1) "Deimos"
2) "Ares"
3) "Vanth"
```
> Tips: [`SET`]如果键已存在，即使该键关联的是非字符串值，`and` 命令也会替换键中已存储的任何值。因此，`and`  [`SET`]执行的是赋值操作。

-----
- 字符串应用
	字符串作为计数器
```
> SET total_crashes 0
OK
> INCR total_crashes
(integer) 1
> INCRBY total_crashes 10
(integer) 11
```
>该[`INCR`]命令将字符串值解析为整数，将其加一，最后将得到的值设置为新值。还有其他类似的命令，例如[`INCRBY`]和 [`DECR`] [`DECRBY`]。它们的内部执行过程始终相同，只是方式略有不同。

- 限制
 >请注意，1，Redis 的单个 string 最大可到 512MB
    2，大多数字符串命令像 GET、SET，基本是 O(1)，也就是耗时几乎不随字符串长度增加 但 SUBSTR / GETRANGE / SETRANGE 这类“按位置随机读写一段内容”的命令，可能是 O(n)，也就是字符串越大、操作范围越大，越慢

-----

## List 列表

Redis 列表是字符串值的链表。Redis 列表常用于：
- 实现栈和队列。
- 为后台工作系统构建队列管理。