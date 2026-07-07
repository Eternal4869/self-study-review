# Day01：ArrayList与LinkedList详解

## 今日目标
- 深入理解ArrayList的底层实现原理
- 深入理解LinkedList的底层实现原理
- 掌握两者的区别和适用场景
- 能够回答面试中的相关问题

---

## 一、ArrayList底层原理

### 1.1 底层数据结构

ArrayList底层是一个**Object数组**，名为`elementData`。

```java
public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    // 默认初始容量
    private static final int DEFAULT_CAPACITY = 10;
    
    // 存储元素的数组
    transient Object[] elementData;
    
    // 实际元素个数
    private int size;
}
```

**关键点：**
- `elementData`是`transient`修饰的，说明不参与默认序列化（ArrayList自定义了序列化逻辑）
- `size`是实际元素个数，`elementData.length`是数组容量

### 1.2 默认初始容量

```java
// 无参构造器：创建空数组（容量为0，不是10！）
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 带参构造器：指定初始容量
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    }
}
```

**面试问题：ArrayList默认容量是10吗？**
> 答：不是。无参构造器创建的是空数组（容量为0），当第一次`add()`时才会扩容为10。

### 1.3 扩容机制

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 确保容量足够
    elementData[size++] = e;            // 添加元素
    return true;
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        // 如果是空数组，取默认容量(10)和minCapacity的较大值
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;  // 修改次数+1（用于fail-fast机制）
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);  // 需要扩容
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5倍扩容
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);  // 复制数组
}
```

**扩容流程总结：**
1. 计算新容量 = 旧容量 + 旧容量 >> 1（即1.5倍）
2. 如果新容量不够，直接用所需最小容量
3. 使用`Arrays.copyOf()`复制到新数组

**面试问题：ArrayList为什么是1.5倍扩容？**
> 答：1.5倍是一个折中选择。太小会导致频繁扩容（性能差），太大会浪费内存。1.5倍扩容时，新数组大小为1.5倍，而Java数组最大长度是`Integer.MAX_VALUE`，1.5倍可以避免溢出。

### 1.4 线程安全性

ArrayList**不是线程安全的**。多线程环境下会出现：
- `IndexOutOfBoundsException`
- 元素丢失
- 数据不一致

**线程安全的替代方案：**
1. `Collections.synchronizedList(new ArrayList<>())`
2. `CopyOnWriteArrayList`（读多写少场景）

### 1.5 时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| get(int index) | O(1) | 直接通过下标访问 |
| add(E e) | O(1) 均摊 | 尾部添加，均摊O(1) |
| add(int index, E e) | O(n) | 需要移动元素 |
| remove(int index) | O(n) | 需要移动元素 |
| contains(Object o) | O(n) | 遍历查找 |

---

## 二、LinkedList底层原理

### 2.1 底层数据结构

LinkedList底层是一个**双向链表**。

```java
public class LinkedList<E> extends AbstractSequentialList<E> implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    transient int size = 0;
    transient Node<E> first;  // 头节点
    transient Node<E> last;   // 尾节点
}

// 节点结构
private static class Node<E> {
    E item;      // 存储的元素
    Node<E> next;   // 后继节点
    Node<E> prev;   // 前驱节点
    
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

**关键点：**
- 双向链表：每个节点有`prev`和`next`两个指针
- `first`指向头节点，`last`指向尾节点
- 支持从头尾两个方向遍历

### 2.2 添加元素

**尾部添加：**
```java
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;  // 链表为空
    else
        l.next = newNode;  // 原尾节点指向新节点
    size++;
    modCount++;
}
```

**头部添加：**
```java
void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

**指定位置添加：**
```java
void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

### 2.3 查询元素

```java
Node<E> node(int index) {
    // 如果index在前半部分，从头开始遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // 如果index在后半部分，从尾开始遍历
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

**优化：** 查询时会判断index在前半部分还是后半部分，选择更近的方向遍历。

### 2.4 时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| get(int index) | O(n) | 需要遍历链表 |
| add(E e) | O(1) | 尾部添加 |
| add(int index, E e) | O(n) | 需要先找到位置 |
| remove(int index) | O(n) | 需要先找到位置 |
| addFirst(E e) | O(1) | 头部添加 |
| addLast(E e) | O(1) | 尾部添加 |
| contains(Object o) | O(n) | 遍历查找 |

---

## 三、Vector简介

Vector是线程安全的ArrayList，但已经**过时**，不推荐使用。

```java
public class Vector<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    protected Object[] elementData;
    protected int elementCount;
    
