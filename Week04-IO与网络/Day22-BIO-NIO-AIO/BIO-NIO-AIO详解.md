# Day22 详解 — BIO/NIO/AIO

## 〇、最新JDK IO特性（面试必知）

### 0.1 JDK IO演进时间线

```
JDK IO演进：

JDK 1.0 (1996)  →  JDK 1.4 (2002)  →  JDK 7 (2011)  →  JDK 11 (2018)  →  JDK 21 (2023)
     │                │                │                │                │
     ▼                ▼                ▼                ▼                ▼
   BIO              NIO              AIO             HttpClient       Virtual Threads
(阻塞IO)          (非阻塞IO)       (异步IO)         标准化            (虚拟线程)★★★
```

### 0.2 Virtual Threads（JDK21+，面试高频）

**Virtual Threads（虚拟线程）** 是JDK 21正式发布的特性（JEP 444），彻底改变了Java的并发IO模型。

```
虚拟线程 vs 平台线程：

平台线程（传统）：
┌─────────────────────────────────────────────────────────┐
│  1个平台线程 = 1个OS线程                                │
│  线程数量受限于OS线程数量（通常几千个）                  │
│  线程切换开销大                                         │
└─────────────────────────────────────────────────────────┘

虚拟线程（JDK21+）：
┌─────────────────────────────────────────────────────────┐
│  1个虚拟线程 ≠ 1个OS线程                                │
│  多个虚拟线程共享少量OS线程（M:N调度）                   │
│  可创建数百万个虚拟线程                                  │
│  IO阻塞时自动释放OS线程                                 │
└─────────────────────────────────────────────────────────┘
```

**虚拟线程核心特点**：

| 特性 | 说明 |
|------|------|
| **轻量级** | 创建成本极低，可创建数百万个 |
| **M:N调度** | 多个虚拟线程映射到少量OS线程 |
| **自动挂起** | IO阻塞时自动释放OS线程 |
| **兼容性** | 支持ThreadLocal，兼容现有代码 |
| **不需池化** | 每个任务创建一个虚拟线程，无需池化 |

**虚拟线程使用示例**：

```java
// JDK21+ 虚拟线程示例
public class VirtualThreadDemo {
    public static void main(String[] args) throws Exception {
        // 方式1：使用Executors创建虚拟线程执行器
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, 100_000).forEach(i -> {
                executor.submit(() -> {
                    Thread.sleep(Duration.ofSeconds(1));
                    return i;
                });
            });
        } // 自动关闭并等待所有任务完成
        
        // 方式2：直接创建虚拟线程
        Thread vThread = Thread.ofVirtual()
            .name("my-virtual-thread")
            .start(() -> {
                System.out.println("Running in virtual thread");
            });
        vThread.join();
        
        // 方式3：使用Thread.Builder
        Thread vt = Thread.ofVirtual()
            .name("task-" + 1)
            .unstarted(() -> {
                System.out.println("Task executed");
            });
        vt.start();
    }
}
```

**虚拟线程与IO**：

```java
// 虚拟线程 + IO = 简单的同步代码 + 高并发
public class VirtualThreadIODemo {
    public static void main(String[] args) throws Exception {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            // 10万个并发HTTP请求
            List<Future<String>> futures = IntStream.range(0, 100_000)
                .mapToObj(i -> executor.submit(() -> {
                    // 同步IO代码，但在虚拟线程中运行
                    // IO阻塞时自动释放OS线程
                    return fetchURL("https://api.example.com/data/" + i);
                }))
                .toList();
            
            // 获取所有结果
            for (Future<String> future : futures) {
                System.out.println(future.get());
            }
        }
    }
    
    static String fetchURL(String url) throws IOException {
        try (var in = URI.create(url).toURL().openStream()) {
            return new String(in.readAllBytes(), StandardCharsets.UTF_8);
        }
    }
}
```

### 0.3 HttpClient（JDK11+标准）

