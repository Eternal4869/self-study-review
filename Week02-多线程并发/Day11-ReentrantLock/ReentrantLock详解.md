# Day11 详解 — ReentrantLock

## 一、ReentrantLock基础

### 1.1 ReentrantLock简介

ReentrantLock是Java并发包（java.util.concurrent）中的**可重入互斥锁**，功能比synchronized更强大。

```java
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    
    public void method() {
        lock.lock();  // 获取锁
        try {
            // 临界区代码
        } finally {
            lock.unlock();  // 必须在finally中释放锁
        }
    }
}
```

### 1.2 synchronized vs ReentrantLock

| 对比项 | synchronized | ReentrantLock |
|--------|-------------|---------------|
| **实现层面** | JVM内置 | Java API |
| **锁获取** | 自动获取释放 | 手动获取释放 |
| **可中断** | ❌ | ✅ `lockInterruptibly()` |
| **超时获取** | ❌ | ✅ `tryLock(timeout)` |
| **公平性** | 非公平 | 可选公平/非公平 |
| **条件变量** | 单一（wait/notify） | 多个Condition |
| **绑定条件** | 对象监视器 | Condition对象 |

---

## 二、公平锁与非公平锁

### 2.1 公平锁

```java
// 公平锁：按照线程请求顺序获取锁
ReentrantLock fairLock = new ReentrantLock(true);  // true表示公平锁

// 非公平锁（默认）
ReentrantLock unfairLock = new ReentrantLock(false);  // 或无参构造
```

### 2.2 公平锁vs非公平锁

| 对比项 | 公平锁 | 非公平锁 |
|--------|--------|----------|
| **顺序** | 按请求顺序 | 抢占式 |
| **性能** | 较低 | 较高 |
| **适用场景** | 公平性要求高 | 吞吐量要求高 |

### 2.3 非公平锁为什么性能更好？

```
非公平锁:
┌─────────────────────────────────────────┐
│ 1. 线程A释放锁                           │
│ 2. 线程C尝试获取锁（直接获取成功）        │
│ 3. 线程B还在唤醒中                       │
└─────────────────────────────────────────┘
省去了线程唤醒的开销
```

### 2.4 源码分析

```java
// ReentrantLock源码（简化版）
public class ReentrantLock {
    private final Sync sync;
    
    // 非公平锁实现
    static final class NonfairSync extends Sync {
        protected final boolean initialTryLock() {
            // 先尝试CAS获取锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(current);
            else if (current == getExclusiveOwnerThread())
                // 可重入
                setState(state + 1);
            else
                // 失败，加入队列
                enqueue(current);
            return true;
        }
    }
    
    // 公平锁实现
    static final class FairSync extends Sync {
        protected final boolean initialTryLock() {
            if (!hasQueuedPredecessors()) {
                // 检查队列中是否有等待线程
                if (compareAndSetState(0, 1))
                    setExclusiveOwnerThread(current);
                else if (current == getExclusiveOwnerThread())
                    setState(state + 1);
                else
                    enqueue(current);
                return true;
            }
            return false;
        }
    }
}
```

---

## 三、可中断锁

### 3.1 lockInterruptibly()

```java
public class InterruptibleLockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    
    public void method() throws InterruptedException {
        lock.lockInterruptibly();  // 可中断地获取锁
        try {
            // 临界区代码
        } finally {
            lock.unlock();
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        InterruptibleLockDemo demo = new InterruptibleLockDemo();
        
        Thread t1 = new Thread(() -> {
            demo.method();
            System.out.println("t1执行完毕");
        });
        
        Thread t2 = new Thread(() -> {
            demo.method();
            System.out.println("t2执行完毕");
        });
        
        t1.start();
        Thread.sleep(100);
        t2.start();  // t2等待锁
        
        Thread.sleep(200);
        t2.interrupt();  // 中断t2
        t2.join();
    }
}
// 输出：
// t1执行完毕
// t2执行完毕（或抛出InterruptedException）
```

