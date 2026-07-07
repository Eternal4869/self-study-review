# Day03：ConcurrentHashMap原理详解

## 今日目标
- 深入理解JDK 7 ConcurrentHashMap的Segment分段锁机制
- 完整掌握JDK 8 ConcurrentHashMap的CAS + synchronized实现
- 理解size()的baseCount + CounterCell设计思想
- 掌握get()为什么不需要加锁的原理
- 了解JDK 11+对ConcurrentHashMap的改进
- 能够回答面试中的ConcurrentHashMap相关问题

---

## 一、ConcurrentHashMap概述

### 1.1 为什么需要ConcurrentHashMap

HashMap不是线程安全的，并发put会导致数据丢失。Hashtable虽然线程安全，但使用方法级synchronized，所有操作互斥，效率极低。ConcurrentHashMap应运而生——**分段加锁**，不同段的操作可以并行。

| 特性 | HashMap | Hashtable | ConcurrentHashMap |
|------|---------|-----------|-------------------|
| 线程安全 | 否 | 是（全表锁） | 是（分段锁/细粒度锁） |
| null键值 | 允许 | 不允许 | 不允许 |
| 性能 | 最好（单线程） | 最差 | 好（并发场景） |
| 并发度 | 1 | 1 | JDK7: 16段 / JDK8: 桶级 |

### 1.2 数据结构演变

```
JDK 7：Segment[] + HashEntry[]
         │
         └─ 每个Segment继承ReentrantLock，内部维护一个HashEntry数组

JDK 8：Node[] + 链表/红黑树（与HashMap结构一致）
         │
         └─ 放弃分段锁，改用CAS + synchronized（桶级别锁）
```

