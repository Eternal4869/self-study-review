# Day17 详解 — 垃圾收集器（CMS、G1、ZGC）

## 〇、CMS废弃与收集器演进（面试必知）

**重要提示**：CMS（Concurrent Mark Sweep）已在JDK 14中被标记为废弃（JEP 363），并在后续版本中移除。

```
收集器演进时间线：

JDK 1.4 (2002)  →  JDK 9 (2017)  →  JDK 14 (2019)  →  JDK 21 (2023)
     │                │                │                 │
     ▼                ▼                ▼                 ▼
   引入CMS        G1成为默认        CMS废弃           ZGC分代化

JDK 23 (2024)
     │
     ▼
 ZGC默认分代模式
```

**当前推荐收集器（2024+）**：
- **通用场景**：G1（JDK9+默认）
- **超大堆 + 低延迟**：ZGC（JDK21+推荐）
- **极致低延迟**：ZGC -XX:MaxGCPauseMillis=1
- **高吞吐量**：Parallel Scavenge

---

## 一、垃圾收集器概览

### 1.1 收集器分类

```
垃圾收集器演进历史：

Serial (1999)  →  ParNew (2002)  →  Parallel Scavenge (2002)
    │                  │                     │
    ▼                  ▼                     ▼
Serial Old (1999)   CMS (2002)          Parallel Old (2008)
    │                   │                     │
    │              JDK14废弃                  │
    │                   │                     │
    ▼                   ▼                     ▼
                        G1 (2014) ← JDK9默认
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
    ZGC (2011)    Shenandoah (2012)   未来
    JDK11引入      JDK12引入
    JDK21分代化
    JDK23默认分代
```

### 1.2 收集器组合关系

```
新生代收集器          老年代收集器
┌─────────────┐      ┌─────────────┐
│ Serial      │ ←──→ │ Serial Old  │  组合1: Client模式
│ ParNew      │ ←──→ │ CMS         │  组合2: 互联网应用
│             │ ←──→ │ Serial Old  │  组合3: CMS后备
│ Parallel    │ ←──→ │ Parallel Old│  组合4: 吞吐量优先
└─────────────┘      └─────────────┘

┌─────────────┐
│     G1      │      整堆收集器，不需要配合
└─────────────┘
```

---

## 二、Serial/Serial Old收集器

### 2.1 Serial收集器

**特点**：
- 新生代收集器
- **单线程**收集
- 使用**复制算法**
- GC时必须**STW（Stop The World）**
- Client模式下的默认新生代收集器

```
Serial GC工作流程：

    应用线程          GC线程
        │               │
        ▼               │
    ┌───────┐           │
    │ 运行中 │           │
    └───┬───┘           │
        │               │
        │  GC触发       │
        ▼               │
    ┌───────┐      ┌───────┐
    │ STW   │ ←──→ │ 标记   │
    │       │      │ 复制   │
    └───────┘      └───────┘
        │               │
        │  GC完成       │
        ▼               │
    ┌───────┐           │
    │ 恢复  │           │
    └───────┘           │
```

### 2.2 Serial Old收集器

**特点**：
- 老年代收集器
- **单线程**收集
- 使用**标记-整理算法**
- Client模式下的默认老年代收集器
- CMS的后备方案（Concurrent Mode Failure时使用）

---

## 三、ParNew收集器

### 3.1 ParNew特点

**特点**：
- 新生代收集器
- **多线程**收集（Serial的多线程版本）
- 使用**复制算法**
- GC时**STW**
- **唯一能与CMS配合**的新生代收集器

```
ParNew GC工作流程（4线程）：

    应用线程    GC1    GC2    GC3    GC4
        │       │      │      │      │
        ▼       │      │      │      │
    ┌───────┐   │      │      │      │
    │ 运行中 │   │      │      │      │
    └───┬───┘   │      │      │      │
        │       │      │      │      │
        │  GC触发 │      │      │      │
        ▼       │      │      │      │
    ┌───────┐ ┌─────┐┌─────┐┌─────┐┌─────┐
    │ STW   │ │Eden ││Eden ││Eden ││Eden │
    │       │ │复制 ││复制 ││复制 ││复制 │
    └───────┘ └─────┘└─────┘└─────┘└─────┘
        │       │      │      │      │
        │  GC完成 │      │      │      │
        ▼       │      │      │      │
    ┌───────┐   │      │      │      │
    │ 恢复  │   │      │      │      │
    └───────┘   │      │      │      │
```

