# Day09 详解 — synchronized原理

## 一、synchronized基本使用

### 1.1 三种使用方式

```java
// 方式一：修饰实例方法（锁住当前对象）
public synchronized void methodA() {
    // 同步代码
}

// 方式二：修饰静态方法（锁住Class对象）
public static synchronized void methodB() {
    // 同步代码
}

// 方式三：修饰代码块（锁住指定对象）
public void methodC() {
    synchronized (this) {  // 或 synchronized (ClassName.class)
        // 同步代码
    }
}
```

### 1.2 锁对象对照表

| synchronized位置 | 锁对象 | 示例 |
|------------------|--------|------|
| 实例方法 | 当前实例 `this` | `this` |
| 静态方法 | Class对象 | `ClassName.class` |
| 代码块 | 括号中的对象 | `new Object()`、`lock` |

### 1.3 代码验证

```java
public class SynchronizedLockTest {
    private static final Object lock = new Object();
    
    // 修饰实例方法 - 锁this
    public synchronized void instanceMethod() {
        System.out.println(Thread.currentThread().getName() + ": 实例方法");
    }
    
    // 修饰静态方法 - 锁Class
    public static synchronized void staticMethod() {
        System.out.println(Thread.currentThread().getName() + ": 静态方法");
    }
    
    public void blockMethod() {
        synchronized (lock) {  // 锁lock对象
            System.out.println(Thread.currentThread().getName() + ": 代码块");
        }
    }
}
```

---

## 二、synchronized底层原理

### 2.1 Monitor对象

每个Java对象都关联着一个**Monitor对象**（监视器锁），这是synchronized实现的基础。

```
Java对象结构:
┌─────────────────────┐
│       Object        │
├─────────────────────┤
│    对象头(Header)    │ ← 包含指向Monitor的指针
├─────────────────────┤
│    实例数据         │
├─────────────────────┤
│    对齐填充         │
└─────────────────────┘

Monitor结构:
┌─────────────────────────────┐
│         Monitor             │
├─────────────────────────────┤
│  Owner (持有锁的线程)       │
├─────────────────────────────┤
│  EntryList (等待锁的队列)   │
├─────────────────────────────┤
│  WaitSet (调用wait的队列)   │
└─────────────────────────────┘
```

### 2.2 monitorenter和monitorexit指令

```java
// Java代码
public void syncMethod() {
    synchronized (this) {
        System.out.println("同步代码块");
    }
}

// 字节码指令
public void syncMethod();
    Code:
       0: aload_0           // 加载this
       1: dup               // 复制栈顶
       2: astore_1          // 存储到局部变量
       3: monitorenter      // 获取锁
       4: getstatic         // System.out
       7: ldc               // "同步代码块"
       9: invokevirtual     // println
      12: aload_1
      13: monitorexit       // 释放锁
      14: goto 22
      17: astore_2          // 异常处理
      18: aload_1
      19: monitorexit       // 异常时也释放锁
      20: aload_2
      21: athrow
      22: return
```

### 2.3 monitorenter指令详解

1. 如果Monitor的进入数为0，则该线程进入Monitor，将进入数设置为1，该线程为Monitor的所有者
2. 如果该线程已经占有该Monitor，只是重新进入，则进入数+1
3. 如果其他线程已经占用了Monitor，则该线程进入阻塞状态，直到Monitor的进入数为0，再重新尝试获取Monitor的所有权

### 2.4 monitorexit指令详解

1. 执行monitorexit的线程必须是Monitor的所有者
2. 执行后Monitor的进入数减1，如果减1后进入数为0，则该线程退出Monitor
3. 如果该线程不是Monitor的所有者，会抛出IllegalMonitorStateException异常

---

## 三、锁升级过程

