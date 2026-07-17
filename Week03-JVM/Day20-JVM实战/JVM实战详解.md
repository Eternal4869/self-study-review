# Day20 详解 — JVM实战

## 一、Arthas安装与使用

### 1.1 Arthas安装

```bash
# 方式1：直接下载
curl -O https://arthas.aliyun.com/arthas-boot.jar

# 方式2：使用一键安装脚本（推荐）
curl -L https://arthas.aliyun.com/install.sh | sh

# 方式3：Maven仓库下载
# https://mvnrepository.com/artifact/com.taobao.arthas/arthas-boot
```

### 1.2 启动Arthas

```bash
# 启动Arthas
java -jar arthas-boot.jar

# 输出示例：
# [INFO] arthas-boot version: 3.6.9
# [INFO] Found existing java process, please choose one and input PID.
# 1: 12345 com.example.Application
# 2: 12346 org.apache.catalina.startup.Bootstrap
# 
# 输入进程ID（如1），回车即可连接

# 连接成功后
# [arthas@12345]$
```

### 1.3 基础命令

```bash
# version - 查看Arthas版本
version

# help - 查看帮助
help

# cls - 清屏
cls

# history - 历史命令
history

# quit/exit - 退出Arthas（不影响Java进程）
quit

# stop - 关闭Arthas Server
stop
```

---

## 二、Arthas核心命令详解

### 2.1 dashboard - 实时面板

```bash
# 实时监控面板
dashboard

# 输出示例：
# ID    NAME                         GROUP          PRIORITY  STATE    %CPU    TIME    INTERRUPT DAEMON
# 1     main                         main           5         WAITING  0.0     0:0     false     false
# 22    Timer-0                      main           5         TIMED_WAITING 0.1  0:0     false     true
# 23    http-nio-8080-exec-1         main           5         WAITING  0.0     0:0     false     true
#
# Memory                    used     total    max      usage
# heap                      256M     512M     1024M    25.00%
# par_space (young)         64M      128M     256M     25.00%
# tenured (old)             128M     256M     768M     16.67%
# metaspace                 32M      64M      256M     12.50%
# compressed_class_space    8M       16M      128M     6.25%
# direct                    16M      16M      -        100.00%
#
# Runtime
# os.name:       Windows 10
# os.arch:       amd64
# java.version:  17.0.2
# java.home:     C:\Program Files\Java\jdk-17

# 退出dashboard
# 输入 q 或者 Ctrl+C
```

### 2.2 thread - 线程分析

```bash
# 查看所有线程
thread

# 查看指定线程堆栈
thread <id>

# 查看最忙的N个线程
thread -n 3

# 查看所有线程状态统计
thread --state WAITING

# 检测死锁
thread -b

# 输出示例：
# Found one Java-level deadlock:
# ==============================
# "Thread-1":
#   waiting to lock monitor 0x00007f8b5c003838 (object 0x00000007aab4b880, a java.lang.Object),
#   which is held by "Thread-0"
# "Thread-0":
#   waiting to lock monitor 0x00007f8b5c006188 (object 0x00000007aab4b890, a java.lang.Object),
#   which is held by "Thread-1"
```

### 2.3 watch - 方法监控

```bash
# 基本用法：监控方法入参和返回值
watch com.example.UserService getUserById '{params, returnObj}' -x 2

# 监控方法执行耗时
watch com.example.UserService getUserById '{params, returnObj, #cost}' -x 2

# 条件过滤：只监控返回值为null的情况
watch com.example.UserService getUserById '{params, returnObj}' 'returnObj == null' -x 2

# 异常时触发
watch com.example.UserService getUserById '{params, throwExp}' -e -x 2

# 监控方法执行前
watch com.example.UserService getUserById '{params}' -b -x 2

# 输出示例：
# Method execution data:
#  {params=[1001], returnObj=User{id=1001, name='张三', age=25}}
#  Method execution cost: 15.23 ms
```

### 2.4 trace - 调用链路跟踪

```bash
# 跟踪方法调用链路
trace com.example.UserService getUserById

# 输出示例：
# +---[0.523ms] com.example.UserMapper#selectById
# +---[5.123ms] com.example.RedisService#get
#   +---[4.890ms] redis.clients.jedis.Jedis#get
# +---[0.234ms] com.example.UserService#convertToDTO
# +---[0.123ms] return: UserDTO{id=1001, name='张三'}

# 跳过JDK内部方法
trace com.example.UserService getUserById '#cost > 100'

# 只显示耗时超过100ms的调用
```

### 2.5 stack - 调用栈查看

