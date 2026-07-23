# Day30 详解 — Bean生命周期

## 〇、最新JDK/Spring版本中的Bean生命周期变化

### 0.1 基线变化：Spring 6.x + JDK 17+

Spring 6.x（2022年发布）基于JDK 17作为**最低版本要求**，带来了大量底层变化：

```
Spring 5.x (JDK 8)              Spring 6.x (JDK 17+)
──────────────────────────────────────────────────
javax.annotation.*       →      jakarta.annotation.*
javax.servlet.*          →      jakarta.servlet.*
反射 + CGLIB              →      反射 + CGLIB + Record支持
JDK动态代理               →      JDK动态代理 + LambdaMetaFactory
```

### 0.2 Jakarta EE迁移

Spring 6.x / Spring Boot 3.x 全面切换到**Jakarta EE 9+**命名空间。这对Bean生命周期的影响：

| 旧命名空间（Spring 5.x） | 新命名空间（Spring 6.x） |
|---|---|
| `javax.annotation.PostConstruct` | `jakarta.annotation.PostConstruct` |
| `javax.annotation.PreDestroy` | `jakarta.annotation.PreDestroy` |
| `javax.annotation.Resource` | `jakarta.annotation.Resource` |
| `javax.inject.Inject` | `jakarta.inject.Inject` |

```java
// ❌ Spring 5.x 的写法（Spring 6.x 中已废弃）
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

// ✅ Spring 6.x / Spring Boot 3.x 正确写法
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;

@Component
public class MyService {

    @PostConstruct  // 现在来源于 jakarta.annotation
    public void init() {
        System.out.println("初始化完成");
    }

    @PreDestroy  // 现在来源于 jakarta.annotation
    public void cleanup() {
        System.out.println("销毁前清理");
    }
}
```

**注意**：`@Resource`注解虽然包名从 `javax.annotation.Resource` 变为 `jakarta.annotation.Resource`，但Spring 6.x**仍然向后兼容**旧的 `javax.annotation.Resource`。不过建议迁移到 `jakarta.annotation`。

### 0.3 Spring Boot 3.2+ 虚拟线程对Bean生命周期的影响

```java
// Spring Boot 3.2+ 支持虚拟线程
@Configuration
public class VirtualThreadConfig {

    @Bean
    public AsyncTaskExecutor asyncTaskExecutor() {
        return new TaskExecutorAdapter(
            Executors.newVirtualThreadPerTaskExecutor()
        );
    }

    @Bean
    public TomcatProtocolHandlerCustomizer<?> tomcatVirtualThreads() {
        return protocolHandler -> {
            protocolHandler.setExecutor(
                Executors.newVirtualThreadPerTaskExecutor()
            );
        };
    }
}
```

- **Bean初始化方法在平台线程中执行**：即使启用了虚拟线程，Spring IoC容器的初始化流程（包括 `@PostConstruct`、`InitializingBean`、`BeanPostProcessor`）仍然在**主线程（平台线程）**中同步执行。
- **虚拟线程不改变Bean的作用域语义**：单例Bean仍然只有一个实例，多例Bean每次获取创建新实例。
- **Request/Session Scope的Bean**：当请求处理切换到虚拟线程时，scope相关的Bean可能在不同的虚拟线程中被创建/访问，需要注意ThreadLocal的使用 — **应在虚拟线程中使用 `ScopedValue`（JDK 20+）代替ThreadLocal**。

### 0.4 Spring Boot 3.x AOT编译与GraalVM原生镜像

Spring Boot 3.x引入了**AOT（Ahead-of-Time）编译**支持，可以将应用编译为GraalVM原生镜像：

```
标准JVM模式                         AOT原生镜像模式
─────────────────                  ────────────────
运行时反射加载Bean       →         编译期BeanDefinition固化
运行时生成CGLIB代理      →         编译期代理类生成
运行时调用@PostConstruct  →         编译期生成初始化调用链
延迟初始化支持            →         所有Bean关闭时全部初始化
条件注解(@Conditional*)   →         编译期求值，固定结果
```

```java
// AOT模式下的Bean定义 hint
@Configuration
@ImportRuntimeHints(MyRuntimeHints.class)
public class AppConfig {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
}

// 为原生镜像提供反射/代理/资源hint
public class MyRuntimeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(MyEntity.class,
            MemberCategory.INVOKE_DECLARED_CONSTRUCTORS,
            MemberCategory.INVOKE_DECLARED_METHODS);
    }
}
```

**AOT对Bean生命周期的影响**：
- `@PostConstruct` / `@PreDestroy` 的调用由编译期确定的代码直接执行（不依赖运行时反射）
- `BeanPostProcessor` 的处理逻辑在编译期完成，不在运行时拦截
- **不再支持在原生镜像中动态注册Bean**（无运行时类加载）
- 所有 `@Lazy` Bean在编译期就会被决定是否需要初始化

---

## 一、Bean生命周期概览

### 1.1 完整生命周期全景图

