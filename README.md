# Java职业转型计划 — 执行指南

## 📁 目录结构

```
JavaCareerPlan/
├── 学习进度跟踪.md          # 总进度跟踪
├── Week01-Java基础/         # 每日任务文件
├── Week02-多线程并发/
├── Week03-JVM/
├── Week04-IO与网络/
├── Week05-Spring核心/
├── Week06-SpringBoot深入/
├── Week07-SpringCloud/
├── Week08-中间件/
├── Week09-项目包装/
├── Week10-Demo项目/
├── Week11-面试冲刺/
├── Week12-面试冲刺续/
├── LeetCode刷题记录/        # LeetCode刷题统计
└── 简历与面试/              # 简历模板和面试准备
```

---

## 🚀 快速开始

### Step 1：开始学习
1. 打开 `Week01-Java基础/Day01-ArrayList与LinkedList.md`
2. 按照任务清单学习
3. 完成后在任务前打勾 `[x]`
4. 在LeetCode刷对应题目
5. 记录学习笔记

### Step 2：跟踪进度
1. 每天更新 `学习进度跟踪.md` 中的每日记录
2. 每周日更新周完成状态
3. 完成里程碑检查点后更新状态

### Step 3：准备面试
1. 第9周开始看 `简历与面试/简历模板.md`
2. 第11周开始看 `简历与面试/面试准备手册.md`
3. 模拟面试并记录复盘

---

## 📋 每日任务格式

每个Day文件包含：
- **📖 理论学习**：需要学习的知识点（带checkbox）
- **🎯 验收标准**：学完后能回答什么问题
- **💻 LeetCode**：当天要刷的题目（带链接）
- **📝 学习笔记**：记录学习心得

---

## ⏰ 每日时间分配

```
09:00-12:00  理论学习（视频/书籍）
12:00-13:30  午餐+休息
13:30-15:30  LeetCode刷题（1-2道中等/1道困难）
15:30-17:30  项目实战/简历优化
19:00-21:00  复习笔记 + 模拟面试
```

---

## 📅 12周详细时间规划

### 第1-4周：Java核心基础（共28天）

| 周次 | 日期范围 | 主题 | 每日任务 |
|------|---------|------|---------|
| **W1** | Day 1-7 | Java基础（集合+面向对象） | 理论3h + LeetCode 1-2题 |
| Day 1 | 周一 | ArrayList与LinkedList | 底层结构、扩容机制、时间复杂度 |
| Day 2 | 周二 | HashMap底层原理 | 数组+链表+红黑树、put()流程 |
| Day 3 | 周三 | ConcurrentHashMap | JDK7分段锁 vs JDK8 CAS+synchronized |
| Day 4 | 周四 | LinkedHashMap与LRU | accessOrder、手写LRU缓存 |
| Day 5 | 周五 | 泛型与异常体系 | 类型擦除、PECS、checked/unchecked |
| Day 6 | 周六 | String与字符串 | 不可变性、常量池、intern() |
| Day 7 | 周日 | **复习日** | 默写HashMap put()流程、ConcurrentHashMap原理 |
| **W2** | Day 8-14 | 多线程与并发 | 理论3h + LeetCode 1-2题 |
| Day 8 | 周一 | 线程基础 | 创建方式、6种状态、start() vs run() |
| Day 9 | 周二 | synchronized原理 | Monitor对象、锁升级（偏向→轻量→重量） |
| Day 10 | 周三 | volatile与CAS | 内存屏障、Unsafe、ABA问题 |
| Day 11 | 周四 | ReentrantLock | 公平锁/非公平锁、Condition、ReadWriteLock |
| Day 12 | 周五 | 线程池 | 7个参数、4种拒绝策略、execute()流程 |
| Day 13 | 周六 | JUC并发包 | CountDownLatch、CyclicBarrier、Semaphore |
| Day 14 | 周日 | **复习日** | 默写线程池execute()流程、线程状态转换图 |
| **W3** | Day 15-21 | JVM | 理论3h + LeetCode 1-2题 |
| Day 15 | 周一 | JVM内存模型 | 堆/方法区/栈、各区域作用 |
| Day 16 | 周二 | 垃圾回收 | GC Roots、标记-清除/复制/标记-整理 |
| Day 17 | 周三 | CMS和G1收集器 | CMS 4阶段、G1 Region化设计 |
| Day 18 | 周四 | 类加载机制 | 双亲委派、打破双亲委派（SPI/Tomcat） |
| Day 19 | 周五 | JVM调优 | 常用参数、OOM排查、jstack/jmap |
| Day 20 | 周六 | JVM实战 | Arthas使用、GC日志分析 |
| Day 21 | 周日 | **复习日** | 默画JVM内存模型、CMS vs G1对比表 |
| **W4** | Day 22-28 | IO与网络 | 理论3h + LeetCode 1-2题 |
| Day 22 | 周一 | BIO/NIO/AIO | 三种IO模型对比、适用场景 |
| Day 23 | 周二 | NIO核心组件 | Channel、Buffer、Selector |
| Day 24 | 周三 | Netty核心概念 | EventLoop、ChannelPipeline、Reactor模式 |
| Day 25 | 周四 | TCP三次握手四次挥手 | 流程图、TIME_WAIT状态 |
| Day 26 | 周五 | HTTP/HTTPS | TLS握手流程、HTTP2/3区别 |
| Day 27 | 周六 | Netty实战 | 编解码器、心跳机制、断线重连 |
| Day 28 | 周日 | **阶段性复习** | 闭卷默写：HashMap/线程池/JVM/TCP |

