# Day29 详解 — Spring IOC启动流程

## 〇、Spring Framework版本演进（2025视角）

### 0.1 Spring版本时间线

```
Spring Framework & Spring Boot 版本演进：

2017         2020         2022         2023         2024         2025
  │            │            │            │            │            │
  ▼            ▼            ▼            ▼            ▼            ▼
Spring 5.0   Spring 5.3   Spring 6.0  Spring 6.1  Spring 6.2  Spring 7.x
JDK 8+       JDK 8-17     JDK 17+     JDK 17+     JDK 17+     (预计)
  │            │            │            │            │
  ▼            ▼            ▼            ▼            ▼
Boot 2.0     Boot 2.7     Boot 3.0    Boot 3.2    Boot 3.4
             (2.x终点)    (新起点)    (Virtual     (AOT增强)
                                        Threads)
```

### 0.2 关键Breaking Changes

| 版本 | 变化 | 影响 |
|------|------|------|
| **Spring 6.0 / Boot 3.0** | `javax.*` → `jakarta.*` | 全局包名替换，使用`jakarta.servlet`等 |
| **Spring 6.0 / Boot 3.0** | JDK 17 baseline | 必须使用JDK 17+编译运行 |
| **Spring 6.0 / Boot 3.0** | 移除`spring.factories`部分支持 | 推荐用`AutoConfiguration.imports` |
| **Spring Boot 3.2** | 正式支持Virtual Threads | 通过`spring.threads.virtual.enabled=true`启用 |
| **Spring 6.1** | 引入JDK 21 Record支持 | Bean定义可使用Java Record |

### 0.3 Jakarta EE迁移（面试高频）

Spring Boot 3.x最显著的变化是命名空间从`javax.*`迁移到`jakarta.*`：

```java
// ❌ Spring Boot 2.x 写法（已废弃）
import javax.annotation.PostConstruct;
import javax.servlet.http.HttpServletRequest;
import javax.persistence.Entity;

// ✅ Spring Boot 3.x 写法
import jakarta.annotation.PostConstruct;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.persistence.Entity;
```

### 0.4 Spring Boot 3.x AOT编译支持

Spring Boot 3.0+引入了**AOT（Ahead-of-Time）编译**支持，通过GraalVM将Spring应用编译为原生镜像：

```java
// AOT编译时预处理
// 运行时动态代理 → 编译时静态代理
// 运行时反射 → 编译时注册
// 启动时间从秒级降到毫秒级

// 构建原生镜像命令
// mvn -Pnative native:compile
// 或 gradle bootBuildImage
```

**AOT vs JVM运行对比**：

| 指标 | JVM模式 | AOT原生镜像 |
|------|---------|-------------|
| 启动时间 | 2-5秒 | 0.05-0.1秒 |
| 内存占用 | 200-500MB | 50-100MB |
| 峰值性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 动态代理 | ✅ 支持 | ⚠️ 需预处理 |
| 适用场景 | 长期运行服务 | 短生命周期/Serverless |

### 0.5 Spring Boot 3.2+ Virtual Threads支持

```java
// application.yml (Spring Boot 3.2+)
// spring.threads.virtual.enabled=true

@Configuration
public class VirtualThreadConfig {

    @Bean
    public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
        return protocolHandler -> {
            // Tomcat使用虚拟线程处理请求
            protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        };
    }

    @Bean
    public AsyncTaskExecutor applicationTaskExecutor() {
        // 异步任务使用虚拟线程
        return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
    }
}
```

---

## 一、Spring容器概述

### 1.1 IoC与DI概念

**IoC（控制反转，Inversion of Control）** 是Spring框架的核心思想：将对象的**创建权**和**管理权**从调用方转移到容器。

```
传统编程模式（自己new对象）：
┌─────────────────────────────────────────────┐
│  ServiceA ───── 主动创建 ─────→ ServiceB    │
│  调用方控制对象的创建和生命周期               │
└─────────────────────────────────────────────┘

IoC模式（容器管理对象）：
┌─────────────────────────────────────────────┐
│  ServiceA ───── 被动注入 ─────→ ServiceB    │
│                     ↑                       │
│              ┌──────┴──────┐                │
│              │   IoC容器    │                │
│              │ (Spring容器) │                │
│              └─────────────┘                │
└─────────────────────────────────────────────┘
```

**DI（依赖注入，Dependency Injection）** 是IoC的一种实现方式，容器在创建对象时自动将其依赖的对象注入进来。

