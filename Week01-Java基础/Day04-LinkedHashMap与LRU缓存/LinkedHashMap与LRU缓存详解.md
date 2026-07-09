# Day04：LinkedHashMap与LRU缓存详解

## 今日目标
- 深入理解LinkedHashMap的底层实现原理（HashMap + 双向链表）
- 掌握accessOrder参数对遍历顺序的影响
- 理解LRU缓存的设计思想和实现方式
- 掌握removeEldestEntry()回调机制
- 了解TreeMap的红黑树有序特性
- 掌握WeakReference和SoftReference的区别与应用场景
- 能够手写LRU缓存（基于LinkedHashMap和自定义实现）
- 能够回答面试中的相关问题

---

## 一、LinkedHashMap概述

### 1.1 为什么需要LinkedHashMap

HashMap的遍历顺序是不确定的（取决于hash分布），而LinkedHashMap在HashMap的基础上增加了一条**双向链表**，使得遍历顺序可以按照**插入顺序**或**访问顺序**排列。

| 特性 | HashMap | LinkedHashMap | TreeMap |
|------|---------|---------------|---------|
| 底层结构 | 数组+链表/红黑树 | 数组+链表/红黑树+双向链表 | 红黑树 |
| 顺序 | 无序 | 插入顺序/访问顺序 | Key自然排序/Comparator |
| null键 | 允许 | 允许 | 不允许（除非自定义Comparator） |
| 性能 | O(1) | O(1)（略低于HashMap） | O(log n) |
| 线程安全 | 否 | 否 | 否 |

### 1.2 继承关系

```
LinkedHashMap<K,V>
    └── extends HashMap<K,V>
        └── extends AbstractMap<K,V>
            └── implements Map<K,V>
```

**关键设计：LinkedHashMap直接继承HashMap**，只增加了双向链表来维护顺序。

---

## 二、LinkedHashMap底层原理

### 2.1 核心数据结构

```java
public class LinkedHashMap<K,V> extends HashMap<K,V>
        implements Map<K,V> {
    
    // 双向链表的头节点（最早插入/访问的元素）
    transient LinkedHashMap.Entry<K,V> head;
    
    // 双向链表的尾节点（最近插入/访问的元素）
    transient LinkedHashMap.Entry<K,V> tail;
    
    // 遍历顺序：false=插入顺序，true=访问顺序
    final boolean accessOrder;
    
    // 链表节点（继承HashMap.Node）
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before;  // 前驱指针
        Entry<K,V> after;   // 后继指针
        
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
}
```

**关键设计：**
- `Entry`继承自`HashMap.Node`，增加了`before`和`after`两个指针
- `head`指向链表头部（最老的元素）
- `tail`指向链表尾部（最新的元素）
- `accessOrder`决定链表的维护策略

### 2.2 双向链表图解

```
accessOrder = false（插入顺序）：
head → [A] ⇄ [B] ⇄ [C] ⇄ [D] ← tail
       ↓      ↓      ↓      ↓
      (最早插入)              (最晚插入)

accessOrder = true（访问顺序）：
每次get/put都会将节点移到tail
head → [A] ⇄ [B] ⇄ [C] ⇄ [D] ← tail
       ↓      ↓      ↓      ↓
      (最久未访问)          (最近访问)
```

### 2.3 构造方法

```java
public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}

// 默认构造：插入顺序
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    this.accessOrder = false;  // 默认插入顺序
}

// 默认构造：accessOrder=false
public LinkedHashMap() {
    super();
    this.accessOrder = false;
}
```

**面试问题：LinkedHashMap默认是插入顺序还是访问顺序？**
> 答：默认是**插入顺序**（accessOrder=false）。只有在构造时显式传入`accessOrder=true`才会变成访问顺序。

---

## 三、LinkedHashMap核心方法

### 3.1 afterNodeAccess() — 访问后调整链表

```java
// 在HashMap.get()/getOrDefault()等访问操作后调用
void afterNodeAccess(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> last;
    // 如果accessOrder=true且e不是尾节点
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        
        p.after = null;
        
        // 1. 将p从链表中摘除
        if (b != null)
            b.after = a;
        else
            head = a;
        
        if (a != null)
            a.before = b;
        else
            last = b;
        
        // 2. 将p插入到tail之后
        if (last != null)
            last.after = p;
        else
            head = p;
        
        p.before = last;
        tail = p;
        ++modCount;
    }
}
```

**流程图解（accessOrder=true时）：**

