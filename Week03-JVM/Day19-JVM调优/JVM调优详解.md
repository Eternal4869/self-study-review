# Day19 详解 — JVM调优

## 一、常用JVM参数

### 1.1 堆内存参数

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `-Xms` | 初始堆大小 | `-Xms512m` |
| `-Xmx` | 最大堆大小 | `-Xmx1024m` |
| `-Xmn` | 新生代大小 | `-Xmn256m` |
| `-XX:NewRatio` | 老年代:新生代比例 | `-XX:NewRatio=2` |
| `-XX:SurvivorRatio` | Eden:S0:S1比例 | `-XX:SurvivorRatio=8` |
| `-XX:MaxTenuringThreshold` | 晋升老年代的年龄阈值 | `-XX:MaxTenuringThreshold=15` |

### 1.2 方法区参数

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `-XX:MetaspaceSize` | 元空间初始大小 | `-XX:MetaspaceSize=128m` |
| `-XX:MaxMetaspaceSize` | 元空间最大大小 | `-XX:MaxMetaspaceSize=256m` |

### 1.3 栈参数

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `-Xss` | 线程栈大小 | `-Xss256k` |

### 1.4 GC参数（2024+更新）

| 参数 | 说明 | 示例值 | 备注 |
|------|------|--------|------|
| `-XX:+UseG1GC` | 使用G1收集器 | - | JDK9+默认 |
| `-XX:MaxGCPauseMillis` | GC最大停顿时间 | `-XX:MaxGCPauseMillis=200` | - |
| `-XX:+UseZGC` | 使用ZGC收集器 | - | JDK21+推荐 |
| `-XX:+ZGenerational` | ZGC分代模式 | - | JDK23+默认 |
| `-XX:+UseShenandoahGC` | 使用Shenandoah收集器 | - | JDK12+ |
| `-XX:+UseConcMarkSweepGC` | 使用CMS收集器 | - | **JDK14已废弃** |
| `-XX:+PrintGCDetails` | 打印GC详细信息 | - | JDK8 |
| `-Xlog:gc*` | GC日志（JDK9+） | - | 推荐 |

### 1.5 OOM相关参数

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `-XX:+HeapDumpOnOutOfMemoryError` | OOM时导出堆转储 | - |
| `-XX:HeapDumpPath` | 堆转储文件路径 | `-XX:HeapDumpPath=/tmp/heap.hprof` |

### 1.6 参数配置示例

```bash
# 通用场景 - G1收集器（推荐）
java -Xms4g -Xmx4g \
     -Xmn2g \
     -XX:MetaspaceSize=256m \
     -XX:MaxMetaspaceSize=512m \
     -Xss512k \
     -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=200 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/app/heapdump.hprof \
     -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags \
     -jar app.jar

# 大堆 + 低延迟场景 - ZGC（JDK21+推荐）
java -Xms16g -Xmx32g \
     -XX:+UseZGC \
     -XX:+ZGenerational \
     -XX:MaxGCPauseMillis=5 \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=/var/log/app/heapdump.hprof \
     -Xlog:gc*:file=/var/log/app/gc.log:time,uptime,level,tags \
     -jar app.jar
```

---

## 二、OOM类型及排查思路

### 2.1 常见OOM类型

| OOM类型 | 原因 | 排查方法 |
|--------|------|---------|
| **Java heap space** | 堆内存不足 | 分析堆转储（heap dump） |
| **Metaspace** | 元空间不足 | 检查动态生成类、增加元空间 |
| **StackOverflowError** | 栈溢出 | 检查递归深度 |
| **Unable to create new native thread** | 无法创建新线程 | 检查线程数限制 |
| **GC overhead limit exceeded** | GC开销超过限制 | 分析GC日志 |

### 2.2 OOM排查流程

```
OOM排查流程：

┌─────────────────────────────────────────────────────────┐
│  1. 获取异常信息                                         │
│     - OOM类型                                           │
│     - 异常堆栈                                          │
│     - 触发时间                                          │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  2. 查看GC日志                                          │
│     - GC频率和耗时                                       │
│     - 堆内存使用情况                                     │
│     - 是否有Full GC                                     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  3. 分析堆转储（Heap Dump）                              │
│     - 使用MAT/VisualVM分析                              │
│     - 找出占用内存最大的对象                             │
│     - 分析对象引用链                                    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  4. 定位代码问题                                        │
│     - 内存泄漏：对象被意外持有引用                       │
│     - 内存溢出：对象确实需要这么多内存                   │
│     - 修复并测试                                        │
└─────────────────────────────────────────────────────────┘
```

