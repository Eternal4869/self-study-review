# Day06 详解 — String与字符串

## 今日目标
- 理解String不可变性原理
- 掌握StringBuilder和StringBuffer的区别
- 理解字符串常量池的工作原理
- 掌握String.intern()方法的使用
- 能够回答面试中的字符串相关问题

---

## 一、String不可变性原理

### 1.1 底层结构（final char[] value）

String类在Java中被设计为不可变类，其底层实现是一个**final修饰的字符数组**。

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    // 用于存储字符串的字符数组，final修饰
    private final char value[];
    
    // 缓存字符串的哈希码
    private int hash; // Default to 0
    
    // JDK 9+ 使用 byte[] + coder 压缩存储
    // private final byte[] value;
    // private final byte coder;
}
```

**关键点：**
- `value`数组被`final`修饰，意味着**引用不可变**（一旦初始化就不能指向其他数组）
- 但数组本身的内容是可变的（理论上），不过String类没有提供任何修改`value`数组的方法
- JDK 9+ 对String进行了优化，使用`byte[]` + `coder`来节省内存（Latin1编码只需1字节，UTF-16编码需要2字节）

### 1.2 为什么设计为不可变

**1. 字符串常量池的需要**
- 字符串常量池要求相同的字面量共享同一个对象
- 如果字符串可变，修改一个字符串会影响其他引用

**2. 安全性**
- String常用于网络连接、文件路径、类加载等敏感操作
- 可变字符串可能导致安全漏洞

**3. 线程安全**
- 不可变对象天然线程安全，无需同步

**4. HashCode缓存**
- 字符串的哈希码在创建时就可以计算并缓存
- HashMap等集合中作为key时性能更好

**5. 性能优化**
- 字符串池可以有效利用内存
- 减少内存占用和GC压力

### 1.3 不可变的影响

✅ **优势：**
```java
String s1 = "hello";
String s2 = "hello";
// s1和s2指向同一个对象，节省内存
System.out.println(s1 == s2); // true

// 作为HashMap的key，hashCode不变
Map<String, Integer> map = new HashMap<>();
map.put("hello", 1);
// 即使获取key后，哈希码不会改变
```

❌ **劣势：**
```java
// 字符串拼接产生大量临时对象
String result = "";
for (int i = 0; i < 10000; i++) {
    result += i; // 每次都创建新的String对象
}

// 更高效的方式
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10000; i++) {
    sb.append(i); // 在同一个对象上操作
}
String result = sb.toString();
```

**面试问题：String为什么是不可变的？**
> 答：String被设计为不可变主要有以下原因：
> 1. **字符串常量池要求**：相同的字面量需要共享同一个对象
> 2. **安全性**：String常用于网络连接、文件路径等敏感操作
> 3. **线程安全**：不可变对象天然线程安全
> 4. **HashCode缓存**：可以缓存哈希码，提高HashMap等集合的性能

---

## 二、StringBuilder vs StringBuffer

### 2.1 StringBuilder（非线程安全）

StringBuilder是可变的字符序列，提供了与StringBuffer兼容的API，但**不保证同步**。

```java
public final class StringBuilder extends AbstractStringBuilder implements java.io.Serializable, CharSequence {
    // 继承自AbstractStringBuilder，内部也是char[]数组
    // 但value数组不是final的，可以扩容
    
    @Override
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }
    
    @Override
    public String toString() {
        // 创建新的String对象
        return new String(value, 0, count);
    }
}
```

**特点：**
- 非线程安全，性能略高
- 适用于单线程环境下的字符串拼接
- 默认初始容量为16

### 2.2 StringBuffer（线程安全）

StringBuffer是线程安全的可变字符序列，其方法使用了**synchronized**修饰。

```java
public final class StringBuffer extends AbstractStringBuilder implements java.io.Serializable, CharSequence {
    // 使用toStringCache缓存toString结果
    private transient char[] toStringCache;
    
    @Override
    public synchronized StringBuffer append(String str) {
        toStringCache = null; // 修改时清空缓存
        super.append(str);
        return this;
    }
    