    // 默认容量10
    public Vector() {
        this(10);
    }
    
    // 2倍扩容
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                         capacityIncrement : oldCapacity);
        // capacityIncrement默认为0，所以是2倍扩容
    }
}
```

**Vector vs ArrayList：**

| 特性 | Vector | ArrayList |
|------|--------|-----------|
| 线程安全 | 是（synchronized） | 否 |
| 扩容倍数 | 2倍 | 1.5倍 |
| 性能 | 较差（锁开销） | 较好 |
| 推荐使用 | 否 | 是 |

---

## 四、ArrayList vs LinkedList对比

| 特性 | ArrayList | LinkedList |
|------|-----------|------------|
| 底层结构 | 数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 尾部添加 | O(1) 均摊 | O(1) |
| 头部添加 | O(n) | O(1) |
| 指定位置添加 | O(n) | O(n) |
| 删除元素 | O(n) | O(1)（已知节点） |
| 内存占用 | 较少（连续） | 较多（指针开销） |
| 实现的接口 | List, RandomAccess | List, Deque |

**使用场景：**
- **ArrayList**：频繁查询、尾部增删、不确定元素数量
- **LinkedList**：频繁头部增删、实现队列/栈、内存敏感场景

---

## 五、面试常见问题

### Q1：为什么ArrayList查询快？
> **答：** ArrayList底层是数组，数组在内存中是连续存储的，支持通过下标直接访问元素（O(1)）。而LinkedList是链表，需要从头或尾遍历才能找到元素（O(n)）。

### Q2：为什么LinkedList增删快？
> **答：** LinkedList增删只需要修改指针指向，不需要移动元素。但这个"快"有个前提：已经找到了要操作的位置。如果需要先查找位置，那总耗时还是O(n)。所以实际场景中，LinkedList的增删优势并不明显。

### Q3：ArrayList扩容过程？
> **答：**
> 1. 添加元素时，检查是否需要扩容
> 2. 如果需要，计算新容量 = 旧容量 + 旧容量 >> 1（1.5倍）
> 3. 使用`Arrays.copyOf()`创建新数组并复制元素
> 4. 将引用指向新数组

### Q4：如何线程安全地使用List？
> **答：**
> 1. `Collections.synchronizedList(new ArrayList<>())` — 加锁，性能一般
> 2. `CopyOnWriteArrayList` — 写时复制，读多写少场景
> 3. `List.of()` — 不可变列表（Java 9+）
> 4. 手动加锁 — `synchronized(list) { ... }`

### Q5：ArrayList的elementData为什么是transient的？
> **答：** 因为elementData数组可能有空洞（容量>实际元素数），直接序列化会浪费空间。ArrayList自定义了`writeObject()`和`readObject()`方法，只序列化实际元素。

---

## 六、代码示例

### 6.1 ArrayList基本使用

```java
import java.util.ArrayList;
import java.util.List;

public class ArrayListDemo {
    public static void main(String[] args) {
        // 创建ArrayList
        List<String> list = new ArrayList<>();
        
        // 添加元素
        list.add("Java");
        list.add("Python");
        list.add("C++");
        
        // 获取元素
        String first = list.get(0);  // O(1)
        System.out.println(first);   // Java
        
        // 在指定位置插入
        list.add(1, "Go");  // O(n)
        
        // 删除元素
        list.remove(0);  // O(n)
        
        // 遍历
        for (String s : list) {
            System.out.println(s);
        }
        
        // 查看容量
        System.out.println("Size: " + list.size());
    }
}
```

### 6.2 LinkedList基本使用

```java
import java.util.LinkedList;
import java.util.List;