### 3.2 相关参数

```bash
# 启用ParNew收集器
-XX:+UseParNewGC

# 设置并行GC线程数
-XX:ParallelGCThreads=4  # 默认等于CPU核数
```

---

## 四、Parallel Scavenge/Parallel Old收集器

### 4.1 Parallel Scavenge收集器

**特点**：
- 新生代收集器
- **多线程**收集
- 使用**复制算法**
- **吞吐量优先**（而非停顿时间优先）

### 4.2 Parallel Old收集器

**特点**：
- 老年代收集器
- **多线程**收集
- 使用**标记-整理算法**
- JDK8默认收集器组合

### 4.3 吞吐量 vs 停顿时间

```
吞吐量优先 vs 停顿时间优先：

吞吐量 = 运行用户代码时间 / (运行用户代码时间 + GC时间)

示例：运行100分钟，GC花了1分钟
吞吐量 = 100 / (100 + 1) = 99%

┌─────────────────────────────────────────────────────────┐
│  吞吐量优先（Parallel）                                  │
│  ┌─┐    ┌─┐    ┌─┐    ┌─┐                              │
│  │ │    │ │    │ │    │ │    ← GC停顿                   │
│  └─┘    └─┘    └─┘    └─┘                              │
│  ████████████████████████████  ← 用户代码运行            │
│                                                         │
│  特点：GC停顿较长，但总吞吐量高                          │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  停顿时间优先（CMS/G1）                                  │
│  ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐ ┌┐                              │
│  ││ ││ ││ ││ ││ ││ ││ ││  ← 短暂GC停顿                 │
│  └┘ └┘ └┘ └┘ └┘ └┘ └┘ └┘                              │
│  ████████████████████████████  ← 用户代码运行            │
│                                                         │
│  特点：GC频繁但每次停顿短，总吞吐量较低                   │
└─────────────────────────────────────────────────────────┘
```

### 4.4 相关参数

```bash
# 启用Parallel收集器（JDK8默认）
-XX:+UseParallelGC          # 新生代
-XX:+UseParallelOldGC       # 老年代

# 设置GC线程数
-XX:ParallelGCThreads=4

# 设置最大GC停顿时间（毫秒）
-XX:MaxGCPauseMillis=200    # 目标：每次GC停顿不超过200ms

# 设置吞吐量大小
-XX:GCTimeRatio=99          # GC时间占比不超过1/(1+99)=1%

# 自适应调节策略（默认开启）
-XX:+UseAdaptiveSizePolicy  # 自动调整新生代大小、Survivor比例等
```

---

## 五、CMS收集器详解

### 5.1 CMS概述

**CMS（Concurrent Mark Sweep）** 是一种以**最短回收停顿时间**为目标的收集器，适合互联网应用。

**特点**：
- 老年代收集器
- 使用**标记-清除**算法
- **并发**收集（GC线程与用户线程同时工作）
- **STW时间短**
- **浮动垃圾**问题

### 5.2 CMS的4个阶段

```
CMS收集器工作流程：

用户线程:  ████████████████████████████████████████████████████████
          │         │              │         │
          │         │              │         │
GC线程:    │    ┌────┴────┐   ┌────┴────┐    │
          │    │ 初始标记 │   │ 重新标记 │    │
          │    │(STW)    │   │ (STW)   │    │
          │    └────┬────┘   └────┬────┘    │
          │         │              │         │
          │    ┌────┴──────────────┴────┐   │
          │    │      并发标记           │   │
          │    │   (与用户线程并行)       │   │
          │    └────────────────────────┘   │
          │                                  │
          │         ┌────────────────────┐   │
          │         │      并发清除       │   │
          │         │  (与用户线程并行)   │   │
          │         └────────────────────┘   │

时间轴: ─────────────────────────────────────────────────────→
```