### 3.2 tryLock()无超时获取

```java
public class TryLockDemo {
    private final ReentrantLock lock = new ReentrantLock();
    
    public void method() {
        if (lock.tryLock()) {  // 尝试获取锁，不阻塞
            try {
                System.out.println(Thread.currentThread().getName() + ": 获取锁");
                // 临界区代码
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println(Thread.currentThread().getName() + ": 未获取锁");
        }
    }
}
```

### 3.3 tryLock(timeout)超时获取

```java
public class TryLockTimeoutDemo {
    private final ReentrantLock lock = new ReentrantLock();
    
    public void method() throws InterruptedException {
        if (lock.tryLock(3, TimeUnit.SECONDS)) {  // 等待3秒
            try {
                System.out.println(Thread.currentThread().getName() + ": 获取锁");
                // 临界区代码
            } finally {
                lock.unlock();
            }
        } else {
            System.out.println(Thread.currentThread().getName() + ": 获取锁超时");
        }
    }
}
```

---

## 四、Condition条件变量

### 4.1 Condition简介

Condition提供了比wait/notify更强大的线程间通信机制，支持**多个条件队列**。

```java
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ConditionDemo {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();   // 非满条件
    private final Condition notEmpty = lock.newCondition();  // 非空条件
    
    public void producer() throws InterruptedException {
        lock.lock();
        try {
            // 生产者等待条件
            while (isFull()) {
                notFull.await();  // 等待notFull条件
            }
            produce();
            notEmpty.signalAll();  // 通知消费者
        } finally {
            lock.unlock();
        }
    }
    
    public void consumer() throws InterruptedException {
        lock.lock();
        try {
            // 消费者等待条件
            while (isEmpty()) {
                notEmpty.await();  // 等待notEmpty条件
            }
            consume();
            notFull.signalAll();  // 通知生产者
        } finally {
            lock.unlock();
        }
    }
}
```

### 4.2 Condition vs wait/notify

| 对比项 | Condition | wait/notify |
|--------|-----------|-------------|
| **条件数量** | 多个 | 单一 |
| **实现方式** | Lock + Condition | synchronized |
| **精确唤醒** | ✅ | ❌ |
| **使用场景** | 生产者消费者模型 | 简单同步 |

### 4.3 精确唤醒示例

```java
public class ConditionExactDemo {
    private int flag = 0;  // 0: A执行, 1: B执行, 2: C执行
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condA = lock.newCondition();
    private final Condition condB = lock.newCondition();
    private final Condition condC = lock.newCondition();
    
    public void printA() throws InterruptedException {
        lock.lock();
        try {
            while (flag != 0) {
                condA.await();
            }
            System.out.println("A");
            flag = 1;
            condB.signalAll();  // 精确唤醒B
        } finally {
            lock.unlock();
        }
    }
    
    public void printB() throws InterruptedException {
        lock.lock();
        try {
            while (flag != 1) {
                condB.await();
            }
            System.out.println("B");
            flag = 2;
            condC.signalAll();  // 精确唤醒C
        } finally {
            lock.unlock();
        }
    }
    
    public void printC() throws InterruptedException {
        lock.lock();
        try {
            while (flag != 2) {
                condC.await();
            }
            System.out.println("C");
            flag = 0;
            condA.signalAll();  // 精确唤醒A
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 五、读写锁ReadWriteLock

### 5.1 ReadWriteLock简介

ReadWriteLock将锁分为**读锁**和**写锁**，允许多个读线程同时访问，但写线程是独占的。

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockDemo {
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();
    private int count = 0;
    
    public int read() {
        readLock.lock();  // 获取读锁
        try {
            return count;
        } finally {
            readLock.unlock();
        }
    }
    
    public void write(int value) {
        writeLock.lock();  // 获取写锁
        try {
            count = value;
        } finally {
            writeLock.unlock();
        }
    }
}
```

### 5.2 读写锁规则

