# Day33 详解 — Spring事件机制与条件装配

## 〇、Spring事件机制版本演进

| 版本 | 特性变化 |
|------|----------|
| **Spring 2.x** | ApplicationEvent + ApplicationListener 接口式监听 |
| **Spring 3.x** | 泛型事件支持，PayloadApplicationEvent |
| **Spring 4.2+** | `@EventListener` 注解、`@TransactionalEventListener` 事务事件、条件监听（condition / SpEL） |
| **Spring 5.x** | `@EventListener` 原生支持异步（配合 `@Async`），支持响应式事件 |
| **Spring 6.x / Boot 3.x** | 基于 Jakarta EE，事件核心 API **基本不变**；Boot 3.x 内置条件注解更丰富 |

### Spring事件 vs 消息队列

| 维度 | Spring事件 | 消息队列（Kafka/RabbitMQ） |
|------|-----------|---------------------------|
| **作用范围** | 同一JVM进程内 | 跨进程、跨服务 |
| **可靠性** | 丢失不可恢复 | 持久化、重试、死信队列 |
| **性能** | 极低延迟（微秒级） | 毫秒级 |
| **事务支持** | 天然支持事务事件 | 最终一致性 |
| **适用场景** | 业务解耦、日志、通知 | 微服务间异步通信 |

> **原则**：进程内解耦用Spring事件，跨服务通信用消息队列。

---

## 一、事件机制概述

### 1.1 观察者模式

Spring事件机制是**观察者模式**的标准实现：

- **Subject（被观察者）**：事件发布者 `ApplicationEventPublisher`
- **Event（事件）**：`ApplicationEvent`
- **Observer（观察者）**：`ApplicationListener`

### 1.2 Spring事件机制架构

```
┌──────────────────────┐
│  ApplicationContext   │  ← 同时也是 EventPublisher + EventMulticaster 的持有者
│                       │
│  publishEvent(event)  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────┐
│  ApplicationEventMulticaster │  ← 广播器（默认 SimpleApplicationEventMulticaster）
│                              │
│  multicastEvent(event)       │
│  ┌────────────────────────┐ │
│  │ 1. resolveDefaultEventType(event) │
│  │ 2. getApplicationListeners(event) │
│  │ 3. invokeListener(listener,event) │
│  └────────────────────────┘ │
└──────────┬───────────────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
┌────────┐ ┌────────┐ ┌────────┐
│Listener│ │Listener│ │Listener│  ← 多个监听器并行/串行执行
│   A    │ │   B    │ │   C    │
└────────┘ └────────┘ └────────┘
```

**关键角色**：

| 角色 | 接口/类 | 说明 |
|------|---------|------|
| **事件** | `ApplicationEvent` | 所有事件的基类 |
| **发布者** | `ApplicationEventPublisher` | `ApplicationContext` 实现了该接口 |
| **广播器** | `ApplicationEventMulticaster` | 负责将事件分发给所有匹配的监听器 |
| **监听器** | `ApplicationListener<E>` / `@EventListener` | 接收并处理事件 |

---

## 二、ApplicationEvent和ApplicationListener

### 2.1 ApplicationEvent基类

```java
public abstract class ApplicationEvent extends EventObject implements Serializable {

    private final long timestamp;  // 事件发生时间戳

    public ApplicationEvent(Object source) {
        super(source);
        this.timestamp = System.currentTimeMillis();
    }

    public final long getTimestamp() {
        return this.timestamp;
    }
}
```

> Spring 4.2 起，自定义事件不再强制继承 `ApplicationEvent`，普通 POJO 即可。

### 2.2 ApplicationListener接口

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    void onApplicationEvent(E event);
}
```

### 2.3 传统方式实现事件监听

```java
// ① 定义事件
public class OrderCreatedEvent extends ApplicationEvent {

    private final Long orderId;
    private final BigDecimal amount;

    public OrderCreatedEvent(Object source, Long orderId, BigDecimal amount) {
        super(source);
        this.orderId = orderId;
        this.amount = amount;
    }
    // getters...
}

// ② 实现监听器（实现接口方式）
@Component
public class OrderCreatedListener implements ApplicationListener<OrderCreatedEvent> {

    @Override
    public void onApplicationEvent(OrderCreatedEvent event) {
        System.out.println("收到订单创建事件: " + event.getOrderId());
    }
}

// ③ 发布事件
@Component
public class OrderService {