### 3.1 锁升级路径

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁（自JDK15禁用，JDK18废弃）
无锁 → 轻量级锁 → 重量级锁
无锁 → 重量级锁
```

**注意**：锁升级是单向的，不会降级（偏向锁除外，会因竞争而撤销）

### 3.2 偏向锁（JDK18已经废弃）

**原理**：
- 在对象头中记录持有锁的线程ID
- 下次同一线程进入时，只需比较线程ID，无需CAS操作
- 适合只有一个线程访问同步块的场景

**工作流程**：
```
线程A第一次访问:
┌─────────────────────────┐
│  1. CAS设置线程ID到对象头 │
│  2. 成功 → 获取偏向锁     │
│  3. 失败 → 等待全局安全点  │
│  4. 撤销偏向锁，升级轻量级 │
└─────────────────────────┘

线程A再次访问:
┌─────────────────────────┐
│  1. 比较线程ID是否一致    │
│  2. 一致 → 直接访问       │
│  3. 不一致 → 撤销偏向锁   │
└─────────────────────────┘
```

**代码验证**：
```java
// 偏向锁延迟启动验证
// JVM参数：-XX:+UseBiasedLocking -XX:BiasedLockingStartupDelay=0
public class BiasedLockingTest {
    public static void main(String[] args) throws InterruptedException {
        // 等待偏向锁启动（默认延迟4秒）
        Thread.sleep(5000);
        
        Object obj = new Object();
        
        Thread t1 = new Thread(() -> {
            synchronized (obj) {
                // 验证偏向锁
                // 使用jol-core: Layout.parseInstance(obj).toPrintable()
            }
        });
        
        t1.start();
        t1.join();
    }
}
```

### 3.3 轻量级锁

**原理**：
- 线程在栈帧中创建锁记录（Lock Record）
- 将对象头中的Mark Word复制到锁记录
- CAS尝试将对象头的Mark Word指向锁记录
- 适合多个线程交替访问（无竞争）的场景

**工作流程**：
```
线程尝试获取轻量级锁:
┌─────────────────────────────────────┐
│ 1. 在栈帧中创建Lock Record          │
│ 2. 复制Mark Word到Lock Record       │
│ 3. CAS将对象头指向Lock Record       │
│    ├─ 成功 → 获取轻量级锁           │
│    └─ 失败 → 自旋等待               │
│         ├─ 自旋成功 → 获取轻量级锁   │
│         └─ 自旋失败 → 升级重量级锁   │
└─────────────────────────────────────┘
```

**自旋锁**：
```java
// 自旋锁原理（简化版）
while (!cas(obj.markWord, null, lockRecord)) {
    // 自旋等待
    // JDK6+使用自适应自旋
}
```

### 3.4 重量级锁

**原理**：
- 依赖操作系统的Mutex Lock（互斥锁）
- 涉及用户态和内核态的切换
- 适合多个线程同时访问同步块的场景

**工作流程**：
```
线程获取重量级锁:
┌─────────────────────────────────────┐
│ 1. JVM创建ObjectMonitor             │
│ 2. 设置Owner为当前线程              │
│ 3. 其他线程进入EntryList等待        │
│ 4. 当前线程调用wait()进入WaitSet    │
│ 5. 当前线程释放锁，唤醒EntryList    │
└─────────────────────────────────────┘
```

---

## 四、锁升级对比

### 4.1 三种锁的对比

| 对比项 | 偏向锁 | 轻量级锁 | 重量级锁 |
|--------|--------|----------|----------|
| **适用场景** | 单线程访问 | 交替访问 | 并发访问 |
| **实现方式** | CAS+Thread ID | CAS+Lock Record | OS Mutex |
| **性能开销** | 最低 | 较低 | 最高 |
| **线程阻塞** | 不阻塞 | 自旋等待 | 阻塞等待 |
| **是否公平** | 非公平 | 非公平 | 非公平 |

### 4.2 JDK版本对锁的优化

| JDK版本 | 优化内容 |
|---------|---------|
| JDK6 | 引入偏向锁、轻量级锁、自适应自旋 |
| JDK15 | 移除偏向锁（JEP 374） |
| JDK18 | 默认禁用偏向锁 |

### 4.3 为什么JDK15移除偏向锁？

```java
// 偏向锁的性能问题：
// 1. 撤销偏向锁需要STW（Stop The World）
// 2. 现代应用中竞争频繁，偏向锁优势不大
// 3. 维护成本高，代码复杂
// 4. JDK6+的轻量级锁已经足够高效
```

---

## 五、synchronized与Lock对比

### 5.1 核心对比

| 对比项 | synchronized | Lock |
|--------|-------------|------|
| **实现层面** | JVM内置 | Java API |
| **锁获取** | 自动获取释放 | 手动获取释放 |
| **可中断** | ❌ | ✅ `lockInterruptibly()` |
| **公平性** | 非公平 | 可选公平/非公平 |
| **条件变量** | 单一（wait/notify） | 多个Condition |
| **性能** | JDK6+优化后差不多 | 差不多 |

### 5.2 使用建议

```java
// 优先使用synchronized的情况：
// 1. 简单的同步需求
// 2. 不需要高级特性（可中断、公平锁）
// 3. 代码块小，执行时间短

