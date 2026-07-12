# Day07 详解 — Week1 复习总结

## 今日目标
- 系统回顾Week1的所有知识点
- 通过默写检验学习成果
- 查漏补缺，巩固薄弱环节
- 为进入Week2多线程学习打好基础

---

## 一、本周知识体系总览

### 1.1 ArrayList核心要点

| 特性 | 说明 |
|------|------|
| 底层结构 | **Object[] elementData** 动态数组 |
| 默认容量 | 10（首次add时扩容） |
| 扩容机制 | **1.5倍扩容**（oldCapacity + (oldCapacity >> 1)） |
| 扩容操作 | **Arrays.copyOf()** 创建新数组并复制 |
| 线程安全 | ❌ **非线程安全**，多线程需用Collections.synchronizedList或CopyOnWriteArrayList |
| 随机访问 | ✅ O(1)，通过下标直接访问 |
| 插入/删除 | ❌ O(n)，需要移动元素 |

**核心流程：add()方法**
```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 检查是否需要扩容
    elementData[size++] = e;            // 在末尾添加元素
    return true;
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5倍扩容
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    elementData = Arrays.copyOf(elementData, newCapacity); // 复制数组
}
```

**易错点：**
- ❌ 在for循环中使用remove()会导致跳过元素
- ✅ 正确做法：使用Iterator的remove()或倒序遍历删除

---

### 1.2 HashMap核心要点

| 特性 | 说明 |
|------|------|
| 底层结构 | **数组 + 链表 + 红黑树**（JDK 1.8） |
| 默认容量 | 16 |
| 负载因子 | **0.75**（默认） |
| 树化阈值 | **8**（链表长度≥8且数组长度≥64时转红黑树） |
| 退化阈值 | **6**（红黑树节点≤6时退化为链表） |
| 扩容机制 | **2倍扩容**（容量翻倍，重新hash分布） |
| 线程安全 | ❌ **非线程安全**，多线程需用ConcurrentHashMap |

**put()流程（重点默写）：**
```
1. 计算key的hash值：(h = key.hashCode()) ^ (h >>> 16)  // 高16位异或低16位
2. 计算数组下标：(n - 1) & hash  // n为数组长度
3. 判断该位置是否为空：
   - 空：直接插入新节点
   - 非空：
     a. key相同（equals判断）：覆盖value
     b. key不同：
        - 若为TreeNode：红黑树插入
        - 若为链表：尾插法遍历，节点数≥8且数组长度≥64则树化
4. 检查是否需要扩容：size > threshold（容量×负载因子）
```

**hash()方法优化：**
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
**为什么高16位异或低16位？** 让高位也参与下标计算，减少hash冲突。

---

### 1.3 ConcurrentHashMap核心要点

| 特性 | 说明 |
|------|------|
| 底层结构 | **数组 + 链表 + 红黑树**（JDK 1.8） |
| 线程安全 | ✅ **线程安全**（CAS + synchronized） |
| 锁粒度 | **Node级别**（锁单个桶，不是整个数组） |
| 并发度 | 数组长度（默认16，最多16个线程并发写） |
| 不允许null | key和value都不能为null |

**get()流程（重点默写）：**
```
1. 计算hash值：(h = key.hashCode()) ^ (h >>> 16)
2. 定位桶位置：(n - 1) & hash
3. 遍历该桶的链表/红黑树：
   - 链表：逐个比较hash和equals
   - 红黑树：O(logn)查找
4. 返回对应的value
```
**为什么get()不需要加锁？**
- Node的val和next都用**volatile**修饰，保证可见性
- 链表头节点的修改通过CAS/synchronized保证原子性

**put()流程（JDK 1.8）：**
```
1. 计算hash值，定位桶位置
2. 若桶为空：CAS操作插入新节点
3. 若桶非空：
   - 当前节点的hash ==MOVED(-1)：正在扩容，帮助扩容
   - 否则：synchronized锁住该桶的头节点
4. 遍历链表/红黑树，插入或更新节点
5. 检查是否需要扩容
```

---

### 1.4 LinkedHashMap与LRU缓存核心要点