    @Autowired
    private ApplicationEventPublisher publisher;

    public void createOrder(Long orderId, BigDecimal amount) {
        // 业务逻辑...
        publisher.publishEvent(new OrderCreatedEvent(this, orderId, amount));
    }
}
```

### 2.4 完整示例代码

```java
@SpringBootApplication
public class EventDemoApplication implements CommandLineRunner {

    @Autowired
    private OrderService orderService;

    public static void main(String[] args) {
        SpringApplication.run(EventDemoApplication.class, args);
    }

    @Override
    public void run(String... args) {
        orderService.createOrder(1001L, new BigDecimal("99.99"));
    }
}
```

---

## 三、@EventListener注解

### 3.1 基本用法

`@EventListener` 提供声明式事件监听，**无需实现接口**：

```java
@Component
public class OrderEventListener {

    @EventListener
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Spring自动识别参数类型 = 要监听的事件类型
        System.out.println("[@EventListener] 订单创建: " + event.getOrderId());
    }
}
```

> 方法名随意，关键在**参数类型**。支持**一个方法监听多种类型**（多个参数）。

```java
// 监听多种类型事件 —— 不推荐，建议一个方法一个类型
@EventListener
public void handleEvents(ApplicationEvent event) {  // 监听所有事件
    if (event instanceof OrderCreatedEvent orderEvent) {
        // ...
    }
}
```

### 3.2 条件监听（condition属性 / SpEL表达式）

通过 SpEL 实现**条件过滤**，只有满足条件才触发监听：

```java
@Component
public class ConditionalEventListener {

    // 只有金额 > 100 的订单才触发
    @EventListener(condition = "#event.amount > 100")
    public void handleLargeOrder(OrderCreatedEvent event) {
        System.out.println("大额订单!" + event.getAmount());
    }

    // 结合自定义判断
    @EventListener(condition = "#event.orderId % 2 == 0")
    public void handleEvenIdOrder(OrderCreatedEvent event) {
        System.out.println("偶数ID订单: " + event.getOrderId());
    }
}
```

### 3.3 @TransactionalEventListener（事务事件监听）

**核心痛点**：业务在事务中操作数据库，但事件监听应该等事务提交后再执行（如发送消息、清理缓存）。

`@TransactionalEventListener` 绑定到**事务阶段**（Spring 4.2+）：

| 阶段 | 枚举值 | 含义 |
|------|--------|------|
| 提交后 | `TransactionPhase.AFTER_COMMIT` | 事务提交成功后执行（**默认**） |
| 回滚后 | `TransactionPhase.AFTER_ROLLBACK` | 事务回滚后执行 |
| 完成后 | `TransactionPhase.AFTER_COMPLETION` | 事务结束（不论成功或失败） |
| 提交前 | `TransactionPhase.BEFORE_COMMIT` | 事务提交前执行 |

```java
@Component
public class TransactionalEventListenerDemo {

    // ✅ 事务提交后发送邮件（默认 AFTER_COMMIT）
    @TransactionalEventListener
    public void sendEmailAfterCommit(OrderCreatedEvent event) {
        // 此时数据库已提交，数据一定可见
        emailService.send(event.getOrderId(), "订单已创建");
    }

    // ✅ 回滚后记录异常
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void logOnRollback(OrderCreatedEvent event) {
        log.error("订单创建失败，事务回滚: {}", event.getOrderId());
    }

    // ✅ 不论成败都执行
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void cleanup(OrderCreatedEvent event) {
        // 清理临时资源
    }
}
```

> ⚡ **注意**：`@TransactionalEventListener` 必须在**事务中发布事件**才能生效；非事务环境不会触发。

### 3.4 方法参数自动注入

`@EventListener` 方法支持**自动注入**多个参数：

```java
@EventListener
public void handleWithContext(
        OrderCreatedEvent event,       // ① 事件对象
        @Autowired MyService service   // ② 可以注入Bean（一般用字段注入即可）
) {
    service.process(event.getOrderId());
}
```

---

## 四、异步事件处理

### 4.1 @EnableAsync + @Async

Spring事件默认是**同步阻塞**的 —— 事件发布者会等待所有监听器执行完才返回。异步只需两步：

```java
@Configuration
@EnableAsync  // ① 启用异步支持
public class AsyncConfig {
}
```

```java
@Component
public class AsyncEventListener {

