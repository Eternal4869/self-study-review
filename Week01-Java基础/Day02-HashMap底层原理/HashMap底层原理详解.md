# Day02：HashMap底层原理详解

## 今日目标
- 深入理解HashMap的底层数据结构（数组+链表+红黑树）
- 掌握hash()扰动函数的设计思想
- 完整走通put()流程
- 理解扩容机制和链表转红黑树的触发条件
- 能够回答面试中的HashMap相关问题

---

## 一、HashMap底层原理概述

### 1.1 数据结构演变

JDK 7：**数组 + 链表**
JDK 8：**数组 + 链表 + 红黑树**

当链表长度过长时，查询退化为O(n)。JDK 8引入红黑树，将极端情况下的查询优化为O(log n)。

**关键点：**
- 数组的每个位置称为一个**桶（Bucket）**
- 同一个桶中冲突的元素用链表存储
- 当链表长度 >= 8 且数组长度 >= 64 时，链表转为红黑树

### 1.2 核心属性

```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    // 默认初始容量：16（必须是2的幂次）
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
    
    // 最大容量：2^30
    static final int MAXIMUM_CAPACITY = 1 << 30;
    
    // 默认负载因子：0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    // 链表转红黑树的阈值：8
    static final int TREEIFY_THRESHOLD = 8;
    
    // 红黑树转链表的阈值：6
    static final int UNTREEIFY_THRESHOLD = 6;
    
    // 链表转红黑树的最小数组容量：64
    static final int MIN_TREEIFY_CAPACITY = 64;
    
    // 存储数据的Node数组
    transient Node<K,V>[] table;
    
    // 实际存储的键值对数量
    transient int size;
    
    // 扩容阈值 = capacity × loadFactor
    int threshold;
    
    // 负载因子
    final float loadFactor;
}
```

**面试问题：HashMap默认容量是多少？**
> 答：默认容量是16。但注意，`new HashMap()` 并不会立即分配数组，而是在第一次 `put()` 时才通过 `resize()` 初始化。

---

## 二、hash()方法与扰动函数

### 2.1 hash()实现

```java
static final int hash(Object key) {
    int h;
    // key.hashCode() 的高16位 ^ 低16位
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**为什么要做异或运算？**
- `hashCode()` 返回32位int，高16位在 `hash & (n-1)` 计算桶位置 `index = (n - 1) & hash` 时根本用不到（n远小于2^16），这时，即使两个元素hash后的高位值不同，计算时仍只有地位参与计算，一旦二者低位元素相同，就会hash冲突。
- 将高16位异或到低16位，让高位的信息也参与桶位置计算，减少冲突

### 2.2 为什么容量必须是2的幂次？

桶位置计算公式：`index = hash & (n - 1)`

当 n 是2的幂次时，`n - 1` 的二进制全是1（如16-1=15=0b1111），`hash & (n-1)` 相当于取模但效率更高。

当 $n = 16$ 时，$n - 1 = 15$，其二进制是 1111（低位全部是 1）。当任何一个 hash 值与 1111 进行 &（按位与）运算时：高位因为遇到了 0，全部被清零。低位因为遇到了 1，原封不动地被保留了下来。

这低位保留下来的值，范围恰好是 0000 到 1111（即十进制的 0 到 15），完美对应了数组的下标。HashMap 借此成功把昂贵的取模运算，变成了瞬间完成的位运算。

如果 n 不是2的幂次，某些桶位置永远不会被使用，导致空间浪费和冲突增加。

### 2.3 tableSizeFor() 保证容量为2的幂次

```java
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

**原理：** 通过无符号右移和或运算，将最高位1以下的所有位都置为1，再加1就得到大于等于cap的最小2的幂次。

例如：`cap=10` → `n=9(0b1001)` → 一系列位运算后 `n=15(0b1111)` → 返回 `16`

---

## 三、put()流程详解（核心）