### 1.3 核心属性（JDK 8）

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
        implements ConcurrentMap<K,V>, Serializable {
    
    // 存储数据的Node数组
    transient volatile Node<K,V>[] table;
    
    // 扩容时的辅助数组
    transient volatile Node<K,V>[] nextTable;
    
    // 基础计数器（CAS更新）
    transient volatile long baseCount;
    
    // CounterCell数组（用于高并发计数）
    transient volatile CounterCell[] counterCells;
    
    // 控制标志：-1:正在初始化, -N:正在扩容, 其他:扩容阈值
    private transient volatile int sizeCtl;
    
    // 默认初始容量
    static final int DEFAULT_CAPACITY = 16;
    
    // 链表转红黑树的阈值
    static final int TREEIFY_THRESHOLD = 8;
    
    // 红黑树转链表的阈值
    static final int UNTREEIFY_THRESHOLD = 6;
    
    // 链表转红黑树的最小数组容量
    static final int MIN_TREEIFY_CAPACITY = 64;
}
```

**面试问题：sizeCtl字段的作用？**
> 答：sizeCtl是ConcurrentHashMap的控制标志。`sizeCtl = -1`表示正在初始化；`sizeCtl = -N`表示有N-1个线程正在协助扩容；正数表示扩容阈值（容量 × 负载因子）。

---

## 二、JDK 7 ConcurrentHashMap（Segment分段锁）

### 2.1 核心数据结构

```java
public class ConcurrentHashMap<K, V> extends AbstractMap<K, V>
        implements ConcurrentMap<K, V>, Serializable {
    
    // Segment数组（每个Segment是一个ReentrantLock）
    final Segment<K,V>[] segments;
    
    // 默认并发度（Segment数量）
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    
    // 每个Segment内部的HashEntry数组
    static final class Segment<K,V> extends ReentrantLock {
        transient volatile HashEntry<K,V>[] table;
        transient int count;      // Segment内元素数量
        transient int modCount;   // 修改次数
        transient int threshold;  // 扩容阈值
        final float loadFactor;   // 负载因子
    }
    
    // HashEntry（链表节点）
    static final class HashEntry<K,V> {
        final int hash;
        final K key;
        volatile V value;
        volatile HashEntry<K,V> next;
    }
}
```

**关键设计：**
- `Segment` 继承 `ReentrantLock`，每个Segment就是一个锁
- 不同Segment上的操作互不影响，可以并行
- 默认16个Segment，最多支持16个线程并发写

### 2.2 put()流程

```java
public V put(K key, V value) {
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    
    // 1. 计算key的hash（经过两次散列）
    int hash = hash(key);
    
    // 2. 根据hash的高几位确定Segment位置
    //    segmentShift=28, segmentMask=15
    //    用hash的高4位定位Segment
    int j = (hash >>> segmentShift) & segmentMask;
    if ((s = (Segment<K,V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null)
        s = ensureSegment(j);  // 延迟初始化
    
    // 3. 在该Segment上执行put
    return s.put(key, hash, value, false);
}

// Segment.put() —— 本质是ReentrantLock + 数组链表操作
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    Lock lock = this.lock();
    lock.lock();  // 加锁（锁的是这个Segment）
    try {
        int c = count;
        if (c++ > threshold)  // 超过阈值则扩容
            rehash();
        
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        
        // 遍历链表
        while (e != null && e.hash != hash && e.key != null)
            e = e.next;
        
        if (e != null) {
            // 找到相同key，覆盖value
            V old = e.value;
            if (!onlyIfAbsent)
                e.value = value;
            return old;
        } else {
            // 在链表头部插入新节点（头插法）
            HashEntry<K,V> newEntry = new HashEntry<>(hash, key, value, first);
            tab[index] = newEntry;
            count = c;  // 写count前不需要加锁，因为已经持有Segment锁
            return null;
        }
    } finally {
        lock.unlock();
    }
}
```

### 2.3 get()流程

```java
public V get(Object key) {
    Segment<K,V> s;
    HashEntry<K,V>[] tab;
    int h = hash(key);
    
    // 定位Segment
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        
        // 遍历链表（HashEntry的value和next是volatile的）
        for (HashEntry<K,V> e = (HashEntry<K,V>)UNSAFE.getObjectVolatile(
                 tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

**get()为什么不需要加锁？**
- HashEntry的`value`和`next`都是`volatile`修饰的
- `volatile`保证了可见性——写线程的修改对读线程立即可见
- 读操作不修改数据，所以不需要加锁
- 但注意：如果get()时正在进行扩容，可能看到部分更新的状态

### 2.4 size()流程

```java
public int size() {
    // 先尝试无锁计算（不精确但很快）
    long sum = 0;
    long check = 0;
    Segment<K,V>[] segments = segments;
    for (int i = 0; i < segments.length; ++i) {
        sum += segments[i].count;
        check += segments[i].modCount;
    }
    
    // 如果有修改，尝试加锁重新计算（最多3次）
    if (check != 0) {
        sum = 0;
        for (int i = 0; i < segments.length; ++i) {
            segments[i].lock();
            sum += segments[i].count;
        }
        for (int i = 0; i < segments.length; ++i) {
            segments[i].unlock();
        }
    }
    
    // 溢出处理
    return (sum < Integer.MAX_VALUE) ? (int)sum : Integer.MAX_VALUE;
}
```

### 2.5 JDK 7 的局限性

1. **并发度受限**：默认只有16个Segment，最多16个线程并行写
2. **Segment不能动态扩容**：Segment数组一旦创建就不能增加
3. **size()代价大**：需要遍历所有Segment，甚至需要加锁
4. **内存占用大**：每个Segment都是一个独立的锁对象

---

## 三、JDK 8 ConcurrentHashMap（CAS + synchronized）

### 3.1 核心数据结构

```java
// 节点定义（与HashMap一致）
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;
    volatile Node<K,V> next;
}

// 红黑树节点
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;
    boolean red;
}

// 转发节点（扩容时使用）
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
}
```

**关键变化：**
- 放弃Segment，使用Node数组（与HashMap一致）
- 链表节点的`val`和`next`都是`volatile`
- 新增ForwardingNode，用于扩容时标记已迁移的桶

### 3.2 put()流程（核心）

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    
    int hash = spread(key.hashCode());  // 扰动函数
    int binCount = 0;
    
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        
        // 1. table为空 → 初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        
        // 2. 桶位置为空 → CAS直接插入（无锁）
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;  // CAS成功，跳出循环
        }
        
        // 3. 正在扩容 → 帮助迁移
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        
        // 4. 桶不为空 → synchronized锁住头节点
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {  // 确认头节点未被修改
                    if (fh >= 0) {
                        // 链表操作
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash && 
                                ((ek = e.key) == key || (key != null && key.equals(ek)))) {
                                // 找到相同key，覆盖value
                                V oldVal = e.val;
                                if (!onlyIfAbsent || oldVal == null)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                // 尾插法插入新节点
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeNode) {
                        // 红黑树操作
                        TreeNode<K,V> p = ((TreeNode<K,V>)f).putTreeVal(tab, hash, key, value);
                        if (p != null) {
                            V oldVal = p.val;
                            if (!onlyIfAbsent || oldVal == null)
                                p.val = value;
                        }
                    }
                }
            }
            
            // 5. 链表长度 >= 8 → 转红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, hash);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    
    // 6. 增加计数
    addCount(1L, binCount);
    return null;
}
```

### 3.3 put流程图解

```
put(key, value)
    │
    ▼
spread(hash) 扰动函数
    │
    ▼
table 为空？ ──是──→ initTable() 初始化（CAS控制）
    │否
    ▼
桶位置为空？ ──是──→ CAS直接插入（无锁！）
    │否
    ▼
是ForwardingNode？ ──是──→ helpTransfer() 帮助迁移
    │否
    ▼
synchronized锁住桶头节点
    │
    ├── 是链表 → 遍历链表（覆盖/尾插）
    └── 是红黑树 → putTreeVal()
    │
    ▼
链表长度 >= 8？ ──是──→ treeifyBin()（需数组长度 >= 64）
    │否
    ▼
addCount() 增加计数
```

### 3.4 CAS操作详解

```java
// 获取桶位置的值（volatile读）
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>) U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}