    @Async  // ② 标记监听方法为异步
    @EventListener
    public void handleAsync(OrderCreatedEvent event) {
        System.out.println(Thread.currentThread().getName() + " 异步处理事件");
    }
}
```

### 4.2 配置异步事件执行器

默认使用 `SimpleAsyncTaskExecutor`（每次新建线程），生产环境必须**自定义线程池**：

```java
@Configuration
@EnableAsync
public class AsyncEventConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);           // 核心线程数
        executor.setMaxPoolSize(8);            // 最大线程数
        executor.setQueueCapacity(200);        // 队列容量
        executor.setThreadNamePrefix("event-");// 线程名前缀
        executor.setRejectedExecutionHandler(
            new CallerRunsPolicy()             // 拒绝策略：由调用线程执行
        );
        executor.initialize();
        return executor;
    }
}
```

### 4.3 异步事件完整示例

```java
// 事件
public class UserRegisterEvent {
    private final Long userId;
    private final String username;
    // 构造器 + getter...
}

// 同步监听（与业务一起提交）
@Component
public class UserRegisterSyncListener {
    @EventListener
    public void saveToDatabase(UserRegisterEvent event) {
        // 同步写入数据库，在事务内
    }
}

// 异步监听（不阻塞主流程）
@Component
public class UserRegisterAsyncListener {
    @Async
    @EventListener
    public void sendWelcomeEmail(UserRegisterEvent event) {
        // 异步发邮件
        System.out.println("发送欢迎邮件: " + event.getUsername());
    }

    @Async
    @EventListener
    public void initUserData(UserRegisterEvent event) {
        // 异步初始化用户数据
    }
}
```

### 4.4 Virtual Threads 做异步事件处理（Spring Boot 3.2+）

Spring Boot 3.2+ 支持**虚拟线程**，特别适合大量轻量级异步任务：

```java
@Configuration
@EnableAsync
public class VirtualThreadAsyncConfig {

    @Bean("virtualThreadExecutor")
    public Executor virtualThreadExecutor() {
        // Java 21 Virtual Threads
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}

@Component
public class VirtualThreadEventListener {

    @Async("virtualThreadExecutor")  // 指定使用虚拟线程执行器
    @EventListener
    public void handleEvent(OrderCreatedEvent event) {
        System.out.println(Thread.currentThread() + " - 虚拟线程处理事件");
    }
}
```

### 4.5 同步 vs 异步对比表

| 维度 | 同步事件 | 异步事件 |
|------|---------|---------|
| **执行方式** | 发布者线程内串行执行 | 独立线程池并行执行 |
| **事务** | 与发布者在同一事务 | 独立事务或非事务 |
| **异常处理** | 异常直接抛给发布者 | 异常在异步线程中，**发布者无感知** |
| **顺序** | 严格有序 | 无序，谁先完成谁先 |
| **适用场景** | 业务强依赖的后续操作 | 日志、邮件、通知等非关键操作 |
| **发布耗时** | = 所有监听器耗时之和 | ≈ 0，发布后立即返回 |

---

## 五、自定义事件实战

### 5.1 用户注册事件（完整示例）

```java
// ========== ① 定义事件 ==========
public class UserRegisterEvent {

    private final Long userId;
    private final String username;
    private final String email;
    private final LocalDateTime registerTime;

    public UserRegisterEvent(Long userId, String username, String email) {
        this.userId = userId;
        this.username = username;
        this.email = email;
        this.registerTime = LocalDateTime.now();
    }
    // getters...
}

// ========== ② 业务服务（发布事件）==========
@Service
public class UserService {

    @Autowired
    private ApplicationEventPublisher publisher;

    @Transactional  // 事务中发布事件
    public void register(String username, String email) {
        Long userId = saveUser(username, email);  // 写入数据库
        // 发布事件（参数不再是ApplicationEvent子类也可以）
        publisher.publishEvent(new UserRegisterEvent(userId, username, email));
    }
}

// ========== ③ 同步监听 ==========
@Component
public class UserRegisterSyncListener {

    @EventListener
    public void initUserProfile(UserRegisterEvent event) {
        userProfileService.init(event.getUserId());  // 初始化用户资料
    }
}

// ========== ④ 事务监听 ==========
@Component
public class UserRegisterTransactionalListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterCommit(UserRegisterEvent event) {
        // 事务提交后发送MQ消息通知下游
        mqProducer.send("user.register", event);
    }
}