```
访问前：head → [A] ⇄ [B] ⇄ [C] ⇄ [D] ← tail
                         ↑
                       get(B)

访问后：head → [A] ⇄ [C] ⇄ [D] ⇄ [B] ← tail
                                   ↑
                                 (移到末尾)
```

### 3.2 afterNodeInsertion() — 插入后处理

```java
// 在HashMap.put()后调用
void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K,V> first;
    // 如果evict=true、head不为空且removeEldestEntry()返回true
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        // 删除链表头部节点（最老的元素）
        removeNode(hash(key), key, null, false, true);
    }
}
```

**这就是LRU缓存的核心机制**：通过重写`removeEldestEntry()`控制是否删除最老的元素。

### 3.3 afterNodeRemoval() — 删除后处理

```java
void afterNodeRemoval(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    
    // 将节点从双向链表中摘除
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}
```

### 3.4 遍历顺序

```java
// LinkedHashMap重写了迭代器
final class LinkedHashIterator {
    LinkedHashMap.Entry<K,V> next;
    LinkedHashMap.Entry<K,V> current;
    
    LinkedHashIterator() {
        // 从head开始遍历
        next = head;
    }
    
    public final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (e == null)
            throw new NoSuchElementException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        next = e.after;  // 沿着after指针遍历
        current = e;
        return e;
    }
}
```

**遍历顺序：**
- `accessOrder=false`：按插入顺序（head → tail）
- `accessOrder=true`：按访问顺序（最久未访问 → 最近访问）

---

## 四、LRU缓存实现

### 4.1 LRU概述

LRU（Least Recently Used，最近最少使用）是一种常用的缓存淘汰策略。当缓存满了时，优先淘汰**最久未访问**的元素。

### 4.2 基于LinkedHashMap实现LRU

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        // accessOrder=true → 按访问顺序排列
        // 初始容量、负载因子1.0（满时才扩容）、访问顺序
        super(capacity, 1.0f, true);
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 当size超过capacity时，删除最老的元素
        return size() > capacity;
    }
    
    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);
        
        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        System.out.println(cache);  // {1=A, 2=B, 3=C}
        
        // 访问key=1，它会被移到链表末尾
        cache.get(1);
        System.out.println(cache);  // {2=B, 3=C, 1=A}
        
        // 插入新元素key=4，触发淘汰
        cache.put(4, "D");
        System.out.println(cache);  // {3=C, 1=A, 4=D} — key=2被淘汰
    }
}
```

**为什么负载因子是1.0？**
> LRU缓存需要在元素数量达到capacity时立即淘汰，所以使用1.0的负载因子（不预留空间），配合`removeEldestEntry()`实现精确控制。

### 4.3 removeEldestEntry()源码分析

```java
// LinkedHashMap默认实现
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;  // 默认不删除
}

// LRU实现
@Override
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > capacity;  // 超过容量就删除
}
```

**调用时机：**
1. `put()`插入新元素后调用
2. `putAll()`批量插入后对每个元素调用
3. `putVal()`内部调用`afterNodeInsertion()`

### 4.4 LRU工作流程

```
容量=3，操作序列：put(1,A), put(2,B), put(3,C), get(1), put(4,D)

1. put(1,A): [1] → head=1, tail=1
2. put(2,B): [1]⇄[2] → head=1, tail=2
3. put(3,C): [1]⇄[2]⇄[3] → head=1, tail=3
4. get(1):   [2]⇄[3]⇄[1] → head=2, tail=1 (1移到末尾)
5. put(4,D): [3]⇄[1]⇄[4] → 淘汰key=2 (head)
```

---

## 五、手写LRU缓存（自定义实现）

### 5.1 使用HashMap + 双向链表

```java
import java.util.HashMap;

public class MyLRUCache<K, V> {
    
    // 双向链表节点
    private static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev;
        Node<K, V> next;
        
        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
    
    private final int capacity;
    private final HashMap<K, Node<K, V>> map;
    private final Node<K, V> head;  // 哨兵头节点（最老）
    private final Node<K, V> tail;  // 哨兵尾节点（最新）
    
