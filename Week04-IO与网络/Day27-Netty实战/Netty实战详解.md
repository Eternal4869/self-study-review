# Day27 详解 — Netty实战

## 〇、Netty最新特性（面试必知）

### 0.1 Netty版本演进

```
Netty版本演进：

Netty 4.0 (2013)  →  Netty 4.1 (2015)  →  Netty 5.x (废弃)  →  Netty 4.1.100+ (当前)
       │                  │                    │                    │
       ▼                  ▼                    ▼                    ▼
    重大重构           长期支持             项目废弃            支持JDK21+
```

### 0.2 Netty与Virtual Threads

```java
// Netty 4.1.100+ 支持Virtual Threads
EventLoopGroup group = new NioEventLoopGroup();

// 使用虚拟线程执行器
ExecutorService virtualExecutor = Executors.newVirtualThreadPerTaskExecutor();

ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(group)
         .channel(NioServerSocketChannel.class)
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             protected void initChannel(SocketChannel ch) {
                 ch.pipeline().addLast(
                     new MyDecoder(),
                     new MyEncoder(),
                     new MyHandler()
                 );
             }
         });
```

---

## 一、编解码器原理

### 1.1 编码器（Encoder）

编码器将**出站数据**从应用程序格式转换为网络格式。

```java
// 自定义编码器
public class MyEncoder extends MessageToByteEncoder<MyMessage> {
    @Override
    protected void encode(ChannelHandlerContext ctx, MyMessage msg, ByteBuf out) {
        // 写入消息类型
        out.writeByte(msg.getType());
        // 写入消息长度
        int length = msg.getData().length;
        out.writeInt(length);
        // 写入消息数据
        out.writeBytes(msg.getData());
    }
}
```

### 1.2 解码器（Decoder）

解码器将**入站数据**从网络格式转换为应用程序格式。

```java
// 自定义解码器
public class MyDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 至少需要5字节（1字节类型 + 4字节长度）
        if (in.readableBytes() < 5) {
            return;
        }
        
        // 标记当前读位置
        in.markReaderIndex();
        
        // 读取消息类型
        byte type = in.readByte();
        // 读取消息长度
        int length = in.readInt();
        
        // 检查数据是否完整
        if (in.readableBytes() < length) {
            in.resetReaderIndex();
            return;
        }
        
        // 读取消息数据
        byte[] data = new byte[length];
        in.readBytes(data);
        
        // 构建消息对象
        MyMessage msg = new MyMessage(type, data);
        out.add(msg);
    }
}
```

### 1.3 常用编解码器

| 编解码器 | 说明 | 适用场景 |
|---------|------|---------|
| **StringDecoder/Encoder** | 字符串编解码 | 文本协议 |
| **ObjectDecoder/Encoder** | Java对象编解码 | 序列化对象 |
| **HttpRequestDecoder** | HTTP请求解码 | HTTP服务 |
| **HttpResponseEncoder** | HTTP响应编码 | HTTP客户端 |
| **ProtobufDecoder** | Protobuf解码 | Protobuf协议 |

---

## 二、心跳机制

### 2.1 IdleStateHandler

Netty使用`IdleStateHandler`实现心跳检测。

```java
// 心跳检测配置
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
         .channel(NioServerSocketChannel.class)
         .childHandler(new ChannelInitializer<SocketChannel>() {
             @Override
             protected void initChannel(SocketChannel ch) {
                 ch.pipeline().addLast(
                     // 超时检测：60秒没有读事件触发
                     new IdleStateHandler(60, 0, 0, TimeUnit.SECONDS),
                     new MyDecoder(),
                     new MyEncoder(),
                     new MyHandler()
                 );
             }
         });
```

### 2.2 心跳处理

```java
// 心跳处理器
public class MyHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            switch (event.state()) {
                case READER_IDLE:
                    System.out.println("读空闲超时，关闭连接: " + ctx.channel().remoteAddress());
                    ctx.close();
                    break;
                case WRITER_IDLE:
                    System.out.println("写空闲超时");
                    break;
                case ALL_IDLE:
                    System.out.println("读写空闲超时");
                    break;
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
}
```

---

## 三、断线重连

### 3.1 客户端断线重连

```java
// 客户端断线重连
public class ClientReconnect {
    public static void main(String[] args) {
        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(new NioEventLoopGroup())
                 .channel(NioSocketChannel.class)
                 .handler(new ChannelInitializer<SocketChannel>() {
                     @Override
                     protected void initChannel(SocketChannel ch) {
                         ch.pipeline().addLast(new ClientHandler());
                     }
                 });
        
        connect(bootstrap, "localhost", 8080);
    }
    
    private static void connect(Bootstrap bootstrap, String host, int port) {
        bootstrap.connect(host, port).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) {
                if (future.isSuccess()) {
                    System.out.println("连接成功");
                } else {
                    System.out.println("连接失败，3秒后重连...");
                    future.channel().eventLoop().schedule(() -> {
                        connect(bootstrap, host, port);
                    }, 3, TimeUnit.SECONDS);
                }
            }
        });
    }
}
```

