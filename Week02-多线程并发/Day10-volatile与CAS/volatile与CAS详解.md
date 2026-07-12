# Day10 详解 — volatile与CAS

## 一、volatile关键字

### 1.1 volatile的两个作用

| 作用 | 说明 | 实现原理 |
|------|------|---------|
| **保证可见性** | 一个线程修改后，其他线程立即可见 | 内存屏障 |
| **禁止指令重排** | 保证代码执行顺序 | 内存屏障 |
| **不保证原子性** | 复合操作不是线程安全 | 需要配合锁/CAS |

### 1.2 volatile内存语义

```
volatile写操作:
┌─────────────────────────────────────────┐
│  StoreStore屏障 → 写入变量 → StoreLoad屏障 │
└─────────────────────────────────────────┘

volatile读操作:
┌─────────────────────────────────────────┐
│  LoadLoad屏障 → 读取变量 → LoadStore屏障   │
└─────────────────────────────────────────┘
```

### 1.3 代码验证可见性

```java
public class VolatileVisibilityTest {
    private static volatile boolean flag = false;
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            while (!flag) {
                // 等待flag变为true
            }
            System.out.println("t1检测到flag变化");
        });
        
        Thread t2 = new Thread(() -> {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            flag = true;  // 修改volatile变量
            System.out.println("t2修改flag为true");
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
// 输出：
// t2修改flag为true
// t1检测到flag变化
```

### 1.4 代码验证不保证原子性

```java
public class VolatileAtomicTest {
    private static volatile int count = 0;
    
    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    count++;  // 非原子操作
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        System.out.println("count = " + count);
        // 预期10000，实际小于10000
    }
}
```

### 1.5 为什么volatile不保证原子性？

```java
// count++ 实际包含3个操作：
// 1. 读取count的值
// 2. count加1
// 3. 写入新值到count

// 线程A: 读取count=10
// 线程B: 读取count=10
// 线程A: count=11，写入
// 线程B: count=11，写入
// 结果：count=11，而不是期望的12
```

---

## 二、内存屏障

### 2.1 内存屏障类型

| 屏障类型 | 作用 | 说明 |
|---------|------|------|
| **LoadLoad屏障** | 保证读操作顺序 | 在Load1和Load2之间 |
| **StoreStore屏障** | 保证写操作顺序 | 在Store1和Store2之间 |
| **LoadStore屏障** | 保证读写顺序 | 在Load和Store之间 |
| **StoreLoad屏障** | 保证写读顺序 | 在Store和Load之间，最耗性能 |

### 2.2 volatile内存屏障插入策略

```java
// volatile写
volatile_write();
// 插入StoreStore屏障（之前的写操作不能重排到volatile写之后）
// 插入StoreLoad屏障（volatile写之后的操作不能重排到volatile写之前）

// volatile读
volatile_read();
// 插入LoadLoad屏障（之后的读操作不能重排到volatile读之前）
// 插入LoadStore屏障（之后的写操作不能重排到volatile读之前）
```

### 2.3 指令重排示例

```java
// 没有volatile时可能重排
int a = 1;      // 1
int b = 2;      // 2
a = a + 3;      // 3
a = a * b;      // 4

// 可能的重排顺序：1, 3, 2, 4 或 2, 4, 1, 3

// 使用volatile禁止重排
volatile int a = 1;  // 1
int b = 2;           // 2
a = a + 3;           // 3
a = a * b;           // 4
// 1, 2, 3, 4 顺序执行
```

---

## 三、MESI缓存一致性协议

### 3.1 缓存行状态

| 状态 | 说明 | 含义 |
|------|------|------|
| **Modified** | 已修改 | 只有当前缓存有数据，已修改 |
| **Exclusive** | 独占 | 只有当前缓存有数据，未修改 |
| **Shared** | 共享 | 多个缓存都有数据，未修改 |
| **Invalid** | 无效 | 缓存数据已失效 |

### 3.2 MESI工作流程

```
读操作流程:
┌─────────────────────────────────────────────────────┐
│ CPU1读取变量x                                        │
│ 1. 本地缓存未命中                                    │
│ 2. 广播Read Invalidate消息到总线                     │
│ 3. 其他CPU响应，标记本地缓存为Invalid                 │
│ 4. 从内存加载数据，状态为Exclusive                    │
└─────────────────────────────────────────────────────┘

写操作流程:
┌─────────────────────────────────────────────────────┐
│ CPU1写入变量x                                        │
│ 1. 检查缓存行状态                                    │
│ 2. 如果是Shared状态，发送Invalidate消息              │
│ 3. 其他CPU将本地缓存标记为Invalid                    │
│ 4. CPU1获得Exclusive所有权，写入数据                 │
└─────────────────────────────────────────────────────┘
```