```java
// JDK11+ HttpClient示例
public class HttpClientDemo {
    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        
        // 同步请求
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://api.example.com/data"))
            .header("Accept", "application/json")
            .GET()
            .build();
        
        HttpResponse<String> response = client.send(request, 
            HttpResponse.BodyHandlers.ofString());
        
        System.out.println("状态码: " + response.statusCode());
        System.out.println("响应体: " + response.body());
        
        // 异步请求
        client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body)
            .thenAccept(System.out::println)
            .join();
    }
}
```

---

## 一、IO基本概念

### 1.1 同步 vs 异步

| 概念 | 说明 | 示例 |
|------|------|------|
| **同步** | 调用者主动等待结果 | 同步方法调用 |
| **异步** | 被调用者通过回调通知结果 | CompletableFuture、回调函数 |

### 1.2 阻塞 vs 非阻塞

| 概念 | 说明 | 示例 |
|------|------|------|
| **阻塞** | 线程被挂起，等待结果返回 | BIO、synchronized |
| **非阻塞** | 线程不等待，立即返回 | NIO、Selector |

### 1.3 组合类型

```
IO模型组合：

┌─────────────┬─────────────┬─────────────────────────────────┐
│             │   阻塞      │   非阻塞                        │
├─────────────┼─────────────┼─────────────────────────────────┤
│   同步      │  同步阻塞   │  同步非阻塞                     │
│             │  (BIO)      │  (NIO轮询)                     │
├─────────────┼─────────────┼─────────────────────────────────┤
│   异步      │  异步阻塞   │  异步非阻塞                     │
│             │  (不存在)   │  (AIO)                         │
└─────────────┴─────────────┴─────────────────────────────────┘
```

---

## 二、BIO（Blocking IO）

### 2.1 BIO模型

**BIO（Blocking IO）** 是同步阻塞IO模型，一个连接对应一个线程。

```
BIO工作流程：

客户端 ──→ 服务端
            │
            ▼
        ┌─────────┐
        │ 阻塞等待│ ← 客户端连接
        │ 连接    │
        └────┬────┘
             │
             ▼
        ┌─────────┐
        │ 创建线程│ ← 为每个客户端创建一个线程
        │ 处理    │
        └────┬────┘
             │
             ▼
        ┌─────────┐
        │ 阻塞读写│ ← 等待客户端数据
        │ 数据    │
        └─────────┘
```

### 2.2 BIO代码示例

```java
// BIO服务端
public class BioServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("BIO服务端启动，监听端口8080");
        
        while (true) {
            // 阻塞等待客户端连接
            Socket socket = serverSocket.accept();
            System.out.println("客户端连接: " + socket.getInetAddress());
            
            // 为每个客户端创建一个新线程
            new Thread(() -> {
                try {
                    InputStream in = socket.getInputStream();
                    byte[] buffer = new byte[1024];
                    int len;
                    
                    // 阻塞读取数据
                    while ((len = in.read(buffer)) != -1) {
                        String data = new String(buffer, 0, len);
                        System.out.println("收到数据: " + data);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}

// BIO客户端
public class BioClient {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("localhost", 8080);
        OutputStream out = socket.getOutputStream();
        
        out.write("Hello BIO".getBytes());
        out.flush();
        
        socket.close();
    }
}
```

### 2.3 BIO的问题

| 问题 | 说明 |
|------|------|
| **线程开销大** | 每个连接需要一个线程 |
| **资源浪费** | 大部分时间线程在等待IO |
| **并发限制** | 无法处理大量并发连接 |

---

## 三、NIO（Non-blocking IO）

### 3.1 NIO模型

**NIO（Non-blocking IO）** 是同步非阻塞IO模型，使用多路复用器（Selector）实现单线程处理多个连接。

```
NIO工作流程：

客户端 ──→ 服务端
            │
            ▼
        ┌─────────────┐
        │   Selector  │ ← 多路复用器，监听多个Channel
        │  (选择器)    │
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    │          │          │
    ▼          ▼          ▼
┌───────┐ ┌───────┐ ┌───────┐
│Channel│ │Channel│ │Channel│ ← 多个Channel
│  1    │ │  2    │ │  3    │
└───────┘ └───────┘ └───────┘
```

