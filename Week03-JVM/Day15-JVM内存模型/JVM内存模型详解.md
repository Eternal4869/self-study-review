# Day15 详解 — JVM内存模型

## 〇、JDK版本演进与JVM变化（面试必知）

### 0.1 JDK LTS版本时间线

| 版本 | 发布时间 | 类型 | JVM相关重要变化 |
|------|---------|------|----------------|
| **JDK 7** | 2012 | LTS | G1引入，永久代存在 |
| **JDK 8** | 2014 | LTS | **元空间替代永久代**，字符串常量池移到堆中 |
| **JDK 9** | 2017 | 非LTS | **模块化系统**，G1成为默认GC |
| **JDK 11** | 2018 | LTS | **ZGC（实验性）**，Epsilon GC，移除Java EE模块 |
| **JDK 17** | 2021 | LTS | **ZGC/Shenandoah生产就绪**，移除Security Manager |
| **JDK 21** | 2023 | LTS | **Virtual Threads（虚拟线程）**，**分代ZGC**，Pattern Matching |
| **JDK 25** | 2025 | LTS | 最新LTS版本，持续演进 |

### 0.2 面试高频：JDK 8 vs JDK 17 vs JDK 21

```
JDK版本演进关键变化：

JDK 8 (2014)
├── 元空间替代永久代
├── Lambda表达式
├── Stream API
└── Optional类

JDK 9 (2017)
├── 模块化系统（Jigsaw）
├── G1成为默认GC
├── JShell交互式编程
└── 接口私有方法

JDK 11 (2018)
├── ZGC（实验性）
├── HttpClient标准化
├── var局部变量类型推断
└── 移除Java EE/CORBA模块

JDK 17 (2021)
├── ZGC/Shenandoah生产就绪
├── Sealed Classes（密封类）
├── Pattern Matching for instanceof
└── 移除Security Manager

JDK 21 (2023)
├── Virtual Threads（虚拟线程）★★★
├── Generational ZGC（分代ZGC）★★★
├── Pattern Matching for switch
├── Record Patterns
└── Sequenced Collections
```

---

## 一、JVM运行时数据区总览

### 1.1 线程私有 vs 线程共享

JVM运行时数据区分为两大类：**线程私有**和**线程共享**。

```
┌─────────────────────────────────────────────────────────────────┐
│                        JVM 运行时数据区                          │
├─────────────────────────┬───────────────────────────────────────┤
│      线程私有（独享）     │         线程共享（共享）               │
├─────────────────────────┼───────────────────────────────────────┤
│  ┌───────────────────┐  │  ┌─────────────────────────────────┐  │
│  │   虚拟机栈         │  │  │           堆（Heap）            │  │
│  │  (VM Stack)       │  │  │   对象实例 + 数组               │  │
│  ├───────────────────┤  │  ├─────────────────────────────────┤  │
│  │   本地方法栈       │  │  │         方法区                  │  │
│  │ (Native Method    │  │  │    (Method Area)                │  │
│  │     Stack)        │  │  │   类信息 + 常量 + 静态变量       │  │
│  ├───────────────────┤  │  └─────────────────────────────────┘  │
│  │   程序计数器       │  │                                       │
│  │ (Program Counter) │  │                                       │
│  └───────────────────┘  │                                       │
└─────────────────────────┴───────────────────────────────────────┘
```

### 1.2 各区域特点对比

| 区域 | 线程共享 | 存储内容 | OOM | 说明 |
|------|---------|---------|-----|------|
| **程序计数器** | 私有 | 当前执行字节码行号 | ❌ 唯一不会OOM | 唯一没有规定OOM的区域 |
| **虚拟机栈** | 私有 | 栈帧（局部变量表等） | ✅ StackOverflowError / OOM | 每个方法创建一个栈帧 |
| **本地方法栈** | 私有 | Native方法调用 | ✅ StackOverflowError / OOM | 与虚拟机栈类似 |
| **堆** | 共享 | 对象实例、数组 | ✅ Java heap space | GC主要管理区域 |
| **方法区** | 共享 | 类信息、常量、静态变量 | ✅ Metaspace | JDK8后由元空间实现 |

---

## 二、堆内存结构

### 2.1 堆的分代结构

堆是JVM中最大的一块内存区域，几乎所有的对象实例都在这里分配内存。