```java
// ❌ 没有DI：自己管理依赖
public class OrderService {
    private UserRepository userRepository = new UserRepositoryImpl(); // 硬编码
}

// ✅ 使用DI：容器注入依赖
@Service
public class OrderService {
    private final UserRepository userRepository;

    // 构造器注入（推荐方式）
    public OrderService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

### 1.2 BeanFactory体系

`BeanFactory`是Spring IoC容器的**根接口**，定义了获取Bean、判断Bean是否存在等最基本的操作。

```
BeanFactory接口继承体系：

    ┌──────────────────────────────────────┐
    │          BeanFactory                 │  根接口
    │  + getBean(name)                     │
    │  + containsBean(name)                │
    │  + isSingleton(name)                 │
    │  + getType(name)                     │
    └──────────┬───────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────┐
    │       HierarchicalBeanFactory        │  支持父子容器
    │  + getParentBeanFactory()            │
    │  + containsLocalBean(name)           │
    └──────────┬───────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────┐
    │        ListableBeanFactory           │  批量获取Bean
    │  + getBeanDefinitionNames()          │
    │  + getBeansOfType(type)              │
    │  + getBeanDefinitionCount()          │
    └──────────┬───────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────┐
    │      AutowireCapableBeanFactory      │  自动装配
    │  + autowireBean(existingBean)        │
    │  + configureBean(existingBean)       │
    │  + resolveDependency(descriptor)     │
    └──────────┬───────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────┐
    │     ConfigurableBeanFactory          │  可配置工厂
    │  + addBeanPostProcessor(bpp)         │
    │  + registerScope(scopeName, scope)   │
    │  + setParentBeanFactory(parent)      │
    └──────────┬───────────────────────────┘
               │
               ▼
    ┌──────────────────────────────────────┐
    │   ConfigurableListableBeanFactory    │  完整功能的BeanFactory
    │  + getBeanDefinition(name)           │
    │  + preInstantiateSingletons()        │
    │  + freezeConfiguration()             │
    └──────────────────────────────────────┘
```

**核心实现类**：

```
BeanFactory实现体系：

    DefaultListableBeanFactory ← 默认实现，refresh()中实际使用的类型
            ↑
    XmlBeanFactory（已废弃@Deprecated）
            ↑
    DefaultListableBeanFactory
```

### 1.3 ApplicationContext继承体系

`ApplicationContext`是`BeanFactory`的**子接口**，提供了更高级的容器功能。

```
ApplicationContext继承体系：

    ┌─────────────────────────┐
    │      BeanFactory        │  IoC容器基础
    └───────────┬─────────────┘
                │
    ┌───────────▼─────────────┐
    │  ApplicationContext     │  应用上下文（核心接口）
    │  + getEnvironment()     │
    │  + publishEvent(e)      │
    │  + getMessage(key,args) │
    └───────────┬─────────────┘
                │
      ┌─────────┴──────────┐
      │                    │
      ▼                    ▼
┌──────────────┐  ┌──────────────────┐
│ Configurable │  │  WebApplication  │
│ Application  │  │     Context      │
│   Context    │  │                  │
└──────┬───────┘  └────────┬─────────┘
       │                   │
       ▼                   ▼
┌──────────────┐  ┌──────────────────┐
│  Abstract    │  │  AnnotationConfig│
│ Application  │  │  Application     │
│   Context    │  │     Context      │  ← Spring Boot使用
└──────────────┘  └──────────────────┘
```

**常用ApplicationContext实现**：

| 实现类 | 适用场景 | 引入版本 |
|--------|----------|----------|
| `AnnotationConfigApplicationContext` | 基于注解的独立应用 | Spring 3.0 |
| `AnnotationConfigServletWebServerApplicationContext` | Spring Boot Web应用 | Boot 2.0 |
| `AnnotationConfigReactiveWebServerApplicationContext` | Spring Boot WebFlux应用 | Boot 2.0 |
| `ClassPathXmlApplicationContext` | 基于XML的传统应用（了解） | Spring 1.0 |

---

## 二、BeanFactory vs ApplicationContext

### 2.1 核心区别对比表

| 对比维度 | BeanFactory | ApplicationContext |
|----------|-------------|-------------------|
| **关系** | 根接口 | 子接口（继承BeanFactory） |
| **Bean实例化时机** | **延迟加载**（首次getBean时） | **预加载**（容器启动时） |
| **BeanPostProcessor** | 需手动注册 | 自动注册 |
| **BeanFactoryPostProcessor** | 需手动注册 | 自动注册 |
| **国际化（MessageSource）** | ❌ 不支持 | ✅ 支持 |
| **事件发布** | ❌ 不支持 | ✅ 支持（ApplicationEvent） |
| **环境抽象（Environment）** | ❌ 不支持 | ✅ 支持 |
| **资源加载（ResourceLoader）** | ❌ 不支持 | ✅ 支持 |
| **AOP集成** | 需手动处理 | 自动集成 |
| **使用场景** | 内存敏感/嵌入式设备 | **企业应用（首选）** |

### 2.2 什么时候使用BeanFactory

**绝大多数场景下应使用ApplicationContext**。仅在以下极端场景才考虑BeanFactory：

```java
// 极低内存环境（如IoT设备），延迟初始化可节省启动内存
BeanFactory factory = new DefaultListableBeanFactory();
// 需手动完成ApplicationContext自动做的所有事情
// 不推荐在企业应用中使用
```

### 2.3 ApplicationContext的额外功能

```java
// ApplicationContext的额外能力演示
@Component
public class AppContextDemo implements ApplicationContextAware {

    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;