### 3.2 NIO核心组件

| 组件 | 说明 |
|------|------|
| **Channel** | 数据通道，双向的（可读可写） |
| **Buffer** | 数据缓冲区 |
| **Selector** | 多路复用器，选择就绪的Channel |

### 3.3 NIO代码示例

```java
// NIO服务端
public class NioServer {
    public static void main(String[] args) throws IOException {
        // 创建Selector
        Selector selector = Selector.open();
        
        // 创建ServerSocketChannel并注册到Selector
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false); // 非阻塞模式
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        
        System.out.println("NIO服务端启动，监听端口8080");
        
        while (true) {
            // 阻塞等待就绪事件
            selector.select();
            
            // 遍历就绪的SelectionKey
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();
                
                if (key.isAcceptable()) {
                    // 处理新连接
                    SocketChannel clientChannel = serverChannel.accept();
                    clientChannel.configureBlocking(false);
                    clientChannel.register(selector, SelectionKey.OP_READ);
                } else if (key.isReadable()) {
                    // 处理读事件
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    int len = channel.read(buffer);
                    if (len > 0) {
                        buffer.flip();
                        String data = new String(buffer.array(), 0, len);
                        System.out.println("收到数据: " + data);
                    }
                }
            }
        }
    }
}
```

---

## 四、AIO（Asynchronous IO）

### 4.1 AIO模型

**AIO（Asynchronous IO）** 是异步非阻塞IO模型，基于回调机制。

```
AIO工作流程：

客户端 ──→ 服务端
            │
            ▼
        ┌─────────────┐
        │  异步操作   │ ← 发起异步IO操作
        │  (回调)     │
        └──────┬──────┘
               │ IO完成时
               ▼
        ┌─────────────┐
        │  回调通知   │ ← 自动回调完成方法
        │  处理结果   │
        └─────────────┘
```

### 4.2 AIO代码示例

```java
// AIO服务端
public class AioServer {
    public static void main(String[] args) throws IOException {
        AsynchronousServerSocketChannel serverChannel = 
            AsynchronousServerSocketChannel.open()
                .bind(new InetSocketAddress(8080));
        
        System.out.println("AIO服务端启动，监听端口8080");
        
        // 异步等待连接
        serverChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel result, Void attachment) {
                // 接受到新连接
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                
                // 异步读取数据
                result.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer bytesRead, ByteBuffer buffer) {
                        buffer.flip();
                        String data = new String(buffer.array(), 0, bytesRead);
                        System.out.println("收到数据: " + data);
                    }
                    
                    @Override
                    public void failed(Throwable exc, ByteBuffer buffer) {
                        exc.printStackTrace();
                    }
                });
                
                // 继续接受新连接
                serverChannel.accept(null, this);
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });
        
        // 阻塞主线程
        Thread.currentThread().join();
    }
}
```

---

## 五、三种IO模型对比

### 5.1 对比表

| 特性 | BIO | NIO | AIO |
|------|-----|-----|-----|
| **IO模型** | 同步阻塞 | 同步非阻塞 | 异步非阻塞 |
| **线程模型** | 1连接1线程 | 多路复用 | 回调通知 |
| **并发能力** | 低 | 高 | 高 |
| **编程复杂度** | 低 | 高 | 高 |
| **适用场景** | 连接数少 | 连接数多 | 连接数多 |

### 5.2 选择建议

```
IO模型选择：

┌─────────────────────────────────────────────────────────┐
│  场景1：连接数少，数据量大                               │
│  → 选择BIO                                              │
│  例如：传统的数据库连接                                  │
├─────────────────────────────────────────────────────────┤
│  场景2：连接数多，数据量小                               │
│  → 选择NIO                                              │
│  例如：聊天服务器、即时通讯                              │
├─────────────────────────────────────────────────────────┤
│  场景3：连接数多，需要高并发                             │
│  → 选择NIO + Reactor模式（Netty）                       │
│  例如：Web服务器、游戏服务器                             │
├─────────────────────────────────────────────────────────┤
│  场景4：JDK21+，追求简单+高并发                         │
│  → 选择Virtual Threads + 同步IO                        │
│  例如：微服务、REST API                                  │
└─────────────────────────────────────────────────────────┘
```