| 特性 | 说明 |
|------|------|
| 底层结构 | **HashMap + 双向链表** |
| 遍历顺序 | **插入顺序** 或 **访问顺序**（accessOrder参数） |
| accessOrder | false=插入顺序，true=访问顺序 |
| removeEldestEntry | 重写此方法实现LRU淘汰策略 |

**LRU缓存实现原理（重点默写）：**
```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        // accessOrder=true，按访问顺序排序
        super(capacity, 0.75f, true);
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        // 超过容量时移除最老的元素
        return size() > capacity;
    }
}
```

**LRU工作流程：**
```
1. 访问key时：get()或put()
   - 若key存在：移动到链表尾部（最新访问）
   - 若key不存在：插入链表尾部
2. 当size > capacity时：
   - removeEldestEntry()返回true
   - 自动移除链表头部的节点（最久未访问）
```

**易错点：**
- ❌ accessOrder=false时，LRU不生效
- ✅ 必须设置accessOrder=true

---

### 1.5 泛型与异常核心要点

**泛型核心概念：**

| 概念 | 说明 |
|------|------|
| 类型擦除 | 编译后泛型信息被擦除，替换为Object或上界类型 |
| 通配符 | `? extends T`（上界）、`? super T`（下界）、`?`（无界） |
| PECS原则 | **Producer Extends, Consumer Super** |
| 泛型方法 | `<T> T method(T param)` - 独立于类的泛型参数 |

**类型擦除示例：**
```java
// 编译前
List<String> list = new ArrayList<>();
list.add("hello");
String s = list.get(0);

// 编译后（类型擦除）
List list = new ArrayList();
list.add("hello");
String s = (String) list.get(0); // 强制转换
```

**异常体系：**
```
Throwable
├── Error（系统级错误，不可恢复）
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception（可恢复）
    ├── RuntimeException（非受检异常）
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   └── ClassCastException
    └── 受检异常（必须处理）
        ├── IOException
        └── SQLException
```

**try-with-resources：**
```java
// 自动关闭实现AutoCloseable的资源
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    String line = br.readLine();
} // 自动调用br.close()
```

---

### 1.6 String与字符串核心要点

| 特性 | 说明 |
|------|------|
| 不可变性 | **final char[] value**，字符串常量池 |
| 字符串常量池 | JDK 7+在堆中，JDK 6在方法区 |
| intern()方法 | 将字符串放入常量池并返回引用 |
| StringBuilder | 可变，线程不安全，性能好 |
| StringBuffer | 可变，线程安全（synchronized），性能稍差 |

**String vs StringBuilder vs StringBuffer：**

| 类型 | 可变性 | 线程安全 | 性能 | 适用场景 |
|------|--------|----------|------|----------|
| String | ❌ 不可变 | ✅ 安全 | 慢（频繁拼接时） | 字符串不常变化 |
| StringBuilder | ✅ 可变 | ❌ 不安全 | 快 | 单线程字符串操作 |
| StringBuffer | ✅ 可变 | ✅ 安全 | 较快 | 多线程字符串操作 |

**字符串常量池：**
```java
String s1 = "hello";       // 常量池
String s2 = "hello";       // 常量池（相同引用）
String s3 = new String("hello"); // 堆内存（新对象）

System.out.println(s1 == s2);      // true（同一引用）
System.out.println(s1 == s3);      // false（不同对象）
System.out.println(s1.equals(s3)); // true（值相同）
```

---

## 二、面试高频问题默写

### 2.1 ArrayList相关

**问题1：ArrayList的扩容机制是什么？**

```
答：ArrayList默认初始容量为10，当添加元素导致容量不足时，
会进行扩容。扩容后的容量为原来的1.5倍（oldCapacity + (oldCapacity >> 1)）。
扩容操作使用Arrays.copyOf()创建新数组并复制所有元素。

具体流程：
1. 计算新容量：newCapacity = oldCapacity + (oldCapacity >> 1)
2. 若新容量小于所需最小容量，则使用最小容量
3. 调用Arrays.copyOf()创建新数组并复制
```

**问题2：如何安全地遍历ArrayList并删除元素？**