---

### 第5-8周：Spring全家桶 + 中间件（共28天）

| 周次 | 日期范围 | 主题 | 每日任务 |
|------|---------|------|---------|
| **W5** | Day 29-35 | Spring核心 | 理论3h + LeetCode 1-2题 |
| Day 29 | 周一 | Spring IOC启动流程 | BeanFactory→ApplicationContext→refresh() |
| Day 30 | 周二 | Bean生命周期 | 实例化→属性注入→初始化→销毁 |
| Day 31 | 周三 | AOP原理 | JDK动态代理 vs CGLIB |
| Day 32 | 周四 | 循环依赖 | 三级缓存、@Transactional原理 |
| Day 33 | 周五 | Spring事件机制 | ApplicationEvent、条件注解 |
| Day 34 | 周六 | 事务传播机制 | 7种传播机制、事务失效场景 |
| Day 35 | 周日 | **复习日** | 默写IOC流程、Bean生命周期 |
| **W6** | Day 36-42 | Spring Boot深入 | 理论3h + LeetCode 1-2题 |
| Day 36 | 周一 | 自动配置原理 | @EnableAutoConfiguration、spring.factories |
| Day 37 | 周二 | Starter机制 | 自定义Starter步骤 |
| Day 38 | 周三 | 配置加载顺序 | 17种加载位置和优先级 |
| Day 39 | 周四 | Actuator | 健康检查、指标监控 |
| Day 40 | 周五 | 异常处理 | @ControllerAdvice + @ExceptionHandler |
| Day 41 | 周六 | 日志框架 | SLF4J + Logback |
| Day 42 | 周日 | **复习日** | 默写自动配置原理、Starter开发步骤 |
| **W7** | Day 43-49 | Spring Cloud | 理论3h + LeetCode 1-2题 |
| Day 43 | 周一 | Nacos注册中心 | 服务注册/发现、AP/CP模式 |
| Day 44 | 周二 | Nacos配置中心 | 动态刷新、命名空间/分组 |
| Day 45 | 周三 | Gateway | 路由、断言、过滤器链、限流 |
| Day 46 | 周四 | Feign | 声明式调用、负载均衡、fallback |
| Day 47 | 周五 | Sentinel | 流量控制、熔断降级 |
| Day 48 | 周六 | 分布式事务 | Seata的AT/TCC/Saga模式 |
| Day 49 | 周日 | **复习日** | 闭卷画出微服务架构图 |
| **W8** | Day 50-56 | 中间件 | 理论3h + LeetCode 1-2题 |
| Day 50 | 周一 | MySQL索引 | B+树、聚簇索引、覆盖索引 |
| Day 51 | 周二 | MySQL事务 | 隔离级别、MVCC原理 |
| Day 52 | 周三 | MySQL优化 | EXPLAIN、慢SQL、索引失效 |
| Day 53 | 周四 | Redis基础 | 数据结构、持久化、缓存策略 |
| Day 54 | 周五 | Redis高可用 | 主从、哨兵、Cluster、分布式锁 |
| Day 55 | 周六 | Kafka基础 | 架构、生产者、消费者、消息不丢失 |
| Day 56 | 周日 | **阶段性复习** | 闭卷默写：IOC/Bean/SpringBoot/MySQL/Redis |

---

### 第9-10周：项目包装 + 简历（共14天）