```
                          Bean生命周期完整流程
                          ═══════════════════

  ┌─────────────────────────────────────────────────────────────────┐
  │                    1. 实例化（Instantiation）                      │
  │  推断构造方法 → 反射/CGLIB创建对象 → BeanWrapper包装对象            │
  └────────────────────────────┬────────────────────────────────────┘
                               ↓
  ┌─────────────────────────────────────────────────────────────────┐
  │             2. 属性填充（Populate Properties）                     │
  │  @Autowired → @Value → @Resource → @Inject                       │
  │  由 InstantiationAwareBeanPostProcessor 处理                      │
  └────────────────────────────┬────────────────────────────────────┘
                               ↓
  ┌─────────────────────────────────────────────────────────────────┐
  │              3. Aware 接口回调（Aware Callbacks）                  │
  │  BeanNameAware → BeanClassLoaderAware → BeanFactoryAware          │
  │  → EnvironmentAware → EmbeddedValueResolverAware                  │
  │  → ResourceLoaderAware → ApplicationEventPublisherAware           │
  │  → MessageSourceAware → ApplicationContextAware                   │
  └────────────────────────────┬────────────────────────────────────┘
                               ↓
  ┌─────────────────────────────────────────────────────────────────┐
  │     4. BeanPostProcessor 前置处理                                  │
  │  postProcessBeforeInitialization(Object bean, String beanName)    │
  │  典型：@PostConstruct 注解处理（CommonAnnotationBeanPostProcessor）│
  └────────────────────────────┬────────────────────────────────────┘
                               ↓
  ┌─────────────────────────────────────────────────────────────────┐
  │     5. 初始化（Initialization）                                   │
  │  Step 1: @PostConstruct 方法（jakarta.annotation）                │
  │  Step 2: InitializingBean.afterPropertiesSet()                   │
  │  Step 3: init-method（配置的 init-method）                        │
  └────────────────────────────┬────────────────────────────────────┘
                               ↓
  ┌─────────────────────────────────────────────────────────────────┐
  │     6. BeanPostProcessor 后置处理                                  │
  │  postProcessAfterInitialization(Object bean, String beanName)     │
  │  典型：AOP代理创建（AbstractAutoProxyCreator）                      │
  └────────────────────────────┬────────────────────────────────────┘
                               ↓
  ┌─────────────────────────────────────────────────────────────────┐
  │                    7. Bean 就绪（Ready）                           │
  │  Bean放⼊⼀级缓存（singletonObjects），可被应⽤程序使⽤              │
  │  触发 ContextRefreshedEvent 事件                                   │
  └────────────────────────────┬────────────────────────────────────┘
                               ↓
  ┌─────────────────────────────────────────────────────────────────┐
  │                    8. 销毁（Destruction）                          │
  │  Step 1: @PreDestroy 方法（jakarta.annotation）                   │
  │  Step 2: DisposableBean.destroy()                                │
  │  Step 3: destroy-method（配置的 destroy-method）                   │
  │  ⚠️ 仅单例Bean执⾏销毁，多例Bean由调⽤者管理                         │
  └─────────────────────────────────────────────────────────────────┘
```

### 1.2 大阶段划分

| 阶段 | 核心动作 | 关键接口/注解 | 常见扩展点 |
|---|---|---|---|
| **实例化** | 创建Bean实例 | Constructor | `InstantiationAwareBeanPostProcessor` |
| **属性填充** | 注入依赖 | `@Autowired`, `@Value`, `@Resource` | `InstantiationAwareBeanPostProcessor.postProcessProperties()` |
| **初始化** | 执行初始化逻辑 | `@PostConstruct`, `InitializingBean`, `init-method` | `BeanPostProcessor` |
| **使用** | 业务调用 | — | — |
| **销毁** | 清理资源 | `@PreDestroy`, `DisposableBean`, `destroy-method` | — |

**一句话总结**：Spring Bean的生命周期从"创建对象"开始，经过"填充属性"和"初始化"，最终在容器关闭时"销毁"，全程有十多个**扩展点**可以介入。

---

## 二、实例化阶段

### 2.1 推断构造方法

Spring在创建Bean时，需要先确定使用哪个构造方法。

```
推断构造方法策略（优先级从高到低）
═══════════════════════════════════

1. 有 @Autowired 标注的构造方法
   ↓ 如果有多个 @Autowired 构造方法怎么办？
   → @Autowired(required = true) 必须只能有一个
   → 多个 @Autowired(required = false) 则会按"最⼤参数匹配"原则选择

2. 多个符合条件的构造方法，且其中一个是默认无参构造
   → Spring 选择无参构造方法

3. 只有一个构造方法（默认策略）
   → 直接使用该构造方法，参数自动从 IoC 容器获取
   （Spring 4.3+ 单构造器自动注入，无需 @Autowired）

4. 有多个构造方法且都没有 @Autowired
   → Spring 选择无参构造方法
   → 无参构造器不存在 → 抛出 BeanCreationException
```