        // 1. 国际化消息
        String msg = ctx.getMessage("user.welcome",
            new Object[]{"张三"}, Locale.CHINA);

        // 2. 发布事件
        ctx.publishEvent(new UserRegisteredEvent(this, userId));

        // 3. 获取环境配置
        String profile = ctx.getEnvironment()
            .getProperty("spring.profiles.active");

        // 4. 加载资源
        Resource resource = ctx.getResource("classpath:data.sql");
    }
}
```

---

## 三、refresh()方法详解

### 3.1 refresh()完整流程（12+步骤）

`refresh()`方法是Spring IoC容器的**核心入口**，所有初始化逻辑都在这一个方法中完成。该方法定义在`AbstractApplicationContext`中。

```
refresh() 12大步骤全景流程图：

┌─────────────────────────────────────────────────────────────────────────┐
│                          refresh() 启动                                  │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 1: prepareRefresh()                                                 │
│ ├─ 设置容器启动时间                                                      │
│ ├─ 设置closed=false, active=true 状态标志                                │
│ ├─ 初始化PropertySources（环境变量、系统属性）                            │
│ └─ 校验必填属性（validateRequiredProperties）                            │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 2: obtainFreshBeanFactory()                                         │
│ ├─ 如果已有BeanFactory，销毁它并关闭                                     │
│ ├─ 创建新的DefaultListableBeanFactory（默认实现）                        │
│ ├─ 加载BeanDefinition（解析XML/注解/JavaConfig）                         │
│ └─ 给每个Bean分配唯一ID                                                  │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 3: prepareBeanFactory(beanFactory)                                  │
│ ├─ 设置类加载器（ClassLoader）                                           │
│ ├─ 设置SpEL表达式解析器                                                  │
│ ├─ 注册默认的PropertyEditor（类型转换器）                                │
│ └─ 添加ApplicationContextAwareProcessor等特殊BeanPostProcessor           │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 4: postProcessBeanFactory(beanFactory)                              │
│ ├─ 模板方法，留给子类扩展                                                │
│ ├─ 如：ServletContextAwareProcessor在此注册（Web容器）                   │
│ └─ 子类可在此注册特殊的BeanPostProcessor                                 │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 5: invokeBeanFactoryPostProcessors(beanFactory)                     │
│ ├─ ★★★ 核心步骤：执行所有BeanFactoryPostProcessor                       │
│ ├─ 先执行BeanDefinitionRegistryPostProcessor（可注册新BeanDefinition）  │
│ │   ├─ ConfigurationClassPostProcessor ← 解析@Configuration类           │
│ │   │   (扫描@ComponentScan、处理@Bean、@Import等)                       │
│ │   └─ 解析所有配置类，生成BeanDefinition并注册                          │
│ ├─ 再执行普通BeanFactoryPostProcessor                                    │
│ └─ 此时BeanDefinition已全部就绪，但尚未实例化Bean                        │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 6: registerBeanPostProcessors(beanFactory)                          │
│ ├─ 注册所有BeanPostProcessor实例到容器                                   │
│ ├─ BeanPostProcessor在Bean初始化前后执行                                 │
│ ├─ 关键处理器：AutowiredAnnotationBeanPostProcessor（处理@Autowired）    │
│ ├─ 关键处理器：CommonAnnotationBeanPostProcessor（处理@PostConstruct等）│
│ └─ 注意：仅注册，尚未使用（Bean实例化时才用）                            │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 7: initMessageSource()                                              │
│ ├─ 初始化MessageSource（国际化支持）                                     │
│ ├─ 若用户未定义messageSource Bean，创建DelegatingMessageSource           │
│ └─ 支持从messages.properties等多语言文件读取                             │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 8: initApplicationEventMulticaster()                                │
│ ├─ 初始化事件多播器（SimpleApplicationEventMulticaster）                 │
│ ├─ 将事件广播给所有注册的监听器                                          │
│ └─ 可配置线程池实现异步事件                                              │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 9: onRefresh()                                                      │
│ ├─ 模板方法，留给子类扩展                                                │
│ ├─ Spring Boot中：启动内嵌Web服务器（Tomcat/Jetty/Undertow）             │
│ └─ 此处是Web服务器启动的入口                                             │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 10: registerListeners()                                             │
│ ├─ 注册所有ApplicationListener Bean到事件多播器                          │
│ ├─ 广播earlyApplicationEvents（早期事件）                                │
│ └─ 事件机制从此正式生效                                                  │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 11: finishBeanFactoryInitialization(beanFactory) ★★★                │
│ ├─ 实例化所有非延迟加载的单例Bean（真正干活的步骤）                      │
│ ├─ 冻结配置（freezeConfiguration），防止后续修改                         │
│ ├─ beanFactory.preInstantiateSingletons()                                │
│ │   └─ 遍历所有BeanDefinition                                            │
│ │       └─ getBean(beanName) → 触发完整的Bean生命周期                     │
│ │           ├─ 实例化（createBeanInstance）                               │
│ │           ├─ 属性填充（populateBean）                                   │
│ │           ├─ 初始化（initializeBean）                                   │
│ │           │   ├─ Aware接口回调                                         │
│ │           │   ├─ BeanPostProcessor.postProcessBeforeInitialization     │
│ │           │   ├─ @PostConstruct                                        │
│ │           │   └─ BeanPostProcessor.postProcessAfterInitialization      │
│ │           └─ 放入单例池（singletonObjects）                            │
│ └─ 自此，所有单例Bean就绪                                                │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ Step 12: finishRefresh()                                                 │
│ ├─ 清除资源缓存                                                          │
│ ├─ 初始化LifecycleProcessor（回调SmartLifecycle.start()）                │
│ ├─ 发布ContextRefreshedEvent事件（容器刷新完毕）                         │
│ └─ 注册到LiveBeansView（JMX支持）                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 核心步骤深入