// ========== ⑤ 异步监听 ==========
@Component
public class UserRegisterAsyncListener {

    @Async
    @EventListener
    public void sendWelcomeEmail(UserRegisterEvent event) {
        emailService.sendWelcome(event.getEmail(), event.getUsername());
    }

    @Async
    @EventListener
    public void initializeRecommendation(UserRegisterEvent event) {
        recommendationService.initNewUser(event.getUserId());
    }
}
```

### 5.2 Spring Boot 内置事件

Spring Boot在启动过程中发布一系列事件，**监听方式与自定义事件完全相同**：

| 事件 | 时机 |
|------|------|
| `ApplicationStartingEvent` | 刚启动，Environment 尚未准备（**需注册到 spring.factories 中监听**） |
| `ApplicationEnvironmentPreparedEvent` | Environment 准备完成 |
| `ApplicationContextInitializedEvent` | ApplicationContext 初始化完成 |
| `ApplicationPreparedEvent` | Bean定义加载完成，刷新前 |
| `ApplicationStartedEvent` | 上下文刷新完成，但 CommandLineRunner 未执行 |
| `ApplicationReadyEvent` | **一切就绪**，应用完全可用 |
| `ApplicationFailedEvent` | 启动失败 |
| `ContextRefreshedEvent` | ApplicationContext 初始化或刷新完成 |

```java
@Component
public class AppLifecycleListener {

    @EventListener
    public void onStarted(ApplicationStartedEvent event) {
        System.out.println("应用启动完成，准备执行任务...");
    }

    @EventListener
    public void onReady(ApplicationReadyEvent event) {
        System.out.println("应用完全就绪，可以接收请求！");
    }
}
```

---

## 六、@Conditional条件注解

### 6.1 @Conditional基础

`@Conditional` 是条件装配的核心元注解，根据条件决定**是否注册Bean**：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Conditional {
    Class<? extends Condition>[] value();  // 条件类必须实现 Condition 接口
}

@FunctionalInterface
public interface Condition {
    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

### 6.2 Spring Boot 内置条件注解

Spring Boot 将 `@Conditional` 封装为一系列易用的条件注解，核心原理一致：

| 注解 | 作用 |
|------|------|
| `@ConditionalOnClass(name = "...")` | 类路径存在指定类时生效 |
| `@ConditionalOnMissingClass("...")` | 类路径**缺失**指定类时生效 |
| `@ConditionalOnBean(type = "...")` | 容器中存在指定Bean时生效 |
| `@ConditionalOnMissingBean(type = "...")` | 容器中**不存在**指定Bean时生效 |
| `@ConditionalOnProperty(prefix, name, havingValue)` | 配置文件属性匹配时生效 |
| `@ConditionalOnExpression(...)` | SpEL表达式结果为true时生效 |
| `@ConditionalOnWebApplication` | 当前为Web应用时生效 |
| `@ConditionalOnNotWebApplication` | 当前为非Web应用时生效 |
| `@ConditionalOnJava(range)` | JDK版本在指定范围时生效 |

```java
// ✅ 只有引入Redis依赖时才装配
@Configuration
@ConditionalOnClass(name = "org.springframework.data.redis.core.RedisTemplate")
public class RedisAutoConfiguration {
    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        return new RedisTemplate<>();
    }
}

// ✅ 只有属性 app.cache.type=redis 时才装配Redis缓存
@Configuration
@ConditionalOnProperty(prefix = "app.cache", name = "type", havingValue = "redis")
public class CacheConfig {
    // ...
}

// ✅ 只有容器中不存在 DataSource Bean 时才创建嵌入式数据库
@Bean
@ConditionalOnMissingBean(DataSource.class)
public DataSource embeddedDataSource() {
    return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
}
```

### 6.3 自定义Condition

```java
// 自定义条件：检测操作系统类型
public class WindowsCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        return System.getProperty("os.name").toLowerCase().contains("win");
    }
}

// 使用自定义条件
@Configuration
public class OSServiceConfig {

    @Bean
    @Conditional(WindowsCondition.class)
    public FileService windowsFileService() {
        return new WindowsFileService();  // 只有 Windows 才注册
    }