```
答：有三种方式：

方式一：使用Iterator（推荐）
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    if (it.next() < 5) {
        it.remove(); // 安全删除
    }
}

方式二：倒序遍历
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i) < 5) {
        list.remove(i);
    }
}

方式三：使用removeIf()（Java 8+）
list.removeIf(n -> n < 5);

❌ 错误方式：正序for循环删除会导致跳过元素
```

**问题3：ArrayList和LinkedList的区别？**

```
答：
ArrayList：
- 底层：Object[]数组
- 随机访问：O(1)
- 插入/删除：O(n)（需要移动元素）
- 内存：连续，占用较少

LinkedList：
- 底层：双向链表
- 随机访问：O(n)（需要遍历）
- 插入/删除：O(1)（已知位置时）
- 内存：不连续，每个节点额外存储前后指针

选择建议：
- 频繁随机访问 → ArrayList
- 频繁在头部/中间插入删除 → LinkedList
- 实际场景：大多数情况ArrayList更优（CPU缓存友好）
```

---

### 2.2 HashMap相关

**问题1：HashMap的put()流程是什么？（重点）**

```
答：
1. 计算key的hash值：(h = key.hashCode()) ^ (h >>> 16)
   - 高16位异或低16位，让高位也参与下标计算

2. 计算数组下标：(n - 1) & hash
   - n为数组长度（必须是2的幂）

3. 判断该位置是否为空：
   - 若为空：直接插入新节点
   - 若非空：
     a. 比较hash值，若相同且key的equals为true：覆盖value
     b. 否则：
        - 若节点是TreeNode：调用红黑树的插入方法
        - 若是链表：遍历链表，尾插法添加新节点
          - 若链表长度≥8且数组长度≥64：树化为红黑树
          - 若数组长度<64：先扩容而不是树化

4. 检查是否需要扩容：
   - 若size > threshold（容量×负载因子）：扩容为原来的2倍
```

**问题2：HashMap为什么线程不安全？**

```
答：
1. 数据丢失：
   - 多线程并发put时，可能同时计算出相同的桶位置
   - 一个线程的插入可能被另一个覆盖

2. 死循环（JDK 1.7）：
   - JDK 1.7使用头插法，扩容时链表反转
   - 多线程扩容可能导致链表成环，get()时死循环
   - JDK 1.8使用尾插法解决了此问题

3. size不准确：
   - size++不是原子操作

解决方案：
- 使用Collections.synchronizedMap()
- 使用ConcurrentHashMap（推荐）
```

**问题3：HashMap的容量为什么是2的幂？**

```
答：
1. 减少hash冲突：
   - 下标计算：(n - 1) & hash
   - 当n是2的幂时，n-1的二进制全为1
   - 这样hash的低位都能参与下标计算

2. 扩容时重新分布高效：
   - 扩容后容量为2n，n-1变成2n-1
   - 新增的位决定了元素是留在原位还是移动到原位+n
   - 可以通过hash & oldCap快速判断

示例：
n=16 (10000), n-1=15 (01111)
hash & 15 只看hash的低4位

扩容到32 (100000), n-1=31 (011111)
hash & 31 看hash的低5位
新增的第5位决定元素是否移动
```

---

### 2.3 ConcurrentHashMap相关

**问题1：ConcurrentHashMap的get()流程是什么？（重点）**

```
答：
1. 计算hash值：(h = key.hashCode()) ^ (h >>> 16)

2. 定位桶位置：(n - 1) & hash

3. 遍历该桶的链表/红黑树：
   - 若是链表：逐个比较hash和equals
   - 若是红黑树：O(logn)查找

4. 返回对应的value

为什么不需要加锁？
- Node的val和next都用volatile修饰，保证可见性
- 链表头节点的修改通过CAS/synchronized保证原子性
- 读操作不会改变数据结构，天然线程安全
```

**问题2：ConcurrentHashMap在JDK 1.7和1.8的区别？**

```
答：
JDK 1.7：
- 底层：分段数组（Segment[]）+ 链表
- 锁：Segment（继承ReentrantLock），锁整个段
- 并发度：Segment数量（默认16）
- 扩容：单个Segment内部扩容

JDK 1.8：
- 底层：Node[] + 链表 + 红黑树（与HashMap相同）
- 锁：CAS + synchronized（锁单个桶的头节点）
- 并发度：数组长度（可动态扩展）
- 扩容：多线程协作扩容（transfer()）

JDK 1.8优化：
1. 锁粒度更细：从段级别降到桶级别
2. 数据结构升级：引入红黑树，查询效率O(logn)
3. 扩容优化：多线程协助，提高扩容效率
4. 内存占用更少：去掉了Segment的开销
```