    public MyLRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }
    
    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) {
            return null;
        }
        // 移到末尾（最近访问）
        moveToTail(node);
        return node.value;
    }
    
    public void put(K key, V value) {
        Node<K, V> node = map.get(key);
        if (node != null) {
            // key已存在，更新value并移到末尾
            node.value = value;
            moveToTail(node);
            return;
        }
        
        // key不存在，新建节点
        node = new Node<>(key, value);
        map.put(key, node);
        addToTail(node);
        
        // 超过容量，删除head后的节点（最久未访问）
        if (map.size() > capacity) {
            Node<K, V> removed = head.next;
            removeNode(removed);
            map.remove(removed.key);
        }
    }
    
    // 将节点添加到tail之前（最新位置）
    private void addToTail(Node<K, V> node) {
        node.prev = tail.prev;
        node.next = tail;
        tail.prev.next = node;
        tail.prev = node;
    }
    
    // 将节点移到末尾
    private void moveToTail(Node<K, V> node) {
        removeNode(node);
        addToTail(node);
    }
    
    // 从链表中删除节点
    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    public int size() {
        return map.size();
    }
    
    public void printCache() {
        Node<K, V> cur = head.next;
        System.out.print("LRU Cache (head → tail): ");
        while (cur != tail) {
            System.out.print("[" + cur.key + "=" + cur.value + "] ");
            cur = cur.next;
        }
        System.out.println();
    }
    
    public static void main(String[] args) {
        MyLRUCache<Integer, String> cache = new MyLRUCache<>(3);
        
        cache.put(1, "A");
        cache.put(2, "B");
        cache.put(3, "C");
        cache.printCache();  // [1=A] [2=B] [3=C]
        
        cache.get(1);
        cache.printCache();  // [2=B] [3=C] [1=A] — 1移到末尾
        
        cache.put(4, "D");
        cache.printCache();  // [3=C] [1=A] [4=D] — 2被淘汰
    }
}
```

### 5.2 复杂度分析

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| get() | O(1) | HashMap查找O(1) + 链表移动O(1) |
| put() | O(1) | HashMap插入O(1) + 链表操作O(1) |
| 空间 | O(capacity) | HashMap + 双向链表 |

### 5.3 哨兵节点的作用

使用哨兵节点（dummy head/tail）可以简化链表操作，避免大量判空逻辑：
- `head.next`始终是链表第一个真实节点
- `tail.prev`始终是链表最后一个真实节点
- 不需要特殊处理空链表的情况

---

## 六、WeakReference与SoftReference

### 6.1 引用类型概览

Java提供了四种引用强度从强到弱：

| 引用类型 | 回收时机 | 用途 |
|---------|---------|------|
| 强引用（Strong） | 永不回收（可达时） | 普通对象引用 |
| 软引用（Soft） | 内存不足时回收 | 缓存 |
| 弱引用（Weak） | 下次GC时回收 | ThreadLocalMap、WeakHashMap |
| 虚引用（Phantom） | 随时可回收 | 堆外内存管理 |

### 6.2 WeakReference

```java
import java.lang.ref.WeakReference;

public class WeakReferenceDemo {
    public static void main(String[] args) {
        // 强引用
        Object strong = new Object();
        
        // 弱引用
        WeakReference<Object> weak = new WeakReference<>(strong);
        
        System.out.println(weak.get());  // java.lang.Object@xxx
        
        // 将strong置为null
        strong = null;
        
        // 触发GC
        System.gc();
        
        // 弱引用对象已被回收
        System.out.println(weak.get());  // null
    }
}
```

**特点：**
- 只要发生GC，弱引用对象一定会被回收
- 不会影响对象的生命周期
- 典型应用：`WeakHashMap`、`ThreadLocalMap`

### 6.3 SoftReference

```java
import java.lang.ref.SoftReference;

public class SoftReferenceDemo {
    public static void main(String[] args) {
        // 软引用
        SoftReference<byte[]> soft = new SoftReference<>(new byte[1024 * 1024 * 10]); // 10MB
        
        System.out.println(soft.get());  // [B@xxx
        
        // 触发GC（内存充足时）
        System.gc();
        System.out.println(soft.get());  // [B@xxx（不会被回收）
        
        // 内存不足时，软引用对象会被回收
        // try {
        //     byte[] bytes = new byte[1024 * 1024 * 100]; // 100MB，可能触发OOM
        // } catch (OutOfMemoryError e) {
        //     System.out.println(soft.get());  // null（已被回收）
        // }
    }
}
```

**特点：**
- 内存充足时不回收，内存不足时才回收
- 适合做缓存（内存够就用，不够就释放）
- 典型应用：图片缓存、页面缓存

### 6.4 WeakReference vs SoftReference 对比

| 特性 | WeakReference | SoftReference |
|------|--------------|---------------|
| 回收时机 | GC时立即回收 | 内存不足时回收 |
| 生命周期 | 短（下次GC） | 长（直到内存不足） |
| 适用场景 | 不需要长期持有的缓存 | 可以长期持有的缓存 |
| 典型应用 | WeakHashMap、ThreadLocal | 图片缓存、数据缓存 |

### 6.5 WeakHashMap

```java
import java.util.WeakHashMap;