### 3.1 putVal() 完整流程

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    
    // 1. 如果table为空或长度为0，先初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    
    // 2. 计算桶位置，如果桶为空，直接放入新节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        
        // 3. 桶不为空，判断第一个节点是否就是目标key
        if (p.hash == hash && 
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        
        // 4. 如果是红黑树节点，走红黑树插入逻辑
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        
        // 5. 链表遍历
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 到达链表尾部，插入新节点（尾插法）
                    p.next = newNode(hash, key, value, null);
                    // 如果链表长度 >= 8，转红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 找到相同key，跳出循环
                if (e.hash == hash && 
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 6. 如果e不为null，说明是覆盖已有key的value
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    
    ++modCount;
    // 7. 超过扩容阈值则扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

### 3.2 put流程图解

```
put(key, value)
    │
    ▼
计算 hash(key)
    │
    ▼
table 为空？ ──是──→ resize() 初始化
    │否
    ▼
计算桶位置 index = hash & (n-1)
    │
    ▼
桶为空？ ──是──→ 直接放入 newNode
    │否
    ▼
桶中第一个节点是目标key？ ──是──→ 覆盖value
    │否
    ▼
是红黑树节点？ ──是──→ putTreeVal() 红黑树插入
    │否
    ▼
遍历链表：
  ├── 找到相同key → 覆盖value
  └── 到达尾部 → 尾插法插入新节点
        │
        ▼
      链表长度 >= 8？ ──是──→ treeifyBin()（需数组长度 >= 64）
        │否
        ▼
      size > threshold？ ──是──→ resize() 扩容
```

---

## 四、链表与红黑树

### 4.1 链表转红黑树

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果数组长度 < 64，优先扩容而不是转红黑树
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            // ... 构建红黑树
        } while ((e = e.next) != null);
        // 替换链表为红黑树
        if ((tab[index] = hd) != null)
            hd.treeify(tab);
    }
}
```

**两个条件缺一不可：**
1. 链表长度 >= 8（`TREEIFY_THRESHOLD`）
2. 数组长度 >= 64（`MIN_TREEIFY_CAPACITY`）

**为什么还需要数组长度 >= 64？**
> 因为数组较小时，扩容比转红黑树更高效。扩容后链表会分散到更多桶中，很多情况下链表长度自然降下来了。

### 4.2 红黑树转链表

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    for (Node<K,V> q = this; q != null; q = q.next) {
        Node<K,V> p = new Node<>(q.hash, q.key, q.value, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}
```

当 `resize()` 拆分红黑树后，如果子链表长度 <= 6（`UNTREEIFY_THRESHOLD`），则将红黑树退化回链表。

### 4.3 TreeNode结构

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // 父节点
    TreeNode<K,V> left;    // 左子节点
    TreeNode<K,V> right;   // 右子节点
    TreeNode<K,V> prev;    // 前驱节点（用于删除后维持链表）
    boolean red;            // 颜色（红/黑）
}
```

**TreeNode同时具备链表和树的特性**：既可以通过 `parent/left/right` 遍历红黑树，也可以通过 `prev/next` 遍历链表。

---

## 五、扩容机制

### 5.1 resize() 流程

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    
    if (oldCap > 0) {
        // 已达最大容量，不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 正常扩容：容量翻倍，阈值翻倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    else if (oldThr > 0) // 使用threshold作为初始容量（带参构造器）
        newCap = oldThr;
    else {               // 默认值
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    threshold = newThr;
    
    // 创建新数组，迁移数据
    Node<K,V>[] newTab = (Node<K,V>[]) new Node[newCap];
    table = newTab;
    
    if (oldTab != null) {
        // 遍历旧数组，迁移每个桶中的元素
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;  // help GC
                
                if (e.next == null)
                    // 桶中只有一个节点，直接放到新位置
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 红黑树拆分
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                    // 链表拆分（JDK 8 优化）
                    Node<K,V> loHead = null, loTail = null;  // 低位链
                    Node<K,V> hiHead = null, hiTail = null;  // 高位链
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 关键：用 hash & oldCap 判断新位置
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        } else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;  // 低位链：原位置
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;  // 高位链：原位置+旧容量
                    }
                }
            }
        }
    }
    return newTab;
}
```

### 5.2 链表迁移的巧妙设计

JDK 8 的链表迁移不需要像 JDK 7 那样重新计算每个元素的桶位置。

**原理：** 新容量 = 旧容量 × 2，`hash & (newCap - 1)` 只比 `hash & (oldCap - 1)` 多用了一个高位（即 `oldCap` 那一位）。

- 如果 `hash & oldCap == 0`：元素留在原位置 `j`
- 如果 `hash & oldCap == 1`：元素移动到 `j + oldCap`

这样每条链表最多被拆成两条，效率很高。

### 5.3 为什么负载因子是0.75？

**时间-空间折中：**
- 负载因子太小（如0.5）：空间浪费大，但冲突少，查询快
- 负载因子太大（如1.0）：空间利用率高，但冲突多，查询慢
- 0.75 是一个折中值

**泊松分布理论：** 在理想随机hash下，桶中节点数服从泊松分布。负载因子为0.75时，每个桶中出现8个节点的概率约为千万分之六，几乎不可能发生。这就是为什么选择8作为链表转红黑树的阈值。

---

## 六、JDK 7 vs JDK 8 对比