### 3.3 volatile与MESI

```java
// volatile写入时的MESI流程
volatile int count = 0;

// 线程A执行: count = 1
// 1. CPU1缓存行状态为Shared（其他CPU也有）
// 2. CPU1发送Invalidate消息
// 3. CPU2将缓存行标记为Invalid
// 4. CPU1写入count=1，状态变为Modified
// 5. 其他CPU需要重新从内存读取
```

---

## 四、CAS原理

### 4.1 CAS是什么？

CAS（Compare And Swap）是**比较并交换**，是一种无锁的原子操作。

```
CAS操作过程:
┌─────────────────────────────────────────┐
│ 1. 读取当前值V                           │
│ 2. 计算新值N                             │
│ 3. 比较V与期望值A                        │
│    ├─ 相等 → 写入新值N                   │
│    └─ 不等 → 重试（自旋）                │
└─────────────────────────────────────────┘
```

### 4.2 Unsafe类实现CAS

```java
// Unsafe类的CAS方法
public final class Unsafe {
    // CAS操作int类型
    public final native boolean compareAndSwapInt(
        Object o, long offset, int expected, int update
    );
    
    // CAS操作long类型
    public final native boolean compareAndSwapLong(
        Object o, long offset, long expected, long update
    );
    
    // CAS操作引用类型
    public final native boolean compareAndSwapObject(
        Object o, long offset, Object expected, Object update
    );
}

// 使用示例
public class CASTest {
    private volatile int count = 0;
    private static final Unsafe unsafe;
    private static final long countOffset;
    
    static {
        try {
            Field f = Unsafe.class.getDeclaredField("theUnsafe");
            f.setAccessible(true);
            unsafe = (Unsafe) f.get(null);
            countOffset = unsafe.objectFieldOffset(
                CASTest.class.getDeclaredField("count")
            );
        } catch (Exception e) {
            throw new Error(e);
        }
    }
    
    public void increment() {
        while (true) {
            int current = count;
            int next = current + 1;
            if (unsafe.compareAndSwapInt(this, countOffset, current, next)) {
                break;
            }
        }
    }
}
```

### 4.3 AtomicInteger示例

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerTest {
    private static AtomicInteger count = new AtomicInteger(0);
    
    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    count.incrementAndGet();  // 原子操作
                }
            });
            threads[i].start();
        }
        
        for (Thread t : threads) {
            t.join();
        }
        
        System.out.println("count = " + count.get());  // 输出10000
    }
}
```

---

## 五、ABA问题

### 5.1 什么是ABA问题？

```
ABA问题示例:
┌─────────────────────────────────────────────────────┐
│ 线程1: 读取A → 准备替换为C                           │
│ 线程2: 读取A → 修改为B → 修改回A                    │
│ 线程1: 比较发现还是A → 替换为C（成功）               │
└─────────────────────────────────────────────────────┘