#### 3.2.1 BeanFactoryPostProcessor vs BeanDefinitionRegistryPostProcessor

这是面试**高频考点**。两者都在Step 5中执行，但功能和顺序不同：

```
执行顺序：
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  1. BeanDefinitionRegistryPostProcessor（优先执行）      │
│     ├─ 实现 PriorityOrdered 接口的                       │
│     ├─ 实现 Ordered 接口的                               │
│     └─ 普通无排序的                                      │
│                                                          │
│  2. BeanFactoryPostProcessor（后执行）                   │
│     ├─ 实现 PriorityOrdered 接口的                       │
│     ├─ 实现 Ordered 接口的                               │
│     └─ 普通无排序的                                      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

```java
// BeanDefinitionRegistryPostProcessor：可以注册新的BeanDefinition
// 最典型的实现：ConfigurationClassPostProcessor
// 负责解析@Configuration、@ComponentScan、@Bean等注解

// BeanFactoryPostProcessor：只能修改已注册的BeanDefinition
// 最典型的实现：PropertySourcesPlaceholderConfigurer
// 负责解析${}占位符，替换为真实值
```

| 对比维度 | BeanFactoryPostProcessor | BeanDefinitionRegistryPostProcessor |
|----------|--------------------------|-------------------------------------|
| 执行阶段 | refresh() Step 5（后） | refresh() Step 5（先） |
| 核心能力 | 修改已有BeanDefinition的属性 | 注册新的BeanDefinition |
| 能否新增Bean | ❌ | ✅ |
| 典型实现 | PropertySourcesPlaceholderConfigurer | ConfigurationClassPostProcessor |

```java
// 自定义BeanDefinitionRegistryPostProcessor示例
@Component
public class MyBeanDefinitionRegistryPostProcessor
        implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(
            BeanDefinitionRegistry registry) {
        // 动态注册BeanDefinition
        BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .genericBeanDefinition(MyService.class);
        registry.registerBeanDefinition("myService",
            builder.getBeanDefinition());
    }

    @Override
    public void postProcessBeanFactory(
            ConfigurableListableBeanFactory beanFactory) {
        // 修改已有BeanDefinition的属性
        BeanDefinition bd = beanFactory.getBeanDefinition("myService");
        bd.setLazyInit(true); // 改为延迟加载
    }
}
```

#### 3.2.2 finishBeanFactoryInitialization详解

这是**最重要**的步骤，所有非延迟单例Bean在此实例化。

```
finishBeanFactoryInitialization() 内部流程：

beanFactory.preInstantiateSingletons()
    │
    ├─ 遍历所有BeanDefinition名称列表
    │
    ├─ 判断条件：非抽象 && 单例 && 非延迟加载
    │   │
    │   ├─ 是FactoryBean？
    │   │   ├─ YES → 使用FactoryBean创建逻辑（&beanName）
    │   │   └─ NO  → getBean(beanName)
    │   │
    │   └─ getBean() → doGetBean() 完整流程：
    │       ├─ 检查单例缓存（singletonObjects）
    │       ├─ 未命中 → 获取BeanDefinition
    │       ├─ 处理依赖（depends-on）
    │       ├─ 根据scope创建：
    │       │   ├─ singleton → createBean → 放入单例池
    │       │   ├─ prototype → createBean（不缓存）
    │       │   └─ request/session → scope中创建
    │       ├─ createBean过程（简化）：
    │       │   ├─ 实例化（反射/Supplier/CGLIB代理）
    │       │   ├─ 属性填充（依赖注入）
    │       │   └─ 初始化（Aware + BPP + @PostConstruct + InitializingBean）
    │       └─ 返回Bean实例
    │
    └─ 所有单例创建完毕后，调用SmartInitializingSingleton.afterSingletonsInstantiated()
```

#### 3.2.3 事件多播器初始化

```java
// Step 8: initApplicationEventMulticaster()
// 默认实现：SimpleApplicationEventMulticaster