### 2.3 Java heap space 排查

```java
// 内存泄漏示例
public class MemoryLeakDemo {
    private static final Map<String, Object> cache = new HashMap<>();
    
    public void addToCache(String key, Object value) {
        cache.put(key, value);  // 不断添加，从不删除
    }
}

// 内存溢出示例
public class OOMDemo {
    public static void main(String[] args) {
        // 创建一个超大对象
        byte[] array = new byte[Integer.MAX_VALUE]; // 约2GB
    }
}
```

**排查步骤**：
1. 检查是否有大对象分配
2. 检查是否有内存泄漏（集合只添加不删除）
3. 使用 `jmap -dump:format=b,file=heap.hprof <pid>` 导出堆转储
4. 使用MAT分析堆转储

### 2.4 Metaspace 排查

**常见原因**：
- 动态生成大量类（CGLIB、反射）
- 加载大量第三方jar包
- JSP编译产生大量类

**解决方案**：
```bash
# 增加元空间大小
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=512m

# 监控元空间使用情况
-XX:+PrintMetaspaceStats
```

### 2.5 StackOverflowError 排查

```java
// 递归过深导致栈溢出
public class StackOverflowDemo {
    public static void main(String[] args) {
        method();
    }
    
    public static void method() {
        method();  // 无限递归
    }
}
```

**解决方案**：
1. 检查递归终止条件
2. 增加栈大小 `-Xss2m`
3. 使用迭代替代递归

---

## 三、JVM监控工具

### 3.1 jps - 查看Java进程

```bash
# 查看所有Java进程
jps -l

# 输出示例：
# 12345 com.example.Application
# 12346 sun.tools.jps.Jps

# 查看进程参数
jps -v <pid>

# 查看主类名和进程ID
jps -m
```

### 3.2 jstat - 监控JVM状态

```bash
# 查看GC统计信息（每1000ms输出一次，共输出10次）
jstat -gcutil <pid> 1000 10

# 输出示例：
#   S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
#   0.00  99.34  45.23  21.45  95.12  93.21   125    1.234     3    0.456   1.690

# 各列含义：
# S0/S1: Survivor区使用率
# E: Eden区使用率
# O: 老年代使用率
# M: 元空间使用率
# CCS: 压缩类空间使用率
# YGC/YGCT: 新生代GC次数/耗时
# FGC/FGCT: Full GC次数/耗时
# GCT: GC总耗时

# 查看GC详情
jstat -gc <pid> 1000 10

# 查看类加载统计
jstat -class <pid>
```

### 3.3 jstack - 分析线程

```bash
# 导出线程堆栈
jstack <pid> > thread_dump.txt

# 查看死锁
jstack -l <pid>

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

```java
// 死锁示例
public class DeadlockDemo {
    private static final Object lock1 = new Object();
    private static final Object lock2 = new Object();
    
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (lock1) {
                System.out.println("Thread-1 holding lock1");
                try { Thread.sleep(100); } catch (Exception e) {}
                synchronized (lock2) {
                    System.out.println("Thread-1 holding lock2");
                }
            }
        });
        
        Thread t2 = new Thread(() -> {
            synchronized (lock2) {
                System.out.println("Thread-2 holding lock2");
                try { Thread.sleep(100); } catch (Exception e) {}
                synchronized (lock1) {
                    System.out.println("Thread-2 holding lock1");
                }
            }
        });
        
        t1.start();
        t2.start();
    }
}
```

### 3.4 jmap - 生成堆转储

```bash
# 导出堆转储文件
jmap -dump:format=b,file=heap.hprof <pid>

# 查看堆内存使用情况
jmap -heap <pid>

# 输出示例：
# Heap Configuration:
#    MinHeapFreeRatio         = 40
#    MaxHeapFreeRatio         = 70
#    MaxHeapSize              = 4294967296 (4096.0MB)
#    NewSize                  = 2147483648 (2048.0MB)
#    OldSize                  = 2147483648 (2048.0MB)

# 查看对象统计
jmap -histo <pid> | head -20

# 输出示例：
#  num     #instances         #bytes  class name
#    1:        1234567      123456789  [B
#    2:         567890       56789012  java.util.HashMap$Node
#    3:         123456       12345678  java.lang.String
```

### 3.5 jinfo - 查看JVM参数

```bash
# 查看JVM参数
jinfo -flags <pid>