// CAS设置桶位置的值
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) {
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

// 设置桶位置的值（volatile写）
static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
    U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
}
```

**CAS用于桶为空的情况**：如果桶为空，直接CAS插入，不需要加锁。这是JDK 8的重大优化——无锁插入。

### 3.5 initTable() 初始化

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // sizeCtl < 0 表示正在初始化或扩容
        if ((sc = sizeCtl) < 0)
            Thread.yield();  // 让出CPU，等待初始化完成
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // CAS将sizeCtl设为-1，表示正在初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] tab = (Node<K,V>[])new Node[n];
                    table = tab;
                    sc = n - (n >>> 2);  // 0.75n
                }
            } finally {
                sizeCtl = sc;  // 恢复sizeCtl为阈值
            }
            break;
        }
    }
    return tab;
}
```

**设计要点：**
- 使用CAS确保只有一个线程执行初始化
- 其他线程通过`Thread.yield()`让出CPU
- 初始化完成后恢复sizeCtl为扩容阈值

---

## 四、扩容机制详解

### 4.1 扩容触发条件

```java
// addCount()中检查是否需要扩容
private final void addCount(long x, int check) {
    // ... 更新计数 ...
    
    if (check >= 0) {
        Node<K,V>[] tab, newTab, nextTab;
        int sc;
        while (s > (long)(sc = sizeCtl) && (tab = table) != null && 
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                // 正在扩容
                if ((nextTab = nextTable) == null)
                    break;
                int transfer;
                if ((transfer & (1 << RESIZE_STAMP_SHIFT)) == 0)
                    break;
                // ...
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc, rs + 2))
                // 触发扩容
                transfer(tab, null);
        }
    }
}
```