#### 阶段1：初始标记（Initial Mark）- STW

**作用**：标记GC Roots直接关联的对象

**特点**：
- 需要STW（Stop The World）
- 速度很快（只标记直接关联对象）

```
初始标记：

GC Roots ──┬──→ 对象A (标记)
           ├──→ 对象B (标记)
           └──→ 对象C (标记)

对象D (未标记，因为不是直接关联)
对象E (未标记)
```

#### 阶段2：并发标记（Concurrent Mark）

**作用**：从GC Roots出发，遍历整个对象图

**特点**：
- 与用户线程**并发执行**
- 耗时最长，但不需要STW
- 可能产生**浮动垃圾**

```
并发标记：

GC Roots ──┬──→ 对象A (标记)
           │      │
           │      └──→ 对象D (标记)
           ├──→ 对象B (标记)
           │      │
           │      └──→ 对象E (标记)
           └──→ 对象C (标记)

对象F (未被任何GC Root引用，标记为可回收)
```

#### 阶段3：重新标记（Remark）- STW

**作用**：修正并发标记期间因用户线程运行而产生变动的标记结果

**特点**：
- 需要STW
- 比初始标记稍长，但远比并发标记短
- 使用**增量更新（Incremental Update）**算法

```
重新标记期间的变化：

并发标记期间：
用户线程创建了 新对象G
用户线程断开了 对象B → 对象E 的引用

重新标记会修正这些变化
```

#### 阶段4：并发清除（Concurrent Sweep）

**作用**：清除标记为可回收的对象

**特点**：
- 与用户线程**并发执行**
- 使用**标记-清除**算法，会产生内存碎片

### 5.3 CMS的缺点

#### 缺点1：浮动垃圾

```
浮动垃圾产生过程：

并发标记阶段：
┌─────────────────────────────────────────┐
│  GC线程正在标记对象                      │
│  此时用户线程创建了新对象F               │
│  对象F在本次GC中不会被标记               │
└─────────────────────────────────────────┘
                    ↓
并发清除阶段：
┌─────────────────────────────────────────┐
│  对象F不会被清除                         │
│  但它实际上是垃圾（没有引用）             │
│  → 浮动垃圾                              │
└─────────────────────────────────────────┘
```

**解决方案**：
- 让CMS在并发清除阶段预留一部分空间给新对象
- 当老年代空间使用率达到阈值时触发CMS GC

```bash
# 设置CMS触发阈值
-XX:CMSInitiatingOccupancyFraction=70  # 老年代使用70%时触发CMS
-XX:+UseCMSInitiatingOccupancyOnly     # 只在达到阈值时触发
```

#### 缺点2：内存碎片

**问题**：CMS使用标记-清除算法，会产生大量内存碎片

**解决方案**：
- 使用 `-XX:+UseCMSCompactAtFullCollection` 在Full GC时进行碎片整理
- 使用 `-XX:CMSFullGCsBeforeCompaction` 设置多少次Full GC后进行一次碎片整理

```bash
# Full GC时进行碎片整理
-XX:+UseCMSCompactAtFullCollection

# 每10次Full GC后进行一次碎片整理
-XX:CMSFullGCsBeforeCompaction=10
```

#### 缺点3：CPU敏感

**问题**：CMS是并发收集器，会占用CPU资源

**影响**：
- 默认回收线程数 = (CPU核数 + 3) / 4
- CPU核数少时，GC线程占用比例高

```bash
# 设置CMS线程数
-XX:ParallelCMSThreads=4
```

### 5.4 CMS相关参数汇总

```bash
# 启用CMS收集器
-XX:+UseConcMarkSweepGC

# CMS触发阈值
-XX:CMSInitiatingOccupancyFraction=70

# 只在达到阈值时触发CMS
-XX:+UseCMSInitiatingOccupancyOnly

# Full GC时碎片整理
-XX:+UseCMSCompactAtFullCollection

# 多少次Full GC后整理一次
-XX:CMSFullGCsBeforeCompaction=10

# CMS线程数
-XX:ParallelCMSThreads=4

# 并发CMS阶段的线程数
-XX:ConcGCThreads=2
```

