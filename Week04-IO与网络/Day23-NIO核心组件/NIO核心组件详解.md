# Day23 详解 — NIO核心组件

## 一、NIO概述

### 1.1 NIO三大核心组件

```
NIO核心组件关系：

┌─────────────────────────────────────────────────────────┐
│                        NIO                              │
├─────────────┬─────────────┬─────────────────────────────┤
│   Channel   │   Buffer    │        Selector             │
│   (通道)    │   (缓冲区)  │        (选择器)             │
├─────────────┼─────────────┼─────────────────────────────┤
│  数据传输   │  数据存储   │  事件监听                    │
│  双向读写   │  读写中转   │  多路复用                    │
└─────────────┴─────────────┴─────────────────────────────┘
```

---

## 二、Channel（通道）

### 2.1 Channel概述

Channel是NIO中数据传输的通道，**双向的**（可读可写），与传统的Stream（单向）不同。

### 2.2 Channel类型

| Channel类型 | 说明 | 适用场景 |
|-------------|------|---------|
| **FileChannel** | 文件IO通道 | 文件读写 |
| **SocketChannel** | TCP客户端通道 | 客户端连接 |
| **ServerSocketChannel** | TCP服务端通道 | 服务端监听 |
| **DatagramChannel** | UDP通道 | UDP通信 |

### 2.3 Channel代码示例

```java
// FileChannel示例
public class FileChannelDemo {
    public static void main(String[] args) throws IOException {
        // 写入文件
        FileOutputStream fos = new FileOutputStream("test.txt");
        FileChannel writeChannel = fos.getChannel();
        
        ByteBuffer buffer = ByteBuffer.wrap("Hello NIO".getBytes());
        writeChannel.write(buffer);
        writeChannel.close();
        
        // 读取文件
        FileInputStream fis = new FileInputStream("test.txt");
        FileChannel readChannel = fis.getChannel();
        
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        readChannel.read(readBuffer);
        
        readBuffer.flip();
        String data = new String(readBuffer.array(), 0, readBuffer.limit());
        System.out.println("读取内容: " + data);
        
        readChannel.close();
    }
}
```

---

## 三、Buffer（缓冲区）

### 3.1 Buffer概述

Buffer是NIO中数据读写的中转容器，本质是一个数组。

### 3.2 Buffer核心属性

```
Buffer核心属性：

capacity（容量）：缓冲区最大容量，创建后不可变
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  position                              limit             │
│    │                                   │                 │
│    ▼                                   ▼                 │
├─────────────────────────────────────────────────────────┤
│  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │
│  0  1  2  3  4  5  6  7  8  9  10 11 12 13 14 15      │
│                                                         │
│  capacity = 16                                          │
│  position = 当前读写位置                                │
│  limit = 可读写边界                                     │
└─────────────────────────────────────────────────────────┘
```

| 属性 | 说明 |
|------|------|
| **capacity** | 缓冲区最大容量，创建后不可变 |
| **position** | 当前读写位置，下一个要读/写的索引 |
| **limit** | 可读写边界，limit之后的数据不可访问 |

### 3.3 Buffer核心方法

```java
// Buffer读写流程
public class BufferDemo {
    public static void main(String[] args) {
        // 1. 创建Buffer
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        System.out.println("初始状态: " + buffer); // position=0, limit=capacity
        
        // 2. 写入数据
        buffer.put("Hello".getBytes());
        System.out.println("写入后: " + buffer); // position=5, limit=capacity
        
        // 3. flip()切换到读模式
        buffer.flip();
        System.out.println("flip后: " + buffer); // position=0, limit=5
        
        // 4. 读取数据
        byte[] bytes = new byte[buffer.remaining()];
        buffer.get(bytes);
        System.out.println("读取: " + new String(bytes));
        
        // 5. clear()或compact()重置Buffer
        buffer.clear(); // position=0, limit=capacity
    }
}
```

### 3.4 flip/clear/rewind区别

```
Buffer模式切换：

写模式（初始）：
┌─────────────────────────────────────────┐
│  position = 写入数据量                   │
│  limit = capacity                       │
└─────────────────────────────────────────┘
                 │
                 │ flip()
                 ▼
┌─────────────────────────────────────────┐
│  position = 0                           │
│  limit = 写入数据量                      │
└─────────────────────────────────────────┘
                 │
                 │ clear() 或 compact()
                 ▼
┌─────────────────────────────────────────┐
│  position = 0                           │
│  limit = capacity                       │
└─────────────────────────────────────────┘
```

| 方法 | 作用 |
|------|------|
| **flip()** | 写模式 → 读模式，position=0, limit=之前position |
| **clear()** | 重置Buffer，position=0, limit=capacity |
| **compact()** | 保留未读数据，position=未读数据量 |
| **rewind()** | 重新读取，position=0 |

---

## 四、Selector（选择器）

### 4.1 Selector概述

Selector是NIO的多路复用器，可以监控多个Channel的事件，实现单线程处理多个连接。

### 4.2 SelectionKey事件类型