| 特性 | JDK 7 | JDK 8 |
|------|-------|-------|
| 数据结构 | 数组 + 链表 | 数组 + 链表 + 红黑树 |
| 插入方式 | 头插法 | 尾插法 |
| 扩容迁移 | 重新计算 hash & (newCap-1) | 高位链/低位链优化 |
| hash计算 | 4次扰动（高16位异或4次） | 1次扰动 |
| 扩容时机 | 插入前检查 | 插入后检查 |
| 线程安全 | 不安全（死循环风险） | 不安全（数据覆盖） |

### JDK 7 头插法死循环问题

在JDK 7中，多线程并发扩容时，头插法会导致链表形成**环形结构**。一旦形成环，后续的 `get()` 操作会陷入**无限循环**，导致CPU 100%。

JDK 8改用**尾插法**，避免了环形链表问题，但HashMap仍然不是线程安全的（并发put可能丢数据）。

---

## 七、面试常见问题

### Q1：HashMap底层数据结构？
> **答：** JDK 8中，HashMap底层是**数组+链表+红黑树**。数组的每个位置称为一个桶，当出现hash冲突时用链表存储。当链表长度>=8且数组长度>=64时，链表转为红黑树以提高查询效率。

### Q2：为什么容量必须是2的幂次？
> **答：** 桶位置计算公式是 `hash & (n-1)`。当n是2的幂次时，`n-1`的二进制全是1，`&`运算相当于取模但更快。如果不是2的幂次，某些桶位置永远不会被使用，导致空间浪费和冲突增加。

### Q3：为什么负载因子是0.75？
> **答：** 0.75是时间与空间的折中。理论上，在随机hash下，每个桶中节点数服从泊松分布。当负载因子为0.75时，一个桶中出现8个节点的概率仅为0.00000006，几乎不可能触发红黑树转换，所以选择8作为链表转红黑树的阈值是合理的。

### Q4：JDK 7 HashMap死循环问题？
> **答：** JDK 7使用**头插法**，多线程并发扩容时，链表可能被反转并形成**环形结构**。一旦形成环，后续get()操作会**无限循环**，CPU 100%。JDK 8改用尾插法解决了环形链表问题，但HashMap仍不是线程安全的。

### Q5：HashMap和Hashtable的区别？

| 特性 | HashMap | Hashtable |
|------|---------|-----------|
| 线程安全 | 否 | 是（方法级synchronized） |
| null键值 | 允许1个null键+多个null值 | 不允许null键和null值 |
| 性能 | 较好 | 较差（全表锁） |
| 继承关系 | AbstractMap | Dictionary（已过时） |
| 推荐替代 | — | ConcurrentHashMap |

> **答：** Hashtable是早期的线程安全Map，使用方法级synchronized，性能差。HashMap不安全但性能好。需要线程安全时推荐使用`ConcurrentHashMap`。

---

## 八、代码示例

### 8.1 HashMap基本使用

```java
import java.util.HashMap;
import java.util.Map;

public class HashMapDemo {
    public static void main(String[] args) {
        HashMap<String, Integer> map = new HashMap<>();
        
        // 添加元素
        map.put("Java", 1);
        map.put("Python", 2);
        map.put("C++", 3);
        
        // 获取元素
        Integer value = map.get("Java");
        System.out.println(value);  // 1
        
        // 判断是否存在
        System.out.println(map.containsKey("Java"));    // true
        System.out.println(map.containsValue(2));       // true
        
        // 获取大小
        System.out.println(map.size());  // 3
        
        // 遍历方式1：entrySet
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            System.out.println(entry.getKey() + " = " + entry.getValue());
        }
        
        // 遍历方式2：keySet
        for (String key : map.keySet()) {
            System.out.println(key + " = " + map.get(key));
        }
        
        // 遍历方式3：forEach (Java 8+)
        map.forEach((k, v) -> System.out.println(k + " = " + v));
        
        // putIfAbsent：不存在才放入
        map.putIfAbsent("Java", 100);  // Java已存在，不会覆盖
        System.out.println(map.get("Java"));  // 仍是1
        
        // compute：重新计算值
        map.compute("Java", (k, v) -> v + 10);
        System.out.println(map.get("Java"));  // 11
        
        // getOrDefault：不存在时返回默认值
        int go = map.getOrDefault("Go", 0);
        System.out.println(go);  // 0
    }
}
```

### 8.2 手写简易HashMap