public class WeakHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        WeakHashMap<Object, String> map = new WeakHashMap<>();
        
        // key1和key2都是强引用，但放入WeakHashMap后会被包装成弱引用
        Object key1 = new Object();
        Object key2 = new Object();
        
        map.put(key1, "value1");
        map.put(key2, "value2");
        
        System.out.println("初始大小: " + map.size());  // 2
        
        // 将key1的强引用置为null，key1对应的entry可以被GC回收
        key1 = null;
        
        // key2仍然有强引用持有，不会被回收
        // 触发GC
        System.gc();
        
        // 等待GC完成
        Thread.sleep(100);
        
        // key1对应的条目已被回收，key2还在
        System.out.println("GC后大小: " + map.size());  // 1
        
        // 如果将key2也置为null，再次GC后所有entry都会被回收
        key2 = null;
        System.gc();
        Thread.sleep(100);
        System.out.println("全部置null后大小: " + map.size());  // 0
    }
}
```

**WeakHashMap的特点：**
- Key是弱引用，当Key不再被外部强引用持有时，GC会自动回收该Key及其对应的entry
- 适合做缓存（Key不再被使用时自动清理）
- Value是强引用，但如果Key被回收了，Value也会因为无法访问而被回收

---

## 七、TreeMap简介

### 7.1 底层结构

TreeMap底层是**红黑树**，Key按照自然排序或自定义Comparator排序。

```java
public class TreeMap<K,V> extends AbstractMap<K,V>
        implements NavigableMap<K,V>, Cloneable, java.io.Serializable {
    
    // 红黑树的根节点
    private transient Entry<K,V> root;
    
    // 比较器（可选）
    private final Comparator<? super K> comparator;
    
    // 红黑树节点
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK;
    }
}
```

### 7.2 基本使用

```java
import java.util.TreeMap;
import java.util.Comparator;

public class TreeMapDemo {
    public static void main(String[] args) {
        // 自然排序（Key需要实现Comparable）
        TreeMap<String, Integer> map = new TreeMap<>();
        map.put("banana", 2);
        map.put("apple", 5);
        map.put("cherry", 3);
        
        System.out.println(map);  // {apple=5, banana=2, cherry=3} — 按Key排序
        
        // 自定义排序
        TreeMap<String, Integer> descMap = new TreeMap<>(Comparator.reverseOrder());
        descMap.put("banana", 2);
        descMap.put("apple", 5);
        descMap.put("cherry", 3);
        
        System.out.println(descMap);  // {cherry=3, banana=2, apple=5} — 降序
        
        // 导航方法
        System.out.println(map.firstKey());   // apple
        System.out.println(map.lastKey());    // cherry
        System.out.println(map.lowerKey("banana"));   // apple（小于banana的最大Key）
        System.out.println(map.higherKey("banana"));  // cherry（大于banana的最小Key）
        
        // 范围查询
        System.out.println(map.subMap("apple", true, "cherry", true));
        // {apple=5, banana=2, cherry=3}
    }
}
```

### 7.3 TreeMap vs HashMap vs LinkedHashMap

| 特性 | HashMap | LinkedHashMap | TreeMap |
|------|---------|---------------|---------|
| 底层结构 | 数组+链表/红黑树 | 数组+链表/红黑树+双向链表 | 红黑树 |
| 顺序 | 无序 | 插入/访问顺序 | Key自然排序 |
| get/put | O(1) | O(1) | O(log n) |
| 最小/最大Key | 需要遍历 | 需要遍历 | O(log n) |
| 适用场景 | 快速查找 | 需要顺序 | 需要排序 |

---

## 八、面试常见问题

### Q1：LinkedHashMap的底层数据结构是什么？
> **答：** LinkedHashMap继承自HashMap，底层是**数组 + 链表/红黑树**（HashMap的结构），另外增加了一条**双向链表**来维护元素的顺序（插入顺序或访问顺序）。

### Q2：LinkedHashMap的accessOrder参数有什么作用？
> **答：** accessOrder决定链表的维护策略。`false`表示按**插入顺序**排列（默认），`true`表示按**访问顺序**排列。当accessOrder=true时，每次get()或put()已存在的key都会将该节点移到链表末尾，这就是LRU缓存的基础。

### Q3：如何用LinkedHashMap实现LRU缓存？
> **答：** 继承LinkedHashMap，构造时设置accessOrder=true，重写removeEldestEntry()方法，当size()超过容量时返回true。这样每次put新元素时，如果超过容量就会自动删除链表头部（最久未访问）的元素。

### Q4：removeEldestEntry()方法在什么时候被调用？
> **答：** 在put()、putAll()等插入操作后调用。具体流程是：put() → afterNodeInsertion() → removeEldestEntry()。如果返回true，则调用removeNode()删除链表头部节点（最老的元素）。

### Q5：WeakReference和SoftReference有什么区别？
> **答：** WeakReference在**下次GC时立即回收**，不关心内存是否充足；SoftReference只在**内存不足时才回收**，内存充足时会保留。WeakReference适合不需要长期持有的缓存（如WeakHashMap），SoftReference适合可以长期持有的缓存（如图片缓存）。

### Q6：WeakHashMap的Key为什么会被自动回收？
> **答：** WeakHashMap的Key是弱引用，当Key不再被外部强引用持有时，GC会自动回收该Key。GC后，WeakHashMap会在内部通过ReferenceQueue感知到Key被回收，然后删除对应的条目。

### Q7：HashMap、LinkedHashMap、TreeMap如何选择？
> **答：**
> - 需要快速查找，不关心顺序 → **HashMap**
> - 需要保持插入顺序或实现LRU → **LinkedHashMap**
> - 需要按Key排序或范围查询 → **TreeMap**

### Q8：LinkedHashMap的性能比HashMap差多少？
> **答：** 差距很小。LinkedHashMap只是在HashMap的基础上增加了维护双向链表的开销（每次put/get后调整链表），时间复杂度仍然是O(1)。在大多数场景下，性能差异可以忽略不计。

---

## 九、代码示例

### 9.1 LinkedHashMap按插入顺序遍历

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LinkedHashMapInsertionOrder {
    public static void main(String[] args) {
        // 默认插入顺序
        LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
        map.put("C", 3);
        map.put("A", 1);
        map.put("B", 2);
        
        // 遍历顺序 = 插入顺序
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            System.out.println(entry.getKey() + " = " + entry.getValue());
        }
        // 输出：C=3, A=1, B=2
    }
}
```