public class LinkedListDemo {
    public static void main(String[] args) {
        // 创建LinkedList
        LinkedList<String> list = new LinkedList<>();
        
        // 添加元素
        list.add("Java");
        list.add("Python");
        list.add("C++");
        
        // 头部添加
        list.addFirst("Go");
        
        // 尾部添加
        list.addLast("Rust");
        
        // 获取元素
        String first = list.getFirst();  // O(1)
        String last = list.getLast();    // O(1)
        
        // 删除元素
        list.removeFirst();  // O(1)
        list.removeLast();   // O(1)
        
        // 遍历
        for (String s : list) {
            System.out.println(s);
        }
        
        // 作为队列使用
        list.offer("Queue");  // 入队
        list.poll();          // 出队
        
        // 作为栈使用
        list.push("Stack");   // 入栈
        list.pop();           // 出栈
    }
}
```

### 6.3 手写ArrayList扩容逻辑

```java
public class MyArrayList<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private Object[] elementData;
    private int size;
    
    public MyArrayList() {
        this.elementData = new Object[DEFAULT_CAPACITY];
        this.size = 0;
    }
    
    public void add(E e) {
        // 检查是否需要扩容
        if (size == elementData.length) {
            grow();
        }
        elementData[size++] = e;
    }
    
    private void grow() {
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5倍
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    @SuppressWarnings("unchecked")
    public E get(int index) {
        if (index < 0 || index >= size) {
            throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        }
        return (E) elementData[index];
    }
    
    public int size() {
        return size;
    }
    
    public static void main(String[] args) {
        MyArrayList<Integer> list = new MyArrayList<>();
        for (int i = 0; i < 15; i++) {
            list.add(i);
            System.out.println("Added " + i + ", size=" + list.size());
        }
    }
}
```

---

## 七、今日LeetCode练习

### [1. 两数之和](https://leetcode.cn/problems/two-sum/)
**题目：** 给定一个整数数组 nums 和一个整数目标值 target，请你在该数组中找出和为目标值的两个整数，并返回它们的数组下标。

**思路：** 使用HashMap存储已经遍历过的数字和下标，空间换时间。

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int complement = target - nums[i];
            if (map.containsKey(complement)) {
                return new int[] { map.get(complement), i };
            }
            map.put(nums[i], i);
        }
        return new int[] {};
    }
}
```

### [49. 字母异位词分组](https://leetcode.cn/problems/group-anagrams/)
**题目：** 给你一个字符串数组，请你将字母异位词组合在一起。字母异位词是由重新排列源单词的所有字母得到的一个新单词。

**思路：** 将排序后的字符串作为key，相同的key就是字母异位词。

```java
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new HashMap<>();
        for (String str : strs) {
            char[] chars = str.toCharArray();
            Arrays.sort(chars);
            String key = new String(chars);
            map.computeIfAbsent(key, k -> new ArrayList<>()).add(str);
        }
        return new ArrayList<>(map.values());
    }
}
```

---

## 八、今日验收检查

完成学习后，请确认你能回答以下问题：

- [ ] ArrayList底层是什么数据结构？
- [ ] ArrayList默认容量是多少？什么时候变为10？
- [ ] ArrayList扩容是几倍？怎么计算的？
- [ ] ArrayList为什么不是线程安全的？
- [ ] LinkedList底层是什么数据结构？
- [ ] LinkedList的Node节点包含哪些字段？
- [ ] 为什么说ArrayList查询快、LinkedList增删快？
- [ ] 如何线程安全地使用List？
- [ ] ArrayList的elementData为什么是transient的？

---

## 九、明日预告

**Day 2：HashMap底层原理**
- HashMap的数据结构（数组+链表+红黑树）
- put()流程详解
- 扩容机制和hash冲突处理
- 链表转红黑树的条件

---

**学习时长：** 3小时理论 + 1小时LeetCode
**完成日期：** 2026/7/6