**问题3：ConcurrentHashMap的size()方法准确吗？**

```
答：不一定完全准确，但足够实用。

原因：
- size()遍历所有Node计数，但遍历过程中可能有并发修改
- 使用baseCount + CounterCell数组分散计数，减少竞争

保证准确性的方式：
1. 使用sumCount()方法汇总所有计数
2. 但对于高并发场景，可能有±1的误差

如果需要精确计数：
- 使用Collections.synchronizedMap() + 单独的锁
- 或者使用LongAdder等原子类手动计数
```

---

### 2.4 其他集合相关

**问题1：HashMap和Hashtable的区别？**

答：
| 特性 | HashMap | Hashtable |
|------|---------|-----------|
| 线程安全 | ❌ 不安全 | ✅ 安全（synchronized） |
| null键值 | 允许1个null键，多个null值 | 不允许null键和null值 |
| 性能 | 高（无锁） | 低（全表锁） |
| 继承 | AbstractMap | Dictionary（已过时） |
| 推荐 | ✅ 推荐 | ❌ 不推荐（遗留类） |

替代Hashtable：
- 线程安全Map：ConcurrentHashMap
- 线程安全List：CopyOnWriteArrayList
- 线程安全Set：CopyOnWriteArraySet

**问题2：ConcurrentHashMap可以放null值吗？为什么？**

```
答：不可以。ConcurrentHashMap的key和value都不能为null。

原因：
1. 二义性问题：
   - map.get(key)返回null时，无法区分"key不存在"还是"value为null"
   - 在多线程环境下，这个二义性会导致问题

2. 设计哲学：
   - ConcurrentHashMap用于高并发场景，null值会增加复杂度
   - 强制使用明确的值，避免歧义

对比：
- HashMap：允许null键和null值
- Hashtable：不允许null键和null值
- ConcurrentHashMap：不允许null键和null值
```

**问题3：什么是fail-fast？HashMap会fail-fast吗？**

```
答：
fail-fast：在遍历集合时，如果集合被修改（结构修改），
遍历会立即抛出ConcurrentModificationException。

原理：
- 集合维护modCount变量，记录修改次数
- 迭代器创建时记录expectedModCount = modCount
- 每次next()检查modCount == expectedModCount
- 若不相等，说明被并发修改，抛出异常

HashMap会fail-fast吗？
- 单线程：是的，for循环中remove()会触发
- 多线程：不一定，ConcurrentModificationException不是保证会抛出

安全遍历方式：
1. 使用Iterator的remove()方法
2. 使用ConcurrentHashMap（不会抛出异常，但可能丢失数据）
3. 使用Collections.synchronizedMap() + 手动同步
```

---

## 三、LeetCode复习

### 本周LeetCode题目清单

| 题号 | 题目 | 难度 | 核心知识点 |
|------|------|------|------------|
| 1 | 两数之和 | Easy | HashMap查找 |
| 15 | 三数之和 | Medium | 双指针 + 排序 |
| 42 | 接雨水 | Hard | 双指针 / 单调栈 |
| 206 | 反转链表 | Easy | 链表操作 |
| 21 | 合并两个有序链表 | Easy | 链表合并 |
| 141 | 环形链表 | Easy | 快慢指针 |
| 142 | 环形链表II | Medium | 快慢指针 + 数学 |
| 25 | K个一组翻转链表 | Hard | 链表分组翻转 |
| 146 | LRU缓存 | Medium | LinkedHashMap |
| 23 | 合并K个升序链表 | Hard | 优先队列 / 分治 |

### 易错点总结

**两数之和（1）：**
```java
// ❌ 错误：两次遍历O(n²)
for (int i = 0; i < nums.length; i++) {
    for (int j = i + 1; j < nums.length; j++) {
        if (nums[i] + nums[j] == target) { ... }
    }
}

// ✅ 正确：一次遍历O(n)
Map<Integer, Integer> map = new HashMap<>();
for (int i = 0; i < nums.length; i++) {
    int complement = target - nums[i];
    if (map.containsKey(complement)) {
        return new int[]{map.get(complement), i};
    }
    map.put(nums[i], i);
}
```

