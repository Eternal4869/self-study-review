# Day24 详解 — Netty核心概念

## 〇、Netty与最新JDK特性（面试必知）

### 0.1 Netty演进

```
Netty演进时间线：

Netty 3.x (2008)  →  Netty 4.x (2013)  →  Netty 5.x (废弃)  →  Netty 4.1+ (当前)
       │                  │                    │                    │
       ▼                  ▼                    ▼                    ▼
    早期版本           重大重构              项目废弃            稳定版本
```

### 0.2 Netty与Virtual Threads（JDK21+）

Netty 4.1.100+开始支持JDK21的Virtual Threads：

```java
// Netty + Virtual Threads（JDK21+）
EventLoopGroup group = new NioEventLoopGroup();

// 使用虚拟线程执行器
ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();

ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(group)
         .channel(NioServerSocketChannel.class)
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             protected void initChannel(SocketChannel ch) {
                 ch.pipeline().addLast(new MyHandler());
             }
         });
```

---

## 一、Netty整体架构

### 1.1 Netty分层架构

```
Netty架构分层：

┌─────────────────────────────────────────────────────────┐
│                    用户层                                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
│  │ Bootstrap   │ │ Codec       │ │ Handler     │       │
│  │ (启动器)    │ │ (编解码器)  │ │ (处理器)    │       │
│  └─────────────┘ └─────────────┘ └─────────────┘       │
├─────────────────────────────────────────────────────────┤
│                    核心层                                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
│  │ Channel     │ │ EventLoop   │ │ Channel     │       │
│  │ (通道)      │ │ (事件循环)  │ │ Pipeline    │       │
│  └─────────────┘ └─────────────┘ └─────────────┘       │
├─────────────────────────────────────────────────────────┤
│                    传输层                                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
│  │ NIO         │ │ Epoll       │ │ Native      │       │
│  │ (Java NIO)  │ │ (Linux)     │ │ (本地)      │       │
│  └─────────────┘ └─────────────┘ └─────────────┘       │
└─────────────────────────────────────────────────────────┘
```

---

## 二、Reactor线程模型

### 2.1 单Reactor单线程

```
单Reactor单线程：

┌─────────────────────────────────────────────────────────┐
│                    Reactor线程                           │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐       │
│  │  接收连接   │ │  处理IO     │ │  业务处理   │       │
│  │  (Accept)   │ │  (Read/     │ │  (Business) │       │
│  │             │ │   Write)    │ │             │       │
│  └─────────────┘ └─────────────┘ └─────────────┘       │
└─────────────────────────────────────────────────────────┘

缺点：一个线程处理所有任务，容易成为瓶颈
```

### 2.2 单Reactor多线程

```
单Reactor多线程：

┌─────────────────────────────────────────────────────────┐
│                    Reactor线程                           │
│  ┌─────────────┐ ┌─────────────┐                       │
│  │  接收连接   │ │  处理IO     │                       │
│  └─────────────┘ └─────────────┘                       │
└─────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    工作线程池                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │
│  │ Thread1 │ │ Thread2 │ │ Thread3 │ │ Thread4 │      │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘      │
└─────────────────────────────────────────────────────────┘

优点：IO和业务处理分离
缺点：Reactor线程仍可能成为瓶颈
```

### 2.3 主从Reactor（Netty默认）

```
主从Reactor（Netty默认模型）：

┌─────────────────────────────────────────────────────────┐
│                    Main Reactor                         │
│  ┌─────────────┐                                       │
│  │  接收连接   │ ← BossGroup                          │
│  └─────────────┘                                       │
└─────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    Sub Reactor                          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │
│  │Selector1│ │Selector2│ │Selector3│ │Selector4│      │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘      │
└─────────────────────────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│                    工作线程池                            │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │
│  │ Thread1 │ │ Thread2 │ │ Thread3 │ │ Thread4 │      │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘      │
└─────────────────────────────────────────────────────────┘

Netty默认模型：BossGroup处理连接，WorkerGroup处理IO
```

---

## 三、EventLoop和EventLoopGroup

### 3.1 核心概念

| 概念 | 说明 |
|------|------|
| **EventLoop** | 事件循环，处理Channel上的所有IO事件 |
| **EventLoopGroup** | 事件循环组，管理多个EventLoop |
| **BossGroup** | 处理客户端连接 |
| **WorkerGroup** | 处理已连接的IO读写 |

### 3.2 代码示例

```java
// Netty服务端
public class NettyServer {
    public static void main(String[] args) throws Exception {
        // BossGroup：处理客户端连接
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        // WorkerGroup：处理已连接的IO
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                     .channel(NioServerSocketChannel.class)
                     .childHandler(new ChannelInitializer<SocketChannel>() {
                         @Override
                         protected void initChannel(SocketChannel ch) {
                             ch.pipeline().addLast(new MyHandler());
                         }
                     });
            
            ChannelFuture future = bootstrap.bind(8080).sync();
            System.out.println("Netty服务端启动，监听端口8080");
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

// 自定义Handler
public class MyHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("收到消息: " + buf.toString(CharsetUtil.UTF_8));
        
        // 回写消息
        ctx.writeAndFlush(Unpooled.copiedBuffer("Echo: " + buf, CharsetUtil.UTF_8));
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

---

## 四、ChannelPipeline和ChannelHandler

### 4.1 Pipeline结构

```
ChannelPipeline结构：

