
> IO 模型是理解 Netty、Kafka、Redis 等高性能中间件的基础。

---

## 一、知识树

```
IO 与 NIO
├── 三种 IO 模型
│   ├── BIO（阻塞 IO）
│   ├── NIO（非阻塞 IO）
│   └── AIO（异步 IO）
├── NIO 三大核心
│   ├── Buffer（缓冲区）
│   ├── Channel（通道）
│   └── Selector（选择器/多路复用器）
├── 多路复用技术
│   ├── select / poll / epoll
│   └── epoll ET vs LT
├── Reactor 模式
│   ├── 单 Reactor 单线程
│   ├── 单 Reactor 多线程
│   └── 主从 Reactor 多线程
├── Netty 基础
│   └── EventLoop / Channel / Pipeline / Handler
└── 面试题
```

---

## 二、三种 IO 模型

### 1. BIO（Blocking IO）

```
一请求一线程模型：

Client1 ──▶ Server ──▶ Thread1（read 阻塞等待数据）
Client2 ──▶ Server ──▶ Thread2（read 阻塞等待数据）
Client3 ──▶ Server ──▶ Thread3（read 阻塞等待数据）

问题：
  - 并连接数多时线程数膨胀
  - 线程创建销毁开销大
  - 大部分时间线程在阻塞等待（浪费资源）
```

```java
// BIO 服务端
ServerSocket serverSocket = new ServerSocket(8080);
while (true) {
    Socket socket = serverSocket.accept(); // 阻塞等待连接
    new Thread(() -> {
        try (InputStream in = socket.getInputStream()) {
            byte[] buf = new byte[1024];
            int len = in.read(buf); // 阻塞等待数据
        } catch (IOException e) { e.printStackTrace(); }
    }).start();
}
```

### 2. NIO（Non-blocking IO）

```
一个线程处理多个连接：

                  ┌─ Channel1（有数据可读）
Selector（轮询）──┤─ Channel2（无数据，跳过）
                  └─ Channel3（有数据可读）

核心：Selector + Channel + Buffer
优势：少量线程即可处理大量连接
```

```java
// NIO 核心 API
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false); // 非阻塞
serverChannel.register(selector, SelectionKey.OP_ACCEPT); // 注册事件

while (true) {
    selector.select(); // 阻塞直到有事件就绪
    Set<SelectionKey> keys = selector.selectedKeys();
    for (SelectionKey key : keys) {
        if (key.isAcceptable()) { /* 处理连接 */ }
        if (key.isReadable())   { /* 处理读取 */ }
    }
}
```

### 3. AIO（Asynchronous IO）

```
真正的异步 IO：
  应用发起 read 请求 → 立即返回 → 内核完成数据拷贝后回调通知

注意：
  - Windows 使用 IOCP 实现（真正异步）
  - Linux 的 AIO 并不完善，Netty 使用 epoll 模拟（本质还是 NIO）
  - Java AIO（AsynchronousChannel）在 Linux 上底层仍用 epoll
```

### 三种模型对比

| 维度 | BIO | NIO | AIO |
|------|-----|-----|-----|
| **阻塞方式** | 全程阻塞 | 非阻塞（事件驱动） | 异步（回调通知） |
| **线程模型** | 一连接一线程 | 一线程处理多连接 | 由 OS 内核完成 |
| **适用连接数** | 少（< 1000） | 多（万级） | 多（万级以上） |
| **编程复杂度** | 简单 | 较复杂 | 复杂 |
| **Linux 支持** | 原生 | epoll | 不完善 |
| **典型应用** | 传统 Tomcat（默认） | Netty / Kafka | Windows IOCP |

---

## 三、NIO 三大核心组件

### 1. Buffer（缓冲区）

```
ByteBuffer 内部结构：

  0     position         limit      capacity
  ┌───────┬────────────────┬──────────┐
  │ 已读  │   可读/写数据   │  未使用  │
  └───────┴────────────────┴──────────┘
          ▲ mark（可选标记位）

- capacity：缓冲区总容量（创建后不可变）
- position：当前读写位置
- limit：读写上限
- mark：标记 position 的快照（可选）
```

```java
ByteBuffer buf = ByteBuffer.allocate(1024); // 分配堆内缓冲区
// ByteBuffer buf = ByteBuffer.allocateDirect(1024); // 堆外内存（零拷贝场景）

buf.put("Hello".getBytes()); // 写入数据，position 前进
buf.flip();                   // 切换为读模式：limit=position, position=0
byte[] data = new byte[buf.remaining()];
buf.get(data);                // 读取数据
buf.clear();                  // 清空（重置 position=0, limit=capacity）
```