---

## 六、G1收集器详解

### 6.1 G1概述

**G1（Garbage-First）** 是JDK9默认的垃圾收集器，目标是在**可控的停顿时间**内完成垃圾收集。

**特点**：
- **整堆收集器**（新生代+老年代）
- 使用**Region化内存**布局
- **可预测停顿**（设置目标停顿时间）
- 使用**标记-整理** + **复制**算法
- **并发**收集

### 6.2 G1的Region化内存设计

```
G1的内存布局：

┌────┬────┬────┬────┬────┬────┬────┬────┐
│ E  │ E  │ E  │ S  │ S  │    │    │    │  E = Eden
├────┼────┼────┼────┼────┼────┼────┼────┤  S = Survivor
│    │    │    │    │    │ O  │ O  │ O  │  O = Old
├────┼────┼────┼────┼────┼────┼────┼────┤  H = Humongous（大对象）
│ O  │ O  │ O  │ H  │ H  │    │    │    │  空 = 空闲Region
├────┼────┼────┼────┼────┼────┼────┼────┤
│    │    │    │    │    │    │    │    │
└────┴────┴────┴────┴────┴────┴────┴────┘

每个Region大小相同（1-32MB），可通过-XX:G1HeapRegionSize设置
```

**Region类型**：
- **Eden**：新生代Eden区
- **Survivor**：新生代Survivor区
- **Old**：老年代
- **Humongous**：存放超过Region 50%大小的大对象

### 6.3 G1的可预测停顿

```
G1的停顿预测模型：

用户设置目标停顿时间：
-XX:MaxGCPauseMillis=200  # 目标：每次GC不超过200ms

G1的工作方式：
┌─────────────────────────────────────────────────────────┐
│  1. 跟踪每个Region的回收价值（回收收益/回收成本）       │
│  2. 在目标停顿时间内，优先回收价值最高的Region           │
│  3. 动态调整回收策略，尽量满足停顿时间目标              │
└─────────────────────────────────────────────────────────┘

Region回收价值示例：
┌────┬────┬────┬────┬────┐
│ R1 │ R2 │ R3 │ R4 │ R5 │
│价值│价值│价值│价值│价值│
│ 10 │ 5  │ 8  │ 3  │ 7  │
└────┴────┴────┴────┴────┘
         ↓
优先回收: R1(10) → R3(8) → R5(7) → ...
```

### 6.4 G1的回收过程

#### Young GC

```
G1 Young GC流程：

┌─────────────────────────────────────────────────────────┐
│  1. 所有Eden Region被填满                               │
│  2. 触发Young GC                                       │
│  3. 将存活对象复制到Survivor Region或Old Region          │
│  4. 清空所有Eden Region                                 │
│  5. 部分Survivor Region晋升到Old Region                 │
└─────────────────────────────────────────────────────────┘

Young GC特点：
- STW执行
- 采用复制算法
- 停顿时间可控
```

#### Mixed GC

```
G1 Mixed GC流程：

┌─────────────────────────────────────────────────────────┐
│  1. 初始标记（Initial Mark）- 与Young GC同时执行        │
│  2. 并发标记（Concurrent Mark）- 与用户线程并发          │
│  3. 最终标记（Final Mark）- 处理STAB队列                │
│  4. 筛选回收（Cleanup）- 选择价值高的Region回收          │
│  5. 执行Mixed GC - 回收Old Region + Eden/Survivor       │
└─────────────────────────────────────────────────────────┘

Mixed GC触发条件：
- 并发标记结束后
- 老年代占用率达到阈值（默认45%）
```

#### Full GC

```
G1 Full GC触发条件：

┌─────────────────────────────────────────────────────────┐
│  1. 内存分配失败（老年代没有足够空间）                   │
│  2. Mixed GC速度跟不上内存分配速度                      │
│  3. 堆内存使用超过阈值                                  │
└─────────────────────────────────────────────────────────┘

Full GC特点：
- STW执行
- 使用Serial Old收集器（单线程）
- 非常耗时，应尽量避免
```