```bash
# 查看方法调用栈
stack com.example.UserService getUserById

# 输出示例：
#     @com.example.UserController#getUser
#     at com.example.UserService#getUserById
#     at com.example.Application#main
# 
#     Executes on:
#     12345@NioEndpoint$Acceptor@0.0.0.0:8080
```

### 2.6 jad - 反编译

```bash
# 反编译类
jad com.example.UserService

# 反编译指定方法
jad com.example.UserService getUserById

# 输出示例：
# @SpringService
# public class UserService {
#     @Resource
#     private UserMapper userMapper;
#     
#     @Resource
#     private RedisService redisService;
#     
#     public User getUserById(Long id) {
#         String cacheKey = "user:" + id;
#         User user = redisService.get(cacheKey);
#         if (user == null) {
#             user = userMapper.selectById(id);
#             if (user != null) {
#                 redisService.set(cacheKey, user, 3600);
#             }
#         }
#         return user;
#     }
# }
```

### 2.7 sc - 搜索类

```bash
# 搜索类
sc com.example.*

# 搜索匹配的类
sc *UserService*

# 查看类的详细信息
sc -d com.example.UserService
```

### 2.8 sm - 搜索方法

```bash
# 搜索类中的方法
sm com.example.UserService *

# 查看方法详情
sm -d com.example.UserService getUserById
```

### 2.9 heapdump - 堆转储

```bash
# 导出堆转储
heapdump /tmp/heap.hprof

# 只导出live对象
heapdump -l /tmp/heap.hprof

# 使用Arthas分析堆转储
heapdump /tmp/heap.hprof
# 会自动分析并给出嫌疑对象
```

---

## 三、GC日志分析

### 3.1 GC日志参数配置

```bash
# JDK8 GC日志参数
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCTimeStamps
-Xloggc:/path/to/gc.log
-XX:+PrintHeapAtGC
-XX:+PrintGCApplicationStoppedTime

# JDK9+ GC日志参数（统一日志框架）
-Xlog:gc*:file=/path/to/gc.log:time,uptime,level,tags
-Xlog:gc*:file=/path/to/gc.log:time,uptime,level,tags,heap
```

### 3.2 GC日志格式解读

```
# Minor GC日志示例
[2024-01-15T10:30:15.123+0800] [GC (Allocation Failure) [PSYoungGen: 262144K->43520K(307200K)] 262144K->43520K(1013760K), 0.0234567 secs] [Times: user=0.15 sys=0.02, real=0.02 secs]

# Full GC日志示例
[2024-01-15T10:35:20.456+0800] [Full GC (Metadata GC Threshold) [PSYoungGen: 0K->0K(307200K)] [ParOldGen: 102400K->102400K(348160K)] 102400K->102400K(655360K), [Metaspace: 65536K->65536K(1114112K)], 0.1234567 secs] [Times: user=0.85 sys=0.05, real=0.12 secs]

# 各字段含义：
# [GC / Full GC]: GC类型
# [PSYoungGen / ParOldGen]: 收集器类型
# 262144K->43520K(307200K): GC前->GC后(该区域总大小)
# 262144K->43520K(1013760K): 堆GC前->堆GC后(堆总大小)
# 0.0234567 secs: GC耗时
# [Times: user=0.15 sys=0.02, real=0.02 secs]: 用户时间、系统时间、实际时间
```

### 3.3 GC日志分析工具

| 工具 | 特点 | 网址 |
|------|------|------|
| **GCEasy** | 在线分析，可视化好 | https://gceasy.io |
| **GCViewer** | 开源桌面工具 | https://github.com/chewiebug/GCViewer |
| **HPjmeter** | HP出品，功能强大 | - |

### 3.4 GCEasy使用

```
GCEasy分析流程：

┌─────────────────────────────────────────────────────────┐
│  1. 上传GC日志文件                                       │
│     访问 https://gceasy.io                              │
│     选择GC日志文件上传                                   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  2. 查看分析报告                                         │
│     - GC统计信息                                        │
│     - 停顿时间分析                                      │
│     - 吞吐量分析                                        │
│     - 内存使用分析                                      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  3. 优化建议                                             │
│     - 推荐的JVM参数                                     │
│     - 问题诊断                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 四、JFR（Java Flight Recorder）

### 4.1 JFR概述

**JFR（Java Flight Recorder）** 是JDK自带的低开销性能分析工具，可以在生产环境中使用。

### 4.2 启动JFR

```bash
# 启动时开启JFR
java -XX:+FlightRecorder \
     -XX:StartFlightRecording=duration=60s,filename=recording.jfr \
     -jar app.jar

# 动态开启JFR（针对已运行的Java进程）
# 使用jcmd命令
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# 停止JFR
jcmd <pid> JFR.stop

