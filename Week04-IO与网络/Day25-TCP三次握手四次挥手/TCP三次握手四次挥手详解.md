# Day25 详解 — TCP三次握手四次挥手

## 一、TCP报文结构

### 1.1 TCP头部字段

```
TCP头部结构（20字节）：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port          │       Destination Port          │
├───────────────────────────────┼───────────────────────────────┤
│                        Sequence Number                         │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                       │
├───────┼───────┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│  Data │       │C│E│U│A│P│R│S│F│                               │
│ Offset│ Rsrvd │W│C│R│C│S│S│Y│I│            Window             │
│       │       │R│E│G│K│H│T│N│N│                               │
├───────┴───────┴─┴─┴─┴─┴─┴─┴─┴─┼───────────────────────────────┤
│           Checksum             │       Urgent Pointer          │
└───────────────────────────────────────────────────────────────┘

关键字段：
- SYN: 同步序列号，用于建立连接
- ACK: 确认号有效
- FIN: 发送方完成数据发送
- RST: 重置连接
- SEQ: 序列号
```

---

## 二、三次握手（建立连接）

### 2.1 三次握手流程

```
三次握手流程：

    客户端                                    服务端
      │                                         │
      │  1. SYN=1, SEQ=x                       │
      │────────────────────────────────────────→│
      │                                         │
      │  2. SYN=1, ACK=1, SEQ=y, ACK=x+1      │
      │←────────────────────────────────────────│
      │                                         │
      │  3. ACK=1, SEQ=x+1, ACK=y+1           │
      │────────────────────────────────────────→│
      │                                         │
      │           连接建立完成                   │
```

### 2.2 每一步的作用

| 步骤 | 方向 | 标志位 | 序列号 | 作用 |
|------|------|--------|--------|------|
| **第一次** | 客户端→服务端 | SYN=1 | SEQ=x | 客户端请求建立连接 |
| **第二次** | 服务端→客户端 | SYN=1, ACK=1 | SEQ=y, ACK=x+1 | 服务端同意连接，确认收到 |
| **第三次** | 客户端→服务端 | ACK=1 | SEQ=x+1, ACK=y+1 | 客户端确认收到 |

### 2.3 为什么需要三次握手？

```
两次握手的问题：

场景：历史连接请求

客户端发送SYN（网络延迟）
    │
    │  （客户端超时，重新发送SYN）
    │
    ▼
服务端收到第一个SYN
    │
    │  发送SYN+ACK
    │
    ▼
客户端收到SYN+ACK
    │
    │  连接建立（但客户端已放弃这个连接）
    │
    ▼
第一个SYN到达服务端
    │
    │  服务端发送SYN+ACK
    │
    ▼
客户端收到SYN+ACK
    │
    │  如果是两次握手，客户端会认为连接建立
    │  但实际上客户端已经放弃了这个连接
    │
    ▼
三次握手解决：客户端发送ACK确认
    │
    │  如果是历史连接，客户端不会发送ACK
    │  服务端收到RST或超时后关闭连接
```

**三次握手的核心作用**：
1. **防止历史连接**：确认连接是当前有效的
2. **同步序列号**：双方交换初始序列号
3. **防止资源浪费**：避免服务端为无效连接分配资源

---

## 三、四次挥手（断开连接）

### 3.1 四次挥手流程

```
四次挥手流程：

    客户端                                    服务端
      │                                         │
      │  1. FIN=1, SEQ=u                        │
      │────────────────────────────────────────→│
      │                                         │
      │  2. ACK=1, SEQ=v, ACK=u+1              │
      │←────────────────────────────────────────│
      │                                         │
      │  （服务端可能还有数据要发送）              │
      │                                         │
      │  3. FIN=1, ACK=1, SEQ=w, ACK=u+1      │
      │←────────────────────────────────────────│
      │                                         │
      │  4. ACK=1, SEQ=u+1, ACK=w+1           │
      │────────────────────────────────────────→│
      │                                         │
      │           连接断开完成                   │
```

### 3.2 每一步的作用

| 步骤 | 方向 | 标志位 | 作用 |
|------|------|--------|------|
| **第一次** | 客户端→服务端 | FIN=1 | 客户端请求断开连接 |
| **第二次** | 服务端→客户端 | ACK=1 | 服务端确认收到，半关闭状态 |
| **第三次** | 服务端→客户端 | FIN=1, ACK=1 | 服务端也请求断开连接 |
| **第四次** | 客户端→服务端 | ACK=1 | 客户端确认收到，完全关闭 |