### 2. Channel（通道）

```
Channel 是双向的数据通道（区别于 Stream 的单向）

主要实现：
  FileChannel         → 文件读写
  SocketChannel       → TCP 客户端
  ServerSocketChannel → TCP 服务端
  DatagramChannel     → UDP

Channel 特点：
  - 双向（可读可写）
  - 可异步
  - 必须配合 Buffer 使用
```

### 3. Selector（多路复用器）

```
Selector 是 NIO 的核心：

  Thread
    │
    ▼
  Selector（管理多个 Channel）
    ├── Channel1 → OP_ACCEPT（连接就绪）
    ├── Channel2 → OP_READ（读就绪）
    ├── Channel3 → OP_WRITE（写就绪）
    └── Channel4 → OP_CONNECT（连接完成）

一个 Selector 线程可以管理数千个 Channel
```

---

## 四、多路复用技术：select / poll / epoll

| 维度 | select | poll | epoll |
|------|--------|------|-------|
| **最大连接数** | 1024（FD_SETSIZE） | 无限制 | 无限制 |
| **数据结构** | bitmap | 链表（pollfd） | 红黑树 + 就绪链表 |
| **工作方式** | 每次调用遍历全部 FD | 每次调用遍历全部 FD | 事件驱动，只返回就绪 FD |
| **内存拷贝** | 每次调用需要拷贝 | 每次调用需要拷贝 | 只在注册时拷贝（mmap 共享内存） |
| **时间复杂度** | O(n) | O(n) | O(1) |
| **适用场景** | 少量连接 | 中等连接 | 大量连接 |

### epoll 的 ET vs LT

```
LT（水平触发，Level Triggered）—— epoll 默认模式：
  只要 FD 上有数据可读，每次 epoll_wait 都会返回该 FD
  编程简单，不容易漏事件
  适用于：阻塞 IO、简单场景

ET（边缘触发，Edge Triggered）：
  只在 FD 状态变化时通知一次（新数据到达时通知一次）
  必须一次性读完所有数据（循环 read 直到 EAGAIN）
  效率更高，但编程复杂
  适用于：非阻塞 IO + 高性能场景（Nginx / Redis 使用 ET）
```

---

## 五、Reactor 模式

### 1. 单 Reactor 单线程

```
       Reactor Thread
       ┌──────────────────────┐
       │ Selector             │
       │   ├── Accept         │
       │   ├── Read           │
       │   └── Write          │
       └──────────────────────┘

问题：一个线程处理所有事件，性能瓶颈
适用：小规模场景（Redis 6.0 之前使用此模型）
```

### 2. 单 Reactor 多线程

```
       Reactor Thread（只负责 Accept + IO 读写）
       ┌──────────────────┐
       │ Selector         │
       │  ├── Accept      │
       │  ├── Read        │──▶ Worker Thread Pool ──▶ 业务处理
       │  └── Write       │◀── Worker Thread Pool ◀── 处理结果
       └──────────────────┘

优势：IO 和业务处理分离
问题：Reactor 单线程仍是瓶颈（高并发 Accept + IO）
```

### 3. 主从 Reactor 多线程

```
       Main Reactor（只负责 Accept）
       ┌──────────────────┐
       │ Selector         │
       │  └── Accept      │
       └────────┬─────────┘
                │ 新连接分配给 Sub Reactor
       ┌────────▼─────────┐
       │ Sub Reactor 1    │  Sub Reactor 2    Sub Reactor N
       │  ├── Read        │   ├── Read         ├── Read
       │  └── Write       │   └── Write        └── Write
       └──────────────────┘
                │
       Worker Thread Pool（业务处理）

Netty 模型：BossGroup（Main Reactor）+ WorkerGroup（Sub Reactor）
```

---

## 六、Netty 基础（了解级）

### 核心组件

| 组件 | 说明 |
|------|------|
| **EventLoop** | 事件循环（= Reactor 线程），处理 Channel 的 IO 操作 |
| **EventLoopGroup** | EventLoop 组，BossGroup 处理 Accept，WorkerGroup 处理 IO |
| **Channel** | 网络连接的抽象，代表一个打开的 Socket |
| **Pipeline** | ChannelPipeline，Channel 的处理链 |
| **Handler** | 处理器，分为 Inbound（入站）和 Outbound（出站） |
| **ByteBuf** | Netty 自己的字节缓冲区（比 JDK ByteBuffer 更好用） |