### 9.2 LinkedHashMap按访问顺序遍历

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LinkedHashMapAccessOrder {
    public static void main(String[] args) {
        // accessOrder=true
        LinkedHashMap<String, Integer> map = new LinkedHashMap<>(16, 0.75f, true);
        map.put("C", 3);
        map.put("A", 1);
        map.put("B", 2);
        
        System.out.println("访问前：" + map);
        // {C=3, A=1, B=2}
        
        // 访问A
        map.get("A");
        System.out.println("访问A后：" + map);
        // {C=3, B=2, A=1} — A移到末尾
        
        // 访问C
        map.get("C");
        System.out.println("访问C后：" + map);
        // {B=2, A=1, C=3} — C移到末尾
    }
}
```

### 9.3 基于LinkedHashMap的LRU缓存（完整版）

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 1.0f, true);
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
    
    @Override
    public String toString() {
        StringBuilder sb = new StringBuilder("LRU Cache [");
        boolean first = true;
        for (Map.Entry<K, V> entry : entrySet()) {
            if (!first) sb.append(" -> ");
            sb.append(entry.getKey()).append(":").append(entry.getValue());
            first = false;
        }
        sb.append("]");
        return sb.toString();
    }
    
    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);
        
        cache.put(1, "Java");
        cache.put(2, "Python");
        cache.put(3, "C++");
        System.out.println(cache);  // [1:Java -> 2:Python -> 3:C++]
        
        cache.get(1);
        System.out.println(cache);  // [2:Python -> 3:C++ -> 1:Java]
        
        cache.put(4, "Go");
        System.out.println(cache);  // [3:C++ -> 1:Java -> 4:Go] — key=2被淘汰
        
        cache.get(3);
        System.out.println(cache);  // [1:Java -> 4:Go -> 3:C++]
        
        cache.put(5, "Rust");
        System.out.println(cache);  // [4:Go -> 3:C++ -> 5:Rust] — key=1被淘汰
    }
}
```

### 9.4 使用WeakReference实现缓存