| 操作 | 读锁 | 写锁 |
|------|------|------|
| **读锁** | ✅ 共享 | ❌ 阻塞 |
| **写锁** | ❌ 阻塞 | ❌ 阻塞 |

### 5.3 读写锁适用场景

```java
// 场景：缓存系统
// 读多写少：使用读写锁提高并发性能

public class Cache<K, V> {
    private final Map<K, V> map = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    
    public V get(K key) {
        rwLock.readLock().lock();
        try {
            return map.get(key);
        } finally {
            rwLock.readLock().unlock();
        }
    }
    
    public void put(K key, V value) {
        rwLock.writeLock().lock();
        try {
            map.put(key, value);
        } finally {
            rwLock.writeLock().unlock();
        }
    }
}
```

---

## 六、Condition实现生产者消费者

### 6.1 经典生产者消费者模式

```java
public class ProducerConsumer {
    private final int[] buffer = new int[10];
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public void produce(int item) throws InterruptedException {
        lock.lock();
        try {
            // 缓冲区满，等待
            while (count == buffer.length) {
                notFull.await();
            }
            // 生产
            buffer[count++] = item;
            System.out.println("生产: " + item + ", 数量: " + count);
            // 通知消费者
            notEmpty.signalAll();
        } finally {
            lock.unlock();
        }
    }
    
    public int consume() throws InterruptedException {
        lock.lock();
        try {
            // 缓冲区空，等待
            while (count == 0) {
                notEmpty.await();
            }
            // 消费
            int item = buffer[--count];
            System.out.println("消费: " + item + ", 数量: " + count);
            // 通知生产者
            notFull.signalAll();
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## 七、LeetCode题目解析

### 7.1 [1117. Building H2O（Medium）](https://leetcode.cn/problems/building-h2o/)

**题目描述**：
设计程序让线程按顺序产生H和O，最终生成H2O。需要2个H线程和1个O线程配合。

**思路分析**：
使用 `synchronized` + `CountDownLatch` 或 `ReentrantLock` + `Condition`。

```java
class H2O {
    private int hCount = 0;
    private int oCount = 0;
    private final Object lock = new Object();

    public void hydrogen(Runnable releaseHydrogen) throws InterruptedException {
        synchronized (lock) {
            while (hCount >= 2) {
                lock.wait();
            }
            hCount++;
            releaseHydrogen.run();
            if (hCount == 2 && oCount == 1) {
                hCount = 0;
                oCount = 0;
            }
            lock.notifyAll();
        }
    }

    public void oxygen(Runnable releaseOxygen) throws InterruptedException {
        synchronized (lock) {
            while (oCount >= 1) {
                lock.wait();
            }
            oCount++;
            releaseOxygen.run();
            if (hCount == 2 && oCount == 1) {
                hCount = 0;
                oCount = 0;
            }
            lock.notifyAll();
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

1. **ReentrantLock功能强大**：支持可中断、超时、公平锁、多个Condition
2. **公平锁vs非公平锁**：公平锁按顺序获取，非公平锁性能更好
3. **Condition精确唤醒**：支持多个条件队列，可精确唤醒特定线程
4. **ReadWriteLock读写分离**：读锁共享，写锁独占，适合读多写少场景
5. **必须在finally中释放锁**：防止死锁

### 验收清单

- [ ] 能对比synchronized和ReentrantLock的区别
- [ ] 能说明公平锁和非公平锁的性能差异
- [ ] 能解释Condition和wait/notify的区别
- [ ] 能说出ReadWriteLock的适用场景
- [ ] 能使用ReentrantLock实现生产者消费者模式

---

## 九、练习建议

1. **源码阅读**：阅读ReentrantLock源码，理解AQS原理
2. **实现锁**：尝试用AQS实现一个简单的锁
3. **生产者消费者**：使用Lock+Condition实现生产者消费者
4. **读写锁实践**：实现一个线程安全的缓存

---

## 十、扩展阅读

- 《Java并发编程的艺术》第5章
- AbstractQueuedSynchronizer(AQS)原理
- JDK源码：ReentrantLock、Condition