### 3.3 为什么需要四次挥手？

```
TCP全双工通信：

客户端 ←────────────────────→ 服务端
      │                     │
      │  数据可以双向传输     │
      │                     │
      ▼                     ▼

关闭连接时：
1. 客户端发送FIN → 服务端确认（第二次挥手）
2. 服务端可能还有数据要发送
3. 服务端发送FIN → 客户端确认（第四次挥手）

所以需要四次挥手，因为TCP是全双工的
```

### 3.4 TIME_WAIT状态

```
TIME_WAIT状态：

四次挥手完成后，主动关闭方进入TIME_WAIT状态
等待时间：2MSL（Maximum Segment Lifetime）

┌─────────────────────────────────────────────────────────┐
│  TIME_WAIT作用：                                         │
│  1. 确保最后一个ACK能到达服务端                          │
│  2. 确保旧连接的数据包在网络中消失                        │
└─────────────────────────────────────────────────────────┘

问题：大量TIME_WAIT会占用端口资源

解决方案：
1. 设置 net.ipv4.tcp_tw_reuse = 1（允许复用TIME_WAIT）
2. 设置 net.ipv4.tcp_tw_recycle = 1（快速回收）
3. 使用连接池复用连接
```

---

## 四、TCP状态转换图

```
TCP状态转换：

客户端状态转换：
CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED

服务端状态转换：
CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
```

---

## 五、Java中的TCP实现

### 5.1 Socket编程示例

```java
// TCP客户端
public class TcpClient {
    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("localhost", 8080);
        
        OutputStream out = socket.getOutputStream();
        out.write("Hello TCP".getBytes());
        out.flush();
        
        InputStream in = socket.getInputStream();
        byte[] buffer = new byte[1024];
        int len = in.read(buffer);
        System.out.println("收到响应: " + new String(buffer, 0, len));
        
        socket.close();
    }
}

// TCP服务端
public class TcpServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("TCP服务端启动");
        
        Socket socket = serverSocket.accept();
        
        InputStream in = socket.getInputStream();
        byte[] buffer = new byte[1024];
        int len = in.read(buffer);
        System.out.println("收到数据: " + new String(buffer, 0, len));
        
        OutputStream out = socket.getOutputStream();
        out.write("Echo: ".getBytes());
        out.write(buffer, 0, len);
        out.flush();
        
        socket.close();
        serverSocket.close();
    }
}
```

---

## 六、LeetCode题目解析

### 6.1 [239. 滑动窗口最大值（Hard）](https://leetcode.cn/problems/sliding-window-maximum/)

**题目描述**：
给你一个整数数组 `nums`，有一个大小为 `k` 的滑动窗口，从数组的最左侧移到最右侧。你只可以看到在滑动窗口内的 `k` 个数字。窗口每次向右移动一位。

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if (nums == null || nums.length == 0) return new int[0];
        
        int n = nums.length;
        int[] result = new int[n - k + 1];
        Deque<Integer> deque = new ArrayDeque<>(); // 存储索引
        
        for (int i = 0; i < n; i++) {
            // 移除超出窗口的元素
            while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
                deque.pollFirst();
            }
            
            // 移除比当前元素小的元素
            while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
                deque.pollLast();
            }
            
            deque.offerLast(i);
            
            // 窗口形成后记录最大值
            if (i >= k - 1) {
                result[i - k + 1] = nums[deque.peekFirst()];
            }
        }
        
        return result;
    }
}
```

---

## 七、今日学习要点总结

### 核心要点归纳

1. **三次握手**：SYN → SYN+ACK → ACK，防止历史连接，同步序列号
2. **四次挥手**：FIN → ACK → FIN → ACK，全双工通信需要四次
3. **TIME_WAIT**：等待2MSL，确保最后一个ACK到达
4. **SYN洪水攻击**：攻击者发送大量SYN但不完成第三次握手

### 验收清单

- [ ] 能画出三次握手和四次挥手的完整流程图
- [ ] 能解释为什么两次握手不够
- [ ] 能说明TIME_WAIT的作用和危害
- [ ] 能解释大量TIME_WAIT的解决方案

---

## 八、练习建议

1. **Wireshark抓包**：使用Wireshark观察TCP三次握手和四次挥手
2. **Socket编程**：实现简单的TCP客户端/服务端
3. **状态观察**：使用netstat命令观察TCP状态

---

## 九、扩展阅读

- 《TCP/IP详解》第1卷：协议
- 《图解TCP/IP》
- RFC 793: Transmission Control Protocol