```java
import java.lang.ref.WeakReference;
import java.util.HashMap;
import java.util.Map;

public class WeakCache<K, V> {
    private final Map<K, WeakReference<V>> cache = new HashMap<>();
    
    public void put(K key, V value) {
        cache.put(key, new WeakReference<>(value));
    }
    
    public V get(K key) {
        WeakReference<V> ref = cache.get(key);
        if (ref != null) {
            V value = ref.get();
            if (value != null) {
                return value;
            } else {
                // Value已被GC回收，清理条目
                cache.remove(key);
            }
        }
        return null;
    }
    
    public int size() {
        return cache.size();
    }
    
    public static void main(String[] args) {
        WeakCache<String, byte[]> cache = new WeakCache<>();
        
        cache.put("data1", new byte[1024 * 1024]); // 1MB
        cache.put("data2", new byte[1024 * 1024]); // 1MB
        System.out.println("Size before GC: " + cache.size());  // 2
        
        System.gc();
        
        // GC后，如果byte[]不再被引用，会被回收
        System.out.println("Size after GC: " + cache.size());
    }
}
```

---

## 十、今日LeetCode练习

### [146. LRU缓存](https://leetcode.cn/problems/lru-cache/)（Medium）⭐️ 必做

**题目：** 设计并实现一个满足 LRU (最近最少使用) 缓存约束的数据结构。实现 `LRUCache` 类：
- `LRUCache(int capacity)` 以正整数作为容量初始化 LRU 缓存
- `int get(int key)` 如果关键字 key 存在于缓存中，则返回关键字的值，否则返回 -1
- `void put(int key, int value)` 如果关键字 key 已经存在，则变更其数据值 value；如果不存在，则向缓存中添加该键值对。当缓存容量达到上限时，应该在插入新项之前淘汰最久未使用的项

**要求：** `get` 和 `put` 必须以 O(1) 的时间复杂度运行。

**解法一：使用LinkedHashMap**

```java
import java.util.LinkedHashMap;

class LRUCache extends LinkedHashMap<Integer, Integer> {
    private int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
    
    public int get(int key) {
        return super.getOrDefault(key, -1);
    }
    
    public void put(int key, int value) {
        super.put(key, value);
    }
}
```

**解法二：HashMap + 双向链表（手写）**

```java
import java.util.HashMap;

class LRUCache {
    
    static class Node {
        int key;
        int value;
        Node prev;
        Node next;
        
        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }
    
    private int capacity;
    private HashMap<Integer, Node> map;
    private Node head;
    private Node tail;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node(-1, -1);
        this.tail = new Node(-1, -1);
        head.next = tail;
        tail.prev = head;
    }
    
    public int get(int key) {
        Node node = map.get(key);
        if (node == null) return -1;
        moveToTail(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        Node node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToTail(node);
            return;
        }
        
        node = new Node(key, value);
        map.put(key, node);
        addToTail(node);
        
        if (map.size() > capacity) {
            Node removed = head.next;
            removeNode(removed);
            map.remove(removed.key);
        }
    }
    
    private void addToTail(Node node) {
        node.prev = tail.prev;
        node.next = tail;
        tail.prev.next = node;
        tail.prev = node;
    }
    
    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    private void moveToTail(Node node) {
        removeNode(node);
        addToTail(node);
    }
}
```

**复杂度分析：**
- 时间：get() O(1)，put() O(1)
- 空间：O(capacity)

**面试要点：**
- 面试中通常要求手写（解法二），而不是直接用LinkedHashMap
- 关键是理解HashMap提供O(1)查找，双向链表提供O(1)的插入/删除/移动
- 哨兵节点简化边界处理

---

## 十一、今日验收检查

完成学习后，请确认你能回答以下问题：

- [ ] LinkedHashMap的底层数据结构是什么？
- [ ] accessOrder参数的两种模式分别是什么效果？
- [ ] removeEldestEntry()在什么时候被调用？
- [ ] 如何用LinkedHashMap实现LRU缓存？
- [ ] WeakReference和SoftReference的区别是什么？
- [ ] WeakHashMap的Key为什么会被自动回收？
- [ ] TreeMap的底层结构和特点是什么？
- [ ] HashMap、LinkedHashMap、TreeMap如何选择？
- [ ] 能够手写LRU缓存（HashMap + 双向链表）
- [ ] LRU缓存的get/put时间复杂度是多少？

---

## 十二、明日预告

**Day 05：泛型与异常体系**
- 泛型的类型擦除机制
- PECS原则（Producer Extends, Consumer Super）
- 通配符的使用场景
- Checked Exception vs Unchecked Exception
- 自定义异常的最佳实践

---

**学习时长：** 3小时理论 + 1小时LeetCode
**完成日期：** _______________