| 周次 | 日期范围 | 主题 | 每日任务 |
|------|---------|------|---------|
| **W9** | Day 57-63 | 项目包装 | 包装对日项目+AI项目 |
| Day 57 | 周一 | STAR法则 | 学习项目描述方法 |
| Day 58 | 周二 | 对日项目包装 | 微服务架构、Spanner、Docker |
| Day 59 | 周三 | 对日项目包装 | CI/CD、日志监控、性能优化 |
| Day 60 | 周四 | AI项目包装 | OpenAI API、RAG、LLM应用 |
| Day 61 | 周五 | 简历初稿 | 个人信息+教育+技能+项目 |
| Day 62 | 周六 | 简历优化 | 排版、用词、英文简历 |
| Day 63 | 周日 | **复习日** | 能流畅讲述2个项目经验 |
| **W10** | Day 64-70 | Demo项目 | 搭建电商秒杀系统 |
| Day 64-65 | 周一-周二 | 项目基础 | Spring Boot + Redis + MySQL |
| Day 66-67 | 周三-周四 | 秒杀功能 | Redis预减库存 + Kafka异步下单 |
| Day 68-69 | 周五-周六 | 项目完善 | Sentinel限流 + Actuator监控 |
| Day 70 | 周日 | **项目Review** | GitHub推送、README完善 |

---

### 第11-12周：面试冲刺（共14天）

| 周次 | 日期范围 | 主题 | 每日任务 |
|------|---------|------|---------|
| **W11** | Day 71-77 | 面试冲刺 | 系统设计+算法刷题 |
| Day 71 | 周一 | 系统设计：秒杀 | 多级缓存、库存预扣、异步下单 |
| Day 72 | 周二 | 系统设计：短链 | 发号器、存储、重定向 |
| Day 73 | 周三 | 系统设计：MQ | Kafka架构、消息可靠性 |
| Day 74 | 周四 | 系统设计：分布式锁 | Redis vs Zookeeper |
| Day 75 | 周五 | 算法冲刺 | Hot 100复习（滑动窗口、双指针） |
| Day 76 | 周六 | 算法冲刺 | Hot 100复习（动态规划、贪心） |
| Day 77 | 周日 | **模拟面试** | 45min技术面 + 15min HR面 |
| **W12** | Day 78-84 | 面试冲刺（续） | 行为面试+投递简历 |
| Day 78 | 周一 | 行为面试 | 离职原因、项目难点、职业规划 |
| Day 79 | 周二 | 技术难点 | 微服务拆分、性能优化、故障排查 |
| Day 80 | 周三 | 综合复习 | Java+Spring+中间件 面经 |
| Day 81 | 周四 | 综合复习 | 系统设计面经 |
| Day 82 | 周五 | 投递准备 | 筛选岗位、优化简历 |
| Day 83 | 周六 | 开始投递 | 目标20+岗位 |
| Day 84 | 周日 | **全面复习** | 12周学习成果总结 |

---

## 📊 12周进度一览

```
Week 01 ████████░░░░░░░░ Java基础（集合）
Week 02 ░░░░░░░░░░░░░░░░ 多线程并发
Week 03 ░░░░░░░░░░░░░░░░ JVM
Week 04 ░░░░░░░░░░░░░░░░ IO与网络
Week 05 ░░░░░░░░░░░░░░░░ Spring核心
Week 06 ░░░░░░░░░░░░░░░░ Spring Boot深入
Week 07 ░░░░░░░░░░░░░░░░ Spring Cloud
Week 08 ░░░░░░░░░░░░░░░░ 中间件
Week 09 ░░░░░░░░░░░░░░░░ 项目包装
Week 10 ░░░░░░░░░░░░░░░░ Demo项目
Week 11 ░░░░░░░░░░░░░░░░ 面试冲刺
Week 12 ░░░░░░░░░░░░░░░░ 面试冲刺（续）
```

---

## 📊 关键资源

### 学习资源
- **B站**：韩顺平Java教程、黑马程序员、周志明JVM
- **GitHub**：JavaGuide、CS-Notes、advanced-java
- **书籍**：《Java并发编程的艺术》、《深入理解JVM》、《Netty实战》

### 面试资源
- **牛客网**：Java面经汇总
- **LeetCode**：Hot 100 + 剑指Offer
- **BOSS直聘**：搜索目标岗位JD，对照查漏补缺

---

## ✅ 里程碑检查点

| 检查点 | 时间 | 通过标准 |
|--------|------|---------|
| Java基础 | 第4周末 | 能手写HashMap、解释synchronized原理、画出JVM内存模型 |
| 框架+中间件 | 第8周末 | 能画出微服务架构图、解释Spring自动配置原理 |
| 简历+项目 | 第10周末 | 简历投出去有回音、GitHub项目能跑 |
| 面试冲刺 | 第12周末 | 模拟面试通过率 > 70% |

---

## 💡 Tips

1. **不要跳过复习日**：每周日的复习很重要，巩固记忆
2. **LeetCode重质不重量**：理解比刷题数量更重要
3. **笔记要及时整理**：当天学的内容当天记录
4. **模拟面试要认真**：找朋友或AI模拟，不要跳过
5. **简历要提前准备**：不要等到最后一刻

---

**祝你学习顺利，早日拿到心仪的Offer！**