---

## 四、粘包/拆包问题

### 4.1 问题原因

```
TCP粘包/拆包原因：

1. Nagle算法：小数据包合并发送
2. 缓冲区：数据积攒到一定量才发送
3. MTU限制：数据超过最大传输单元会拆分

发送方：[消息1][消息2][消息3]
         │
         ▼ TCP传输
接收方：[消息1消息2][消息3]  ← 粘包
或：    [消息1][消息2部分][消息2部分+消息3]  ← 拆包
```

### 4.2 解决方案

```java
// 方案1：固定长度
ch.pipeline().addLast(new FixedLengthFrameDecoder(1024));

// 方案2：分隔符
ch.pipeline().addLast(new DelimiterBasedFrameDecoder(1024, 
    Unpooled.copiedBuffer("\r\n", CharsetUtil.UTF_8)));

// 方案3：长度字段（推荐）
ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(
    1024,   // 最大长度
    0,      // 长度字段偏移
    4,      // 长度字段占用字节数
    0,      // 预期长度调整
    4       // 跳过字节数
));
```

---

## 五、Netty实战示例

### 5.1 完整Echo Server

```java
// Netty Echo Server
public class NettyEchoServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                     .channel(NioServerSocketChannel.class)
                     .option(ChannelOption.SO_BACKLOG, 128)
                     .childOption(ChannelOption.SO_KEEPALIVE, true)
                     .childHandler(new ChannelInitializer<SocketChannel>() {
                         @Override
                         protected void initChannel(SocketChannel ch) {
                             ch.pipeline().addLast(
                                 new IdleStateHandler(30, 0, 0, TimeUnit.SECONDS),
                                 new StringDecoder(CharsetUtil.UTF_8),
                                 new StringEncoder(CharsetUtil.UTF_8),
                                 new EchoHandler()
                             );
                         }
                     });
            
            ChannelFuture future = bootstrap.bind(8080).sync();
            System.out.println("Netty Echo Server启动，监听端口8080");
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}

// Echo Handler
public class EchoHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        String message = (String) msg;
        System.out.println("收到: " + message);
        
        // Echo回显
        ctx.writeAndFlush("Echo: " + message + "\r\n");
    }
    
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            System.out.println("心跳超时，关闭连接");
            ctx.close();
        }
        super.userEventTriggered(ctx, evt);
    }
    
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

---

## 六、LeetCode题目解析

### 6.1 [21. 合并两个有序链表（Easy）](https://leetcode.cn/problems/merge-two-sorted-lists/)

```java
class Solution {
    public ListNode mergeTwoLists(ListNode list1, ListNode list2) {
        ListNode dummy = new ListNode(0);
        ListNode curr = dummy;
        
        while (list1 != null && list2 != null) {
            if (list1.val <= list2.val) {
                curr.next = list1;
                list1 = list1.next;
            } else {
                curr.next = list2;
                list2 = list2.next;
            }
            curr = curr.next;
        }
        
        curr.next = (list1 != null) ? list1 : list2;
        return dummy.next;
    }
}
```

### 6.2 [142. 环形链表 II（Medium）](https://leetcode.cn/problems/linked-list-cycle-ii/)

```java
public class Solution {
    public ListNode detectCycle(ListNode head) {
        ListNode slow = head, fast = head;
        
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            
            if (slow == fast) {
                // 找到相遇点
                ListNode ptr = head;
                while (ptr != slow) {
                    ptr = ptr.next;
                    slow = slow.next;
                }
                return ptr; // 入口点
            }
        }
        
        return null; // 无环
    }
}
```

---

## 七、今日学习要点总结

### 核心要点归纳

1. **编解码器**：Encoder（出站编码）和Decoder（入站解码）
2. **心跳机制**：IdleStateHandler检测空闲超时
3. **断线重连**：ChannelFutureListener监听连接状态
4. **粘包/拆包**：FixedLengthFrameDecoder、DelimiterBasedFrameDecoder、LengthFieldBasedFrameDecoder

### 验收清单

- [ ] 能自定义编解码器处理自定义协议
- [ ] 能实现心跳检测和超时处理
- [ ] 能解决TCP粘包/拆包问题
- [ ] 能实现客户端断线重连

---

## 八、练习建议

1. **自定义协议**：设计并实现自定义协议的编解码器
2. **心跳检测**：实现完整的心跳检测机制
3. **断线重连**：实现客户端自动重连

---

## 九、扩展阅读

- 《Netty实战》第6-9章：编解码器、ChannelHandler
- Netty官方文档：https://netty.io/wiki/
- GitHub示例：https://github.com/netty/netty/tree/main/example