```java
@Component
public class UserService {

    private final UserRepository userRepository;
    private final Logger logger;

    // ✅ Spring 4.3+：只有⼀个构造方法，⾃动注⼊，@Autowired 可省略
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
        this.logger = LoggerFactory.getLogger(UserService.class);
    }
}

@Component
public class PaymentService {

    private PaymentGateway gateway;

    // ✅ 多个构造方法时，用 @Autowired 明确指定
    @Autowired
    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }

    // 无 @Autowired，不会被选为实例化构造方法
    public PaymentService() {
        // 默认构造方法
    }
}

@Component
public class ReportService {

    // ✅ Spring 4.3+：多构造器且无 @Autowired 时，选择无参构造
    public ReportService() {
        System.out.println("使⽤默认构造方法");
    }

    public ReportService(DataSource ds) {
        // 不会被选用（多个构造器且都没有 @Autowired 时选无参构造）
    }
}
```

### 2.2 实例化方式

Spring根据情况选择不同的实例化策略：

| 实例化方式 | 适用场景 | 技术实现 |
|---|---|---|
| **反射** | 普通单例Bean | `Constructor.newInstance()` |
| **CGLIB子类化** | 需要代理的Bean（如配置类 @Configuration） | CGLIB动态⽣成子类 |
| **Supplier** | 编程式注册Bean（`registerBean(...)`） | `supplier.get()` |
| **工厂方法** | `@Bean` 标注的方法 | 反射调用工厂方法 |
| **Record** | JDK 16+ Record类型 | Spring 6.x 原生支持 |

```java
// 1. 反射实例化（默认方式）
@Component
public class OrderService {
    public OrderService(OrderRepository repo) {
        // 通过反射调用构造方法
    }
}

// 2. Supplier 实例化（编程式注册）
@Configuration
public class SupplierConfig {

    @Bean
    public Supplier<MyBean> myBeanSupplier() {
        return () -> new MyBean("custom arg");
    }
}
// 或通过 ApplicationContext 编程式注册：
// context.registerBean(MyBean.class, () -> new MyBean("custom arg"));

// 3. CGLIB 子类化（Spring 自动处理 @Configuration 类）
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(); // Spring CGLIB 代理确保单例
    }
}
```

---

## 三、属性填充阶段

### 3.1 @Autowired注入

```java
@Component
public class OrderService {

    // 字段注入
    @Autowired
    private PaymentService paymentService;

    // Setter注入
    private InventoryService inventoryService;

    @Autowired
    public void setInventoryService(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    // 构造器注入（Spring 4.3+ 单构造器可省略 @Autowired）
    private final UserService userService;

    public OrderService(UserService userService) {
        this.userService = userService;
    }

    // required = false：找不到Bean时不报错
    @Autowired(required = false)
    private OptionalService optionalService;
}
```

**@Autowired注入过程**：
1. `AutowiredAnnotationBeanPostProcessor` 扫描 `@Autowired` 注解
2. 按类型（byType）查找候选Bean
3. 多个候选时按名称（byName）匹配
4. 仍无法确定 → `@Qualifier` 或 `@Primary` 决定
5. 都找不到 → `required = false` 时跳过，否则抛 `NoSuchBeanDefinitionException`

**Spring 6.x 中 `@Autowired(required = false)` 的替代方案**：

```java
@Component
public class ModernService {

    // ✅ 推荐：使用 Optional<T>（语义更清晰）
    @Autowired
    public ModernService(Optional<LegacyService> legacyService) {
        this.legacyService = legacyService.orElse(null);
    }

    // ✅ 推荐：使用 @Nullable（Kotlin友好）
    @Autowired
    public ModernService(@Nullable LegacyService legacyService) {
        this.legacyService = legacyService;
    }

    // ✅ 集合注入：注入所有实现该类型的Bean（包括空集合）
    @Autowired
    private List<Plugin> plugins; // 没有Plugin实现时 => 空List，不会报错
}
```

### 3.2 @Value注入

```java
@Component
public class AppProperties {

    // 字面值
    @Value("myApp")
    private String appName;

    // 占位符
    @Value("${server.port:8080}")
    private int port;

    // SpEL表达式
    @Value("#{systemProperties['java.home']}")
    private String javaHome;

    // 默认值（冒号后面）
    @Value("${app.timeout:3000}")
    private long timeout;
}
```

### 3.3 @Resource注入

```java
@Component
public class TaskService {

    // @Resource 是 JSR-250 标准，默认按名称注入
    @Resource(name = "userRepositoryImpl")
    private UserRepository userRepository;

    // 不指定name时，先按字段名匹配，匹配不到再按类型匹配
    @Resource
    private MailSender mailSender; // 先找名为 "mailSender" 的Bean

    // Setter上也支持
    @Resource
    public void setLogService(LogService logService) {
        this.logService = logService;
    }
}
```

### 3.4 三种注入方式对比