**LRU缓存（146）：**
```java
// ❌ 错误：忘记设置accessOrder
super(capacity, 0.75f); // 默认false，LRU不生效

// ✅ 正确：设置accessOrder=true
super(capacity, 0.75f, true); // 按访问顺序排序
```

**反转链表（206）：**
```java
// ❌ 错误：丢失后续节点
ListNode prev = null;
while (curr != null) {
    ListNode next = curr.next;
    curr.next = prev;
    prev = curr;
    curr = next; // 必须用临时变量
}

// ✅ 正确：使用临时变量保存
ListNode prev = null, curr = head;
while (curr != null) {
    ListNode next = curr.next; // 保存下一个
    curr.next = prev;          // 反转
    prev = curr;               // 移动prev
    curr = next;               // 移动curr
}
```

### 解题思路回顾

**HashMap类题目：**
- 两数之和：用Map存 {值: 下标}，边存边查
- 字母异位词分组：排序后的字符串作为key

**链表类题目：**
- 反转链表：三指针（prev, curr, next）
- 环形链表：快慢指针（fast走2步，slow走1步）
- 合并链表：哨兵节点 + 比较连接

**LRU缓存：**
- 核心：LinkedHashMap + accessOrder=true
- 关键方法：重写removeEldestEntry()

---

## 四、知识薄弱点分析

### 常见易错点

| 知识点 | 易错点 | 正确理解 |
|--------|--------|----------|
| HashMap扩容 | 扩容是2倍 | ✅ 2倍扩容，容量保持2的幂 |
| ArrayList扩容 | 扩容是2倍 | ❌ **1.5倍扩容** |
| HashMap树化 | 链表长度≥8就树化 | ❌ 还需要数组长度≥64 |
| HashMap红黑树退化 | 节点≤8退化 | ❌ **节点≤6**才退化 |
| ConcurrentHashMap | 可以放null | ❌ **key和value都不能为null** |
| LRU缓存 | 使用HashMap | ❌ 必须用**LinkedHashMap + accessOrder=true** |
| HashMap线程安全 | JDK 1.8是线程安全的 | ❌ **仍然不安全**，只是解决了死循环 |
| String不可变 | 可以修改字符数组 | ❌ **final char[]**，不可变 |

### 面试常见坑

**坑1：ArrayList的remove()时间复杂度**
```
❌ 错误理解：删除最后一个元素是O(1)
✅ 正确理解：
   - 删除最后一个元素：O(1)
   - 删除中间元素：O(n)，需要移动后续所有元素
   - 总体平均：O(n)
```

**坑2：HashMap的hash()方法**
```
❌ 错误理解：直接用key.hashCode()
✅ 正确理解：(h = key.hashCode()) ^ (h >>> 16)
   - 高16位异或低16位
   - 目的：让高位也参与下标计算，减少冲突
```

**坑3：ConcurrentHashMap的size()**
```
❌ 错误理解：size()是精确值
✅ 正确理解：高并发时可能有±1的误差
   - 使用baseCount + CounterCell数组
   - 遍历时可能有并发修改
```

**坑4：String的equals()和==**
```
❌ 错误理解：==比较值
✅ 正确理解：
   - == 比较引用（内存地址）
   - equals() 比较内容
   - 字符串常量池导致 == 可能返回true（相同字面量）
```

---

## 五、下周学习计划

### Week2学习目标

| 天数 | 主题 | 核心内容 |
|------|------|----------|
| Day08 | 线程基础 | Thread、Runnable、Callable、线程生命周期 |
| Day09 | synchronized | 对象锁、类锁、锁升级（偏向→轻量→重量） |
| Day10 | volatile | 可见性、禁止指令重排、内存屏障 |
| Day11 | ThreadLocal | 原理、内存泄漏、使用场景 |
| Day12 | 线程池 | 核心参数、执行流程、拒绝策略 |
| Day13 | AQS | AbstractQueuedSynchronizer、公平/非公平锁 |
| Day14 | 复习日 | Week2复习总结 |