| 事件 | 说明 | 触发条件 |
|------|------|---------|
| **OP_ACCEPT** | 接收连接 | 服务端收到新连接 |
| **OP_CONNECT** | 连接就绪 | 客户端连接建立完成 |
| **OP_READ** | 读就绪 | 通道可读取数据 |
| **OP_WRITE** | 写就绪 | 通道可写入数据 |

### 4.3 Selector代码示例

```java
// NIO Echo Server
public class NioEchoServer {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false);
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        
        System.out.println("NIO Echo Server启动");
        
        while (true) {
            selector.select();
            
            Iterator<SelectionKey> it = selector.selectedKeys().iterator();
            while (it.hasNext()) {
                SelectionKey key = it.next();
                it.remove();
                
                if (key.isAcceptable()) {
                    handleAccept(key, selector);
                } else if (key.isReadable()) {
                    handleRead(key);
                }
            }
        }
    }
    
    private static void handleAccept(SelectionKey key, Selector selector) throws IOException {
        ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
        SocketChannel clientChannel = serverChannel.accept();
        clientChannel.configureBlocking(false);
        clientChannel.register(selector, SelectionKey.OP_READ);
        System.out.println("新连接: " + clientChannel.getRemoteAddress());
    }
    
    private static void handleRead(SelectionKey key) throws IOException {
        SocketChannel channel = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        
        int bytesRead = channel.read(buffer);
        if (bytesRead == -1) {
            channel.close();
            return;
        }
        
        buffer.flip();
        byte[] data = new byte[buffer.remaining()];
        buffer.get(data);
        
        // Echo回显
        channel.write(ByteBuffer.wrap(data));
    }
}
```

---

## 五、NIO vs BIO对比

| 对比项 | BIO | NIO |
|--------|-----|-----|
| **IO模型** | 同步阻塞 | 同步非阻塞 |
| **线程模型** | 1连接1线程 | 多路复用 |
| **数据处理** | 面向流（Stream） | 面向缓冲区（Buffer） |
| **阻塞点** | read/write阻塞 | selector阻塞 |
| **适用场景** | 连接数少 | 连接数多 |

---

## 六、LeetCode题目解析

### 6.1 [225. 用队列实现栈（Easy）](https://leetcode.cn/problems/implement-stack-using-queues/)

**题目描述**：
使用两个队列实现栈。

```java
class MyStack {
    private Queue<Integer> queue1;
    private Queue<Integer> queue2;
    
    public MyStack() {
        queue1 = new LinkedList<>();
        queue2 = new LinkedList<>();
    }
    
    public void push(int x) {
        queue2.offer(x);
        while (!queue1.isEmpty()) {
            queue2.offer(queue1.poll());
        }
        Queue<Integer> temp = queue1;
        queue1 = queue2;
        queue2 = temp;
    }
    
    public int pop() {
        return queue1.poll();
    }
    
    public int top() {
        return queue1.peek();
    }
    
    public boolean empty() {
        return queue1.isEmpty();
    }
}
```

### 6.2 [150. 逆波兰表达式求值（Medium）](https://leetcode.cn/problems/evaluate-reverse-polish-notation/)

**题目描述**：
根据逆波兰表示法，求表达式的值。

```java
class Solution {
    public int evalRPN(String[] tokens) {
        Stack<Integer> stack = new Stack<>();
        
        for (String token : tokens) {
            switch (token) {
                case "+":
                    stack.push(stack.pop() + stack.pop());
                    break;
                case "-":
                    int b = stack.pop();
                    int a = stack.pop();
                    stack.push(a - b);
                    break;
                case "*":
                    stack.push(stack.pop() * stack.pop());
                    break;
                case "/":
                    int divisor = stack.pop();
                    int dividend = stack.pop();
                    stack.push(dividend / divisor);
                    break;
                default:
                    stack.push(Integer.parseInt(token));
            }
        }
        
        return stack.pop();
    }
}
```

---

## 七、今日学习要点总结

### 核心要点归纳

1. **Channel**：双向数据通道，支持FileChannel、SocketChannel、ServerSocketChannel
2. **Buffer**：数据缓冲区，核心属性capacity/position/limit，flip/clear切换模式
3. **Selector**：多路复用器，单线程处理多个Channel，支持OP_ACCEPT/OP_READ/OP_WRITE事件
4. **NIO vs BIO**：NIO面向缓冲区，BIO面向流；NIO多路复用，BIO 1连接1线程

### 验收清单

- [ ] 能说出Channel和Buffer的关系
- [ ] 能解释Buffer的flip/clear/rewind方法作用
- [ ] 能说明Selector如何实现多路复用
- [ ] 能手写简单的NIO Echo Server

---

## 八、练习建议

1. **Buffer实践**：熟悉Buffer的读写流程和模式切换
2. **Selector实践**：实现NIO Echo Server
3. **对比BIO**：比较BIO和NIO在高并发下的性能差异

---

## 九、扩展阅读

- 《Netty实战》第2章：Java NIO
- Oracle官方文档：Java NIO
- JEP 444: Virtual Threads：https://openjdk.org/jeps/444