// 底层源码逻辑（简化）：
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 1. 检查用户是否自定义了 applicationEventMulticaster Bean
    if (beanFactory.containsLocalBean(
            APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster = beanFactory.getBean(
            APPLICATION_EVENT_MULTICASTER_BEAN_NAME);
    } else {
        // 2. 使用默认的 SimpleApplicationEventMulticaster
        this.applicationEventMulticaster =
            new SimpleApplicationEventMulticaster(beanFactory);
    }
}
```

---

## 四、BeanDefinition加载与注册

### 4.1 BeanDefinition是什么

`BeanDefinition`是Spring中对Bean定义的**元数据描述**，它包含了创建Bean实例所需的所有信息。

```
BeanDefinition = Bean的"配方/说明书"

┌─────────────────────────────────────────────────────────┐
│  BeanDefinition 包含的信息：                             │
│                                                         │
│  ├─ beanClassName      "com.example.UserService"        │
│  ├─ scope              singleton / prototype / request  │
│  ├─ lazyInit           true / false                     │
│  ├─ dependsOn          ["dataSource", "userDao"]        │
│  ├─ constructorArguments 构造器参数值                   │
│  ├─ propertyValues     属性值（setter注入）              │
│  ├─ initMethodName     @PostConstruct对应的方法         │
│  ├─ destroyMethodName  @PreDestroy对应的方法            │
│  ├─ factoryBeanName    工厂Bean名                       │
│  ├─ factoryMethodName  工厂方法名（@Bean方法）           │
│  ├─ primary            是否优先注入                      │
│  └─ autowireMode       AUTOWIRE_BY_TYPE / BY_NAME       │
└─────────────────────────────────────────────────────────┘
```

### 4.2 BeanDefinition注册流程

```
注册流程图：

源代码（注解/XML/JavaConfig）
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  解析器（Reader/Parser）                                 │
│  ├─ AnnotatedBeanDefinitionReader（注解解析）            │
│  ├─ XmlBeanDefinitionReader（XML解析）                   │
│  └─ ClassPathBeanDefinitionScanner（@ComponentScan扫描） │
└─────────────────────────────────┬───────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────┐
│  BeanDefinition（元数据对象）                            │
│  ├─ GenericBeanDefinition                                │
│  ├─ RootBeanDefinition                                   │
│  ├─ AnnotatedGenericBeanDefinition                       │
│  └─ ScannedGenericBeanDefinition                         │
└─────────────────────────────────┬───────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────┐
│  BeanDefinitionRegistry.registerBeanDefinition(name, bd) │
│  ├─ 底层的DefaultListableBeanFactory维护一个ConcurrentHashMap│
│  │   Map<String, BeanDefinition> beanDefinitionMap =    │
│  │       new ConcurrentHashMap<>(256);                   │
│  ├─ 检查名称是否重复                                     │
│  └─ 存入beanDefinitionMap                                │
└─────────────────────────────────────────────────────────┘
```

### 4.3 BeanDefinition常用属性

```java
// BeanDefinition核心属性一览
public class BeanDefinitionDemo {

    public void showBeanDefinition() {
        // 1. 类名
        String beanClassName = "com.example.UserService";

        // 2. 作用域：singleton(默认) / prototype / request / session
        String scope = "singleton";

        // 3. 是否延迟初始化
        boolean lazyInit = false; // 默认false，即启动时初始化

        // 4. 是否抽象Bean（模板Bean）
        boolean isAbstract = false;

        // 5. 是否主候选Bean（@Primary）
        boolean isPrimary = true;

        // 6. 初始化方法名
        String initMethodName = "init";

        // 7. 销毁方法名
        String destroyMethodName = "destroy";

        // 8. 自动装配模式
        int autowireMode = AUTOWIRE_BY_TYPE; // 按类型装配

        // 9. 构造器参数
        ConstructorArgumentValues constructorValues;

        // 10. 属性值（setter注入）
        MutablePropertyValues propertyValues;

        // 11. 工厂Bean名（@Bean定义时非空）
        String factoryBeanName;

        // 12. 工厂方法名（@Bean方法名）
        String factoryMethodName;

        // 13. 依赖的Bean名
        String[] dependsOn = {"dataSource"};
    }
}
```

---

## 五、配置解析方式

### 5.1 XML配置（历史，了解即可）

Spring Boot时代之前的主流配置方式，现已逐渐被注解和JavaConfig替代。

```xml
<!-- applicationContext.xml（传统方式，了解即可） -->
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userService" class="com.example.UserService">
        <property name="userDao" ref="userDao"/>
    </bean>

    <bean id="userDao" class="com.example.UserDao"/>

    <!-- 组件扫描（XML方式启用） -->
    <context:component-scan base-package="com.example"/>

</beans>
```

**2025年面试中**：通常不提XML，但要知道`XmlBeanDefinitionReader`负责解析。

### 5.2 注解驱动（@ComponentScan、@Service、@Repository等）

**注解驱动**是Spring 3.0+推荐的方式，通过类路径扫描自动发现组件。

```
注解扫描流程：

@ComponentScan(basePackages = "com.example")
    │
    ▼
ClassPathBeanDefinitionScanner
    │
    ├─ 扫描指定包下所有.class文件
    │
    ├─ 判断是否有注解标记：
    │   ├─ @Component（通用组件）
    │   ├─ @Service（服务层）
    │   ├─ @Repository（数据访问层）
    │   ├─ @Controller / @RestController（控制层）
    │   └─ @Configuration（配置类）
    │
    ├─ 生成ScannedGenericBeanDefinition
    │
    └─ 注册到BeanDefinitionRegistry