### 6.5 G1相关参数

```bash
# 启用G1收集器（JDK9+默认）
-XX:+UseG1GC

# 设置目标停顿时间（毫秒）
-XX:MaxGCPauseMillis=200

# 设置Region大小（1-32MB，2的幂）
-XX:G1HeapRegionSize=8m

# 设置堆内存大小
-Xms4g -Xmx4g

# G1触发Mixed GC的阈值
-XX:InitiatingHeapOccupancyPercent=45

# 设置新生代比例（默认5%-60%）
-XX:G1NewSizePercent=5
-XX:G1MaxNewSizePercent=60

# 设置Survivor比例
-XX:G1ReservePercent=10
```

---

## 七、ZGC收集器详解（JDK21+推荐）

### 7.1 ZGC概述

**ZGC（Z Garbage Collector）** 是JDK 11引入的超低延迟垃圾收集器，JDK 21引入分代ZGC（JEP 439），JDK 23默认使用分代模式。

### 7.2 ZGC演进历史

```
ZGC演进时间线：

JDK 11 (2018)    JDK 15 (2020)    JDK 21 (2023)    JDK 23 (2024)
     │                │                │                │
     ▼                ▼                ▼                ▼
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ 实验性   │ ──→ │ 生产就绪│ ──→ │ 分代ZGC │ ──→ │ 默认分代│
│ ZGC     │     │ ZGC     │     │ (JEP439)│     │ 模式    │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
```

### 7.3 ZGC核心特点

| 特性 | 说明 |
|------|------|
| **停顿时间** | < 1ms（不随堆大小增加） |
| **最大堆** | 支持TB级（理论16TB+） |
| **分代模式** | JDK21引入，JDK23默认 |
| **着色指针** | 使用64位指针存储元数据 |
| **读屏障** | 使用Load Barrier实现并发处理 |
| **适用场景** | 大堆、低延迟要求高的应用 |

### 7.4 ZGC工作原理

```
ZGC着色指针（Colored Pointers）：

64位指针结构：
┌─────────────────────────────────────────────────────────────┐
│  元数据位  │              对象地址                           │
│  (4位)     │              (60位)                            │
├───────────┼────────────────────────────────────────────────┤
│ Marked0   │  标记位0，表示对象是否被标记                    │
│ Marked1   │  标记位1，表示对象是否被标记                    │
│ Remapped  │  重映射位，表示对象是否已被移动                  │
│ Finalizable│ 可终结位，表示对象是否可终结                   │
└───────────┴────────────────────────────────────────────────┘

ZGC工作流程：
┌─────────────────────────────────────────────────────────────┐
│  1. 并发标记（Concurrent Mark）                             │
│     - 使用着色指针标记存活对象                               │
│     - 几乎不需要STW                                        │
├─────────────────────────────────────────────────────────────┤
│  2. 并发预备重分配（Concurrent Prepare for Relocate）       │
│     - 分析需要重分配的内存区                                │
├─────────────────────────────────────────────────────────────┤
│  3. 并发重分配（Concurrent Relocate）                       │
│     - 使用读屏障和转发表移动对象                             │
│     - 更新着色指针                                          │
└─────────────────────────────────────────────────────────────┘
```

### 7.5 ZGC参数配置

```bash
# 启用ZGC
-XX:+UseZGC

# 分代模式（JDK21+）
-XX:+UseZGC -XX:+ZGenerational
# JDK23+只需 -XX:+UseZGC（默认分代）

# 设置最大停顿时间目标
-XX:MaxGCPauseMillis=5  # 目标5ms（默认）

# 设置堆大小
-Xms4g -Xmx16g

# 设置GC线程数（可选）
-XX:ConcGCThreads=4
```

### 7.6 ZGC vs G1对比