问题：线程1认为没有变化，但实际上经历了A→B→A
```

### 5.2 ABA问题示例代码

```java
public class ABAProblem {
    private static AtomicReference<Integer> ref = new AtomicReference<>(1);
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            int value = ref.get();  // 读取1
            System.out.println("T1读取: " + value);
            
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
            // 尝试将1改为2（期望还是1）
            boolean success = ref.compareAndSet(1, 2);
            System.out.println("T1修改结果: " + success);  // true
        });
        
        Thread t2 = new Thread(() -> {
            int value = ref.get();  // 读取1
            System.out.println("T2读取: " + value);
            
            // 修改为2
            ref.compareAndSet(1, 2);
            System.out.println("T2修改为: 2");
            
            // 修改回1
            ref.compareAndSet(2, 1);
            System.out.println("T2修改为: 1");
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
// 输出：
// T1读取: 1
// T2读取: 1
// T2修改为: 2
// T2修改为: 1
// T1修改结果: true  ← ABA问题
```

### 5.3 解决方案：AtomicStampedReference

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABAFix {
    private static AtomicStampedReference<Integer> ref = 
        new AtomicStampedReference<>(1, 0);  // 值为1，版本号为0
    
    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            int stamp = ref.getStamp();  // 获取版本号
            int value = ref.getReference();  // 获取值
            System.out.println("T1读取: " + value + ", 版本: " + stamp);
            
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            
            // 尝试修改（期望值1，期望版本0）
            boolean success = ref.compareAndSet(1, 2, stamp, stamp + 1);
            System.out.println("T1修改结果: " + success);  // false
        });
        
        Thread t2 = new Thread(() -> {
            int stamp = ref.getStamp();
            System.out.println("T2读取版本: " + stamp);
            
            // 修改为2
            ref.compareAndSet(1, 2, stamp, stamp + 1);
            stamp = ref.getStamp();
            System.out.println("T2修改为: 2, 版本: " + stamp);
            
            // 修改回1
            ref.compareAndSet(2, 1, stamp, stamp + 1);
            stamp = ref.getStamp();
            System.out.println("T2修改为: 1, 版本: " + stamp);
        });
        
        t1.start();
        t2.start();
        t1.join();
        t2.join();
    }
}
// 输出：
// T1读取: 1, 版本: 0
// T2读取版本: 0
// T2修改为: 2, 版本: 1
// T2修改为: 1, 版本: 2
// T1修改结果: false  ← 版本号已变化，修改失败
```

---

## 六、CAS的其他问题

### 6.1 CAS的缺点

| 问题 | 说明 | 解决方案 |
|------|------|---------|
| **ABA问题** | 值变化后又变回原值 | AtomicStampedReference |
| **自旋开销** | 竞争激烈时CPU空转 | 适应性自旋、锁 |
| **只能保证一个变量** | 无法同时CAS多个变量 | 锁、AtomicReference封装 |

### 6.2 CAS自旋优化

```java
// 自旋CAS示例
public class SpinLock {
    private AtomicReference<Thread> owner = new AtomicReference<>();
    
    public void lock() {
        Thread current = Thread.currentThread();
        while (!owner.compareAndSet(null, current)) {
            // 自旋等待
        }
    }
    
    public void unlock() {
        Thread current = Thread.currentThread();
        owner.compareAndSet(current, null);
    }
}
```

---

## 七、LeetCode题目解析

### 7.1 [1116. 打印零与奇偶数（Medium）](https://leetcode.cn/problems/print-zero-even-odd/)

**题目描述**：
三个线程分别打印0、奇数、偶数，要求按0,1,0,2,0,3...的顺序输出。

**思路分析**：
使用 `synchronized` + `volatile` 标志位控制执行顺序。

```java
class ZeroEvenOdd {
    private int n;
    private volatile int flag = 0;  // 0:zero, 1:odd, 2:even
    private final Object lock = new Object();
    
    public ZeroEvenOdd(int n) {
        this.n = n;
    }

    public void zero(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            synchronized (lock) {
                while (flag != 0) {
                    lock.wait();
                }
                printNumber.accept(0);
                if (i % 2 == 1) {
                    flag = 1;  // 下一个打印奇数
                } else {
                    flag = 2;  // 下一个打印偶数
                }
                lock.notifyAll();
            }
        }
    }

    public void even(IntConsumer printNumber) throws InterruptedException {
        for (int i = 2; i <= n; i += 2) {
            synchronized (lock) {
                while (flag != 2) {
                    lock.wait();
                }
                printNumber.accept(i);
                flag = 0;  // 下一个打印0
                lock.notifyAll();
            }
        }
    }

    public void odd(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i += 2) {
            synchronized (lock) {
                while (flag != 1) {
                    lock.wait();
                }
                printNumber.accept(i);
                flag = 0;  // 下一个打印0
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

## 八、今日学习要点总结

### 核心要点归纳

1. **volatile保证可见性**：通过内存屏障实现，不保证原子性
2. **内存屏障类型**：LoadLoad、StoreStore、LoadStore、StoreLoad
3. **MESI协议**：Modified、Exclusive、Shared、Invalid四种状态
4. **CAS原理**：比较并交换，无锁的原子操作
5. **ABA问题**：使用AtomicStampedReference解决

### 验收清单

- [ ] 能解释volatile保证可见性但不保证原子性的原因
- [ ] 能说明CAS的ABA问题及解决方案
- [ ] 能解释内存屏障的作用
- [ ] 能说出MESI协议的四种状态
- [ ] 能解释CAS的三个操作数

---

## 九、练习建议

1. **验证volatile可见性**：写代码验证volatile和非volatile的区别
2. **查看汇编**：使用JITWatch查看volatile生成的汇编指令
3. **Atomic类使用**：熟悉AtomicInteger、AtomicLong、AtomicReference的使用
4. **CAS实现锁**：尝试用CAS实现一个简单的自旋锁

---

## 十、扩展阅读

- 《Java并发编程的艺术》第1章
- 《深入理解Java虚拟机》第12章
- JSR-133内存模型规范