    @Override
    public synchronized String toString() {
        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);
        }
        return new String(toStringCache, true);
    }
}
```

**特点：**
- 线程安全，方法使用synchronized修饰
- 有toStringCache缓存，多次调用toString()时性能更好
- 适用于多线程环境

### 2.3 性能对比

| 特性 | String | StringBuilder | StringBuffer |
|------|--------|---------------|--------------|
| **可变性** | 不可变 | 可变 | 可变 |
| **线程安全** | 是（不可变） | ❌ 否 | ✅ 是 |
| **性能** | 字符串拼接差 | 最好 | 较好 |
| **适用场景** | 字符串不经常变化 | 单线程拼接 | 多线程拼接 |
| **扩容机制** | 不扩容 | 1.5倍扩容 | 1.5倍扩容 |

**代码示例：**
```java
// String拼接 - 产生大量临时对象
long start = System.currentTimeMillis();
String str = "";
for (int i = 0; i < 100000; i++) {
    str += "a"; // 每次都创建新String对象
}
System.out.println("String拼接耗时: " + (System.currentTimeMillis() - start) + "ms");

// StringBuilder拼接 - 性能最好
start = System.currentTimeMillis();
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100000; i++) {
    sb.append("a");
}
str = sb.toString();
System.out.println("StringBuilder拼接耗时: " + (System.currentTimeMillis() - start) + "ms");

// StringBuffer拼接 - 线程安全但有同步开销
start = System.currentTimeMillis();
StringBuffer buffer = new StringBuffer();
for (int i = 0; i < 100000; i++) {
    buffer.append("a");
}
str = buffer.toString();
System.out.println("StringBuffer拼接耗时: " + (System.currentTimeMillis() - start) + "ms");
```

**面试问题：StringBuilder和StringBuffer的区别？**
> 答：主要区别在于线程安全性：
> 1. **StringBuilder**：非线程安全，性能更高，适用于单线程环境
> 2. **StringBuffer**：线程安全（方法使用synchronized修饰），适用于多线程环境
> 3. 两者都继承自AbstractStringBuilder，底层都是char[]数组，扩容机制相同（1.5倍）

---

## 三、字符串常量池

### 3.1 JDK6 vs JDK7+ 位置变化

**JDK 6：字符串常量池在方法区（永久代）**
- 位置：永久代（PermGen）
- 大小：默认只有4MB，容易出现`java.lang.OutOfMemoryError: PermGen space`
- GC：Full GC时才会回收

**JDK 7+：字符串常量池移到堆中**
- 位置：堆（Heap）
- 大小：默认为`-XX:StringTableSize=1009`（可配置）
- GC：Minor GC时就可以回收

**变化原因：**
1. 永久代大小有限，容易OOM
2. 字符串常量池中的字符串可能很多，放在堆中更容易回收
3. 提高GC效率

### 3.2 常量池工作原理

字符串常量池是一个**哈希表**，用于存储字符串字面量的引用。

**工作流程：**
1. 编译期：将字符串字面量存入常量池
2. 类加载：将常量池中的字符串加载到运行时常量池
3. 运行期：使用`String.intern()`方法将字符串放入常量池

```java
// 字面量直接使用常量池中的对象
String s1 = "hello"; // 直接使用常量池中的"hello"
String s2 = "hello"; // 使用同一个对象
System.out.println(s1 == s2); // true

// new String()创建新对象
String s3 = new String("hello"); // 在堆中创建新对象
System.out.println(s1 == s3); // false
System.out.println(s1.equals(s3)); // true
```

### 3.3 代码示例

**示例1：字符串字面量**
```java
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");

// s1和s2指向常量池中的同一个对象
System.out.println(s1 == s2); // true
System.out.println(s1.equals(s2)); // true

// s3是堆中的新对象
System.out.println(s1 == s3); // false
System.out.println(s1.equals(s3)); // true

// intern()方法返回常量池中的引用
String s4 = s3.intern();
System.out.println(s1 == s4); // true
```

**示例2：字符串拼接**
```java
String s1 = "hello";
String s2 = "world";
String s3 = "helloworld";

