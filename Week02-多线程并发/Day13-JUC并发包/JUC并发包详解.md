# Day13 详解 — JUC并发包

## 一、CountDownLatch倒计时器

### 1.1 CountDownLatch简介

CountDownLatch允许一个或多个线程**等待其他线程完成操作**后再继续执行。

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3);  // 计数为3
        
        for (int i = 0; i < 3; i++) {
            final int taskId = i;
            new Thread(() -> {
                try {
                    Thread.sleep(1000);
                    System.out.println("任务" + taskId + "完成");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    latch.countDown();  // 计数减1
                }
            }).start();
        }
        
        latch.await();  // 等待计数变为0
        System.out.println("所有任务完成");
    }
}
// 输出：
// 任务0完成
// 任务1完成
// 任务2完成
// 所有任务完成
```

### 1.2 CountDownLatch核心方法

| 方法 | 说明 |
|------|------|
| `countDown()` | 计数减1 |
| `await()` | 阻塞等待计数变为0 |
| `await(timeout, unit)` | 超时等待 |
| `getCount()` | 获取当前计数 |

### 1.3 使用场景

```java
// 场景1：等待多个线程完成
CountDownLatch latch = new CountDownLatch(5);
for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        try {
            // 执行任务
        } finally {
            latch.countDown();
        }
    }).start();
}
latch.await();  // 等待5个线程完成

// 场景2：子线程等待主线程
CountDownLatch startLatch = new CountDownLatch(1);
for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        startLatch.await();  // 等待开始信号
        // 执行任务
    }).start();
}
startLatch.countDown();  // 发送开始信号
```

---

## 二、CyclicBarrier循环栅栏

### 2.1 CyclicBarrier简介

CyclicBarrier让一组线程**到达一个屏障点时被阻塞**，直到最后一个线程到达屏障点，所有线程才会继续执行。

```java
import java.util.concurrent.CyclicBarrier;

public class CyclicBarrierDemo {
    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(3, () -> {
            System.out.println("所有线程到达屏障点");
        });
        