| 特性 | `@Autowired` | `@Resource` | `@Value` |
|---|---|---|---|
| **来源** | Spring 原生 | JSR-250（Java标准） | Spring 原生 |
| **注入策略** | 默认 byType | 默认 byName | N/A（注入值） |
| **required** | 支持 `required = false` | 不支持（找不到直接抛异常） | 支持默认值 `${key:default}` |
| **@Qualifier** | 支持 | 不支持（用 `name` 属性） | 不支持 |
| **集合注入** | 支持 `List<T>` | 不支持 | 不支持 |
| **优先级** | 最低（最先由Spring处理） | 中等 | 最低 |
| **Jakarta位置** | 无（Spring独有） | `jakarta.annotation.Resource` | 无 |

### 3.5 构造器注入 vs 字段注入 vs Setter注入

```java
// ❌ 字段注入 — 不推荐
@Component
public class BadExample {
    @Autowired
    private UserService userService;  // 隐藏依赖、不利于测试
}

// ⚠️ Setter注入 — 可选依赖时可用
@Component
public class AcceptableExample {
    private UserService userService;

    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
}

// ✅ 构造器注入 — 推荐（Spring官方推荐）
@Component
public class GoodExample {
    private final UserService userService; // final 保障不可变

    public GoodExample(UserService userService) {
        this.userService = userService;
    }
}
```

**为什么推荐构造器注入？**

| 维度 | 构造器注入 | 字段注入 | Setter注入 |
|---|---|---|---|
| **不可变性** | ✅ 可声明 `final` | ❌ 不能 | ❌ 不能 |
| **依赖明确性** | ✅ 构造参数一目了然 | ❌ 隐藏依赖 | ⚠️ 一般 |
| **单元测试** | ✅ 直接new传入Mock | ❌ 需要反射注入 | ✅ 调用setter |
| **循环依赖** | ⚠️ 构造器循环依赖会报错 | ✅ 可解决（但不建议） | ✅ 可解决 |
| **Spring兼容性** | ✅ 完美支持 | ✅ 支持 | ✅ 支持 |
| **空指针安全** | ✅ 编译期保证 | ❌ 运行期风险 | ⚠️ 更可控 |

---

## 四、Aware接口回调

### 4.1 常用Aware接口

Spring的Aware接口让Bean直接感知容器的内部组件。执行顺序如下：

| 序号 | Aware接口 | 用途 | 回调方法 |
|---|---|---|---|
| 1 | `BeanNameAware` | 获取Bean在容器中的名称 | `setBeanName(String name)` |
| 2 | `BeanClassLoaderAware` | 获取加载Bean的ClassLoader | `setBeanClassLoader(ClassLoader cl)` |
| 3 | `BeanFactoryAware` | 获取BeanFactory（IoC容器） | `setBeanFactory(BeanFactory bf)` |
| 4 | `EnvironmentAware` | 获取Environment配置 | `setEnvironment(Environment env)` |
| 5 | `EmbeddedValueResolverAware` | 获取字符串值解析器 | `setEmbeddedValueResolver(StringValueResolver r)` |
| 6 | `ResourceLoaderAware` | 获取资源加载器 | `setResourceLoader(ResourceLoader rl)` |
| 7 | `ApplicationEventPublisherAware` | 获取事件发布器 | `setApplicationEventPublisher(ApplicationEventPublisher p)` |
| 8 | `MessageSourceAware` | 获取国际化消息源 | `setMessageSource(MessageSource ms)` |
| 9 | `ApplicationContextAware` | 获取完整ApplicationContext | `setApplicationContext(ApplicationContext ctx)` |

### 4.2 调用时机与示例

```java
@Component
public class LifecycleDemoBean implements
        BeanNameAware,
        BeanFactoryAware,
        ApplicationContextAware {

    private String beanName;
    private BeanFactory beanFactory;
    private ApplicationContext applicationContext;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("1. BeanNameAware: " + name);
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
        System.out.println("2. BeanFactoryAware: 获取到BeanFactory");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
        System.out.println("3. ApplicationContextAware: 获取到ApplicationContext");
    }
}
```

**⚠️ 注意**：实现Aware接口会让Bean与Spring框架耦合。更好的做法是：
```java
@Component
public class BetterDemo {
    // ✅ 直接注入所需组件，而非通过Aware接口
    private final ApplicationContext ctx;

    public BetterDemo(ApplicationContext ctx) {
        this.ctx = ctx; // 构造器注入，同样可获取ApplicationContext
    }
}
```

---

## 五、BeanPostProcessor详解

`BeanPostProcessor` 是Spring Bean生命周期中**最重要的扩展点**，可以在Bean初始化前后进行干预。

```
                    BeanPostProcessor 作用时机
                    ══════════════════════════

  Bean实例化
       ↓
  属性填充
       ↓
  Aware回调
       ↓
 ┌─────────────────────────────────────────┐
 │ postProcessBeforeInitialization(bean, name) │  ← 前置处理
 └─────────────────────────────────────────┘
       ↓
  @PostConstruct → InitializingBean → init-method
       ↓
 ┌─────────────────────────────────────────┐
 │ postProcessAfterInitialization(bean, name)  │  ← 后置处理
 └─────────────────────────────────────────┘
       ↓
  Bean就绪
```