// 编译期优化：直接拼接字面量
String s4 = "hello" + "world"; // 等同于 "helloworld"
System.out.println(s3 == s4); // true

// 运行时拼接：创建新的String对象
String s5 = s1 + "world"; // 等同于 new StringBuilder().append(s1).append("world").toString()
System.out.println(s3 == s5); // false

// final变量在编译期优化
final String s6 = "hello";
String s7 = s6 + "world"; // 编译期优化为 "helloworld"
System.out.println(s3 == s7); // true
```

**示例3：JDK7+ 常量池位置变化**
```java
// JDK6中，intern()会将字符串复制到永久代
// JDK7+中，intern()直接在堆中记录引用

String s1 = new String("a") + new String("b"); // 堆中创建"ab"
s1.intern(); // JDK7+: 直接在常量池记录堆中"ab"的引用
String s2 = "ab"; // 使用常量池中的引用
System.out.println(s1 == s2); // JDK7+: true, JDK6: false
```

**面试问题：字符串常量池的位置变化？**
> 答：JDK6中字符串常量池在永久代（PermGen），容易出现OOM；JDK7+移到了堆中，便于GC回收，提高了性能。

---

## 四、String.intern()方法

### 4.1 方法原理

`String.intern()`方法的作用是将字符串添加到字符串常量池中，并返回常量池中的引用。

```java
// native方法，由JVM实现
public native String intern();
```

**工作流程：**
1. 如果常量池中已经包含等于此String对象的字符串，则返回常量池中的字符串引用
2. 否则，将此String对象添加到常量池中，并返回此String对象的引用

**JDK7+的变化：**
- JDK6：将字符串从堆复制到永久代，返回永久代中的引用
- JDK7+：在堆中记录常量池的引用，避免复制

### 4.2 使用场景

**场景1：减少重复字符串**
```java
// 从数据库或文件读取大量重复字符串
List<String> list = new ArrayList<>();
for (int i = 0; i < 100000; i++) {
    list.add(new String("hello")); // 创建大量重复对象
}

// 使用intern()减少内存占用
for (int i = 0; i < 100000; i++) {
    list.add(new String("hello").intern()); // 所有"hello"共享同一个引用
}
```

**场景2：字符串比较优化**
```java
String s1 = new String("hello");
String s2 = new String("hello");

// 使用intern()后可以使用==比较
if (s1.intern() == s2.intern()) {
    System.out.println("相同的字符串");
}
```

**场景3：作为HashMap的key**
```java
Map<String, Integer> map = new HashMap<>();
String key = new String("hello");

// 使用intern()确保key的唯一性
map.put(key.intern(), 1);
```

### 4.3 注意事项和风险

**风险1：性能问题**
```java
// 不推荐：频繁调用intern()会有性能开销
for (int i = 0; i < 1000000; i++) {
    String s = new String("hello" + i).intern(); // 每次都要查找常量池
}

// 推荐：只对重复字符串使用intern()
Set<String> set = new HashSet<>();
for (int i = 0; i < 1000000; i++) {
    String s = new String("hello").intern(); // 只有第一次会添加到常量池
    set.add(s);
}
```

**风险2：内存泄漏**
```java
// JDK6中，intern()将字符串放在永久代，Full GC才会回收
// 可能导致永久代OOM

// JDK7+中，常量池在堆中，更容易回收
// 但仍然需要注意使用场景
```

**风险3：字符串比较混淆**
```java
String s1 = new String("hello");
String s2 = "hello";

// intern()返回的是常量池中的引用
System.out.println(s1.intern() == s2); // true

// 但s1本身还是堆中的对象
System.out.println(s1 == s2); // false
System.out.println(s1 == s1.intern()); // false
```

**最佳实践：**
```java
// ✅ 推荐：对重复字符串使用intern()
String key = getKeyFromDB().intern(); // 假设key是重复的