```java
// Netty Server 简易示例
EventLoopGroup bossGroup = new NioEventLoopGroup(1);   // Main Reactor
EventLoopGroup workerGroup = new NioEventLoopGroup();    // Sub Reactor
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(NioServerSocketChannel.class)
 .childHandler(new ChannelInitializer<SocketChannel>() {
     @Override
     protected void initChannel(SocketChannel ch) {
         ch.pipeline().addLast(new MyHandler());
     }
 });
b.bind(8080).sync();
```

### 为什么 Redis / Netty / Nginx 都用 epoll

```
1. epoll 支持百万级连接（select 只有 1024）
2. epoll 只返回就绪的 FD（不需要遍历所有 FD）
3. epoll 使用共享内存（减少内核态/用户态拷贝）
4. epoll 支持 ET 模式（效率更高）
5. Linux 是服务器主流 OS，epoll 是 Linux 下最优方案
```

---

## 七、面试题

### Q1：BIO、NIO、AIO 的区别？🔥🔥🔥🔥

BIO 是同步阻塞，一连接一线程；NIO 是同步非阻塞，使用 Selector 多路复用，一个线程处理多个连接；AIO 是真正的异步 IO，由操作系统完成后回调通知。Linux 下 AIO 不完善，Netty 底层使用 epoll 实现 NIO。

### Q2：NIO 的三大核心组件是什么？🔥🔥🔥

Buffer（缓冲区，读写数据）、Channel（双向通道）、Selector（多路复用器，管理多个 Channel 的事件）。

### Q3：select、poll、epoll 的区别？🔥🔥🔥🔥

select 有 1024 连接限制，每次全量遍历 FD；poll 去掉了限制但仍全量遍历；epoll 使用红黑树管理 FD + 事件驱动回调，只返回就绪的 FD，时间复杂度 O(1)，支持百万级连接。

### Q4：epoll 的 LT 和 ET 模式区别？🔥🔥

LT（水平触发）：有数据就持续通知，编程简单。ET（边缘触发）：只在状态变化时通知一次，必须一次读完所有数据，效率更高。Redis 和 Nginx 使用 ET 模式。

### Q5：Reactor 模式有哪几种？Netty 用的是哪种？🔥🔥🔥

三种：单 Reactor 单线程、单 Reactor 多线程、主从 Reactor 多线程。Netty 使用主从 Reactor 多线程：BossGroup（Accept）+ WorkerGroup（IO 读写）+ 业务线程池。

### Q6：为什么 Redis 单线程还这么快？🔥🔥

单线程避免了锁竞争和上下文切换；数据全在内存（纳秒级）；使用 epoll 多路复用处理大量连接；IO 多路复用 + 内存操作使得单线程足以支撑 10 万+ QPS。

---

## 八、常见误区

| 误区 | 正确理解 |
|------|----------|
| NIO 就是非阻塞 IO | NIO 是 Non-blocking IO，但在 Linux 上本质是 I/O 多路复用（同步非阻塞） |
| AIO 一定比 NIO 快 | Linux 下 AIO 不完善，性能可能不如 epoll（NIO） |
| epoll 是异步 IO | epoll 是同步非阻塞 IO（多路复用），不是异步 IO |
| Netty 用的是 AIO | Netty 废弃了 AIO（Linux 不完善），底层统一使用 NIO（epoll） |
| BIO 完全没用了 | 连接数少的场景（管理端、内部接口）BIO 更简单 |

---

## 九、实战场景

### 高性能 HTTP 服务选型

```
连接数 < 1000：BIO（Tomcat 默认 NIO，简单场景可用 BIO）
连接数 1K-10K：NIO（Netty / Tomcat NIO）
连接数 > 10K：Netty + epoll（C10K 问题）
文件传输：NIO FileChannel.transferTo()（零拷贝）
```

---

## 十、关联知识

- [35-Kafka架构](35-Kafka架构.md) - Kafka 使用 NIO + 零拷贝
- [38-微服务组件](38-微服务组件.md) - 网关（Netty 实现）
- [39-限流熔断降级](39-限流熔断降级.md) - 高并发场景的网络 IO
- [41-系统设计](41-系统设计.md) - 高性能系统设计