    @Bean
    @ConditionalOnMac
    public FileService macFileService() {
        return new MacFileService();
    }
}
```

### 6.4 条件注解在自动装配中的应用

Spring Boot 的**自动装配**大量使用条件注解，以 `DataSourceAutoConfiguration` 为例：

```java
// 简化的自动装配逻辑
@AutoConfiguration                              // 自动配置标记
@ConditionalOnClass(DataSource.class)           // 有DataSource类才生效
@ConditionalOnProperty(prefix = "spring.datasource", name = "url")
                                                // 配置了url才生效
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean    // 用户自定义了就跳过
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

> 条件注解的优先级顺序决定了 **"如果用户自己定义了Bean，就不自动创建"** 的模式。

---

## 七、Profile环境配置

### 7.1 @Profile基本用法

`@Profile` 决定了**Bean仅在指定环境下激活**，本质上是 `@Conditional` 的一个特化实现：

```java
@Configuration
public class DBConfig {

    @Bean
    @Profile("dev")       // 开发环境
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .build();
    }

    @Bean
    @Profile("prod")      // 生产环境
    public DataSource prodDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-db:3306/app");
        ds.setUsername("prod_user");
        return ds;
    }
}
```

### 7.2 激活Profile方式

| 方式 | 示例 |
|------|------|
| **配置文件** | `spring.profiles.active=dev` |
| **环境变量** | `export SPRING_PROFILES_ACTIVE=prod` |
| **JVM参数** | `-Dspring.profiles.active=prod` |
| **启动类设置** | `SpringApplication.setAdditionalProfiles("dev")` |
| **测试注解** | `@ActiveProfiles("test")` |

```yaml
# application.yml
spring:
  profiles:
    active: dev   # 激活dev配置
```

### 7.3 多环境配置实战

Spring Boot 推荐**多配置文件 + Profile分区**：

```
src/main/resources/
├── application.yml              # 公共配置
├── application-dev.yml           # 开发环境
├── application-test.yml          # 测试环境
└── application-prod.yml          # 生产环境
```

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:devdb
    driver-class-name: org.h2.Driver

# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/app?useSSL=true
    hikari:
      maximum-pool-size: 50
```

```java
// 同一个Bean在不同Profile下不同实现
@Service
@Profile("dev")
public class MockPaymentService implements PaymentService {
    @Override
    public void pay(Order order) {
        System.out.println("开发环境：模拟支付");
    }
}

@Service
@Profile("prod")
public class RealPaymentService implements PaymentService {
    @Override
    public void pay(Order order) {
        paymentGateway.charge(order);  // 真实支付
    }
}
```

### 7.4 @Profile vs @Conditional区别

| 维度 | @Profile | @Conditional |
|------|----------|-------------|
| **粒度** | 粗粒度（环境级别） | 细粒度（任意条件） |
| **本质** | `@Conditional(ProfileCondition.class)` | 接口基类 |
| **条件来源** | 配置文件 / JVM参数 | 任意：类路径、属性、系统信息 |
| **常见场景** | 区分dev/test/prod | 自动装配、条件化Bean注入 |
| **可组合性** | 与/或组合 | 通过自定义Condition实现复杂逻辑 |

---

## 八、面试常见问题

**Q1：Spring事件是同步还是异步？默认是什么？**
默认是**同步**的，发布者线程会等待所有监听器执行完。可通过 `@Async` 改为异步。

**Q2：@TransactionalEventListener 与普通 @EventListener 有什么区别？**
`@TransactionalEventListener` 绑定到事务阶段，默认在 `AFTER_COMMIT` 后执行，确保事务提交后才做后续操作（如发MQ消息）。普通 `@EventListener` 在方法调用时立即执行。

**Q3：Spring事件和MQ应该怎么选择？**
进程内解耦（如初始化缓存、记录日志）用Spring事件；跨服务异步通信用MQ。Spring事件丢失不可恢复，不能用于跨进程场景。

**Q4：@ConditionalOnMissingBean 在多个自动配置之间如何确定优先级？**
通过 `@AutoConfigureBefore` / `@AutoConfigureAfter` / `@AutoConfigureOrder` 控制自动配置的顺序，确保"优先级低"的先注册，"优先级高"的通过 `@ConditionalOnMissingBean` 检查后覆盖。

**Q5：Profile 可以同时激活多个吗？**
可以。`spring.profiles.active=dev,mock` 同时激活 dev 和 mock 两个Profile。`@Profile("dev & mock")` 要求两个都激活，`@Profile("dev | mock")` 要求任一激活。

**Q6：`@EventListene​r` 的 condition 表达式里能用哪些变量？**
能用方法参数名（如 `#event`）、方法返回值（`#result`，仅 `@TransactionalEventListener` 的 AFTER_COMMIT）、以及 Spring 容器中的 Bean（`@beanName.method()`）。