### 4.2 transfer() 扩容流程

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    
    // 1. 计算每个线程负责的桶数量（最少16个）
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE;
    
    // 2. 创建新数组
    if (nextTab == null) {
        try {
            nextTab = (Node<K,V>[])new Node[n << 1];
        } catch (Throwable ex) {
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    
    // 3. 每个线程负责一段桶的迁移
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false;
    
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        
        // 4. 从右向左逐个桶迁移
        while (advance) {
            if (--i < bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0)
                finishing = true;
            else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex, 
                    nextBound = (nextIndex > stride ? nextIndex - stride : 0)))
                bound = nextBound;
        }
        
        // 5. 迁移当前桶的数据
        if (i < 0 || i >= n || i + n >= nextn)
            finishing = true;
        
        Node<K,V> f;
        if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);  // 放置ForwardingNode
        else if ((fh = f.hash) == MOVED)
            advance = true;  // 已经迁移过
        else {
            synchronized (f) {
                // 链表拆分（高低位链）
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        // 链表：用hash & n拆分
                        int runBit = f.hash & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        } else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash & n;
                            Node<K,V> pnext = p.next;
                            if ((ph & n) == 0)
                                p.next = ln;
                            else
                                p.next = hn;
                            // ...
                        }
                        // 放入新数组
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    else if (f instanceof TreeNode) {
                        // 红黑树拆分
                        // ...
                    }
                }
            }
        }
    }
}
```

### 4.3 扩容的关键设计

1. **多线程协作**：每个线程负责一段桶的迁移（最少16个）
2. **ForwardingNode**：已迁移的桶放置ForwardingNode，表示数据已转移
3. **高低位链**：与HashMap类似，用 `hash & oldCap` 判断新位置
4. **平滑迁移**：迁移过程中新旧数组并存，put/get可以继续进行
5. **CAS控制**：通过CAS原子性地获取迁移区间

### 4.4 helpTransfer() 帮助扩容

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTable;
    if (tab == null || (nextTable = ((ForwardingNode<K,V>)f).nextTable) == null)
        return tab;
    
    int rs = resizeStamp(tab.length);
    while (nextTable == nextTable && tab == table && sizeCtl < 0) {
        if (syncTransfer || ... == TRANSFER_TRANSFER) {
            // 参与数据迁移
            Node<K,V> next;
            while ((next = transfer.next) != null) {
                // 迁移一个桶
                // ...
            }
        }
    }
    return nextTable;
}
```

**并发扩容**：当多个线程同时put发现正在扩容时，会帮助迁移数据，提高扩容速度。

---

## 五、size() 计数原理

### 5.1 为什么不能简单用一个变量计数

多线程并发修改同一个计数器，即使用volatile修饰也会丢失更新。加锁又太重。

### 5.2 baseCount + CounterCell方案

```java
// 基础计数器
transient volatile long baseCount;

// CounterCell数组
transient volatile CounterCell[] counterCells;

// CounterCell定义
@sun.misc.Contended  // 防止伪共享
static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}
```

**设计思想：**
1. **低竞争时**：CAS更新baseCount，无锁开销
2. **高竞争时**：CAS失败 → 分散到CounterCell数组中，每个线程更新不同的cell
3. **读取时**：sum = baseCount + 所有CounterCell.value

### 5.3 addCount() 流程

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    
    // 1. 如果counterCells不为空或CAS baseCount失败
    if ((as = counterCells) != null || 
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        
        CounterCell a; long v; int m;
        boolean uncontended = true;
        
        // 2. 随机选取一个CounterCell
        int index = (h = ThreadLocalRandom.getProbe()) & (m = as.length - 1);
        
        // 3. CAS更新该cell（失败也没关系，最终会sumAll）
        if (as == null || (m < 0) || 
            (a = as[index]) == null || 
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            
            fullAddCount(x, uncontended);  // 创建或扩容CounterCell
            return;
        }
        
        if (check <= 1)
            return;
        
        // 4. 计算总和
        s = sumCount();
    }
    
    // 5. 检查是否需要扩容
    if (check >= 0) {
        Node<K,V>[] tab, newTab, nextTab;
        int sc;
        while (s > (long)(sc = sizeCtl) && (tab = table) != null && 
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                // 正在扩容
                if ((nextTab = nextTable) == null)
                    break;
                int transfer;
                if ((transfer & (1 << RESIZE_STAMP_SHIFT)) == 0)
                    break;
                // ...
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc, rs + 2))
                // 触发扩容
                transfer(tab, null);
        }
    }
}