# 查看JFR状态
jcmd <pid> JFR.status
```

### 4.3 JFR记录内容

| 事件类型 | 说明 |
|---------|------|
| **CPU Profiling** | CPU采样 |
| **Heap Allocation** | 堆内存分配 |
| **GC** | GC事件 |
| **Threads** | 线程事件 |
| **I/O** | I/O操作 |
| **Method Profiling** | 方法采样 |

### 4.4 分析JFR记录

```bash
# 使用JDK Mission Control（JMC）分析
# 启动JMC
jmc

# 打开.jfr文件进行分析
```

---

## 五、Prometheus + Grafana JVM监控

### 5.1 架构设计

```
JVM监控架构：

┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   JVM应用   │ ───→ │ Micrometer  │ ───→ │ Prometheus  │
│             │      │   Exporter  │      │             │
└─────────────┘      └─────────────┘      └─────────────┘
                                                │
                                                ▼
                                         ┌─────────────┐
                                         │   Grafana   │
                                         │   可视化    │
                                         └─────────────┘
```

### 5.2 添加Micrometer依赖

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### 5.3 配置application.yml

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: my-app
```

### 5.4 Prometheus配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']
        labels:
          application: 'my-app'
```

### 5.5 Grafana仪表板

```
JVM监控仪表板常用指标：

CPU指标：
- jvm.cpu.usage: CPU使用率
- system.cpu.usage: 系统CPU使用率

内存指标：
- jvm.memory.used: 已用内存
- jvm.memory.committed: 已分配内存
- jvm.memory.max: 最大内存

GC指标：
- jvm.gc.pause: GC停顿时间
- jvm.gc.concurrent.phase.time: 并发GC阶段时间
- jvm.gc.memory.promoted: 晋升到老年代的内存
- jvm.gc.memory.allocated: 新生代分配的内存

线程指标：
- jvm.threads.live: 活跃线程数
- jvm.threads.daemon: 守护线程数