```
                              堆内存 (Heap)
    ┌─────────────────────────────────────────────────────┐
    │                                                     │
    │   新生代 (Young Generation)      老年代 (Old)        │
    │   ┌─────────────────────────┐   ┌───────────────┐   │
    │   │         │       │       │   │               │   │
    │   │  Eden   │  S0   │  S1   │   │    Old Gen    │   │
    │   │  80%    │  10%  │  10%  │   │     66%       │   │
    │   │         │       │       │   │               │   │
    │   └─────────────────────────┘   └───────────────┘   │
    │                                                     │
    └─────────────────────────────────────────────────────┘
    
    新生代 : 老年代 = 1 : 2 （默认比例）
    Eden : S0 : S1 = 8 : 1 : 1 （默认比例）
```

### 2.2 各区域详解

#### Eden区
- **作用**：新对象优先在Eden区分配内存
- **特点**：GC频繁发生（Minor GC）
- **大小**：默认占新生代的80%

#### Survivor区（S0/S1）
- **作用**：存放Minor GC后存活的对象
- **特点**：两个区域交替使用，复制算法
- **大小**：各占新生代的10%

#### 老年代
- **作用**：存放长期存活的对象
- **特点**：GC频率低，但回收时间长
- **大小**：默认占整个堆的2/3

### 2.3 对象晋升流程

```
新对象创建
    │
    ▼
┌─────────────┐
│  Eden区分配  │
└──────┬──────┘
       │ Eden区满了，触发Minor GC
       ▼
┌─────────────┐     ┌─────────────┐
│ 存活对象    │────→│  S0/S1区    │
│ (复制到S区) │     │  (年龄+1)   │
└─────────────┘     └──────┬──────┘
                           │ 年龄达到阈值(默认15)
                           ▼
                    ┌─────────────┐
                    │   老年代     │
                    └─────────────┘
```

### 2.4 代码示例

```java
// 对象在堆内存中的分配示例
public class HeapDemo {
    public static void main(String[] args) {
        // 局部变量引用存放在栈中，对象实例存放在堆中
        Person person = new Person("张三");
        
        // 数组也在堆中分配
        int[] array = new int[100];
        
        // 查看堆内存使用情况
        Runtime runtime = Runtime.getRuntime();
        System.out.println("最大堆内存: " + runtime.maxMemory() / 1024 / 1024 + "MB");
        System.out.println("已使用堆内存: " + runtime.totalMemory() / 1024 / 1024 + "MB");
    }
}

class Person {
    private String name;  // 引用类型，name对象也在堆中
    
    public Person(String name) {
        this.name = name;
    }
}
```

---

## 三、虚拟机栈

### 3.1 栈帧结构

每个方法执行时都会创建一个**栈帧（Stack Frame）**，用于存储该方法的局部变量表、操作数栈、动态链接、方法返回地址等信息。

```
虚拟机栈 (VM Stack)
    │
    │  方法A的栈帧
    │  ┌─────────────────────────────────┐
    │  │  局部变量表                      │
    │  │  - 基本类型 (int, long等)        │
    │  │  - 对象引用 (reference)          │
    │  ├─────────────────────────────────┤
    │  │  操作数栈                        │
    │  │  - 方法执行的工作区               │
    │  ├─────────────────────────────────┤
    │  │  动态链接                        │
    │  │  - 指向运行时常量池的方法引用     │
    │  ├─────────────────────────────────┤
    │  │  方法返回地址                    │
    │  │  - 方法退出后回到调用位置         │
    │  └─────────────────────────────────┘
    │
    │  方法B的栈帧
    │  ┌─────────────────────────────────┐
    │  │  ...                            │
    │  └─────────────────────────────────┘
```

### 3.2 栈帧各部分详解

| 组成部分 | 说明 | 示例 |
|---------|------|------|
| **局部变量表** | 存放方法参数和局部变量 | `int a = 10` 中的 `a` |
| **操作数栈** | 方法执行的工作区，压栈/出栈 | `iadd` 操作会从栈中弹出两个int |
| **动态链接** | 指向运行时常量池，支持多态 | 接口调用时确定具体实现 |
| **方法返回地址** | 方法退出后恢复调用者的执行位置 | `return` 后回到调用处 |

### 3.3 栈溢出问题