### 预习建议

**线程基础预习：**
```java
// 1. 理解线程的创建方式
// 方式一：继承Thread
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running");
    }
}

// 方式二：实现Runnable
Runnable runnable = () -> System.out.println("Runnable running");
new Thread(runnable).start();

// 方式三：实现Callable（有返回值）
Callable<Integer> callable = () -> 42;
FutureTask<Integer> task = new FutureTask<>(callable);
new Thread(task).start();
Integer result = task.get(); // 阻塞获取结果
```

**关键概念预习：**
1. **线程生命周期**：NEW → RUNNABLE → BLOCKED/WAITING/TIMED_WAITING → TERMINATED
2. **synchronized原理**：Monitor对象、对象头、锁升级
3. **volatile关键字**：保证可见性、禁止指令重排
4. **线程池核心参数**：corePoolSize、maximumPoolSize、keepAliveTime、workQueue

---

## 六、验收清单

### 本周学习成果自测

#### ArrayList
- [ ] 能默写ArrayList的扩容机制（1.5倍扩容）
- [ ] 理解Arrays.copyOf()的作用
- [ ] 知道如何安全遍历删除元素（Iterator/removeIf/倒序）
- [ ] 理解ArrayList和LinkedList的区别及适用场景

#### HashMap
- [ ] 能默写HashMap的put()流程（hash计算→定位→插入→扩容）
- [ ] 理解hash()方法高16位异或低16位的作用
- [ ] 知道树化条件：链表长度≥8且数组长度≥64
- [ ] 理解扩容为什么是2倍以及如何重新分布
- [ ] 知道HashMap为什么线程不安全（数据丢失/死循环）

#### ConcurrentHashMap
- [ ] 能默写ConcurrentHashMap的get()流程（无锁读取）
- [ ] 理解volatile保证可见性的原理
- [ ] 知道JDK 1.7和1.8的区别（分段锁→CAS+synchronized）
- [ ] 理解为什么key和value都不能为null

#### LinkedHashMap与LRU
- [ ] 理解LinkedHashMap的双向链表结构
- [ ] 知道accessOrder参数的作用（true=访问顺序）
- [ ] 能默写LRU缓存的实现（继承LinkedHashMap+removeEldestEntry）
- [ ] 理解LRU的工作流程（访问时移动到尾部，超容量移除头部）

#### 泛型与异常
- [ ] 理解类型擦除的概念
- [ ] 知道PECS原则（Producer Extends, Consumer Super）
- [ ] 理解异常体系（Error/RuntimeException/受检异常）
- [ ] 掌握try-with-resources的使用

#### String
- [ ] 理解String的不可变性（final char[]）
- [ ] 知道字符串常量池的位置（JDK 7+在堆中）
- [ ] 理解intern()方法的作用
- [ ] 区分String/StringBuilder/StringBuffer的适用场景

#### LeetCode
- [ ] 能独立完成两数之和（HashMap思路）
- [ ] 能独立完成反转链表（三指针）
- [ ] 能独立完成LRU缓存（LinkedHashMap实现）
- [ ] 能独立完成环形链表（快慢指针）
- [ ] 理解本周所有题目的解题思路

---

**本周总结**：Week1主要学习了Java集合框架的核心数据结构（ArrayList、HashMap、ConcurrentHashMap、LinkedHashMap）以及泛型、异常、String等基础知识。重点掌握HashMap和ConcurrentHashMap的底层原理，这是面试高频考点。LRU缓存的实现是 LinkedHashMap 的典型应用，务必掌握。

**下周预告**：Week2将进入Java并发编程，重点学习线程基础、synchronized、volatile、ThreadLocal、线程池和AQS。这是Java后端面试的重中之重，建议提前预习线程的基本概念。

---

## 扩展阅读

1. **JDK源码阅读**：建议从ArrayList、HashMap开始，逐步阅读ConcurrentHashMap源码
2. **深入理解Java虚拟机**：了解对象内存布局、垃圾回收机制
3. **并发编程实战**：为Week2的并发编程打好理论基础
4. **LeetCode刷题**：每天保持1-2道题的节奏，重点练习链表和HashMap相关题目