// 优先使用Lock的情况：
// 1. 需要可中断锁
// 2. 需要公平锁
// 3. 需要多个条件变量
// 4. 需要非阻塞地获取锁
```

---

## 六、LeetCode题目解析

### 6.1 [1115. 交替打印FooBar（Medium）](https://leetcode.cn/problems/print-foobar-alternately/)

**题目描述**：
设计程序让两个线程交替打印 "foo" 和 "bar"。

**思路分析**：
使用 `synchronized` + `volatile` 标志位控制执行顺序。

```java
class FooBar {
    private int n;
    private volatile boolean flag = true;  // true: foo执行, false: bar执行
    private final Object lock = new Object();

    public FooBar(int n) {
        this.n = n;
    }

    public void foo(Runnable printFoo) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            synchronized (lock) {
                while (!flag) {
                    lock.wait();
                }
                printFoo.run();
                flag = false;
                lock.notifyAll();
            }
        }
    }

    public void bar(Runnable printBar) throws InterruptedException {
        for (int i = 0; i < n; i++) {
            synchronized (lock) {
                while (flag) {
                    lock.wait();
                }
                printBar.run();
                flag = true;
                lock.notifyAll();
            }
        }
    }
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

## 七、今日学习要点总结

### 核心要点归纳

1. **synchronized三种用法**：实例方法（锁this）、静态方法（锁Class）、代码块（锁指定对象）
2. **底层原理**：基于Monitor对象，通过monitorenter/monitorexit指令实现
3. **锁升级过程**：无锁 → 偏向锁 → 轻量级锁 → 重量级锁
4. **偏向锁**：适合单线程访问，记录Thread ID，JDK15已移除
5. **轻量级锁**：适合交替访问，CAS+Lock Record
6. **重量级锁**：适合并发访问，依赖OS Mutex

### 验收清单

- [ ] 能解释synchronized的锁升级过程
- [ ] 能说明为什么JDK6要引入锁升级
- [ ] 能解释偏向锁为什么在JDK15被移除
- [ ] 能说出synchronized三种使用方式及其锁对象
- [ ] 能解释monitorenter和monitorexit的作用

---

## 八、练习建议

1. **查看字节码**：使用javap -c查看synchronized代码块的字节码
2. **JOL工具**：使用jol-core查看对象头信息，验证锁状态
3. **JVM参数**：通过-XX:+UseBiasedLocking等参数验证锁升级
4. **源码阅读**：阅读ObjectMonitor源码（HotSpot虚拟机）

---

## 九、扩展阅读

- 《深入理解Java虚拟机》第13章
- 《Java并发编程的艺术》第5章
- HotSpot虚拟机ObjectMonitor源码