### 5.1 前置处理（postProcessBeforeInitialization）

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(Monitored.class)) {
            System.out.println("[BEFORE] Bean '" + beanName + "' 即将初始化");
        }
        return bean; // ⚠️ 必须返回bean（可以是代理对象）
    }
}
```

### 5.2 后置处理（postProcessAfterInitialization）

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean.getClass().isAnnotationPresent(Monitored.class)) {
            System.out.println("[AFTER] Bean '" + beanName + "' 初始化完成");
        }
        return bean; // ⚠️ 可以返回代理对象实现AOP
    }
}
```

### 5.3 实际应用场景

Spring内部通过多个`BeanPostProcessor`实现核心功能：

| BeanPostProcessor子类 | 作用 | 处理阶段 |
|---|---|---|
| `AutowiredAnnotationBeanPostProcessor` | 处理 `@Autowired`、`@Value` 注解 | 属性填充 |
| `CommonAnnotationBeanPostProcessor` | 处理 `@PostConstruct`、`@PreDestroy`、`@Resource` | 前置/后置 |
| `AbstractAutoProxyCreator` | 创建AOP代理对象 | 后置 |
| `AbstractAdvisingBeanPostProcessor` | 为特定Bean添加Advice | 后置 |
| `ApplicationContextAwareProcessor` | 处理所有Aware接口回调 | 前置 |
| `InitDestroyAnnotationBeanPostProcessor` | 处理 `@PostConstruct` / `@PreDestroy` | 前置 |

```java
// AbstractAutoProxyCreator 的简化工作原理
public abstract class AbstractAutoProxyCreator
        extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            // 1. 检查是否需要代理（是否有匹配的切面）
            Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(
                bean.getClass(), beanName, null);
            if (specificInterceptors != DO_NOT_PROXY) {
                // 2. 创建代理对象
                Object proxy = createProxy(
                    bean.getClass(), beanName, specificInterceptors,
                    new SingletonTargetSource(bean));
                return proxy; // 返回代理对象代替原Bean
            }
        }
        return bean;
    }
}
```

**实际使用场景：一个监控Bean初始化耗时的BeanPostProcessor**：

```java
@Component
public class TimingBeanPostProcessor implements BeanPostProcessor {

    private final Map<String, Long> startTimes = new ConcurrentHashMap<>();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        startTimes.put(beanName, System.nanoTime());
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Long start = startTimes.remove(beanName);
        if (start != null) {
            long duration = System.nanoTime() - start;
            if (duration > 100_000_000) { // 超过100ms
                System.out.println("⚠️ Bean '" + beanName +
                    "' 初始化耗时: " + (duration / 1_000_000) + "ms");
            }
        }
        return bean;
    }
}
```

---

## 六、初始化方法

### 6.1 三种初始化方法

| 方式 | 来源 | 耦合度 | 优先级 | 推荐度 |
|---|---|---|---|---|
| `@PostConstruct` | JSR-250 / Jakarta标准 | 低（标准注解） | 最高 | ⭐⭐⭐ |
| `InitializingBean.afterPropertiesSet()` | Spring框架接口 | 高（强耦合Spring） | 中等 | ⭐ |
| `@Bean(initMethod = "...")` 或 XML `init-method` | Spring配置 | 低（无侵入） | 最低 | ⭐⭐ |

### 6.2 执行顺序（关键！）

```
✅ 执行顺序（记住！面试高频）
═══════════════════════════════

  1. @PostConstruct 标注的方法     （最早）
         ↓
  2. InitializingBean.afterPropertiesSet()  （其次）
         ↓
  3. @Bean(initMethod = "xxx") / XML init-method  （最后）
```

### 6.3 完整示例代码

```java
import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.InitializingBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Component
public class InitDemoBean implements InitializingBean {

    // 方式1：@PostConstruct（最先执行）
    @PostConstruct
    public void postConstructInit() {
        System.out.println("1. @PostConstruct — 属性已注入，执行初始化");
    }

    // 方式2：InitializingBean（其次执行）
    @Override
    public void afterPropertiesSet() {
        System.out.println("2. InitializingBean.afterPropertiesSet()");
    }

    // 方式3：init-method（最后执行）
    public void customInit() {
        System.out.println("3. customInit() — @Bean(initMethod = \"customInit\")");
    }

    // 普通构造方法（最先执行，但不属于初始化阶段）
    public InitDemoBean() {
        System.out.println("0. 构造方法 — 对象创建，属性尚未注入");
    }
}

@Configuration
class InitConfig {

    @Bean(initMethod = "customInit")
    public InitDemoBean initDemoBean() {
        return new InitDemoBean();
    }
}
```

**控制台输出**：
```
0. 构造方法 — 对象创建，属性尚未注入
1. @PostConstruct — 属性已注入，执行初始化
2. InitializingBean.afterPropertiesSet()
3. customInit() — @Bean(initMethod = "customInit")
```