| 对比项 | G1 | ZGC |
|--------|-----|-----|
| **停顿时间** | 数十~数百ms | < 1ms |
| **吞吐量** | 高 | 中等 |
| **堆大小** | 数GB~数十GB | 数百MB~TB |
| **内存开销** | 中等 | 较高 |
| **适用场景** | 通用 | 大堆、低延迟 |
| **JDK版本** | JDK9+默认 | JDK21+推荐 |

---

## 八、Shenandoah收集器简介

### 8.1 Shenandoah概述

**Shenandoah** 是Red Hat开发的低延迟垃圾收集器，目标是将停顿时间控制在10ms以内。

### 8.2 Shenandoah特点

| 特性 | 说明 |
|------|------|
| **停顿时间** | < 10ms |
| **并发整理** | 使用Brooks Pointer实现并发整理 |
| **适用场景** | 低延迟要求的应用 |
| **JDK版本** | JDK12+生产就绪 |

### 8.3 Shenandoah参数

```bash
# 启用Shenandoah
-XX:+UseShenandoahGC

# 设置目标停顿时间
-XX:ShenandoahGCHeuristics=adaptive
```

---

## 九、CMS vs G1 vs ZGC对比

### 7.1 核心对比表

| 对比项 | CMS | G1 |
|--------|-----|-----|
| **收集范围** | 老年代 | 整堆（新生代+老年代） |
| **内存布局** | 连续内存 | Region化内存 |
| **算法** | 标记-清除 | 标记-整理 + 复制 |
| **内存碎片** | 有碎片 | 无碎片 |
| **停顿时间** | 不可预测 | 可预测（-XX:MaxGCPauseMillis） |
| **并发程度** | 高（大部分阶段并发） | 高（大部分阶段并发） |
| **适用堆大小** | 中小堆（< 4GB） | 大堆（> 4GB） |
| **JDK版本** | JDK9之前 | JDK9+默认 |

### 7.2 选择建议

```
选择收集器的决策流程：

                    ┌─────────────────┐
                    │  堆内存大小？    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        < 4GB          4GB - 16GB        > 16GB
              │              │              │
              ▼              ▼              ▼
        ┌─────────┐    ┌─────────┐    ┌─────────┐
        │  CMS    │    │  CMS    │    │   G1    │
        │  或 G1  │    │  或 G1  │    │         │
        └─────────┘    └─────────┘    └─────────┘

具体建议：
├─ 小堆（< 4GB）：CMS或G1都可以
├─ 中堆（4-16GB）：CMS或G1，看停顿时间要求
├─ 大堆（> 16GB）：优先G1
├─ 对停顿敏感：G1（可设置目标停顿时间）
└─ JDK9+：默认G1
```

### 7.3 优缺点对比

| 对比项 | CMS优点 | CMS缺点 | G1优点 | G1缺点 |
|--------|---------|---------|--------|--------|
| **停顿** | 停顿短 | 不可预测 | 可预测停顿 | - |
| **碎片** | - | 有碎片 | 无碎片 | - |
| **吞吐** | - | 吞吐量低 | - | 吞吐量较低 |
| **内存** | 适合小堆 | - | 适合大堆 | 内存开销大 |
| **复杂度** | 实现简单 | - | - | 实现复杂 |

---

## 八、代码示例

### 8.1 收集器配置示例

```java
public class CollectorDemo {
    public static void main(String[] args) {
        // 模拟对象创建
        List<byte[]> list = new ArrayList<>();
        
        for (int i = 0; i < 1000; i++) {
            list.add(new byte[1024 * 1024]); // 1MB
        }
        
        // 查看GC信息
        // JVM参数示例：
        // -Xms512m -Xmx512m -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -verbose:gc -XX:+PrintGCDetails
    }
}
```

### 8.2 GC日志分析

```bash
# GC日志参数（JDK8）
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:/path/to/gc.log

# GC日志参数（JDK9+）
-Xlog:gc*:file=/path/to/gc.log:time,uptime,level,tags

# GC日志分析工具
# GCEasy: https://gceasy.io
# GCViewer: https://github.com/chewiebug/GCViewer
```

---

## 九、LeetCode题目解析

### 9.1 [104. 二叉树的最大深度（Easy）](https://leetcode.cn/problems/maximum-depth-of-binary-tree/)

