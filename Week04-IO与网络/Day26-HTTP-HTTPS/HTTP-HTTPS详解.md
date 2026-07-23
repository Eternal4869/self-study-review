# Day26 详解 — HTTP/HTTPS

## 〇、HTTP演进与最新特性（面试必知）

### 0.1 HTTP版本演进

```
HTTP版本演进：

HTTP/1.0 (1996)  →  HTTP/1.1 (1997)  →  HTTP/2 (2015)  →  HTTP/3 (2022)
      │                  │                  │                  │
      ▼                  ▼                  ▼                  ▼
   短连接            持久连接            二进制协议          QUIC协议
   无Host           管线化              多路复用            UDP传输
   无缓存           分块传输            头部压缩            0-RTT
```

### 0.2 HTTP/3与QUIC（最新）

**HTTP/3** 是最新的HTTP协议，基于QUIC（Quick UDP Internet Connections）。

```
HTTP/3 vs HTTP/2：

HTTP/2（基于TCP）：
┌─────────────────────────────────────────────────────────┐
│  HTTP/2                                                  │
├─────────────────────────────────────────────────────────┤
│  TCP连接                                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 多路复用（一个连接多个流）                        │   │
│  │ 问题：TCP队头阻塞                                 │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘

HTTP/3（基于QUIC/UDP）：
┌─────────────────────────────────────────────────────────┐
│  HTTP/3                                                  │
├─────────────────────────────────────────────────────────┤
│  QUIC连接（基于UDP）                                     │
│  ┌─────────────────────────────────────────────────┐   │
│  │ 多路复用（无队头阻塞）                            │   │
│  │ 0-RTT连接建立                                    │   │
│  │ 内置TLS 1.3                                     │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 一、HTTP/1.1 vs HTTP/2

### 1.1 主要区别

| 特性 | HTTP/1.1 | HTTP/2 |
|------|---------|--------|
| **协议格式** | 文本协议 | 二进制协议 |
| **多路复用** | 不支持 | 支持（一个连接多个流） |
| **头部压缩** | 无 | HPACK算法压缩 |
| **服务器推送** | 不支持 | 支持 |
| **队头阻塞** | 存在 | TCP层仍存在 |

### 1.2 HTTP/2多路复用

```
HTTP/1.1 vs HTTP/2多路复用：

HTTP/1.1（队头阻塞）：
请求1 ─────────────────────→ 响应1
                               请求2 ─────────────────────→ 响应2
                                                       请求3 ──→ 响应3
串行执行，一个请求阻塞后续请求

HTTP/2（多路复用）：
请求1 ──┐
请求2 ──┼──→ 同时传输 ──→ 响应1
请求3 ──┘                响应2
                         响应3
并行执行，互不阻塞
```

---

## 二、HTTPS工作原理

### 2.1 HTTPS加密流程

```
HTTPS加密流程：

客户端                                        服务端
  │                                             │
  │  1. ClientHello                              │
  │  - 支持的TLS版本                             │
  │  - 支持的加密套件                            │
  │  - 随机数1                                   │
  │─────────────────────────────────────────────→│
  │                                             │
  │  2. ServerHello                              │
  │  - 选择的TLS版本                             │
  │  - 选择的加密套件                            │
  │  - 随机数2                                   │
  │  - 服务器证书                                │
  │←─────────────────────────────────────────────│
  │                                             │
  │  3. 验证证书                                 │
  │  - 验证证书有效性                            │
  │  - 生成预主密钥                              │
  │  - 用服务器公钥加密预主密钥                   │
  │─────────────────────────────────────────────→│
  │                                             │
  │  4. 生成会话密钥                             │
  │  - 用随机数1+2+预主密钥生成                  │
  │  - 开始加密通信                              │
  │←─────────────────────────────────────────────│
```

### 2.2 TLS 1.2 vs TLS 1.3

| 特性 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| **握手RTT** | 2-RTT | 1-RTT（支持0-RTT） |
| **加密套件** | 多种 | 只保留5种安全套件 |
| **密钥交换** | RSA、DHE | 只支持ECDHE |
| **前向保密** | 可选 | 强制 |
| **安全性** | 较高 | 更高 |

---

## 三、HTTP状态码

### 3.1 状态码分类

| 分类 | 说明 | 常见状态码 |
|------|------|-----------|
| **1xx** | 信息 | 100 Continue |
| **2xx** | 成功 | 200 OK, 201 Created, 204 No Content |
| **3xx** | 重定向 | 301 Moved Permanently, 302 Found, 304 Not Modified |
| **4xx** | 客户端错误 | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| **5xx** | 服务端错误 | 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable |

---

## 四、Java中的HTTP客户端

### 4.1 HttpURLConnection

```java
// 传统方式（JDK8及以前）
public class HttpDemo {
    public static void main(String[] args) throws IOException {
        URL url = new URL("https://api.example.com/data");
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        
        conn.setRequestMethod("GET");
        conn.setConnectTimeout(5000);
        conn.setReadTimeout(5000);
        
        int code = conn.getResponseCode();
        if (code == 200) {
            InputStream in = conn.getInputStream();
            String response = new String(in.readAllBytes(), StandardCharsets.UTF_8);
            System.out.println("响应: " + response);
        }
        
        conn.disconnect();
    }
}
```

### 4.2 HttpClient（JDK11+推荐）

```java
// JDK11+ HttpClient
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

## 五、LeetCode题目解析

### 5.1 [148. 排序链表（Medium）](https://leetcode.cn/problems/sort-list/)

**题目描述**：
给你链表的头结点 `head`，请将其按升序排列并返回排序后的链表。

```java
class Solution {
    public ListNode sortList(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        
        // 找中点
        ListNode slow = head, fast = head.next;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
        }
        
        ListNode mid = slow.next;
        slow.next = null;
        
        // 递归排序
        ListNode left = sortList(head);
        ListNode right = sortList(mid);
        
        // 合并
        return merge(left, right);
    }
    
    private ListNode merge(ListNode l1, ListNode l2) {
        ListNode dummy = new ListNode(0);
        ListNode curr = dummy;
        
        while (l1 != null && l2 != null) {
            if (l1.val <= l2.val) {
                curr.next = l1;
                l1 = l1.next;
            } else {
                curr.next = l2;
                l2 = l2.next;
            }
            curr = curr.next;
        }
        
        curr.next = (l1 != null) ? l1 : l2;
        return dummy.next;
    }
}
```

---

## 六、今日学习要点总结

### 核心要点归纳

1. **HTTP/1.1**：持久连接、管线化、Host头
2. **HTTP/2**：二进制协议、多路复用、头部压缩、服务器推送
3. **HTTP/3**：基于QUIC/UDP，无队头阻塞，0-RTT
4. **HTTPS**：对称加密+非对称加密+CA证书
5. **TLS 1.3**：1-RTT握手，强制前向保密

### 验收清单

- [ ] 能说出HTTP/1.1和HTTP/2的主要区别
- [ ] 能解释HTTPS的加密流程
- [ ] 能说明TLS 1.2和TLS 1.3的区别
- [ ] 能解释HTTP/3为什么基于QUIC

---

## 七、练习建议

1. **Wireshark抓包**：观察HTTP/1.1和HTTP/2的差异
2. **HTTPS配置**：配置本地HTTPS服务
3. **HTTP/2实践**：使用HTTP/2客户端进行测试

---

## 八、扩展阅读

- 《HTTP权威指南》
- RFC 7540: HTTP/2
- RFC 9114: HTTP/3
- MDN Web Docs: HTTP