        for (int i = 0; i < 3; i++) {
            final int taskId = i;
            new Thread(() -> {
                try {
                    System.out.println("线程" + taskId + "到达屏障点");
                    barrier.await();  // 等待其他线程
                    System.out.println("线程" + taskId + "继续执行");
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
// 输出：
// 线程0到达屏障点
// 线程1到达屏障点
// 线程2到达屏障点
// 所有线程到达屏障点
// 线程0继续执行
// 线程1继续执行
// 线程2继续执行
```

### 2.2 CountDownLatch vs CyclicBarrier

| 对比项 | CountDownLatch | CyclicBarrier |
|--------|----------------|---------------|
| **可重用** | ❌ 一次性 | ✅ 可循环使用 |
| **触发条件** | 计数归零 | 所有线程到达屏障点 |
| **使用场景** | 等待多个事件完成 | 多线程同步到达 |
| **回调函数** | 无 | 可指定barrierAction |

### 2.3 CyclicBarrier重用

```java
CyclicBarrier barrier = new CyclicBarrier(2);

// 第一轮
new Thread(() -> {
    try {
        barrier.await();
        System.out.println("第一轮");
    } catch (Exception e) {
        e.printStackTrace();
    }
}).start();

new Thread(() -> {
    try {
        barrier.await();
        System.out.println("第一轮");
    } catch (Exception e) {
        e.printStackTrace();
    }
}).start();

// 第二轮（CyclicBarrier可重用）
new Thread(() -> {
    try {
        barrier.await();
        System.out.println("第二轮");
    } catch (Exception e) {
        e.printStackTrace();
    }
}).start();

new Thread(() -> {
    try {
        barrier.await();
        System.out.println("第二轮");
    } catch (Exception e) {
        e.printStackTrace();
    }
}).start();
```

---

## 三、Semaphore信号量

### 3.1 Semaphore简介

Semaphore用于**控制同时访问特定资源的线程数量**，常用于限流。

```java
import java.util.concurrent.Semaphore;

public class SemaphoreDemo {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(3);  // 同时最多3个线程
        
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            new Thread(() -> {
                try {
                    semaphore.acquire();  // 获取许可
                    System.out.println("线程" + taskId + "获取许可");
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();  // 释放许可
                    System.out.println("线程" + taskId + "释放许可");
                }
            }).start();
        }
    }
}
```

### 3.2 Semaphore核心方法

| 方法 | 说明 |
|------|------|
| `acquire()` | 获取一个许可 |
| `acquire(int permits)` | 获取多个许可 |
| `tryAcquire()` | 尝试获取许可（非阻塞） |
| `release()` | 释放一个许可 |
| `availablePermits()` | 获取可用许可数 |

### 3.3 使用场景：限流

```java
// 数据库连接池限流
public class DatabasePool {
    private final Semaphore semaphore = new Semaphore(10);  // 最多10个连接
    
    public Connection getConnection() throws InterruptedException {
        semaphore.acquire();  // 获取许可
        return createConnection();
    }
    
    public void releaseConnection(Connection conn) {
        closeConnection(conn);
        semaphore.release();  // 释放许可
    }
}

// 接口限流
public class RateLimiter {
    private final Semaphore semaphore = new Semaphore(100);  // QPS=100
    
    public void handleRequest() throws InterruptedException {
        semaphore.acquire();
        try {
            // 处理请求
        } finally {
            semaphore.release();
        }
    }
}
```

---

## 四、CompletableFuture异步编程

### 4.1 CompletableFuture简介

CompletableFuture是Java 8引入的异步编程工具，支持**链式调用**和**组合操作**。

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureDemo {
    public static void main(String[] args) {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            return "Hello";
        }).thenApply(result -> {
            return result + " World";
        }).thenApply(result -> {
            return result.toUpperCase();
        });
        
        System.out.println(future.join());  // 输出: HELLO WORLD
    }
}
```

### 4.2 创建CompletableFuture

```java
// 1. supplyAsync - 有返回值的异步任务
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});

// 2. runAsync - 无返回值的异步任务
CompletableFuture<Void> future2 = CompletableFuture.runAsync(() -> {
    System.out.println("异步任务");
});

// 3. completedFuture - 已完成的Future
CompletableFuture<String> future3 = CompletableFuture.completedFuture("Done");
```

### 4.3 thenApply vs thenCompose

```java
// thenApply - 转换结果（类似Stream的map）
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World");  // String -> String

// thenCompose - 扁平化（类似Stream的flatMap）
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + " World"));

// 区别：thenApply的函数返回普通值，thenCompose的函数返回CompletableFuture
```

### 4.4 组合操作

```java
// thenCombine - 合并两个结果
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> result = future1.thenCombine(future2, 
    (s1, s2) -> s1 + " " + s2);

// thenAcceptBoth - 消费两个结果
future1.thenAcceptBoth(future2, 
    (s1, s2) -> System.out.println(s1 + " " + s2));

// runAfterBoth - 两个都完成后执行
future1.runAfterBoth(future2, 
    () -> System.out.println("都完成了"));

// applyToEither - 任一完成后执行
future1.applyToEither(future2, 
    s -> s + " completed");

// acceptEither - 消费任一结果
future1.acceptEither(future2, 
    s -> System.out.println(s + " completed"));

// runAfterEither - 任一完成后执行
future1.runAfterEither(future2, 
    () -> System.out.println("有一个完成了"));
```

### 4.5 异常处理

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    if (true) throw new RuntimeException("异常");
    return "Hello";
}).exceptionally(ex -> {
    System.out.println("捕获异常: " + ex.getMessage());
    return "默认值";
}).handle((result, ex) -> {
    if (ex != null) {
        return "错误处理";
    }
    return result;
});
```

---

## 五、其他JUC工具类

### 5.1 Atomic系列

```java
import java.util.concurrent.atomic.*;

// 原子整数
AtomicInteger atomicInt = new AtomicInteger(0);
atomicInt.incrementAndGet();  // 原子+1
atomicInt.compareAndSet(1, 2);  // CAS操作

// 原子长整型
AtomicLong atomicLong = new AtomicLong(0);

// 原子引用
AtomicReference<String> atomicRef = new AtomicReference<>("Hello");

// 原子字段更新器
AtomicIntegerFieldUpdater<User> updater = 
    AtomicIntegerFieldUpdater.newUpdater(User.class, "age");
```

### 5.2 ConcurrentLinkedQueue

```java
import java.util.concurrent.ConcurrentLinkedQueue;

// 线程安全的队列
ConcurrentLinkedQueue<String> queue = new ConcurrentLinkedQueue<>();
queue.offer("Hello");
queue.poll();
queue.peek();
```

### 5.3 BlockingQueue

```java
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.SynchronousQueue;

// 阻塞队列
BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);
queue.put("Hello");  // 阻塞等待
queue.take();        // 阻塞等待