```java
public class MyHashMap<K, V> {
    static class Node<K, V> {
        final int hash;
        final K key;
        V value;
        Node<K, V> next;
        
        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }
    }
    
    private static final int DEFAULT_CAPACITY = 16;
    private static final float DEFAULT_LOAD_FACTOR = 0.75f;
    
    private Node<K, V>[] table;
    private int size;
    private int threshold;
    private final float loadFactor;
    
    @SuppressWarnings("unchecked")
    public MyHashMap() {
        this.table = new Node[DEFAULT_CAPACITY];
        this.threshold = (int)(DEFAULT_CAPACITY * DEFAULT_LOAD_FACTOR);
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }
    
    private int hash(K key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
    
    private int indexFor(int hash, int length) {
        return hash & (length - 1);
    }
    
    public void put(K key, V value) {
        if (size >= threshold) {
            resize();
        }
        int h = hash(key);
        int idx = indexFor(h, table.length);
        
        // 遍历链表，找到相同key则覆盖
        Node<K, V> node = table[idx];
        while (node != null) {
            if (node.hash == h && 
                (node.key == key || (key != null && key.equals(node.key)))) {
                node.value = value;
                return;
            }
            node = node.next;
        }
        
        // 头插法插入新节点
        table[idx] = new Node<>(h, key, value, table[idx]);
        size++;
    }
    
    public V get(K key) {
        int h = hash(key);
        int idx = indexFor(h, table.length);
        
        Node<K, V> node = table[idx];
        while (node != null) {
            if (node.hash == h && 
                (node.key == key || (key != null && key.equals(node.key)))) {
                return node.value;
            }
            node = node.next;
        }
        return null;
    }
    
    public int size() {
        return size;
    }
    
    @SuppressWarnings("unchecked")
    private void resize() {
        int newCapacity = table.length * 2;
        Node<K, V>[] newTable = new Node[newCapacity];
        threshold = (int)(newCapacity * loadFactor);
        
        for (int i = 0; i < table.length; i++) {
            Node<K, V> node = table[i];
            while (node != null) {
                Node<K, V> next = node.next;
                int newIdx = indexFor(node.hash, newCapacity);
                node.next = newTable[newIdx];
                newTable[newIdx] = node;
                node = next;
            }
        }
        table = newTable;
    }
    
    public static void main(String[] args) {
        MyHashMap<String, Integer> map = new MyHashMap<>();
        for (int i = 0; i < 20; i++) {
            map.put("key" + i, i);
            System.out.println("Put key" + i + ", size=" + map.size());
        }
        
        System.out.println("\nGet key5 = " + map.get("key5"));
        System.out.println("Get key99 = " + map.get("key99"));
    }
}
```

---

## 九、今日LeetCode练习

### [128. 最长连续序列](https://leetcode.cn/problems/longest-consecutive-sequence/)（Medium）

**题目：** 给定一个未排序的整数数组 `nums`，找出数字连续的最长序列（不要求序列元素在原数组中连续）的长度。请你设计并实现时间复杂度为 O(n) 的算法。

**思路：** 使用HashSet存储所有数字。对于每个数字，如果是序列的起始点（即 `num - 1` 不存在），就向后扩展计数。

```java
class Solution {
    public int longestConsecutive(int[] nums) {
        Set<Integer> set = new HashSet<>();
        for (int num : nums) {
            set.add(num);
        }
        
        int maxLen = 0;
        for (int num : set) {
            // 只从序列起始点开始扩展
            if (!set.contains(num - 1)) {
                int currentNum = num;
                int currentLen = 1;
                
                while (set.contains(currentNum + 1)) {
                    currentNum++;
                    currentLen++;
                }
                
                maxLen = Math.max(maxLen, currentLen);
            }
        }
        return maxLen;
    }
}
```

**复杂度分析：**
- 时间：O(n) — 每个数字最多被访问2次
- 空间：O(n) — HashSet存储

---

## 十、今日验收检查

完成学习后，请确认你能回答以下问题：

- [ ] HashMap底层是什么数据结构？
- [ ] hash()扰动函数的作用是什么？
- [ ] 为什么HashMap的容量必须是2的幂次？
- [ ] 请完整描述put()流程
- [ ] 链表转红黑树需要满足哪两个条件？
- [ ] JDK 8扩容时链表如何迁移？高位链和低位链是什么？
- [ ] 为什么负载因子是0.75？
- [ ] JDK 7和JDK 8的HashMap有什么区别？
- [ ] JDK 7 HashMap为什么会出现死循环？
- [ ] HashMap和Hashtable有什么区别？

---

## 十一、明日预告

**Day 03：ConcurrentHashMap原理**
- ConcurrentHashMap在JDK 7中的分段锁（Segment）机制
- ConcurrentHashMap在JDK 8中的CAS + synchronized
- put()流程对比
- size()的实现原理

---

**学习时长：** 3小时理论 + 1小时LeetCode
**完成日期：** _______________
