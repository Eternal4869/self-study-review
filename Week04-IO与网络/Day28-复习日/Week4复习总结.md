# Day28 详解 — Week4复习总结

## 一、本周核心知识点回顾

### 1.1 知识脉络图

```
Week4 IO与网络 知识脉络：

┌─────────────────────────────────────────────────────────────────────────────┐
│                          IO与网络 知识体系                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
    │   IO模型      │       │   网络协议     │       │   框架        │
    └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
            │                       │                       │
    ┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
    │               │       │               │       │               │
    ▼               ▼       ▼               ▼       ▼               ▼
┌───────┐     ┌───────┐ ┌───────┐     ┌───────┐ ┌───────┐     ┌───────┐
│ BIO   │     │ NIO   │ │ TCP   │     │ HTTP  │ │ Netty │     │Virtual│
│ NIO   │     │ AIO   │ │ 三次  │     │ HTTPS │ │       │     │Threads│
│ AIO   │     │       │ │ 握手  │     │       │ │       │     │(JDK21+)│
└───────┘     └───────┘ └───────┘     └───────┘ └───────┘     └───────┘
```

### 1.2 各Day知识点速览

| Day | 主题 | 核心知识点 |
|-----|------|-----------|
| Day22 | BIO/NIO/AIO | 同步/异步、阻塞/非阻塞、三种IO模型对比 |
| Day23 | NIO核心组件 | Channel、Buffer、Selector多路复用 |
| Day24 | Netty核心概念 | Reactor模型、EventLoop、ChannelPipeline |
| Day25 | TCP三次握手四次挥手 | 握手/挥手流程、TIME_WAIT、状态转换 |
| Day26 | HTTP/HTTPS | HTTP/1.1 vs HTTP/2、HTTPS加密流程 |
| Day27 | Netty实战 | 编解码器、心跳机制、断线重连、粘包拆包 |

---

## 二、IO模型速记

### 2.1 三种IO模型

```
IO模型对比：

BIO（同步阻塞）：
├── 1连接1线程
├── 阻塞在read/write
└── 适用：连接数少

NIO（同步非阻塞）：
├── Selector多路复用
├── 面向Buffer
└── 适用：连接数多

AIO（异步非阻塞）：
├── 回调机制
├── 异步操作
└── 适用：连接数多，高并发
```

### 2.2 Virtual Threads（JDK21+，面试高频）

```
Virtual Threads核心特点：

┌─────────────────────────────────────────────────────────┐
│  轻量级线程，M:N调度                                     │
│  可创建数百万个                                          │
│  IO阻塞时自动释放OS线程                                  │
│  支持ThreadLocal                                        │
│  不需要池化                                              │
└─────────────────────────────────────────────────────────┘

代码示例：
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return i;
        });
    });
}
```

---

## 三、NIO核心组件速记

### 3.1 三大组件

```
NIO三大组件：

Channel（通道）：
├── 双向数据传输
├── FileChannel、SocketChannel、ServerSocketChannel
└── 注册到Selector

Buffer（缓冲区）：
├── 数据读写中转
├── 核心属性：capacity/position/limit
└── flip()切换读写模式

Selector（选择器）：
├── 多路复用
├── 监听Channel事件
└── 单线程处理多个连接
```

### 3.2 Buffer操作速记

```
Buffer模式切换：

写模式 → flip() → 读模式
读模式 → clear()/compact() → 写模式

关键方法：
flip(): position=0, limit=之前position
clear(): position=0, limit=capacity
compact(): 保留未读数据
```

---

## 四、Netty核心概念速记

### 4.1 Netty架构

```
Netty分层架构：

用户层：Bootstrap/Codec/Handler
核心层：Channel/EventLoop/Pipeline
传输层：NIO/Epoll/Native
```

### 4.2 Reactor模型

```
Netty默认模型：主从Reactor

BossGroup：处理客户端连接
WorkerGroup：处理IO读写

代码：
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
         .channel(NioServerSocketChannel.class)
         .childHandler(...);
```

---

## 五、TCP三次握手四次挥手速记

### 5.1 三次握手

```
三次握手（建立连接）：

客户端 → 服务端：SYN=1, SEQ=x
客户端 ← 服务端：SYN=1, ACK=1, SEQ=y, ACK=x+1
客户端 → 服务端：ACK=1, SEQ=x+1, ACK=y+1

作用：防止历史连接，同步序列号
```