```java
// 递归调用导致栈溢出
public class StackOverflowDemo {
    private static int count = 0;
    
    public static void recursion() {
        count++;
        recursion();  // 无限递归
    }
    
    public static void main(String[] args) {
        try {
            recursion();
        } catch (StackOverflowError e) {
            System.out.println("栈深度: " + count);
            // 默认情况下，栈深度约1000-2000层
        }
    }
}
```

### 3.4 设置栈大小

```bash
# 设置线程栈大小（默认512KB）
-Xss256k
-Xss1m
-Xss2m
```

---

## 四、方法区与元空间

### 4.1 方法区的作用

方法区（Method Area）是JVM规范中定义的一个逻辑区域，用于存储：
- **类的元信息**：类名、字段、方法、接口等
- **常量池**：编译期生成的各种字面量和符号引用
- **静态变量**：类的静态字段
- **JIT编译后的代码**：即时编译器编译后的机器码

### 4.2 永久代 vs 元空间

| 对比项 | 永久代（JDK7及以前） | 元空间（JDK8+） |
|--------|---------------------|-----------------|
| **位置** | JVM堆内存中 | 本地内存（Native Memory） |
| **大小限制** | 固定大小，容易OOM | 默认无限制，受物理内存限制 |
| **配置参数** | -XX:PermSize / -XX:MaxPermSize | -XX:MetaspaceSize / -XX:MaxMetaspaceSize |
| **字符串常量池** | 在永久代中 | 移到堆中 |
| **存储内容** | 类信息+常量+静态变量+字符串常量池 | 仅类信息+静态变量+常量 |

### 4.3 为什么用元空间替代永久代？

```
永久代的问题：
┌─────────────────────────────────────┐
│  JVM堆内存                          │
│  ┌────────────────────────────────┐ │
│  │  新生代 + 老年代               │ │
│  ├────────────────────────────────┤ │
│  │  永久代（固定大小）              │ │  ← 容易OOM
│  │  -XX:MaxPermSize=256m          │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘

元空间的改进：
┌─────────────────────────────────────┐
│  JVM堆内存                          │
│  ┌────────────────────────────────┐ │
│  │  新生代 + 老年代               │ │  ← 更大空间
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  本地内存（Native Memory）           │
│  ┌────────────────────────────────┐ │
│  │  元空间（动态扩展）              │ │  ← 受物理内存限制
│  │  -XX:MaxMetaspaceSize=256m     │ │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
```

**主要原因**：
1. **永久代大小固定**，容易出现 `java.lang.OutOfMemoryError: PermGen space`
2. **元空间使用本地内存**，默认只受物理内存限制，更灵活
3. **合并HotSpot与JRockit**的需要，JRockit没有永久代概念

---

## 五、程序计数器

### 5.1 作用

程序计数器（Program Counter Register）是一块很小的内存空间，可以看作是**当前线程所执行的字节码的行号指示器**。

- **字节码解释器**通过改变程序计数器的值来选取下一条需要执行的字节码指令
- 分支、循环、跳转、异常处理、线程恢复等基础功能都依赖程序计数器

### 5.2 为什么是线程私有？

```
线程A执行:                    线程B执行:
┌──────────────────┐         ┌──────────────────┐
│ PC: 100          │         │ PC: 200          │
│ 执行第100行字节码 │         │ 执行第200行字节码 │
└──────────────────┘         └──────────────────┘
        │                            │
        ▼                            ▼
   方法调用后:                  方法调用后:
   PC = 150 (返回位置)         PC = 250 (返回位置)
```

- 如果不使用程序计数器，线程切换后无法恢复到正确的执行位置
- 每个线程都有独立的程序计数器，互不影响

### 5.3 唯一不会OOM的区域

程序计数器是**唯一一个**在JVM规范中没有规定任何 `OutOfMemoryError` 的区域，因为它占用的内存非常小，且大小固定。

---

## 六、本地方法栈

### 6.1 作用

本地方法栈（Native Method Stack）与虚拟机栈的作用类似，区别在于：
- **虚拟机栈**：为JVM执行Java方法（字节码）服务
- **本地方法栈**：为JVM使用Native方法服务

### 6.2 Native方法示例