---

## 七、销毁方法

### 7.1 三种销毁方法

| 方式 | 来源 | 适用场景 |
|---|---|---|
| `@PreDestroy` | JSR-250 / Jakarta标准 | 推荐首选 |
| `DisposableBean.destroy()` | Spring框架接口 | 强耦合，不推荐 |
| `@Bean(destroyMethod = "...")` | Spring配置 | 无侵入，适用第三方类 |

### 7.2 执行顺序

```
✅ 销毁顺序
═══════════════

  1. @PreDestroy    （最早）
         ↓
  2. DisposableBean.destroy()   （其次）
         ↓
  3. @Bean(destroyMethod = "xxx") / XML destroy-method  （最后）
```

```java
import jakarta.annotation.PreDestroy;
import org.springframework.beans.factory.DisposableBean;

@Component
public class DestroyDemoBean implements DisposableBean {

    // 方式1：@PreDestroy（最先执行）
    @PreDestroy
    public void preDestroyCleanup() {
        System.out.println("1. @PreDestroy — 释放数据库连接池");
    }

    // 方式2：DisposableBean（其次执行）
    @Override
    public void destroy() {
        System.out.println("2. DisposableBean.destroy() — 清理资源");
    }

    // 方式3：destroy-method（最后执行）
    public void customDestroy() {
        System.out.println("3. customDestroy() — @Bean(destroyMethod = \"customDestroy\")");
    }
}

@Configuration
class DestroyConfig {

    @Bean(destroyMethod = "customDestroy")
    public DestroyDemoBean destroyDemoBean() {
        return new DestroyDemoBean();
    }
}
```

**⚠️ 注意**：`@Bean` 注解 `destroyMethod` 属性在Spring Boot中默认值为 `"(inferred)"`，会自动推断 `close()` 或 `shutdown()` 作为销毁方法。如果不希望自动推断，设置 `destroyMethod = ""`。

---

## 八、单例 vs 多例Bean生命周期

### 8.1 区别对比

| 维度 | 单例（Singleton） | 多例（Prototype） |
|---|---|---|
| **作用域** | IoC容器中只有⼀个实例 | 每次获取新建实例 |
| **创建时机** | 容器启动时（默认） | 每次 `getBean()` 时 |
| **初始化回调** | ✅ 完整执⾏ | ✅ 完整执⾏ |
| **销毁回调** | ✅ 容器关闭时执行 | ❌ **Spring 不负责销毁** |
| **BeanPostProcessor** | ✅ 完整处理 | ✅ 完整处理 |
| **循环依赖** | ✅ 可解决（三级缓存） | ❌ 无法解决 |
| **内存占用** | 持有引用，不GC | 由调用者释放，GC回收 |
| **适用场景** | 无状态Service、DAO、工具类 | 有状态对象、每次请求数据不同 |

### 8.2 Prototype Bean的特殊性

```java
@Component
@Scope("prototype")
public class PrototypeBean implements DisposableBean {

    public PrototypeBean() {
        System.out.println(">>> PrototypeBean 创建");
    }

    @Override
    public void destroy() {
        // ⚠️ 这段代码永远不会被Spring调用！
        System.out.println(">>> PrototypeBean 销毁 — 不会执行");
    }
}

@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext ctx =
            SpringApplication.run(DemoApplication.class, args);

        // 多次获取多例Bean
        PrototypeBean bean1 = ctx.getBean(PrototypeBean.class);
        PrototypeBean bean2 = ctx.getBean(PrototypeBean.class);
        System.out.println(bean1 == bean2); // false

        ctx.close(); // 容器关闭 — PrototypeBean的@PreDestroy/destroy()不会被调用！
    }
}
```

**如何让Prototype Bean正确释放资源？**

```java
@Component
@Scope("prototype")
public class ManagedPrototypeBean {

    private FileOutputStream outputStream;

    @PostConstruct
    public void init() throws Exception {
        outputStream = new FileOutputStream("data.log");
    }

    // 方式1：提供显式的释放方法，由调用者管理
    public void release() throws Exception {
        outputStream.close();
        System.out.println("资源已释放");
    }
}

// 方式2：使用 ObjectFactory + 自定义销毁注册
@Component
public class PrototypeClient {

    @Autowired
    private ObjectFactory<ManagedPrototypeBean> factory;

    public void process() {
        ManagedPrototypeBean bean = factory.getObject();
        try {
            // 使用bean
        } finally {
            bean.release(); // 显式释放资源
        }
    }
}
```

---

## 九、面试常见问题

### Q1：Bean生命周期的主要阶段是什么？

**答**：实例化 → 属性填充 → Aware回调 → BeanPostProcessor前置处理 → 初始化（@PostConstruct → InitializingBean → init-method）→ BeanPostProcessor后置处理 → Bean就绪 → 销毁（@PreDestroy → DisposableBean → destroy-method）。