---

## 六、LeetCode题目解析

### 6.1 [155. 最小栈（Medium）](https://leetcode.cn/problems/min-stack/)

**题目描述**：
设计一个支持 `push`，`pop`，`top` 操作，并能在常数时间内检索到最小元素的栈。

**思路分析**：
使用辅助栈存储最小值。

```java
class MinStack {
    private Stack<Integer> stack;
    private Stack<Integer> minStack;
    
    public MinStack() {
        stack = new Stack<>();
        minStack = new Stack<>();
    }
    
    public void push(int val) {
        stack.push(val);
        if (minStack.isEmpty() || val <= minStack.peek()) {
            minStack.push(val);
        }
    }
    
    public void pop() {
        int val = stack.pop();
        if (val == minStack.peek()) {
            minStack.pop();
        }
    }
    
    public int top() {
        return stack.peek();
    }
    
    public int getMin() {
        return minStack.peek();
    }
}
```

**复杂度分析**：
- 时间复杂度：O(1)
- 空间复杂度：O(n)

### 6.2 [232. 用栈实现队列（Easy）](https://leetcode.cn/problems/implement-queue-using-stacks/)

**题目描述**：
使用两个栈实现队列。

**思路分析**：
一个栈用于入队，一个栈用于出队。

```java
class MyQueue {
    private Stack<Integer> stackIn;
    private Stack<Integer> stackOut;
    
    public MyQueue() {
        stackIn = new Stack<>();
        stackOut = new Stack<>();
    }
    
    public void push(int x) {
        stackIn.push(x);
    }
    
    public int pop() {
        if (stackOut.isEmpty()) {
            while (!stackIn.isEmpty()) {
                stackOut.push(stackIn.pop());
            }
        }
        return stackOut.pop();
    }
    
    public int peek() {
        if (stackOut.isEmpty()) {
            while (!stackIn.isEmpty()) {
                stackOut.push(stackIn.pop());
            }
        }
        return stackOut.peek();
    }
    
    public boolean empty() {
        return stackIn.isEmpty() && stackOut.isEmpty();
    }
}
```

**复杂度分析**：
- 时间复杂度：push O(1)，pop/amortized O(1)
- 空间复杂度：O(n)

---

## 七、今日学习要点总结

### 核心要点归纳

1. **BIO**：同步阻塞，1连接1线程，适合连接数少的场景
2. **NIO**：同步非阻塞，使用Selector多路复用，适合连接数多的场景
3. **AIO**：异步非阻塞，基于回调机制
4. **Virtual Threads（JDK21+）**：虚拟线程，M:N调度，可创建数百万个，IO阻塞时自动释放OS线程
5. **HttpClient（JDK11+）**：标准化的HTTP客户端API

### 验收清单

- [ ] 能解释同步vs异步、阻塞vs非阻塞的区别
- [ ] 能说明BIO的缺点和NIO的改进
- [ ] 能说出三种IO模型各自的适用场景
- [ ] 能解释Virtual Threads的核心特点（面试高频）
- [ ] 能说出Virtual Threads的使用方式

---

## 八、练习建议

1. **代码实践**：分别实现BIO/NIO/AIO的Echo Server
2. **Virtual Threads实践**：使用JDK21+创建大量虚拟线程
3. **对比测试**：比较BIO和NIO在高并发下的性能差异

---

## 九、扩展阅读

- 《Netty实战》第2章：Java NIO
- JEP 444: Virtual Threads：https://openjdk.org/jeps/444
- JEP 321: HTTP Client (Standard)：https://openjdk.org/jeps/321
- Oracle官方文档：Java NIO和Virtual Threads