```java
// Native方法声明
public class NativeMethodDemo {
    // Object类中的native方法
    // private native void registerNatives();
    
    // 获取当前线程
    // public static native Thread currentThread();
    
    public static void main(String[] args) {
        // 获取当前线程 - 底层调用Native方法
        Thread currentThread = Thread.currentThread();
        System.out.println("当前线程: " + currentThread.getName());
        
        // 获取系统时间 - 底层调用Native方法
        long time = System.currentTimeMillis();
        System.out.println("当前时间戳: " + time);
        
        // 获取对象的hash值
        Object obj = new Object();
        int hash = obj.hashCode();
        System.out.println("Hash值: " + hash);
    }
}
```

---

## 七、各区域OOM场景

### 7.1 堆内存溢出

```java
// 模拟堆内存溢出
// JVM参数: -Xms20m -Xmx20m
public class HeapOOMDemo {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        while (true) {
            list.add(new byte[1024 * 1024]); // 每次添加1MB
        }
    }
}
// 报错: java.lang.OutOfMemoryError: Java heap space
```

### 7.2 虚拟机栈溢出

```java
// 模拟虚拟机栈溢出
// JVM参数: -Xss128k
public class StackOOMDemo {
    private int stackLength = 1;
    
    public void stackLeak() {
        stackLeak();  // 无限递归
    }
    
    public static void main(String[] args) {
        StackOOMDemo oom = new StackOOMDemo();
        try {
            oom.stackLeak();
        } catch (StackOverflowError e) {
            System.out.println("栈深度: " + oom.stackLength);
        }
    }
}
// 报错: java.lang.StackOverflowError
```

### 7.3 元空间溢出

```java
// 模拟元空间溢出（需要大量动态生成类）
// JVM参数: -XX:MetaspaceSize=10m -XX:MaxMetaspaceSize=10m
public class MetaspaceOOMDemo {
    public static void main(String[] args) {
        int count = 0;
        try {
            while (true) {
                count++;
                // 使用CGLIB动态生成类
                // Enhancer enhancer = new Enhancer();
                // enhancer.setSuperclass(Object.class);
                // enhancer.setCallback((MethodInterceptor) (obj, method, objects, proxy) -> proxy.invokeSuper(obj, objects));
                // enhancer.create();
            }
        } catch (Throwable e) {
            System.out.println("生成类数量: " + count);
            e.printStackTrace();
        }
    }
}
// 报错: java.lang.OutOfMemoryError: Metaspace
```

### 7.4 OOM异常汇总

| 区域 | 异常类型 | 常见原因 |
|------|---------|---------|
| **堆** | `Java heap space` | 对象太多、内存泄漏 |
| **虚拟机栈** | `StackOverflowError` | 递归太深、栈帧太大 |
| **元空间** | `Metaspace` | 动态生成类太多 |
| **直接内存** | `Direct buffer memory` | NIO直接内存分配过多 |

---

## 八、JVM参数配置

### 8.1 常用堆参数

```bash
# 堆内存设置
-Xms512m          # 初始堆大小（建议与-Xmx相同）
-Xmx1024m         # 最大堆大小
-Xmn256m          # 新生代大小
-XX:NewRatio=2    # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8  # Eden:S0:S1 = 8:1:1

# 元空间设置
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=256m

# 栈设置
-Xss256k          # 线程栈大小
```

### 8.2 参数查看代码

```java
public class JVMParamsDemo {
    public static void main(String[] args) {
        // 获取JVM内存信息
        Runtime runtime = Runtime.getRuntime();
        
        System.out.println("===== JVM内存信息 =====");
        System.out.println("最大可用内存: " + runtime.maxMemory() / 1024 / 1024 + "MB");
        System.out.println("当前可用内存: " + runtime.totalMemory() / 1024 / 1024 + "MB");
        System.out.println("当前已用内存: " + (runtime.totalMemory() - runtime.freeMemory()) / 1024 / 1024 + "MB");
        System.out.println("当前空闲内存: " + runtime.freeMemory() / 1024 / 1024 + "MB");
        
        // 获取系统属性
        System.out.println("\n===== JVM系统属性 =====");
        System.out.println("Java版本: " + System.getProperty("java.version"));
        System.out.println("Java厂商: " + System.getProperty("java.vendor"));
        System.out.println("操作系统: " + System.getProperty("os.name"));
        System.out.println("CPU核数: " + runtime.availableProcessors());
    }
}
```

---

## 九、LeetCode题目解析