# 查看特定参数
jinfo -flag MaxHeapSize <pid>
jinfo -flag UseG1GC <pid>
```

---

## 四、Arthas使用

### 4.1 Arthas概述

**Arthas** 是阿里巴巴开源的Java诊断工具，支持在线诊断问题。

### 4.2 安装与启动

```bash
# 下载Arthas
curl -O https://arthas.aliyun.com/arthas-boot.jar

# 启动Arthas
java -jar arthas-boot.jar

# 选择要诊断的Java进程
# 输入进程序号，回车进入Arthas控制台
```

### 4.3 常用命令

```bash
# dashboard - 实时监控面板
dashboard

# 输出示例：
# ID    NAME                         GROUP          PRIORITY  STATE
# 1     main                         main           5         WAITING
# 22    Timer-0                      main           5         TIMED_WAITING
#
# Memory:
#                  used     total    max      usage
# heap             256M     512M     1024M    25.00%
# par_space        64M      128M     256M     25.00%
# tenured          128M     256M     768M     16.67%
# metaspace        32M      64M      256M     12.50%

# thread - 查看线程信息
thread

# 查看指定线程堆栈
thread <id>

# 查看死锁
thread -b

# heapdump - 导出堆转储
heapdump /tmp/heap.hprof

# watch - 监视方法执行
watch com.example.Service method '{params, returnObj}' -x 2

# trace - 跟踪方法调用链路
trace com.example.Service method

# jad - 反编译类
jad com.example.Service

# sc - 搜索类
sc com.example.*

# sm - 搜索方法
sm com.example.Service *
```

### 4.4 Arthas实战示例

```bash
# 监控方法入参和返回值
watch com.example.UserService getUserById '{params, returnObj}' -x 3

# 输出示例：
# 方法执行数据:
#  {params=[1001], returnObj=User{id=1001, name='张三'}}

# 跟踪方法调用耗时
trace com.example.UserService getUserById '#cost > 100'

# 输出示例：
#  com.example.UserService#getUserById
#  +---[0.5ms] com.example.UserMapper#selectById
#  +---[2.3ms] com.example.RedisService#get
#  +---[0.2ms] return: User{id=1001, name='张三'}
```

---

## 五、MAT分析堆转储

### 5.1 MAT概述

**MAT（Memory Analyzer Tool）** 是一款强大的堆转储分析工具，可以帮助快速定位内存问题。

### 5.2 使用步骤

```
MAT分析流程：

┌─────────────────────────────────────────────────────────┐
│  1. 获取堆转储文件                                       │
│     jmap -dump:format=b,file=heap.hprof <pid>          │
│     或配置 -XX:+HeapDumpOnOutOfMemoryError              │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  2. 打开MAT                                             │
│     File → Open Heap Dump → 选择heap.hprof             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  3. 查看Leak Suspects Report                           │
│     自动分析可能的内存泄漏点                             │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  4. 分析Dominator Tree                                  │
│     查看占用内存最大的对象                               │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│  5. 查看Histogram                                       │
│     查看各类对象的数量和内存占用                         │
└─────────────────────────────────────────────────────────┘
```

### 5.3 MAT关键功能

| 功能 | 说明 | 用途 |
|------|------|------|
| **Leak Suspects** | 自动分析内存泄漏 | 快速定位问题 |
| **Dominator Tree** | 支配树，显示对象支配关系 | 找出占用内存最大的对象 |
| **Histogram** | 类的直方图，显示对象数量和大小 | 分析对象分布 |
| **OQL** | 对象查询语言，类似SQL | 精确查询对象 |
| **Path to GC Roots** | 查看到GC Roots的路径 | 分析对象为什么无法被回收 |

### 5.4 常见内存问题分析

```java
// 问题1：大数组
// 在Histogram中查找 [B (byte数组)
// 查看哪些对象持有这些大数组

// 问题2：内存泄漏
// 查看Leak Suspects
// 分析Dominator Tree中的大对象

// 问题3：类加载过多
// 在Histogram中按类名排序
// 查看是否有大量重复的类（如动态代理类）
```

---

## 六、VisualVM使用

### 6.1 VisualVM概述

**VisualVM** 是JDK自带的可视化监控工具，可以监控JVM的内存、线程、GC等。

### 6.2 启动方式

```bash
# JDK8及以前
jvisualvm