// ❌ 不推荐：对动态生成的字符串使用intern()
for (int i = 0; i < 100000; i++) {
    String key = ("key" + i).intern(); // 动态字符串，每次都不同
}
```

**面试问题：String.intern()的作用和风险？**
> 答：`intern()`方法将字符串添加到常量池并返回引用，可以减少重复字符串的内存占用。但存在以下风险：
> 1. **性能开销**：频繁调用会有查找常量池的开销
> 2. **内存泄漏**：JDK6中字符串放在永久代，容易OOM
> 3. **使用混淆**：`intern()`返回的引用可能与原对象不同
> 4. **适用场景**：只对重复字符串使用，避免对动态字符串使用

---

## 五、LeetCode题目解析

### [5. 最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)（Medium）

#### 题目描述
给定一个字符串 `s`，找到 `s` 中最长的回文子串。

**示例：**
- 输入：`s = "babad"`
- 输出：`"bab"` 或 `"aba"`
- 解释：`"bab"` 和 `"aba"` 都是最长回文子串

#### 思路分析

**方法1：暴力解法（超时）**
- 枚举所有子串，检查是否回文
- 时间复杂度：O(n³)

**方法2：中心扩展法（推荐）**
- 从每个字符（或两个字符）向两边扩展
- 时间复杂度：O(n²)
- 空间复杂度：O(1)

**方法3：Manacher算法（最优）**
- 时间复杂度：O(n)
- 空间复杂度：O(n)
- 实现复杂，面试中一般不需要

#### Java代码

**方法1：中心扩展法**
```java
class Solution {
    public String longestPalindrome(String s) {
        if (s == null || s.length() < 2) {
            return s;
        }
        
        int start = 0;
        int maxLength = 1;
        
        // 遍历每个字符作为中心
        for (int i = 0; i < s.length(); i++) {
            // 以i为中心的回文（奇数长度）
            int len1 = expandAroundCenter(s, i, i);
            // 以i和i+1为中心的回文（偶数长度）
            int len2 = expandAroundCenter(s, i, i + 1);
            
            int len = Math.max(len1, len2);
            if (len > maxLength) {
                start = i - (len - 1) / 2;
                maxLength = len;
            }
        }
        
        return s.substring(start, start + maxLength);
    }
    