// 求和
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

### 5.4 @Contended注解的作用

- 防止**伪共享（False Sharing）**
- CPU缓存行通常64字节，多个CounterCell可能在同一缓存行
- 当不同线程修改同一缓存行的不同变量时，会导致缓存失效
- `@Contended`注解会让JVM在字段前后填充字节，确保每个CounterCell独占缓存行

**面试问题：什么是伪共享？**
> 答：CPU缓存以缓存行为单位（通常64字节），如果两个线程频繁修改同一缓存行中的不同变量，会导致整个缓存行频繁失效，性能急剧下降。这就是伪共享。解决方案是使用`@Contended`注解填充字节，使每个变量独占缓存行。

---

## 六、get() 原理

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    
    if ((tab = table) != null && (n = tab.length) > 0 && 
        (e = tabAt(tab, (n - 1) & h)) != null) {
        
        if ((eh = e.hash) == h) {
            // 第一个节点就是目标key
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            // 正在扩容，调用ForwardingNode.find()
            return (p = e.find(h, key)) != null ? p.val : null;
        
        // 遍历链表
        while ((e = e.next) != null) {
            if (e.hash == h && 
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

**get()为什么不需要加锁？**
1. `Node.val`和`Node.next`都是`volatile`修饰
2. `volatile`保证了**可见性**：写线程的修改立即对读线程可见
3. `tabAt()`使用`getObjectVolatile`读取，保证看到最新值
4. 读操作不修改数据，所以不需要加锁
5. **ForwardingNode.find()**：扩容时通过find()在新数组中查找

**面试问题：get()时如果正在扩容怎么办？**
> 答：如果get()时遇到ForwardingNode（hash = MOVED），会调用ForwardingNode.find()方法在新数组中继续查找。ForwardingNode的nextTable指向新数组，所以可以无缝切换。

---

## 七、JDK 7 vs JDK 8 全面对比

| 特性 | JDK 7 | JDK 8 |
|------|-------|-------|
| 数据结构 | Segment + HashEntry | Node数组 + 链表/红黑树 |
| 锁机制 | Segment（ReentrantLock） | CAS + synchronized（桶级） |
| 并发度 | 默认16（Segment数量） | 等于桶数量（可动态扩展） |
| get() | 不加锁（volatile） | 不加锁（volatile） |
| put() | 加Segment锁 | CAS（空桶）/ synchronized（非空桶） |
| size() | 遍历所有Segment | baseCount + CounterCell |
| 扩容 | 单线程 | 多线程协助 |
| null键值 | 不允许 | 不允许 |
| 内存效率 | 较低（Segment开销） | 较高 |
| 适用场景 | 中等并发 | 高并发 |

**面试问题：为什么JDK 8放弃分段锁？**
> **答：** 分段锁有三个问题：①并发度受限（默认只有16个Segment）；②Segment不能动态扩容；③size()需要遍历所有Segment。JDK 8改为CAS + synchronized（桶级锁），锁粒度更细，并发度等于桶数量。

---

## 八、面试常见问题

### Q1：ConcurrentHashMap底层数据结构？
> **答：** JDK 8中ConcurrentHashMap底层是**Node数组 + 链表 + 红黑树**，与HashMap结构一致。不同之处在于线程安全机制：使用CAS处理空桶插入，使用synchronized锁住桶头节点处理非空桶。

### Q2：ConcurrentHashMap的get()为什么不需要加锁？
> **答：** 因为Node的val和next都用volatile修饰，volatile保证了写操作对读操作的可见性。读线程可以立即看到写线程的修改，不需要加锁。

### Q3：ConcurrentHashMap的size()怎么实现的？
> **答：** JDK 8采用baseCount + CounterCell方案。低竞争时CAS更新baseCount；高竞争时CAS失败会创建CounterCell数组，每个线程分散到不同的cell更新。最终sum = baseCount + 所有CounterCell.value。使用@Contended注解防止伪共享。

### Q4：ConcurrentHashMap在JDK 8中如何扩容？
> **答：** JDK 8支持多线程协助扩容。每个线程负责一段桶（最少16个），通过ForwardingNode标记已迁移的桶。扩容过程中新旧数组并存，put/get可以继续进行。多个线程同时发现扩容时会协作迁移，提高扩容速度。

### Q5：ConcurrentHashMap为什么不允许null键值？
> **答：** 因为ConcurrentHashMap的get()如果返回null，无法区分"key不存在"还是"value为null"。在多线程环境下，如果允许null值，get()返回null时无法确定是否需要put，这会导致歧义。而HashMap单线程环境下可以通过containsKey()区分。

### Q6：synchronized锁住桶头节点，为什么性能还这么好？
> **答：** 因为锁的粒度非常细——只锁住单个桶的头节点。不同桶的操作完全并行。而且JDK 6+的synchronized已经做了大量优化（偏向锁、轻量级锁、自旋锁等），单对象锁的性能与ReentrantLock相当甚至更好。

### Q7：什么是伪共享？如何解决？
> **答：** CPU缓存以缓存行为单位（通常64字节），如果两个线程频繁修改同一缓存行中的不同变量，会导致整个缓存行频繁失效，性能急剧下降。解决方案是使用`@Contended`注解填充字节，使每个变量独占缓存行。ConcurrentHashMap的CounterCell就使用了这个注解。

---

## 九、代码示例

### 9.1 ConcurrentHashMap基本使用

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class ConcurrentHashMapDemo {
    public static void main(String[] args) {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        
        // 添加元素
        map.put("Java", 1);
        map.put("Python", 2);
        map.put("C++", 3);
        
        // 获取元素（不需要加锁）
        Integer value = map.get("Java");
        System.out.println(value);  // 1
        
        // putIfAbsent：不存在才放入
        map.putIfAbsent("Java", 100);  // Java已存在，不会覆盖
        System.out.println(map.get("Java"));  // 仍是1
        
        // compute：重新计算值
        map.compute("Java", (k, v) -> v + 10);
        System.out.println(map.get("Java"));  // 11
        
        // getOrDefault：不存在时返回默认值
        int go = map.getOrDefault("Go", 0);
        System.out.println(go);  // 0
        
        // merge：合并值
        map.merge("Java", 10, Integer::sum);
        System.out.println(map.get("Java"));  // 21
        
        // 遍历
        map.forEach((k, v) -> System.out.println(k + " = " + v));
    }
}
```

### 9.2 并发安全的计数器

```java
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class ConcurrentCounter {
    private final ConcurrentHashMap<String, AtomicLong> counterMap = new ConcurrentHashMap<>();
    
    public void increment(String key) {
        counterMap.computeIfAbsent(key, k -> new AtomicLong(0)).incrementAndGet();
    }
    
    public long getCount(String key) {
        AtomicLong counter = counterMap.get(key);
        return counter != null ? counter.get() : 0;
    }
    
    public static void main(String[] args) throws InterruptedException {
        ConcurrentCounter counter = new ConcurrentCounter();
        
        // 10个线程并发递增
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 10000; j++) {
                    counter.increment("requests");
                }
            });
            threads[i].start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        // 结果准确：100000
        System.out.println("Total: " + counter.getCount("requests"));
    }
}
```

### 9.3 手写简易ConcurrentHashMap（简化版）

```java
public class MyConcurrentHashMap<K, V> {
    static class Node<K, V> {
        final int hash;
        final K key;
        volatile V value;
        volatile Node<K, V> next;
        
        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
    
    private static final int DEFAULT_CAPACITY = 16;
    private volatile Node<K, V>[] table;
    private volatile int size;
    
    @SuppressWarnings("unchecked")
    public MyConcurrentHashMap() {
        this.table = new Node[DEFAULT_CAPACITY];
    }
    
    private int hash(K key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    
    public V put(K key, V value) {
        int h = hash(key);
        int idx = h & (table.length - 1);
        
        // CAS尝试直接插入（桶为空的情况）
        // 注意：实际实现应使用Unsafe.compareAndSwapObject
        if (table[idx] == null) {
            Node<K, V> newNode = new Node<>(h, key, value, null);
            if (table[idx] == null) {  // 简化版，实际用CAS
                table[idx] = newNode;
                size++;
                return null;
            }
        }
        
        // 桶不为空，synchronized锁住头节点
        synchronized (table[idx]) {
            Node<K, V> node = table[idx];
            while (node != null) {
                if (node.hash == h && 
                    (node.key == key || (key != null && key.equals(node.key)))) {
                    V old = node.value;
                    node.value = value;
                    return old;
                }
                node = node.next;
            }
            // 尾插法
            Node<K, V> newNode = new Node<>(h, key, value, table[idx]);
            table[idx] = newNode;
            size++;
            return null;
        }
    }
    
    public V get(K key) {
        int h = hash(key);
        int idx = h & (table.length - 1);
        
        Node<K, V> node = table[idx];
        while (node != null) {
            if (node.hash == h && 
                (node.key == key || (key != null && key.equals(node.key)))) {
                return node.value;  // volatile读，保证可见性
            }
            node = node.next;
        }
        return null;
    }
    
    public int size() {
        return size;
    }
    
    public static void main(String[] args) throws InterruptedException {
        MyConcurrentHashMap<String, Integer> map = new MyConcurrentHashMap<>();
        
        // 并发测试
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            final int threadId = i;
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    map.put("key-" + threadId + "-" + j, j);
                }
            });
            threads[i].start();
        }
        
        for (Thread thread : threads) {
            thread.join();
        }
        
        System.out.println("Size: " + map.size());
    }
}
```

---

## 十、今日LeetCode练习

### [49. 字母异位词分组](https://leetcode.cn/problems/group-anagrams/)（Medium）

**题目：** 给你一个字符串数组，请你将字母异位词组合在一起。可以按任意顺序返回结果。

**思路：** 使用HashMap，key为排序后的字符串，value为该异位词组的列表。

```java
import java.util.*;