# 或在JDK安装目录的bin目录下
$JAVA_HOME/bin/jvisualvm
```

### 6.3 主要功能

| 功能 | 说明 |
|------|------|
| **概览** | 查看JVM基本信息 |
| **监视** | 查看CPU、内存、GC、类加载情况 |
| **线程** | 查看线程状态，检测死锁 |
| **Sampler** | CPU和内存采样 |
| **插件** | 安装额外插件（如Visual GC） |

---

## 七、LeetCode题目解析

### 7.1 [322. 零钱兑换（Medium）](https://leetcode.cn/problems/coin-change/)

**题目描述**：
给你一个整数数组 `coins`，表示不同面额的硬币；以及一个整数 `amount`，表示总金额。计算并返回可以凑成总金额所需的最少的硬币个数。

**思路分析**：
使用动态规划，`dp[i]` 表示凑成金额 `i` 所需的最少硬币数。

```java
class Solution {
    public int coinChange(int[] coins, int amount) {
        // dp[i]表示凑成金额i所需的最少硬币数
        int[] dp = new int[amount + 1];
        
        // 初始化：除dp[0]=0外，其余设为amount+1（表示不可能）
        Arrays.fill(dp, amount + 1);
        dp[0] = 0;
        
        // 状态转移
        for (int i = 1; i <= amount; i++) {
            for (int coin : coins) {
                if (coin <= i) {
                    dp[i] = Math.min(dp[i], dp[i - coin] + 1);
                }
            }
        }
        
        return dp[amount] > amount ? -1 : dp[amount];
    }
}
```

**复杂度分析**：
- 时间复杂度：O(amount × coins.length)
- 空间复杂度：O(amount)

### 7.2 [300. 最长递增子序列（Medium）](https://leetcode.cn/problems/longest-increasing-subsequence/)

**题目描述**：
给你一个整数数组 `nums`，找到其中最长严格递增子序列的长度。

**思路分析**：
使用动态规划或贪心+二分查找。

```java
// 动态规划解法
class Solution {
    public int lengthOfLIS(int[] nums) {
        int n = nums.length;
        // dp[i]表示以nums[i]结尾的最长递增子序列长度
        int[] dp = new int[n];
        Arrays.fill(dp, 1); // 每个元素自身就是一个子序列
        
        int maxLen = 1;
        
        for (int i = 1; i < n; i++) {
            for (int j = 0; j < i; j++) {
                if (nums[j] < nums[i]) {
                    dp[i] = Math.max(dp[i], dp[j] + 1);
                }
            }
            maxLen = Math.max(maxLen, dp[i]);
        }
        
        return maxLen;
    }
}

// 贪心+二分解法
class Solution2 {
    public int lengthOfLIS(int[] nums) {
        // tails[i]表示长度为i+1的递增子序列的最小末尾元素
        int[] tails = new int[nums.length];
        int size = 0;
        
        for (int num : nums) {
            int left = 0, right = size;
            while (left < right) {
                int mid = left + (right - left) / 2;
                if (tails[mid] < num) {
                    left = mid + 1;
                } else {
                    right = mid;
                }
            }
            tails[left] = num;
            if (left == size) {
                size++;
            }
        }
        
        return size;
    }
}
```

**复杂度分析**：
- 动态规划：时间 O(n²)，空间 O(n)
- 贪心+二分：时间 O(n log n)，空间 O(n)

---

## 八、今日学习要点总结

### 核心要点归纳

1. **常用JVM参数**：-Xms/-Xmx（堆大小）、-Xmn（新生代）、-Xss（栈大小）
2. **OOM排查**：获取异常信息→分析GC日志→导出堆转储→使用MAT分析
3. **jstack**：分析线程问题，检测死锁
4. **jmap**：导出堆转储，查看内存使用
5. **Arthas**：在线诊断工具，watch/trace/dashboard等命令
6. **MAT**：分析堆转储，Leak Suspects、Dominator Tree、Histogram

### 验收清单

- [ ] 能说出常用的JVM调优参数及含义
- [ ] 能使用jstack排查死锁问题
- [ ] 能使用jmap导出堆转储并用MAT分析
- [ ] 能说明线上OOM问题的排查步骤
- [ ] 能使用Arthas的基本命令进行在线诊断

---

## 九、练习建议

1. **参数实践**：配置不同的JVM参数，观察程序运行情况
2. **OOM模拟**：分别模拟堆OOM、栈溢出、元空间OOM
3. **工具使用**：练习使用jps、jstat、jstack、jmap
4. **Arthas实战**：使用Arthas监控和诊断实际问题
5. **MAT分析**：使用MAT分析堆转储文件

---

## 十、扩展阅读

- 《深入理解Java虚拟机》第4章：虚拟机性能监控与故障处理工具
- Arthas官方文档：https://arthas.aliyun.com/
- MAT官方文档：https://eclipse.dev/mat/