    // 中心扩展方法
    private int expandAroundCenter(String s, int left, int right) {
        // 向两边扩展，直到不是回文
        while (left >= 0 && right < s.length() && s.charAt(left) == s.charAt(right)) {
            left--;
            right++;
        }
        // 返回回文长度
        return right - left - 1;
    }
}
```

**方法2：动态规划（更容易理解）**
```java
class Solution {
    public String longestPalindrome(String s) {
        int n = s.length();
        if (n < 2) {
            return s;
        }
        
        // dp[i][j]表示s[i..j]是否是回文
        boolean[][] dp = new boolean[n][n];
        
        int start = 0;
        int maxLength = 1;
        
        // 单个字符都是回文
        for (int i = 0; i < n; i++) {
            dp[i][i] = true;
        }
        
        // 从短到长枚举子串长度
        for (int len = 2; len <= n; len++) {
            for (int i = 0; i <= n - len; i++) {
                int j = i + len - 1;
                
                if (len == 2) {
                    // 两个字符：相等就是回文
                    dp[i][j] = (s.charAt(i) == s.charAt(j));
                } else {
                    // 更长：首尾相等且内部是回文
                    dp[i][j] = (s.charAt(i) == s.charAt(j)) && dp[i + 1][j - 1];
                }
                
                if (dp[i][j] && len > maxLength) {
                    start = i;
                    maxLength = len;
                }
            }
        }
        
        return s.substring(start, start + maxLength);
    }
}
```

#### 复杂度分析

**中心扩展法：**
- 时间复杂度：O(n²)，遍历n个中心，每个中心扩展最多n次
- 空间复杂度：O(1)，只使用常数空间

**动态规划法：**
- 时间复杂度：O(n²)，两层循环
- 空间复杂度：O(n²)，二维DP数组

**面试扩展：**
- 如果面试官问Manacher算法，可以简单介绍其思想
- Manacher算法利用回文的对称性，避免重复计算
- 时间复杂度O(n)，但实现复杂

---

## 六、今日学习要点总结

### 核心要点归纳

1. **String不可变性**
   - 底层是final char[]数组
   - 设计原因：常量池、安全性、线程安全、性能
   - 影响：字符串拼接产生大量临时对象

2. **StringBuilder vs StringBuffer**
   - StringBuilder：非线程安全，性能更好
   - StringBuffer：线程安全（synchronized），适用于多线程
   - 扩容机制：1.5倍

3. **字符串常量池**
   - JDK6：永久代，容易OOM
   - JDK7+：堆中，便于GC回收
   - 工作原理：哈希表存储字符串引用

4. **String.intern()**
   - 作用：将字符串添加到常量池
   - 风险：性能开销、内存泄漏、使用混淆
   - 最佳实践：只对重复字符串使用

### 验收清单

- [ ] 能解释String为什么是不可变的
- [ ] 能说明StringBuilder和StringBuffer的区别
- [ ] 能解释字符串常量池的位置变化
- [ ] 能解释String.intern()的作用和风险
- [ ] 能完成最长回文子串的题目
- [ ] 能解释中心扩展法的原理
- [ ] 能解释动态规划法的原理

---

## 七、练习建议

### 代码练习
1. **字符串操作练习**
   - 实现字符串反转
   - 判断字符串是否是回文
   - 统计字符串中字符出现次数

2. **性能测试**
   - 测试String、StringBuilder、StringBuffer的拼接性能
   - 测试intern()的性能影响

3. **常量池验证**
   - 验证字面量、new String()、intern()的区别
   - 验证JDK版本差异

### 面试准备
1. **高频问题**
   - String为什么是不可变的？
   - StringBuilder和StringBuffer的区别？
   - 字符串常量池的位置变化？
   - String.intern()的作用和风险？
   - 字符串拼接的性能问题？

2. **代码实现**
   - 手写字符串反转
   - 手写回文判断
   - 理解字符串池的工作原理

### LeetCode练习
1. **相关题目**
   - [3. 无重复字符的最长子串](https://leetcode.cn/problems/longest-substring-without-repeating-characters/)（Medium）
   - [28. 找出字符串中第一个匹配项的下标](https://leetcode.cn/problems/find-the-index-of-the-first-occurrence-in-a-string/)（Easy）
   - [647. 回文子串](https://leetcode.cn/problems/palindromic-substrings/)（Medium）

2. **进阶题目**
   - [131. 分割回文串](https://leetcode.cn/problems/palindrome-partitioning/)（Medium）
   - [516. 最长回文子序列](https://leetcode.cn/problems/longest-palindromic-subsequence/)（Medium）

---

## 八、扩展阅读

### 1. 官方文档
- [String类官方文档](https://docs.oracle.com/javase/8/docs/api/java/lang/String.html)
- [StringBuilder类官方文档](https://docs.oracle.com/javase/8/docs/api/java/lang/StringBuilder.html)
- [StringBuffer类官方文档](https://docs.oracle.com/javase/8/docs/api/java/lang/StringBuffer.html)

### 2. 深入理解
- 《Java核心技术 卷I》第4章 - 对象与类
- 《Effective Java》第10章 - 序列化 - 关于String的不可变性
- 《深入理解Java虚拟机》第2章 - Java内存区域 - 字符串常量池

### 3. 实践建议
- 在IDE中调试字符串操作，观察内存变化
- 使用JVisualVM观察字符串常量池
- 阅读String类的源码，理解JDK版本差异

### 4. 常见面试题集
- 字符串相关的面试题通常考察：
  1. 对Java基础的理解深度
  2. 对内存模型的理解
  3. 对性能优化的意识
  4. 解决实际问题的能力

### 5. 学习资源
- [Java String 深度解析](https://www.cnblogs.com/xiaoxi/p/6036548.html)
- [深入理解Java字符串常量池](https://blog.csdn.net/justloveyou_/article/details/60883291)
- [String.intern()方法详解](https://www.cnblogs.com/xiahj/p/8182861.html)

---

**下一步学习建议：**
- 完成本日的LeetCode题目
- 练习字符串操作相关的代码
- 准备字符串相关的面试问题
- 预习Day07的复习日内容