Tomcat指标：
- tomcat.sessions.active: 活跃会话数
- tomcat.sessions.created: 创建的会话数
```

### 5.6 常用Grafana仪表板ID

| 仪表板ID | 名称 | 说明 |
|---------|------|------|
| 4701 | JVM (Micrometer) | Spring Boot JVM监控 |
| 8919 | JVM Monitoring | JVM综合监控 |
| 12856 | JVM Metrics | JVM指标监控 |

---

## 六、GC日志分析实战

### 6.1 常见GC问题诊断

#### 问题1：频繁Minor GC

```
GC日志特征：
[GC (Allocation Failure) [PSYoungGen: 262144K->43520K(307200K)] ...
[GC (Allocation Failure) [PSYoungGen: 262144K->43520K(307200K)] ...
[GC (Allocation Failure) [PSYoungGen: 262144K->43520K(307200K)] ...

原因：
- 新生代空间不足
- 对象创建速度过快

解决方案：
- 增大新生代大小：-Xmn512m
- 优化代码，减少对象创建
```

#### 问题2：Full GC频繁

```
GC日志特征：
[Full GC (Metadata GC Threshold) [PSYoungGen: 0K->0K(307200K)] [ParOldGen: 102400K->102400K(348160K)] ...

原因：
- 老年代空间不足
- 元空间不足
- 内存泄漏

解决方案：
- 增大堆内存：-Xmx2g
- 增大元空间：-XX:MetaspaceSize=256m
- 分析堆转储，查找内存泄漏
```

#### 问题3：GC停顿时间长

```
GC日志特征：
[GC (Allocation Failure) ... 0.5234567 secs]
[Full GC ... 2.1234567 secs]

原因：
- 堆内存过大
- 存在大对象
- 收集器选择不当

解决方案：
- 切换到G1收集器：-XX:+UseG1GC
- 设置目标停顿时间：-XX:MaxGCPauseMillis=200
- 优化大对象分配
```

### 6.2 GC优化建议（2024+更新）

| 场景 | 建议参数 |
|------|---------|
| **通用场景** | -XX:+UseG1GC -XX:MaxGCPauseMillis=200 |
| **低延迟应用** | -XX:+UseZGC（JDK21+推荐） |
| **超大堆（>8G）+ 低延迟** | -XX:+UseZGC -XX:MaxGCPauseMillis=5 |
| **高吞吐量应用** | -XX:+UseParallelGC -XX:ParallelGCThreads=8 |
| **大堆应用（>4G）** | -XX:+UseG1GC -XX:G1HeapRegionSize=16m |
| **小堆应用（<1G）** | -XX:+UseSerialGC |

---

## 七、LeetCode题目解析

### 7.1 [146. LRU缓存（Medium）](https://leetcode.cn/problems/lru-cache/)

**题目描述**：
设计并实现一个满足 LRU (最近最少使用) 缓存 约束的数据结构。实现 `LRUCache` 类：
- `LRUCache(int capacity)` 以正整数作为容量 `capacity` 初始化 LRU 缓存
- `int get(int key)` 如果关键字 `key` 存在于缓存中，则返回关键字的值，否则返回 -1
- `void put(int key, int value)` 如果关键字 `key` 已经存在，则变更其数据值 `value`；如果不存在，则向缓存中插入该组 `key-value`。如果插入操作导致关键字数量超过 `capacity` ，则应该 逐出 最久未使用的关键字

**思路分析**：
使用HashMap + 双向链表实现。HashMap提供O(1)的查找，双向链表维护访问顺序。

```java
class LRUCache {
    private int capacity;
    private Map<Integer, Node> map;
    private Node head;  // 虚拟头节点
    private Node tail;  // 虚拟尾节点
    
    class Node {
        int key;
        int value;
        Node prev;
        Node next;
        
        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node(0, 0);
        this.tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }
    
    public int get(int key) {
        if (!map.containsKey(key)) {
            return -1;
        }
        
        Node node = map.get(key);
        // 移动到头部（最近使用）
        removeNode(node);
        addToHead(node);
        
        return node.value;
    }
    
    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.value = value;
            removeNode(node);
            addToHead(node);
        } else {
            if (map.size() >= capacity) {
                // 删除尾部节点（最久未使用）
                Node tailNode = tail.prev;
                removeNode(tailNode);
                map.remove(tailNode.key);
            }
            
            Node newNode = new Node(key, value);
            map.put(key, newNode);
            addToHead(newNode);
        }
    }
    
    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    private void addToHead(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}

// 使用示例
LRUCache cache = new LRUCache(2);
cache.put(1, 1);
cache.put(2, 2);
cache.get(1);      // 返回 1
cache.put(3, 3);   // 移除 key=2
cache.get(2);      // 返回 -1
```

**复杂度分析**：
- 时间复杂度：get O(1)，put O(1)
- 空间复杂度：O(capacity)

### 7.2 [23. 合并K个升序链表（Hard）](https://leetcode.cn/problems/merge-k-sorted-lists/)

**题目描述**：
给你一个链表数组，每个链表都已经按升序排列。请你将所有链表合并到一个升序链表中，返回合并后的链表。

**思路分析**：
使用优先队列（最小堆）合并K个链表。

```java
class Solution {
    public ListNode mergeKLists(ListNode[] lists) {
        if (lists == null || lists.length == 0) {
            return null;
        }
        
        // 最小堆，按节点值排序
        PriorityQueue<ListNode> queue = new PriorityQueue<>((a, b) -> a.val - b.val);
        
        // 将每个链表的头节点加入堆
        for (ListNode head : lists) {
            if (head != null) {
                queue.offer(head);
            }
        }
        
        ListNode dummy = new ListNode(0);
        ListNode curr = dummy;
        
        while (!queue.isEmpty()) {
            ListNode node = queue.poll();
            curr.next = node;
            curr = curr.next;
            
            if (node.next != null) {
                queue.offer(node.next);
            }
        }
        
        return dummy.next;
    }
}
```

**复杂度分析**：
- 时间复杂度：O(N log k)，其中N是所有节点总数，k是链表数量
- 空间复杂度：O(k)，堆的大小

---

## 八、今日学习要点总结

### 核心要点归纳

1. **Arthas**：在线诊断工具，watch/trace/dashboard/jad等命令
2. **GC日志**：使用 `-Xlog:gc*` 配置，使用GCEasy/GCViewer分析
3. **JFR**：Java Flight Recorder，低开销的性能分析工具
4. **JVM监控**：Prometheus + Grafana搭建监控面板
5. **GC问题诊断**：频繁GC、Full GC、停顿时间长等问题的排查

### 验收清单

- [ ] 能使用Arthas的watch/trace命令在线诊断方法
- [ ] 能配置并分析GC日志
- [ ] 能搭建简单的JVM监控面板
- [ ] 能根据GC日志判断GC是否正常

---

## 九、练习建议

1. **Arthas实战**：使用Arthas监控实际应用的方法执行
2. **GC日志分析**：使用GCEasy分析GC日志
3. **监控搭建**：搭建Prometheus + Grafana JVM监控
4. **JFR使用**：使用JFR进行性能分析

---

## 十、扩展阅读

- Arthas官方文档：https://arthas.aliyun.com/
- Micrometer官方文档：https://micrometer.io/
- Grafana官方文档：https://grafana.com/docs/
- GC日志分析工具：https://gceasy.io