### 5.2 四次挥手

```
四次挥手（断开连接）：

客户端 → 服务端：FIN=1, SEQ=u
客户端 ← 服务端：ACK=1, SEQ=v, ACK=u+1
客户端 ← 服务端：FIN=1, ACK=1, SEQ=w, ACK=u+1
客户端 → 服务端：ACK=1, SEQ=u+1, ACK=w+1

作用：全双工通信需要四次
```

---

## 六、HTTP/HTTPS速记

### 6.1 HTTP版本对比

| 版本 | 特点 |
|------|------|
| HTTP/1.1 | 持久连接、管线化 |
| HTTP/2 | 二进制协议、多路复用、头部压缩 |
| HTTP/3 | 基于QUIC/UDP、无队头阻塞、0-RTT |

### 6.2 HTTPS加密流程

```
HTTPS加密：

ClientHello → ServerHello(证书) → 验证证书 → 生成密钥 → 加密通信

TLS 1.3改进：1-RTT握手，强制前向保密
```

---

## 七、常见面试题汇总

### 7.1 IO模型相关

**Q1: BIO、NIO、AIO的区别？**

A: BIO同步阻塞，1连接1线程；NIO同步非阻塞，Selector多路复用；AIO异步非阻塞，回调机制。

**Q2: 什么是Virtual Threads？**

A: JDK21引入的虚拟线程，M:N调度，可创建数百万个，IO阻塞时自动释放OS线程，不需要池化。

**Q3: NIO的三大组件是什么？**

A: Channel（双向通道）、Buffer（缓冲区）、Selector（多路复用器）。

### 7.2 Netty相关

**Q4: Netty的Reactor模型是什么？**

A: 主从Reactor多线程模型，BossGroup处理连接，WorkerGroup处理IO。

**Q5: 如何解决TCP粘包/拆包问题？**

A: 使用FixedLengthFrameDecoder、DelimiterBasedFrameDecoder或LengthFieldBasedFrameDecoder。

### 7.3 TCP相关

**Q6: 为什么需要三次握手？**

A: 防止历史连接，同步序列号，避免资源浪费。

**Q7: TIME_WAIT的作用是什么？**

A: 等待2MSL，确保最后一个ACK到达，防止旧连接的数据包干扰新连接。

### 7.4 HTTP相关

**Q8: HTTP/1.1和HTTP/2的主要区别？**

A: HTTP/2是二进制协议，支持多路复用、头部压缩、服务器推送。

**Q9: HTTPS的加密流程是什么？**

A: 非对称加密交换密钥，对称加密传输数据，CA证书验证身份。

---

## 八、闭卷默写验收清单

### 8.1 IO模型

- [ ] 能解释同步vs异步、阻塞vs非阻塞
- [ ] 能说出BIO/NIO/AIO的区别和适用场景
- [ ] 能解释Virtual Threads的核心特点

### 8.2 NIO

- [ ] 能说出Channel、Buffer、Selector的作用
- [ ] 能解释Buffer的flip/clear方法
- [ ] 能手写NIO Echo Server

### 8.3 Netty

- [ ] 能解释Reactor模型
- [ ] 能说明ChannelPipeline的工作原理
- [ ] 能解释心跳机制和断线重连

### 8.4 TCP/HTTP

- [ ] 能画出三次握手和四次挥手流程
- [ ] 能解释TIME_WAIT的作用
- [ ] 能说出HTTP/2的主要特性

---

## 九、本周学习建议

### 9.1 强化练习

1. **NIO实践**：实现NIO Echo Server
2. **Netty实践**：实现完整的Netty服务端/客户端
3. **抓包分析**：使用Wireshark观察TCP/HTTP协议

### 9.2 面试准备

1. **背诵面试题**：IO模型、Virtual Threads、TCP握手挥手
2. **代码手写**：NIO Echo Server、Netty基础代码
3. **画图讲解**：Reactor模型、HTTPS加密流程

### 9.3 下周预告

第五周将学习**Spring核心**，包括：
- Spring IOC启动流程
- Bean生命周期
- AOP原理
- 循环依赖解决方案

---

## 十、扩展阅读

- 《Netty实战》- Trustin Lee
- 《TCP/IP详解》第1卷：协议
- 《HTTP权威指南》
- JEP 444: Virtual Threads：https://openjdk.org/jeps/444

---

**本周学习完成！继续加油！** 🚀