### 9.1 [53. 最大子数组和（Medium）](https://leetcode.cn/problems/maximum-subarray/)

**题目描述**：
给你一个整数数组 `nums`，请你找出一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

**思路分析**：
使用分治思想，将数组分成两半，最大子数组和要么在左半边，要么在右半边，要么跨越中点。

```java
class Solution {
    public int maxSubArray(int[] nums) {
        return maxSubArray(nums, 0, nums.length - 1);
    }
    
    private int maxSubArray(int[] nums, int left, int right) {
        // 递归终止条件
        if (left == right) {
            return nums[left];
        }
        
        int mid = left + (right - left) / 2;
        
        // 左半边最大子数组和
        int leftMax = maxSubArray(nums, left, mid);
        // 右半边最大子数组和
        int rightMax = maxSubArray(nums, mid + 1, right);
        // 跨越中点的最大子数组和
        int crossMax = maxCrossArray(nums, left, mid, right);
        
        return Math.max(Math.max(leftMax, rightMax), crossMax);
    }
    
    private int maxCrossArray(int[] nums, int left, int mid, int right) {
        // 从mid向左找最大和
        int leftSum = Integer.MIN_VALUE;
        int sum = 0;
        for (int i = mid; i >= left; i--) {
            sum += nums[i];
            leftSum = Math.max(leftSum, sum);
        }
        
        // 从mid+1向右找最大和
        int rightSum = Integer.MIN_VALUE;
        sum = 0;
        for (int i = mid + 1; i <= right; i++) {
            sum += nums[i];
            rightSum = Math.max(rightSum, sum);
        }
        
        return leftSum + rightSum;
    }
}
```

**复杂度分析**：
- 时间复杂度：O(n log n)
- 空间复杂度：O(log n)，递归栈深度

### 9.2 [1. 两数之和（Easy）](https://leetcode.cn/problems/two-sum/)

**题目描述**：
给定一个整数数组 `nums` 和一个整数目标值 `target`，在该数组中找出和为目标值的两个整数，并返回它们的数组下标。

**思路分析**：
使用哈希表存储已遍历的元素，每次查找 `target - nums[i}` 是否在哈希表中。

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            
            if (map.containsKey(complement)) {
                return new int[]{map.get(complement), i};
            }
            
            map.put(nums[i], i);
        }
        
        return new int[]{};
    }
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(n)

---

## 十、今日学习要点总结

### 核心要点归纳

1. **JVM运行时数据区**分为线程私有（程序计数器、虚拟机栈、本地方法栈）和线程共享（堆、方法区）
2. **堆内存分代**：新生代（Eden:S0:S1 = 8:1:1）+ 老年代（比例1:2）
3. **虚拟机栈**：每个方法创建一个栈帧，包含局部变量表、操作数栈、动态链接、方法返回地址
4. **方法区实现**：JDK7用永久代，JDK8用元空间（本地内存），解决了永久代OOM问题
5. **程序计数器**是唯一不会OOM的区域，线程私有
6. **各区域OOM异常**：堆 → Java heap space，栈 → StackOverflowError，元空间 → Metaspace

### 验收清单

- [ ] 能画出JVM内存模型图，标注各区域及所属关系
- [ ] 能说出堆内存的分代结构及各区域比例（Eden:S0:S1 = 8:1:1，新生代:老年代 = 1:2）
- [ ] 能解释为什么JDK8要用元空间替代永久代
- [ ] 能说明各区域分别会抛出什么OOM异常
- [ ] 能解释对象从Eden到老年代的晋升流程

---

## 十一、练习建议

1. **画图记忆**：亲手画出JVM内存模型图，标注各区域
2. **参数实践**：使用 `-Xms` `-Xmx` 等参数配置JVM，观察内存变化
3. **OOM模拟**：分别模拟堆、栈、元空间OOM，观察异常信息
4. **源码阅读**：查看 `Runtime.getRuntime().maxMemory()` 等方法的实现

---

## 十二、扩展阅读

- 《深入理解Java虚拟机》第2章：Java内存区域与内存溢出异常
- Oracle官方文档：The Java Virtual Machine Specification
- JVM内存模型详解：https://docs.oracle.com/javase/specs/
- JDK 21 Release Notes：https://www.oracle.com/java/technologies/javase/21-relnotes.html
- OpenJDK Projects：https://openjdk.org/projects/