// 数组阻塞队列
BlockingQueue<String> arrayQueue = new ArrayBlockingQueue<>(10);

// 同步队列（不存储元素）
BlockingQueue<String> syncQueue = new SynchronousQueue<>();
```

---

## 六、LeetCode题目解析

### 6.1 [1195. Fizz Buzz Multithreaded（Medium）](https://leetcode.cn/problems/fizz-buzz-multithreaded/)

**题目描述**：
四个线程分别打印"fizz"、"buzz"、"fizzbuzz"、数字，按规则输出。

**思路分析**：
使用 `synchronized` + `volatile` 标志位控制执行顺序。

```java
class FizzBuzz {
    private int n;
    private volatile int flag = 1;  // 1: number, 2: fizz, 3: buzz, 4: fizzbuzz
    private final Object lock = new Object();

    public FizzBuzz(int n) {
        this.n = n;
    }

    // 打印fizz（能被3整除但不能被5整除）
    public void fizz(Runnable printFizz) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            synchronized (lock) {
                while (flag != 2) {
                    lock.wait();
                }
                if (i % 3 == 0 && i % 5 != 0) {
                    printFizz.run();
                }
                flag = nextFlag(i);
                lock.notifyAll();
            }
        }
    }

    // 打印buzz（能被5整除但不能被3整除）
    public void buzz(Runnable printBuzz) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            synchronized (lock) {
                while (flag != 3) {
                    lock.wait();
                }
                if (i % 5 == 0 && i % 3 != 0) {
                    printBuzz.run();
                }
                flag = nextFlag(i);
                lock.notifyAll();
            }
        }
    }

    // 打印fizzbuzz（能被15整除）
    public void fizzbuzz(Runnable printFizzBuzz) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            synchronized (lock) {
                while (flag != 4) {
                    lock.wait();
                }
                if (i % 15 == 0) {
                    printFizzBuzz.run();
                }
                flag = nextFlag(i);
                lock.notifyAll();
            }
        }
    }

    // 打印数字
    public void number(IntConsumer printNumber) throws InterruptedException {
        for (int i = 1; i <= n; i++) {
            synchronized (lock) {
                while (flag != 1) {
                    lock.wait();
                }
                if (i % 3 != 0 && i % 5 != 0) {
                    printNumber.accept(i);
                }
                flag = nextFlag(i);
                lock.notifyAll();
            }
        }
    }

    private int nextFlag(int i) {
        if (i % 15 == 0) return 4;
        if (i % 5 == 0) return 3;
        if (i % 3 == 0) return 2;
        return 1;
    }
}
```

**复杂度分析**：
- 时间复杂度：O(n)
- 空间复杂度：O(1)

---

## 七、今日学习要点总结

### 核心要点归纳

1. **CountDownLatch**：等待多个线程完成，一次性使用
2. **CyclicBarrier**：所有线程到达屏障点，可循环使用
3. **Semaphore**：控制并发访问数量，常用于限流
4. **CompletableFuture**：异步编程，支持链式调用和组合操作
5. **thenApply vs thenCompose**：thenApply转换值，thenCompose扁平化

### 验收清单

- [ ] 能对比CountDownLatch和CyclicBarrier的区别
- [ ] 能说明CompletableFuture的thenApply和thenCompose区别
- [ ] 能解释Semaphore的使用场景（限流）
- [ ] 能使用CompletableFuture进行异步编程
- [ ] 能说出各种JUC工具类的适用场景

---

## 八、练习建议

1. **CountDownLatch实战**：实现等待多个服务启动完成
2. **CyclicBarrier实战**：实现多线程分批执行
3. **Semaphore实战**：实现数据库连接池限流
4. **CompletableFuture实战**：实现异步编排，调用多个服务

---

## 九、扩展阅读

- 《Java并发编程的艺术》第7章
- 《Java并发编程实战》第5章
- CompletableFuture官方文档