### Q2：@PostConstruct、InitializingBean、init-method 的执行顺序？

**答**：`@PostConstruct` 最先，`InitializingBean.afterPropertiesSet()` 其次，`@Bean(initMethod)` / XML `init-method` 最后。执行顺序是由 `CommonAnnotationBeanPostProcessor` 的优先级高于 `InitializingBean` 调用逻辑决定的。

### Q3：BeanPostProcessor 和 BeanFactoryPostProcessor 的区别？

| 对比维度 | BeanPostProcessor | BeanFactoryPostProcessor |
|---|---|---|
| **作用目标** | Bean实例 | BeanFactory（Bean定义） |
| **执行时机** | Bean初始化前后 | 所有Bean定义加载后、实例化前 |
| **操作对象** | 具体的Bean对象 | BeanDefinition元数据 |
| **典型用途** | 创建代理、处理注解 | 修改Bean定义、属性占位符替换 |

### Q4：Spring是如何处理@Autowired注解的？

**答**：`AutowiredAnnotationBeanPostProcessor` 在属性填充阶段扫描Bean中的所有 `@Autowired` 注解。对于每个被注解的字段/方法，它从容器中按类型（byType）查找匹配的Bean，如果有多个则按名称（byName）匹配，还支持 `@Qualifier` 和 `@Primary` 进一步指定。

### Q5：单例Bean和多例Bean的生命周期有何区别？

**答**：核心区别在于**销毁阶段**。单例Bean由Spring容器统一管理销毁，会回调 `@PreDestroy` / `DisposableBean.destroy()` / `destroy-method`。多例Bean的销毁回调**不会被执行**，因为Spring容器在创建并交付后就不再持有多例Bean的引用，资源释放由调用者自行管理。

### Q6：Spring 6.x中Bean生命周期有什么新变化？

**答**：① 全面切换到Jakarta EE命名空间（`jakarta.annotation.PostConstruct` / `PreDestroy`）；② 支持JDK Record类型的Bean自动检测（单构造器 → 属性就是Record组件）；③ 支持GraalVM AOT编译（Bean生命周期在编译期确定）；④ 兼容虚拟线程（但Bean初始化仍在平台线程）。

### Q7：为什么构造器注入能检测循环依赖，而字段注入不能？

**答**：构造器注入发生在**Bean实例化时**，此时Bean还未放入三级缓存，无法提前暴露引用，因此会直接抛出 `BeanCurrentlyInCreationException`。字段注入发生在**属性填充时**，此时Bean的半成品已放入三级缓存，可以被其他Bean引用，从而可能解决循环依赖。

---

## 十、LeetCode题目解析

### 10.1 [278. 第一个错误的版本（Easy）](https://leetcode.cn/problems/first-bad-version/)

**题目描述**：有 `n` 个版本 `[1, 2, ..., n]`，其中一个版本开始之后都是错误的版本。给定API `bool isBadVersion(version)`，找出第一个错误的版本。要求尽量少调用API。

**思路分析**：典型的**二分查找**问题。因为"正确→错误"的分界是单调的，可用二分查找定位第一个错误版本。

```
版本状态：[true, true, true, false, false, false]
                            ↑
                     第一个错误版本（边界）
```

- 如果 `isBadVersion(mid)` 为 true，则第一个错误版本在 `mid` 或其左侧：`right = mid`
- 如果 `isBadVersion(mid)` 为 false，则第一个错误版本在 `mid` 右侧：`left = mid + 1`

```java
public class Solution extends VersionControl {
    public int firstBadVersion(int n) {
        int left = 1, right = n;
        while (left < right) {
            int mid = left + (right - left) / 2; // 防止溢出
            if (isBadVersion(mid)) {
                right = mid; // mid 可能是第一个错误版本
            } else {
                left = mid + 1; // mid 一定不是，往右找
            }
        }
        return left; // left == right 时，就是第一个错误版本
    }
}
```

**复杂度分析**：
- 时间复杂度：O(log n)，每次将搜索范围缩小一半
- 空间复杂度：O(1)，只使用了常数变量

---

### 10.2 [34. 在排序数组中查找元素的第一个和最后一个位置（Medium）](https://leetcode.cn/problems/find-first-and-last-position-of-element-in-sorted-array/)

**题目描述**：给定一个升序数组 `nums` 和目标值 `target`，找出 `target` 在数组中**开始位置**和**结束位置**。不存在返回 `[-1, -1]`。要求 O(log n)。

**思路分析**：两次二分查找 — 第一次找**左边界**（第一个等于target的位置），第二次找**右边界**（最后一个等于target的位置）。

```
nums = [5, 7, 7, 8, 8, 10], target = 8
              ↑     ↑
          leftBound  rightBound
输出：[3, 4]
```

**找左边界**：`nums[mid] >= target` 时 `right = mid`，最终 `left` 停在第一个 `target`。
**找右边界**：`nums[mid] <= target` 时 `left = mid + 1`，最终 `right` 停在最后一个 `target`。