**题目描述**：
给定一个二叉树 `root`，返回其最大深度。最大深度是从根节点到最远叶子节点的最长路径上的节点数。

**思路分析**：
使用递归或迭代（BFS/DFS）求解。

```java
// 递归方式
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        
        int leftDepth = maxDepth(root.left);
        int rightDepth = maxDepth(root.right);
        
        return Math.max(leftDepth, rightDepth) + 1;
    }
}

// BFS方式
class Solution2 {
    public int maxDepth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        
        Queue<TreeNode> queue = new LinkedList<>();
        queue.offer(root);
        int depth = 0;
        
        while (!queue.isEmpty()) {
            int size = queue.size();
            depth++;
            
            for (int i = 0; i < size; i++) {
                TreeNode node = queue.poll();
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }
        }
        
        return depth;
    }
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：递归 O(h)，BFS O(n)

### 9.2 [102. 二叉树的层序遍历（Medium）](https://leetcode.cn/problems/binary-tree-level-order-traversal/)

**题目描述**：
给你二叉树的根节点 `root`，返回其节点值的层序遍历（即逐层地，从左到右访问所有节点）。

**思路分析**：
使用BFS，借助队列实现层序遍历。

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> result = new ArrayList<>();
        
        if (root == null) {
            return result;
        }
        
        Queue<TreeNode> queue = new LinkedList<>();
        queue.offer(root);
        
        while (!queue.isEmpty()) {
            int size = queue.size();
            List<Integer> level = new ArrayList<>();
            
            for (int i = 0; i < size; i++) {
                TreeNode node = queue.poll();
                level.add(node.val);
                
                if (node.left != null) {
                    queue.offer(node.left);
                }
                if (node.right != null) {
                    queue.offer(node.right);
                }
            }
            
            result.add(level);
        }
        
        return result;
    }
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(n)

---

## 十一、今日学习要点总结

### 核心要点归纳

1. **Serial**：单线程收集器，Client模式默认
2. **ParNew**：多线程新生代收集器，配合CMS使用（CMS已废弃）
3. **Parallel**：吞吐量优先，JDK8默认收集器
4. **CMS**：4个阶段（初始标记→并发标记→重新标记→并发清除），**JDK14已废弃**
5. **G1**：Region化内存，可预测停顿，整堆收集器，**JDK9+默认**
6. **ZGC**：停顿<1ms，支持TB级堆，**JDK21+推荐**，JDK23默认分代
7. **Shenandoah**：低延迟收集器，<10ms停顿，JDK12+可用
8. **收集器选择**：通用用G1，大堆+低延迟用ZGC，高吞吐用Parallel

### 验收清单

- [ ] 能说出CMS的4个阶段及各自做了什么
- [ ] 能解释CMS为什么会产生浮动垃圾
- [ ] 能说明G1的Region划分和回收策略
- [ ] 能对比CMS和G1的优缺点及适用场景
- [ ] 能解释Young GC、Mixed GC、Full GC的区别
- [ ] 能说出ZGC的核心特点（停顿<1ms，支持TB级堆）
- [ ] 能说明CMS已被废弃，推荐使用G1或ZGC
- [ ] 能根据应用场景选择合适的收集器

---

## 十二、练习建议

1. **收集器选择**：根据应用场景选择合适的收集器
2. **GC日志分析**：使用GCEasy或GCViewer分析GC日志
3. **参数调优**：实践 `-XX:MaxGCPauseMillis` 等参数
4. **面试准备**：CMS、G1、ZGC都是高频面试题，需要深入理解
5. **ZGC实践**：尝试使用 `-XX:+UseZGC` 参数运行程序，观察GC日志

---

## 十三、扩展阅读

- 《深入理解Java虚拟机》第3章：垃圾收集器
- Oracle官方文档：JVM Garbage Collection Tuning
- G1收集器设计论文：A Study of the Replication Copying Garbage Collector
- JEP 439: Generational ZGC：https://openjdk.org/jeps/439
- JEP 474: ZGC: Generational Mode by Default：https://openjdk.org/jeps/474