```

```java
// 使用示例
@Configuration
@ComponentScan(basePackages = "com.example.demo",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.REGEX,
        pattern = "com.example.demo.exclude.*"))
public class AppConfig {
}

// 被扫描到的组件
@Service
public class UserService {
    @Autowired
    private UserRepository userRepository;
}

@Repository
public class UserRepository {
}
```

**@Component、@Service、@Repository、@Controller的关系**：

| 注解 | 用途 | 额外语义 |
|------|------|----------|
| `@Component` | 通用Spring组件 | 无 |
| `@Service` | 业务逻辑层 | 无额外（语义标记） |
| `@Repository` | 数据访问层 | 自动转换持久层异常（PersistenceExceptionTranslation） |
| `@Controller` | Web控制层 | MVC相关处理 |
| `@RestController` | REST API | = @Controller + @ResponseBody |
| `@Configuration` | 配置类 | 包含@Bean方法，CGLIB代理 |

### 5.3 JavaConfig（@Configuration + @Bean）

**JavaConfig**是Spring 3.0+引入、Spring Boot首选的纯Java配置方式。

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("root");
        config.setPassword("password");
        return new HikariDataSource(config);
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

**@Configuration的CGLIB代理机制**：

```java
// @Configuration类会被CGLIB代理
// 确保@Bean方法的"单例"语义
@Configuration
public class AppConfig {

    @Bean
    public A a() {
        return new A(b()); // 调用b()时，返回的是容器中的单例B，而非new B()
    }

    @Bean
    public B b() {
        return new B();
    }
}

// 等价于（CGLIB代理后的伪代码）：
// class AppConfig$$EnhancerBySpringCGLIB extends AppConfig {
//     @Override
//     public B b() {
//         if (容器中已有B实例) {
//             return 容器中的B实例;
//         }
//         return super.b();
//     }
// }
```

### 5.4 Spring Boot自动装配

**自动装配（Auto-Configuration）**是Spring Boot的**杀手级特性**，它根据classpath中的依赖自动配置Spring应用。

```
自动装配原理：

@SpringBootApplication
    │
    ├─ @SpringBootConfiguration（= @Configuration）
    ├─ @EnableAutoConfiguration ★★★ 核心
    │   └─ @Import(AutoConfigurationImportSelector.class)
    │       └─ 从META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    │           读取所有自动配置类（Spring Boot 3.x新格式）
    │
    └─ @ComponentScan（扫描当前包下组件）
```

**自动配置类注册文件（Spring Boot 3.x）**：

```
旧格式（Spring Boot 2.6之前）：META-INF/spring.factories
新格式（Spring Boot 2.7+ / 3.x）：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports

# org.springframework.boot.autoconfigure.AutoConfiguration.imports
# 每行一个全限定类名（新格式）
org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