```java
public class Solution {
    public int[] searchRange(int[] nums, int target) {
        if (nums == null || nums.length == 0) {
            return new int[]{-1, -1};
        }

        int leftBound = findLeftBound(nums, target);
        int rightBound = findRightBound(nums, target);

        return new int[]{leftBound, rightBound};
    }

    // 查找左边界（第一个等于target的位置）
    private int findLeftBound(int[] nums, int target) {
        int left = 0, right = nums.length - 1;
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (nums[mid] >= target) {
                right = mid - 1; // 往左收缩
            } else {
                left = mid + 1;
            }
        }
        // left 越界或没找到
        if (left >= nums.length || nums[left] != target) {
            return -1;
        }
        return left;
    }

    // 查找右边界（最后一个等于target的位置）
    private int findRightBound(int[] nums, int target) {
        int left = 0, right = nums.length - 1;
        while (left <= right) {
            int mid = left + (right - left) / 2;
            if (nums[mid] <= target) {
                left = mid + 1; // 往右收缩
            } else {
                right = mid - 1;
            }
        }
        // right 越界或没找到
        if (right < 0 || nums[right] != target) {
            return -1;
        }
        return right;
    }
}
```

**复杂度分析**：
- 时间复杂度：O(log n)，两次独立的二分查找
- 空间复杂度：O(1)

---

## 十一、今日学习要点总结

### 核心要点归纳

1. **Bean生命周期 = 实例化 → 属性填充 → 初始化 → 使用 → 销毁**，全程有十几个扩展点
2. **实例化时**Spring推断构造方法：`@Autowired`构造 > 单构造器 > 多构造器中选默认构造器
3. **属性填充**由 `AutowiredAnnotationBeanPostProcessor` 和 `CommonAnnotationBeanPostProcessor` 驱动
4. **Aware接口**按固定顺序回调：BeanNameAware → BeanFactoryAware → ApplicationContextAware
5. **初始化方法执行顺序**：`@PostConstruct` → `InitializingBean.afterPropertiesSet()` → `init-method`
6. **销毁方法执行顺序**：`@PreDestroy` → `DisposableBean.destroy()` → `destroy-method`
7. **BeanPostProcessor**是Spring最强大的扩展点，AOP代理在 `postProcessAfterInitialization` 中创建
8. **多例Bean Spring不负责销毁**，调用者必须自己管理资源释放
9. **JDK 17 / Spring 6.x**中注意Jakarta命名空间迁移：`jakarta.annotation.PostConstruct`
10. **AOT原生镜像**模式下Bean生命周期在编译期确定，不支持运行期动态注册

### 验收清单

- [ ] 能画出Bean生命周期完整流程图（从实例化到销毁的全部步骤）
- [ ] 能说出三种初始化方法的执行顺序（@PostConstruct → InitializingBean → init-method）
- [ ] 能解释BeanPostProcessor的前置和后置处理分别做什么
- [ ] 能说出至少3个BeanPostProcessor的实际应用场景
- [ ] 能说明单例Bean和多例Bean的生命周期区别（特别是销毁阶段）
- [ ] 理解 @Autowired(required = false) 的替代方案（Optional、@Nullable）
- [ ] 知晓Spring 6.x中Jakarta EE迁移的影响（注解包名变化）
- [ ] 完成LeetCode 278（二分查找边界）和34（查找元素区间）

---

## 十二、练习建议

1. **动手写一个BeanPostProcessor**：统计每个Bean的初始化耗时，输出超过阈值的Bean
2. **阅读Spring源码**：从 `AbstractAutowireCapableBeanFactory.doCreateBean()` 方法入手，跟踪完整流程
   - 核心类：`AbstractAutowireCapableBeanFactory`
   - 关键方法：`createBean()` → `doCreateBean()` → `populateBean()` → `initializeBean()`
3. **验证销毁顺序**：分别用三种方式注册销毁方法，观察容器关闭时的输出
4. **对比单例与多例**：创建两个Bean（一个单例一个多例），在销毁方法中打印日志，验证多例Bean的销毁方法是否被调用
5. **循环依赖实验**：创建两个相互依赖的Bean，分别使用构造器注入和字段注入，观察Spring是否能解决循环依赖
6. **升级到Jakarta**：如果你有Spring Boot 2.x项目，尝试迁移到Spring Boot 3.x，处理 `javax → jakarta` 的命名空间变化

---

## 十三、扩展阅读

- [Spring官方文档 - Bean生命周期回调](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-lifecycle)
- [Spring官方文档 - 自定义Bean（BeanPostProcessor）](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-extension-bpp)
- [Jakarta Annotations规范](https://jakarta.ee/specifications/annotations/)
- [Spring Boot 3.x AOT处理文档](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.aot)
- [Spring AOP代理创建源码分析](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-api)
- **推荐阅读源码**：`AbstractAutowireCapableBeanFactory` 中的 `initializeBean()` 方法，是理解整个生命周期的关键入口