---

## 九、LeetCode题目解析

### 9.1 [19. 删除链表的倒数第N个结点（Medium）](https://leetcode.cn/problems/remove-nth-node-from-end-of-list/)

**思路**：快慢指针。快指针先走n步，然后快慢指针同时走，当快指针到达尾部时，慢指针正好在倒数第n+1个节点，删除其next。

```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0, head);  // 哨兵节点
    ListNode fast = dummy, slow = dummy;

    for (int i = 0; i < n; i++) {   // fast先走n步
        fast = fast.next;
    }

    while (fast.next != null) {     // 快慢同时走
        fast = fast.next;
        slow = slow.next;
    }

    slow.next = slow.next.next;     // 删除倒数第n个
    return dummy.next;
}
```

**复杂度**：时间 O(L)，空间 O(1)，L 为链表长度。

### 9.2 [24. 两两交换链表中的节点（Medium）](https://leetcode.cn/problems/swap-nodes-in-pairs/)

**思路**：递归 / 迭代。每两个节点反转一次，递归处理剩余部分最简洁。

```java
// 递归法
public ListNode swapPairs(ListNode head) {
    if (head == null || head.next == null) {
        return head;
    }
    ListNode newHead = head.next;            // 第二个节点变成新头
    head.next = swapPairs(newHead.next);     // 递归处理剩余
    newHead.next = head;                     // 反转指针
    return newHead;
}

// 迭代法
public ListNode swapPairs(ListNode head) {
    ListNode dummy = new ListNode(0, head);
    ListNode prev = dummy;
    while (prev.next != null && prev.next.next != null) {
        ListNode first = prev.next;
        ListNode second = first.next;
        prev.next = second;
        first.next = second.next;
        second.next = first;
        prev = first;
    }
    return dummy.next;
}
```

**复杂度**：时间 O(n)，空间 O(n)（递归栈）/ O(1)（迭代）。

---

## 十、今日学习要点总结

### 核心要点归纳

- **Spring事件 = 观察者模式**，进程内解耦利器
- `ApplicationEventPublisher` 发布事件，`ApplicationListener<T>` 或 `@EventListener` 监听
- 默认**同步执行**，加 `@Async` 变为异步（需 `@EnableAsync`）
- `@TransactionalEventListener` 绑定事务阶段，**最常用 AFTER_COMMIT**
- `@Conditional` 及其派生注解实现**条件装配**，是自动配置的基石
- `@Profile` 是 `@Conditional` 的环境特化版，区分 dev/test/prod
- Spring事件适合日志、通知、缓存更新；跨服务用MQ

### 验收清单

- [ ] 能自定义Spring事件并发布监听（同步+异步）
- [ ] 能解释事件机制的使用场景，与MQ的区别
- [ ] 能使用 `@TransactionalEventListener` 实现事务提交后执行
- [ ] 能使用 `@ConditionalOnClass`、`@ConditionalOnProperty` 等条件注解
- [ ] 能说明 `@Profile` 的作用和使用方式
- [ ] 理解 `@Conditional` 在自动装配中的应用
- [ ] 完成 2 道 LeetCode 链表题目

---

## 十一、练习建议

1. 搭建Spring Boot项目，实现**用户注册 → 发邮件 + 发优惠券 + 初始化推荐**的完整事件链（同步+异步+事务事件）
2. 配置 `application-dev.yml` 和 `application-prod.yml`，用 `@Profile` 区分不同环境的Bean
3. 实现自定义 `Condition`：根据系统时间决定是否开启限流功能
4. 用 Virtual Threads（如果有JDK 21+）实现异步事件处理并对比性能
5. 阅读 `spring-boot-autoconfigure` 源码中任意一个 `XXXAutoConfiguration`，理解条件注解的实际应用

---

## 十二、扩展阅读

- [Spring Framework Reference: Events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html)
- [Spring Boot Reference: Auto-configuration](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)
- [Spring Boot Reference: Profiles](https://docs.spring.io/spring-boot/reference/features/profiles.html)
- [Baeldung: Spring Events](https://www.baeldung.com/spring-events)
- [Baeldung: @Conditional Annotations](https://www.baeldung.com/spring-conditional-annotations)