class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new HashMap<>();
        
        for (String str : strs) {
            // 排序作为key
            char[] chars = str.toCharArray();
            Arrays.sort(chars);
            String sorted = new String(chars);
            
            // 使用computeIfAbsent原子性地添加
            map.computeIfAbsent(sorted, k -> new ArrayList<>()).add(str);
        }
        
        return new ArrayList<>(map.values());
    }
}
```

**复杂度分析：**
- 时间：O(n * k * log k)，n是字符串数量，k是最大字符串长度
- 空间：O(n * k)

**变体思考：** 如果是多线程场景，如何使用ConcurrentHashMap实现？
- 使用`computeIfAbsent`保证线程安全
- 或者先分组再合并

---

## 十一、今日验收检查

完成学习后，请确认你能回答以下问题：

- [ ] ConcurrentHashMap在JDK 7和JDK 8中分别是什么数据结构？
- [ ] JDK 7的Segment是什么？它和ReentrantLock有什么关系？
- [ ] JDK 8为什么放弃分段锁？
- [ ] JDK 8的put()流程是怎样的？
- [ ] get()为什么不需要加锁？
- [ ] size()的baseCount + CounterCell方案是如何工作的？
- [ ] @Contended注解的作用是什么？
- [ ] JDK 8的ConcurrentHashMap如何扩容？
- [ ] ConcurrentHashMap为什么不允许null键值？
- [ ] synchronized锁住桶头节点为什么性能还这么好？

---

## 十二、明日预告

**Day 04：LinkedHashMap与LRU缓存**
- LinkedHashMap底层原理（HashMap + 双向链表）
- accessOrder参数的作用
- LinkedHashMap实现LRU缓存
- 手写LRU缓存（基于LinkedHashMap和自定义实现）

---

**学习时长：** 3小时理论 + 1小时LeetCode
**完成日期：** _______________