客户端数据 ──→ Inbound Handler 1 ──→ Inbound Handler 2 ──→ 业务处理
                                                                    │
客户端数据 ←── Outbound Handler 1 ←── Outbound Handler 2 ←────────┘

数据流向：
Inbound（入站）：从网络到应用程序
Outbound（出站）：从应用程序到网络
```

### 4.2 Handler类型

| Handler类型 | 说明 | 常用类 |
|-------------|------|--------|
| **Inbound** | 处理入站数据 | ChannelInboundHandlerAdapter |
| **Outbound** | 处理出站数据 | ChannelOutboundHandlerAdapter |
| **Duplex** | 同时处理入站和出站 | ChannelDuplexHandler |

### 4.3 Handler代码示例

```java
// 入站Handler
public class MyInboundHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("客户端连接建立");
    }
    
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ByteBuf buf = (ByteBuf) msg;
        System.out.println("收到数据: " + buf.toString(CharsetUtil.UTF_8));
        ctx.fireChannelRead(msg); // 传递给下一个Handler
    }
    
    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        System.out.println("客户端断开连接");
    }
}

// 出站Handler
public class MyOutboundHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        System.out.println("发送数据: " + msg);
        ctx.write(msg, promise); // 传递给下一个Handler
    }
}
```

---

## 五、ByteBuf

### 5.1 ByteBuf vs NIO Buffer

| 对比项 | ByteBuf | NIO Buffer |
|--------|---------|------------|
| **读写索引** | 独立的readerIndex和writerIndex | 共用position |
| **容量扩展** | 自动扩展 | 固定容量 |
| **引用计数** | 支持引用计数 | 不支持 |
| **池化** | 支持池化 | 不支持 |

### 5.2 ByteBuf使用示例

```java
// ByteBuf基本使用
public class ByteBufDemo {
    public static void main(String[] args) {
        // 创建ByteBuf
        ByteBuf buffer = Unpooled.buffer(1024);
        
        // 写入数据
        buffer.writeBytes("Hello Netty".getBytes());
        System.out.println("writerIndex: " + buffer.writerIndex()); // 11
        
        // 读取数据
        byte[] bytes = new byte[buffer.readableBytes()];
        buffer.readBytes(bytes);
        System.out.println("readerIndex: " + buffer.readerIndex()); // 11
        
        // 重置读索引
        buffer.resetReaderIndex();
        
        // 释放ByteBuf
        buffer.release();
    }
}
```

---

## 六、LeetCode题目解析

### 6.1 [20. 有效的括号（Easy）](https://leetcode.cn/problems/valid-parentheses/)

**题目描述**：
给定一个只包含 `'('`、`')'`、`'{'`、`'}'`、`'['` 和 `']'` 的字符串，判断字符串是否有效。

```java
class Solution {
    public boolean isValid(String s) {
        Stack<Character> stack = new Stack<>();
        
        for (char c : s.toCharArray()) {
            if (c == '(' || c == '{' || c == '[') {
                stack.push(c);
            } else {
                if (stack.isEmpty()) return false;
                
                char top = stack.pop();
                if (c == ')' && top != '(') return false;
                if (c == '}' && top != '{') return false;
                if (c == ']' && top != '[') return false;
            }
        }
        
        return stack.isEmpty();
    }
}
```

---

## 七、今日学习要点总结

### 核心要点归纳

1. **Netty架构**：用户层（Bootstrap/Codec/Handler）→ 核心层（Channel/EventLoop/Pipeline）→ 传输层（NIO/Epoll）
2. **Reactor模型**：Netty默认使用主从Reactor多线程模型
3. **EventLoop**：事件循环，处理Channel上的所有IO事件
4. **ChannelPipeline**：Handler链，处理入站和出站数据
5. **ByteBuf**：Netty的缓冲区，比NIO Buffer更强大

### 验收清单

- [ ] 能解释Netty的Reactor线程模型
- [ ] 能说明EventLoop的工作流程
- [ ] 能画出ChannelPipeline的处理器链结构
- [ ] 能对比ByteBuf和NIO Buffer的优缺点

---

## 八、练习建议

1. **Netty实践**：实现一个简单的Echo Server/Client
2. **Handler实践**：自定义编解码器
3. **Pipeline实践**：组合多个Handler实现复杂业务

---

## 九、扩展阅读

- 《Netty实战》第3章：Netty入门
- Netty官方文档：https://netty.io/
- GitHub：https://github.com/netty/netty