```java
// 条件装配的核心注解

@AutoConfiguration // Spring Boot 2.7+新注解，替代@Configuration
@ConditionalOnClass(DataSource.class) // classpath中存在DataSource
@ConditionalOnProperty(name = "spring.datasource.enabled", matchIfMissing = true)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean // 用户未自定义时生效
    @ConditionalOnProperty(name = "spring.datasource.url")
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

**Spring Boot自动装配核心条件注解**：

| 条件注解 | 作用 | 示例 |
|----------|------|------|
| `@ConditionalOnClass` | classpath中存在指定类时生效 | `@ConditionalOnClass(name = "com.mysql.cj.jdbc.Driver")` |
| `@ConditionalOnMissingClass` | classpath中不存在指定类时生效 | 备选方案时使用 |
| `@ConditionalOnBean` | 容器中存在指定Bean时生效 | `@ConditionalOnBean(DataSource.class)` |
| `@ConditionalOnMissingBean` | 容器中不存在指定Bean时生效 | 用户未自定义时启用默认配置 |
| `@ConditionalOnProperty` | 配置属性满足条件时生效 | `@ConditionalOnProperty(name = "feature.enabled")` |
| `@ConditionalOnWebApplication` | Web应用环境下生效 | `@ConditionalOnWebApplication(type = SERVLET)` |
| `@ConditionalOnResource` | classpath中存在指定资源时生效 | `@ConditionalOnResource(resources = "classpath:logging.properties")` |

---

## 六、面试常见问题

**Q1：BeanFactory和FactoryBean的区别？**

| 对比 | BeanFactory | FactoryBean |
|------|-------------|-------------|
| 本质 | IoC容器（接口） | 创建复杂Bean的工厂Bean |
| 作用 | 管理所有Bean | 创建特定类型的Bean |
| 获取 | 本身就是容器 | 通过`&beanName`获取工厂本身，通过`beanName`获取产品 |
| 典型应用 | 容器基础 | 创建MyBatis Mapper代理、Feign客户端等 |

```java
// FactoryBean示例：创建一个复杂的Bean
@Component
public class MyFactoryBean implements FactoryBean<ComplexObject> {
    @Override
    public ComplexObject getObject() {
        return new ComplexObject(/* 复杂构造逻辑 */);
    }
    @Override
    public Class<?> getObjectType() {
        return ComplexObject.class;
    }
    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

**Q2：Spring容器启动时，Bean的实例化顺序是怎样的？**

大致顺序：按**依赖关系**排序后实例化。先实例化被依赖的Bean（如DataSource），再实例化依赖它的Bean（如JdbcTemplate）。`@DependsOn`可显式指定顺序，`@Order`/`@Priority`用于集合注入时的排序（不影响实例化顺序）。

**Q3：Spring如何解决循环依赖？**

Spring通过**三级缓存**解决单例Bean的**setter注入**循环依赖：
- 一级缓存 `singletonObjects`：完全初始化好的Bean
- 二级缓存 `earlySingletonObjects`：早期暴露的半成品Bean
- 三级缓存 `singletonFactories`：Bean工厂（可生成代理对象）

**构造器注入无法解决循环依赖**，会抛出`BeanCurrentlyInCreationException`。

**Q4：refresh()方法中哪些步骤是线程安全的？**

`refresh()`方法本身由`synchronized`关键字保护，确保同一时间只有一个线程执行。Spring通过`startupShutdownMonitor`对象锁保证容器初始化是单线程的，但Bean初始化完成后，容器对外提供Bean的操作是线程安全的（使用`ConcurrentHashMap`）。

**Q5：ApplicationContext和BeanFactory的加载时机有何不同？**

- `BeanFactory`：**懒加载**，`getBean()`时才创建Bean实例
- `ApplicationContext`：**预加载**，`refresh()`时就创建所有非延迟单例Bean

这意味着`ApplicationContext`启动较慢，但运行时性能更好（无首次创建开销）。可通过`@Lazy`注解让特定Bean延迟加载。

**Q6：Spring Boot启动时如何决定启用哪些自动配置？**

通过`@Conditional`系列注解的条件判断：
1. 根据classpath中有哪些类（`@ConditionalOnClass`）
2. 根据用户是否自定义了Bean（`@ConditionalOnMissingBean`）
3. 根据配置文件中的属性值（`@ConditionalOnProperty`）
4. 根据应用类型（Web/非Web，`@ConditionalOnWebApplication`）

只有满足所有条件的自动配置才会生效。

**Q7：Spring Boot 3.x中spring.factories和AutoConfiguration.imports的区别？**

| 维度 | spring.factories (旧) | AutoConfiguration.imports (新) |
|------|-----------------------|-------------------------------|
| 文件路径 | `META-INF/spring.factories` | `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` |
| 格式 | Key-Value（key=自动配置注册Key） | 纯文本，每行一个类名 |
| 引入版本 | Spring Boot 1.0 | Spring Boot 2.7+（3.x推荐） |
| 功能 | 通用SPI机制（多种用途） | 专用于自动配置类注册 |

---

## 七、LeetCode题目解析

### 7.1 [704. 二分查找（Easy）](https://leetcode.cn/problems/binary-search/)

**题目描述**：给定一个`n`个元素有序的（升序）整型数组`nums`和一个目标值`target`，写一个函数搜索`nums`中的`target`，如果目标值存在返回下标，否则返回`-1`。

**思路分析**：经典的二分查找。维护左右指针`left`和`right`，每次取中间位置`mid`与`target`比较。若`nums[mid] < target`，说明目标在右半部分，`left = mid + 1`；若`nums[mid] > target`，说明目标在左半部分，`right = mid - 1`；相等则返回`mid`。

**Java代码**：

```java
public class BinarySearch {
    public int search(int[] nums, int target) {
        int left = 0;
        int right = nums.length - 1;

        while (left <= right) {
            // 防止(left + right)溢出
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) {
                return mid;
            } else if (nums[mid] < target) {
                left = mid + 1; // 目标在右半部分
            } else {
                right = mid - 1; // 目标在左半部分
            }
        }
        return -1; // 未找到
    }
}
```

**复杂度分析**：
- 时间复杂度：**O(log n)**，每次将搜索范围减半
- 空间复杂度：**O(1)**，只使用常数级额外空间

---

### 7.2 [35. 搜索插入位置（Easy）](https://leetcode.cn/problems/search-insert-position/)

**题目描述**：给定一个排序数组和一个目标值，在数组中找到目标值并返回其索引。如果目标值不存在于数组中，返回它将会被按顺序插入的位置。请必须使用时间复杂度为O(log n)的算法。

**思路分析**：本质仍是二分查找。当目标值存在时直接返回索引；当目标值不存在时，最终`left`指针的位置恰好就是目标值应该插入的位置（因为循环退出条件为`left > right`，`left`指向了第一个大于等于`target`的位置）。

**Java代码**：

```java
public class SearchInsertPosition {
    public int searchInsert(int[] nums, int target) {
        int left = 0;
        int right = nums.length - 1;

        while (left <= right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) {
                return mid; // 找到目标值
            } else if (nums[mid] < target) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        // 未找到目标值，left即为插入位置
        return left;
    }
}
```

**复杂度分析**：
- 时间复杂度：**O(log n)**，标准的二分查找
- 空间复杂度：**O(1)**，只使用常数级额外空间

**关键理解**：为什么找不到时返回`left`？

```
示例：nums = [1,3,5,6], target = 2

初始：left=0, right=3, mid=1 → nums[1]=3 > 2 → right=0
      left=0, right=0, mid=0 → nums[0]=1 < 2 → left=1
      left=1, right=0 → 退出循环，left=1正是插入位置 ✓

示例：nums = [1,3,5,6], target = 7

初始：left=0, right=3, mid=1 → nums[1]=3 < 7 → left=2
      left=2, right=3, mid=2 → nums[2]=5 < 7 → left=3
      left=3, right=3, mid=3 → nums[3]=6 < 7 → left=4
      left=4, right=3 → 退出循环，left=4正是插入位置 ✓
```

---

## 八、今日学习要点总结

### 核心要点归纳

1. **IoC控制反转**是Spring的核心理念，将对象的创建和管理交给容器，实现解耦
2. **BeanFactory**是基础IoC容器（延迟加载），**ApplicationContext**是其子接口（预加载 + 事件 + 国际化 + 资源加载）
3. **refresh()方法**是容器启动的核心入口，包含12个步骤，其中Step 5（执行BeanFactoryPostProcessor）、Step 6（注册BeanPostProcessor）、Step 11（实例化单例Bean）最为关键
4. **BeanDefinitionRegistryPostProcessor**（先执行，可注册新BeanDefinition）vs **BeanFactoryPostProcessor**（后执行，可修改已有BeanDefinition）
5. `ConfigurationClassPostProcessor`是Spring注解驱动的核心，负责解析`@Configuration`、`@ComponentScan`、`@Bean`等注解
6. BeanDefinition的注册最终存储在`DefaultListableBeanFactory`的`beanDefinitionMap`（ConcurrentHashMap）中
7. Spring Boot自动装配通过`@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → 读取`AutoConfiguration.imports`文件实现
8. Spring Boot 3.x关键变化：`javax.*` → `jakarta.*`、JDK 17 baseline、支持Virtual Threads（3.2+）、`spring.factories` → `AutoConfiguration.imports`

### 验收清单

- [ ] 能画出Spring IOC启动流程图（至少包含refresh()的12个核心步骤）
- [ ] 能解释BeanFactory和ApplicationContext的区别（至少说出5个区别点）
- [ ] 能说出refresh()方法的核心步骤（特别是Step 5、Step 6、Step 11）
- [ ] 能说明BeanDefinition是如何被加载的（从注解/JavaConfig/XML到beanDefinitionMap的完整路径）
- [ ] 能解释BeanFactoryPostProcessor和BeanDefinitionRegistryPostProcessor的区别和执行顺序
- [ ] 能说明Spring Boot自动装配的原理（@EnableAutoConfiguration → AutoConfiguration.imports → @Conditional条件判断）
- [ ] 完成LeetCode题目704（二分查找）和35（搜索插入位置）

---

## 九、练习建议

1. **源码调试**：在IDEA中新建Spring Boot项目，在`refresh()`方法的12个步骤分别设置断点，逐步调试观察每个步骤做了什么。重点关注Step 5中`ConfigurationClassPostProcessor`如何解析`@ComponentScan`。

2. **画图练习**：手绘Spring IOC启动流程图（至少包含refresh()的12个步骤），尽量标注每个步骤的关键类名和方法名。

3. **代码实践**：
   - 自定义一个`BeanDefinitionRegistryPostProcessor`，动态注册一个Bean
   - 自定义一个`BeanFactoryPostProcessor`，修改已有Bean的属性
   - 通过`@ConditionalOnProperty`控制某个`@Configuration`类是否生效

4. **LeetCode巩固**：独立完成704和35，重点理解二分查找的边界条件（`left <= right` vs `left < right`，`mid`计算公式防止溢出）。

5. **自动装配探索**：查看`spring-boot-autoconfigure`包下的`AutoConfiguration.imports`文件，了解Spring Boot默认提供了哪些自动配置。

---

## 十、扩展阅读

- [Spring Framework 6.x 官方文档 - IoC容器](https://docs.spring.io/spring-framework/reference/core/beans.html)
- [Spring Boot 3.x 官方文档 - 自动装配](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Spring Boot 3.x Migration Guide（javax→jakarta迁移）](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-3.0-Migration-Guide)
- [Spring Boot 3.2 虚拟线程支持](https://spring.io/blog/2023/11/23/spring-boot-3-2-0-available-now/)
- [Spring 6.1 AOT与GraalVM原生镜像](https://docs.spring.io/spring-framework/reference/core/aot.html)
- 《Spring揭秘》王福强 — 深入理解Spring IoC容器实现
- `AbstractApplicationContext.refresh()` 源码注释非常详细，建议直接阅读
