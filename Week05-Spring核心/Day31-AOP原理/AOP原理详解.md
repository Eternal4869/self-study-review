# Day31 详解 — AOP原理

## 今日目标
- 深入理解AOP核心概念（切面、切点、通知、连接点、织入）
- 掌握JDK动态代理和CGLIB动态代理的原理与实现
- 理解Spring AOP代理创建过程和调用流程
- 熟练使用@Aspect注解定义切面和通知
- 完成两道LeetCode算法题

---

## 〇、Spring AOP版本演进（2025视角）

### Spring 5.x → 6.x 的关键变化

Spring AOP从5.x到6.x经历了重大底层重构。Spring 6.x基于**JDK 17+**，引入了对GraalVM原生镜像的支持，AOP代理机制也随之调整。

| 特性 | Spring 5.x | Spring 6.x (Spring Boot 3.x) |
|------|-----------|------|
| **JDK基线** | JDK 8 | JDK 17 |
| **默认代理方式** | JDK代理（有接口时） | **CGLIB代理**（始终） |
| **字节码工具** | CGLIB（Fork版） | Spring内置`spring-core` CGLIB fork，移除net.sf.cglib依赖 |
| **AOT编译** | 不支持 | 支持（GraalVM Native Image） |
| **@EnableAspectJAutoProxy** | proxyTargetClass默认false | Spring Boot自动配置默认proxyTargetClass=true |

### Spring Boot 3.x AOP 默认行为

```java
// Spring Boot 2.x 之前: 有接口→JDK代理, 无接口→CGLIB代理
// Spring Boot 2.0+: 默认使用CGLIB代理（spring.aop.auto=true, spring.aop.proxy-target-class=true）
// Spring Boot 3.x: 延续此策略，且更激进地移除JDK代理的回退逻辑
```

**关键变化**：
- Spring Boot 3.x默认`spring.aop.proxy-target-class=true`
- 即使Bean实现了接口，也**强制使用CGLIB代理**
- 如需使用JDK代理，必须手动设置`spring.aop.proxy-target-class=false`
- 在Native Image中，代理必须在AOT阶段提前生成

### Spring AOP vs AspectJ 对比

| 维度 | Spring AOP | AspectJ |
|------|-----------|---------|
| **实现方式** | 动态代理（运行时） | 编译期/类加载期织入（静态） |
| **代理对象** | ✅ 创建代理对象 | ❌ 不需要代理，直接修改字节码 |
| **连接点范围** | 仅方法执行 | 方法调用/构造函数/字段访问/异常处理等 |
| **性能** | 运行时反射，略慢 | 编译期织入，运行时零开销 |
| **使用复杂度** | 简单（注解即可） | 需要AspectJ编译器或LTW |
| **依赖** | spring-aop | aspectjweaver.jar |
| **Spring兼容** | 原生集成 | 需要@EnableLoadTimeWeaving |

> **结论**：Spring AOP = "够用"的AOP，满足90%场景（事务管理、日志记录、权限校验）。需要更细粒度控制时（如拦截private方法），才需要AspectJ。

**现代@Aspect注解驱动 vs XML配置**：
- ✅ **注解驱动**（推荐）：`@Aspect` + `@Component`，Spring Boot 3.x标准做法
- ❌ **XML配置**：已过时，Spring Boot 3.x不推荐，只有遗留项目中使用
- 本详解统一采用**注解驱动**方式讲解

---

## 一、AOP核心概念

AOP（Aspect-Oriented Programming，面向切面编程）是OOP的补充，用于将**横切关注点**（Cross-cutting Concerns）从业务逻辑中分离出来。

### 1.1 切面（Aspect）

**切面是通知和切点的集合**——它定义了"在何时（通知）、在何处（切点）"执行横切逻辑。

```java
@Aspect     // ← 声明这是一个切面
@Component  // ← 让Spring管理
public class LoggingAspect {
    // 切面 = 切点 + 通知
}
```

**类比**：切面就像"插件"，可以插入到系统的不同位置，而不修改原有代码。

### 1.2 切点（Pointcut）

**切点是一组连接点的谓词表达式**，决定了通知"在何处"执行。Spring使用**AspectJ切点表达式语言**。

```java
// 切点表达式：匹配com.example.service包下所有类的所有public方法
@Pointcut("execution(public * com.example.service.*.*(..))")
public void serviceLayer() {}  // 切点签名，方法体为空
```

常用切点指示器：

| 指示器 | 说明 | 示例 |
|-------|------|------|
| `execution()` | 匹配方法执行 | `execution(* com.example..*.*(..))` |
| `@annotation()` | 匹配带有指定注解的方法 | `@annotation(org.springframework.transaction.annotation.Transactional)` |
| `within()` | 匹配指定类型内的方法 | `within(com.example.service.*)` |
| `@within()` | 匹配带有指定注解的类型 | `@within(org.springframework.stereotype.Service)` |
| `args()` | 匹配参数类型 | `args(java.lang.String)` |
| `@args()` | 匹配参数上有指定注解的方法 | `@args(com.example.NotNull)` |
| `this()` | 匹配代理对象类型 | `this(com.example.UserService)` |
| `target()` | 匹配目标对象类型 | `target(com.example.UserServiceImpl)` |
| `bean()` | 按bean名称匹配（Spring特有） | `bean(userService)` |

### 1.3 通知（Advice）

**通知定义了"在何时"执行切面逻辑**。

| 通知类型 | 注解 | 执行时机 |
|---------|------|---------|
| **前置通知** | `@Before` | 目标方法执行前 |
| **后置返回通知** | `@AfterReturning` | 目标方法正常返回后 |
| **后置异常通知** | `@AfterThrowing` | 目标方法抛出异常后 |
| **后置最终通知** | `@After` | 目标方法执行后（类似finally） |
| **环绕通知** | `@Around` | 包裹目标方法，前后都可执行 |

**执行顺序**（正常情况）：
```
@Around 前 → @Before → 目标方法 → @Around 后 → @AfterReturning → @After
```

**执行顺序**（异常情况）：
```
@Around 前 → @Before → 目标方法抛异常 → @AfterThrowing → @After
```

### 1.4 连接点（Join Point）

**连接点是程序执行过程中能被织入切面的点**。在Spring AOP中，连接点**永远是方法执行**。

```java
@Before("serviceLayer()")
public void logBefore(JoinPoint joinPoint) {
    String methodName = joinPoint.getSignature().getName(); // 获取方法名
    Object[] args = joinPoint.getArgs();                     // 获取参数
    Object target = joinPoint.getTarget();                   // 获取目标对象
}
```

### 1.5 织入（Weaving）

**织入是将切面应用到目标对象，创建代理对象的过程**。

Spring AOP使用**运行时织入**（通过动态代理），在Spring容器初始化Bean时完成。

### 概念关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        切面 (Aspect)                         │
│  ┌─────────────────────────┐  ┌───────────────────────────┐ │
│  │    切点 (Pointcut)       │  │     通知 (Advice)          │ │
│  │  ┌─────────────────┐    │  │  @Before                  │ │
│  │  │ 匹配一组连接点    │    │  │  @After                   │ │
│  │  │ 即"在哪里"      │    │  │  @Around                  │ │
│  │  └─────────────────┘    │  │  即"干什么"               │ │
│  └─────────────────────────┘  └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ 织入(Weaving)
              ┌───────────────────────────────┐
              │         代理对象               │
              │  ┌─────────────────────────┐  │
              │  │  JDK代理 或 CGLIB代理    │  │
              │  └─────────────────────────┘  │
              └───────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────┐
    │     连接点 (Join Point)                          │
    │     method1()  method2()  method3()  ...        │
    │     ↑ 方法执行点（Spring AOP仅支持方法级别）       │
    └─────────────────────────────────────────────────┘
```

---

## 二、JDK动态代理

JDK动态代理是Java原生提供的代理机制，基于**接口**创建代理对象。

### 2.1 InvocationHandler接口

```java
package java.lang.reflect;

@FunctionalInterface
public interface InvocationHandler {
    /**
     * @param proxy  代理对象本身
     * @param method 被调用的方法
     * @param args   方法参数
     * @return 方法返回值
     */
    Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
}
```

**原理**：所有对代理对象的方法调用都会被**拦截并转发**到`invoke()`方法中，由开发者决定如何处理。

### 2.2 Proxy.newProxyInstance

```java
public static Object newProxyInstance(
    ClassLoader loader,      // 类加载器（通常用目标类的类加载器）
    Class<?>[] interfaces,   // 目标类实现的接口列表
    InvocationHandler h      // 调用处理器
)
```

**底层原理**：
1. JVM根据传入的接口列表，动态生成代理类字节码（类名格式：`$Proxy0`, `$Proxy1`...）
2. 该代理类实现了所有传入的接口
3. 代理类持有`InvocationHandler`实例
4. 任何接口方法调用都会委托给`InvocationHandler.invoke()`

### 2.3 完整示例代码

```java
import java.lang.reflect.*;

// 步骤1：定义接口
interface UserService {
    void addUser(String name);
    String getUser(Long id);
}

// 步骤2：实现类（目标对象）
class UserServiceImpl implements UserService {
    @Override
    public void addUser(String name) {
        System.out.println("【目标方法】添加用户: " + name);
    }

    @Override
    public String getUser(Long id) {
        System.out.println("【目标方法】查询用户, id=" + id);
        return "User{id=" + id + ", name='张三'}";
    }
}

// 步骤3：InvocationHandler实现
class LogInvocationHandler implements InvocationHandler {
    private final Object target; // 目标对象

    public LogInvocationHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println(">>> [JDK代理] 方法执行前: " + method.getName());
        long start = System.currentTimeMillis();

        // 调用目标对象的实际方法
        Object result = method.invoke(target, args);

        long end = System.currentTimeMillis();
        System.out.println("<<< [JDK代理] 方法执行后: " + method.getName()
                + ", 耗时=" + (end - start) + "ms");
        return result;
    }
}

// 步骤4：创建代理对象
public class JdkProxyDemo {
    public static void main(String[] args) {
        // 创建目标对象
        UserService target = new UserServiceImpl();

        // 创建代理对象
        UserService proxy = (UserService) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),   // 类加载器
                target.getClass().getInterfaces(),     // 接口列表
                new LogInvocationHandler(target)       // 调用处理器
        );

        // 通过代理调用方法
        proxy.addUser("李四");
        String user = proxy.getUser(1L);
        System.out.println("结果: " + user);

        // 查看代理对象的类名
        System.out.println("代理类: " + proxy.getClass().getName());
        // 输出: 代理类: com.sun.proxy.$Proxy0
    }
}

/* 运行结果:
>>> [JDK代理] 方法执行前: addUser
【目标方法】添加用户: 李四
<<< [JDK代理] 方法执行后: addUser, 耗时=0ms
>>> [JDK代理] 方法执行前: getUser
【目标方法】查询用户, id=1
<<< [JDK代理] 方法执行后: getUser, 耗时=0ms
结果: User{id=1, name='张三'}
代理类: com.sun.proxy.$Proxy0
*/
```

**JDK代理内部结构**：

```
┌──────────────────┐
│   调用方          │
│  userServiceProxy │
└────────┬─────────┘
         │ 调用 addUser("李四")
         ▼
┌──────────────────────────────────────┐
│  $Proxy0 (动态生成的代理类)            │
│  implements UserService              │
│                                      │
│  addUser(String name) {              │
│      handler.invoke(this, m3, args); │ ← 所有方法调用都委托给handler
│  }                                   │
│                                      │
│  持有 InvocationHandler handler      │
└────────┬─────────────────────────────┘
         │ invoke(proxy, method, args)
         ▼
┌──────────────────────────────────────┐
│  LogInvocationHandler                │
│  invoke() {                          │
│      前置增强 ...                     │
│      method.invoke(target, args);    │ ← 反射调用目标方法
│      后置增强 ...                     │
│  }                                   │
└────────┬─────────────────────────────┘
         │ method.invoke()
         ▼
┌──────────────────┐
│  UserServiceImpl  │ ← 真正的目标对象
└──────────────────┘
```

### 2.4 JDK代理的限制

1. **必须基于接口**：目标类必须实现至少一个接口
2. **只能代理接口方法**：即使目标类有public方法，若不在接口中声明，代理无法拦截
3. **仅能拦截方法调用**：不能拦截字段访问、构造器等
4. **代理对象与目标对象不是同一类型**：`$Proxy0 instanceof UserServiceImpl` → ❌ false

---

## 三、CGLIB动态代理

CGLIB（Code Generation Library）是一个强大的字节码生成库，Spring将其fork后内置于`spring-core`中。它通过**继承目标类**来创建代理。

### 3.1 Enhancer和MethodInterceptor

```java
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;

/**
 * MethodInterceptor接口（注意：不是JDK的InvocationHandler）
 * 功能类似，但更强大——提供了MethodProxy避免反射开销
 */
public interface MethodInterceptor {
    Object intercept(Object obj,         // 代理对象
                     Method method,      // 被调用的方法
                     Object[] args,      // 方法参数
                     MethodProxy proxy   // CGLIB方法代理（用于快速调用，无反射）
                    ) throws Throwable;
}
```

**CGLIB vs JDK代理的核心优势**：`MethodProxy`允许直接调用父类方法，**无需反射**，性能更高。

### 3.2 CGLIB代理原理

1. CGLIB通过`Enhancer`在运行时动态生成目标类的子类
2. 代理类**继承**目标类，重写所有非final方法
3. 拦截逻辑通过`MethodInterceptor`实现
4. 代理对象与目标对象是**父子类关系**

```
┌───────────────────────────────────────────┐
│  UserService$$EnhancerByCGLIB$$xxxxx      │
│  extends UserServiceImpl                  │ ← 继承目标类！
│                                           │
│  addUser(String name) {                   │ ← 重写父类方法
│      MethodInterceptor.intercept(         │
│          this, method, args, methodProxy  │
│      );                                   │
│  }                                        │
└────────┬──────────────────────────────────┘
         │ intercept()
         ▼
┌──────────────────────────────────────────┐
│  MethodInterceptor实现                    │
│  intercept() {                           │
│      前置增强 ...                         │
│      methodProxy.invokeSuper(obj, args); │ ← 调用父类方法（无反射！
│      后置增强 ...                         │
│  }                                        │
└────────┬──────────────────────────────────┘
         │ invokeSuper()
         ▼
┌──────────────────┐
│  UserServiceImpl │ ← 父类（目标对象）
└──────────────────┘
```

### 3.3 完整示例代码

```java
import org.springframework.cglib.proxy.Enhancer;
import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;

// 目标类：无需实现接口！
class ProductService {
    public void addProduct(String name) {
        System.out.println("【目标方法】添加商品: " + name);
    }

    public String getProduct(Long id) {
        System.out.println("【目标方法】查询商品, id=" + id);
        return "Product{id=" + id + ", name='手机'}";
    }

    // final方法——CGLIB无法代理！
    public final void finalMethod() {
        System.out.println("【final方法】不会被代理");
    }
}

// MethodInterceptor实现
class LogMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args,
                            MethodProxy proxy) throws Throwable {
        System.out.println(">>> [CGLIB代理] 方法执行前: " + method.getName());
        long start = System.currentTimeMillis();

        // proxy.invokeSuper(): 调用父类（目标类）的方法，无反射开销
        Object result = proxy.invokeSuper(obj, args);

        long end = System.currentTimeMillis();
        System.out.println("<<< [CGLIB代理] 方法执行后: " + method.getName()
                + ", 耗时=" + (end - start) + "ms");
        return result;
    }
}

// 创建CGLIB代理
public class CglibProxyDemo {
    public static void main(String[] args) {
        // 创建Enhancer
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(ProductService.class);    // 设置父类（目标类）
        enhancer.setCallback(new LogMethodInterceptor()); // 设置拦截器

        // 创建代理对象
        ProductService proxy = (ProductService) enhancer.create();

        // 通过代理调用方法
        proxy.addProduct("笔记本电脑");
        String product = proxy.getProduct(100L);
        System.out.println("结果: " + product);

        // final方法不会被拦截
        proxy.finalMethod();

        // 查看代理对象的类名和父子关系
        System.out.println("代理类: " + proxy.getClass().getName());
        System.out.println("是ProductService实例: " + (proxy instanceof ProductService)); // true
    }
}

/* 运行结果:
>>> [CGLIB代理] 方法执行前: addProduct
【目标方法】添加商品: 笔记本电脑
<<< [CGLIB代理] 方法执行后: addProduct, 耗时=0ms
>>> [CGLIB代理] 方法执行前: getProduct
【目标方法】查询商品, id=100
<<< [CGLIB代理] 方法执行后: getProduct, 耗时=0ms
结果: Product{id=100, name='手机'}
【final方法】不会被代理          ← final方法没有被拦截
代理类: com.demo.ProductService$$EnhancerByCGLIB$$e4a3b2c1
是ProductService实例: true
*/
```

### 3.4 CGLIB的限制

| 限制 | 说明 | 规避方案 |
|------|------|----------|
| **final类** | 无法继承，CGLIB无法代理 | ❌ 无解，必须去掉final |
| **final方法** | 子类无法重写，不会被拦截 | 去掉final修饰符 |
| **private方法** | 子类无法访问，不会被拦截 | 改为protected或public |
| **构造器拦截** | 代理类会调用父类构造器 | 确保父类有无参构造器；CGLIB可跳过构造器（`setInterceptDuringConstruction(false)`） |
| **equals/hashCode** | 默认不会被代理 | Spring AOP默认排除了这些方法 |
| **static方法** | 属于类而非实例 | ❌ 无法代理 |
| **自身调用** | 同一个类内`this.method()`不会触发代理 | 注入自身代理对象，用代理调用（或使用AspectJ LTW） |

> **重要提示**：自身调用（self-invocation）是Spring AOP最常见的"坑"！
> ```java
> @Service
> public class UserService {
>     @Transactional
>     public void methodA() {
>         this.methodB();  // ❌ methodB的事务不会生效！因为绕过了代理
>     }
>
>     @Transactional
>     public void methodB() { ... }
> }
> ```

---

## 四、JDK代理 vs CGLIB代理

### 4.1 核心区别对比表

| 维度 | JDK动态代理 | CGLIB代理 |
|------|-----------|-----------|
| **实现原理** | 基于接口 + 反射 | 基于继承 + 字节码生成 |
| **要求** | 目标类必须实现接口 | 目标类不能是final，方法不能是final |
| **代理对象类型** | `com.sun.proxy.$Proxy0` | `目标类$$EnhancerByCGLIB$$xxx` |
| **instanceof目标类** | ❌ false | ✅ true |
| **instanceof接口** | ✅ true | ✅ true（间接） |
| **方法调用方式** | 反射（`method.invoke`） | `MethodProxy.invokeSuper`（FastClass机制，无反射） |
| **性能-创建** | ✅ 更快（JVM内置支持） | 较慢（需要生成字节码+类加载） |
| **性能-调用** | 较慢（反射） | ✅ 更快（FastClass索引） |
| **拦截范围** | 仅接口中声明的方法 | 所有非final的public/protected方法 |
| **Spring Boot 3.x默认** | ❌ 需手动配置 | ✅ 默认使用 |
| **JDK版本要求** | JDK 1.3+ | JDK 1.3+ |
| **依赖** | JDK内置，零依赖 | Spring内置CGLIB Fork（spring-core） |

### 4.2 Spring选择策略

```java
// Spring判断代理方式的简化逻辑
if (proxyTargetClass) {
    return CGLIB_PROXY;                        // 强制CGLIB
} else if (targetClass实现接口) {
    return JDK_PROXY;                          // 使用JDK代理
} else {
    return CGLIB_PROXY;                        // 无接口，必须CGLIB
}
```

**Spring Boot 3.x 具体行为**：

| 配置 | 行为 |
|------|------|
| 默认（`spring.aop.proxy-target-class=true`） | **始终CGLIB**，即使实现了接口 |
| `spring.aop.proxy-target-class=false` | 有接口→JDK代理，无接口→CGLIB |
| 单个Bean：`@Scope(proxyMode=ScopedProxyMode.TARGET_CLASS)` | 强制CGLIB |
| 单个Bean：`@Scope(proxyMode=ScopedProxyMode.INTERFACES)` | 强制JDK代理 |

**为什么Spring Boot 3.x默认CGLIB？**
1. **统一行为**：避免"有接口用JDK，无接口用CGLIB"带来的行为差异（如`@Transactional`自调用陷阱）
2. **性能更高**：CGLIB的FastClass机制避免了反射开销
3. **类型兼容**：`instanceof`可以同时匹配目标类和接口，注入更灵活

```java
// 注入时如果目标类实现了接口，JDK代理只能按接口注入
@Autowired
private UserService userService; // ✅ 接口注入，JDK和CGLIB都支持

@Autowired
private UserServiceImpl userService; // ❌ JDK代理会失败（$Proxy0不是UserServiceImpl子类）
                                      // ✅ CGLIB代理可以
```

---

## 五、Spring AOP代理创建过程

### 5.1 AbstractAutoProxyCreator

`AbstractAutoProxyCreator`是Spring AOP的核心处理器，实现了`BeanPostProcessor`接口，在Bean初始化后进行代理包装。

```java
// AbstractAutoProxyCreator核心逻辑（简化版）
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    // Bean初始化后回调：检查是否需要创建代理
    @Override
    public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (!this.earlyProxyReferences.contains(cacheKey)) {
                // 核心：判断是否需要为这个Bean创建代理
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }

    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        // 1. 获取与这个Bean匹配的所有Advisor（切面=切点+通知）
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(
                bean.getClass(), beanName, null);

        if (specificInterceptors != DO_NOT_PROXY) {
            // 2. 创建代理对象
            Object proxy = createProxy(bean.getClass(), beanName,
                    specificInterceptors, new SingletonTargetSource(bean));
            return proxy;
        }
        return bean; // 不需要代理，返回原Bean
    }
}
```

### 5.2 代理创建时序

```
Spring容器启动
    │
    ▼
1. Bean实例化（Instantiation）
    │
    ▼
2. 属性填充（Populate Properties）
    │
    ▼
3. 初始化（Initialization）
    │  - @PostConstruct
    │  - InitializingBean.afterPropertiesSet()
    │  - init-method
    │
    ▼
4. BeanPostProcessor.postProcessAfterInitialization()  ← AbstractAutoProxyCreator在此拦截
    │
    ├── 4.1 检查所有Advisor的切点是否匹配当前Bean
    │
    ├── 4.2 如果有匹配 → 创建代理对象
    │       ├── 收集匹配的Advisor（切面）
    │       ├── 选择代理方式（JDK / CGLIB）
    │       ├── ProxyFactory.getProxy() 创建代理
    │       └── 返回代理对象（替代原Bean放入容器）
    │
    └── 4.3 如果无匹配 → 返回原Bean
```

### 5.3 代理对象调用流程

#### JDK代理：JdkDynamicAopProxy

```java
// JdkDynamicAopProxy 简化实现
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler {

    private final AdvisedSupport advised; // 包含Advisor链和目标对象

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // 1. 获取该方法的拦截器链（通知链）
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
                method, targetClass);

        if (chain.isEmpty()) {
            // 2a. 没有拦截器，直接调用目标方法
            return AopUtils.invokeJoinpointUsingReflection(target, method, args);
        } else {
            // 2b. 构建责任链：ReflectiveMethodInvocation
            MethodInvocation invocation = new ReflectiveMethodInvocation(
                    proxy, target, method, args, targetClass, chain);
            return invocation.proceed(); // 递归执行拦截器链 + 目标方法
        }
    }
}
```

#### CGLIB代理：CglibAopProxy / DynamicAdvisedInterceptor

```java
// CglibAopProxy内部类 DynamicAdvisedInterceptor
private static class DynamicAdvisedInterceptor implements MethodInterceptor {

    private final AdvisedSupport advised;

    @Override
    public Object intercept(Object proxy, Method method, Object[] args,
                            MethodProxy methodProxy) throws Throwable {
        // 获取拦截器链（与JDK代理相同逻辑）
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
                method, targetClass);

        if (chain.isEmpty()) {
            // 无拦截器：直接用CGLIB的快速调用
            return methodProxy.invoke(target, args);
        } else {
            // 有拦截器：构建CglibMethodInvocation责任链
            CglibMethodInvocation invocation = new CglibMethodInvocation(
                    proxy, target, method, args, targetClass, chain, methodProxy);
            return invocation.proceed();
        }
    }
}
```

**拦截器链执行原理（责任链模式）**：

```
    请求进入
        │
        ▼
┌─────────────────┐   proceed()   ┌─────────────────┐   proceed()   ┌──────┐
│ @Around通知      │ ──────────► │ @Before通知       │ ──────────► │ 目标  │
│ 前置逻辑         │             │                  │             │ 方法  │
│ 调proceed()     │ ◄────────── │                  │ ◄────────── │      │
│ 后置逻辑         │             │                  │             └──────┘
└─────────────────┘             └─────────────────┘
                                       │
                                       ▼
                                ┌─────────────────┐
                                │ @After通知       │ ← 类似finally，保证执行
                                └─────────────────┘
```

---

## 六、@AspectJ注解开发

### 6.1 @Aspect + @Component

Spring Boot 3.x中使用AOP的最小配置：

```java
// 启动类或配置类上无需任何注解，Spring Boot自动配置已启用AOP！
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// 切面类
@Aspect     // 声明这是切面
@Component  // 让Spring管理
public class LoggingAspect {
    // 切点和通知定义...
}
```

> 在Spring Boot 3.x中，AOP自动配置已启用（`spring.aop.auto=true`），不需要手动添加`@EnableAspectJAutoProxy`。传统Spring项目中仍需要加上该注解。

### 6.2 切点表达式详解

```java
// 1. execution表达式 —— 最常用
/**
 * 语法: execution(修饰符 返回值 包.类.方法(参数) throws-异常)
 * 通配符:
 *    *    匹配任意一个单词
 *    ..   匹配任意多个单词（包路径或参数）
 *    +    匹配子类
 */

// 匹配com.example.service包下所有类的所有public方法
@Pointcut("execution(public * com.example.service.*.*(..))")

// 匹配service包及其子包下所有类的所有方法
@Pointcut("execution(* com.example.service..*.*(..))")

// 匹配所有以add开头的方法
@Pointcut("execution(* add*(..))")

// 匹配参数列表第一个是String的方法
@Pointcut("execution(* *(String, ..))")

// 2. @annotation —— 匹配带有指定注解的方法
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")

// 3. within —— 匹配某个包/类下的所有方法
@Pointcut("within(com.example.service.*)")

// 4. bean —— 按bean名称匹配（Spring特有）
@Pointcut("bean(*Service)")     // 匹配名称以Service结尾的bean

// 5. 组合切点表达式
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() {}

@Pointcut("@annotation(com.example.Loggable)")
public void loggableMethods() {}

// 组合：service层中带@Loggable注解的方法
@Pointcut("serviceLayer() && loggableMethods()")
public void loggableServiceMethods() {}
```

### 6.3 五种通知类型 —— 完整示例

```java
@Aspect
@Component
public class CompleteAopAspect {

    // ==================== 切点定义 ====================

    @Pointcut("execution(* com.example.service.UserService.*(..))")
    public void userServiceMethods() {}

    // ==================== 1. @Before 前置通知 ====================
    // 在目标方法执行前运行，无法阻止目标方法执行
    @Before("userServiceMethods()")
    public void before(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArguments();
        System.out.println("[@Before] 准备执行: " + methodName
                + ", 参数: " + Arrays.toString(args));
    }

    // ==================== 2. @AfterReturning 后置返回通知 ====================
    // 目标方法正常返回后执行，可以获取返回值
    // returning属性指定接收返回值的参数名
    @AfterReturning(
        pointcut = "userServiceMethods()",
        returning = "result"
    )
    public void afterReturning(JoinPoint joinPoint, Object result) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("[@AfterReturning] " + methodName
                + " 正常返回, 返回值: " + result);
    }

    // ==================== 3. @AfterThrowing 后置异常通知 ====================
    // 目标方法抛出异常后执行
    // throwing属性指定接收异常的参数名
    @AfterThrowing(
        pointcut = "userServiceMethods()",
        throwing = "ex"
    )
    public void afterThrowing(JoinPoint joinPoint, Exception ex) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("[@AfterThrowing] " + methodName
                + " 抛出异常: " + ex.getMessage());
    }

    // ==================== 4. @After 后置最终通知 ====================
    // 无论目标方法正常返回还是抛出异常，都会执行（类似finally）
    @After("userServiceMethods()")
    public void after(JoinPoint joinPoint) {
        String methodName = joinPoint.getSignature().getName();
        System.out.println("[@After] " + methodName + " 执行完毕（释放资源等）");
    }

    // ==================== 5. @Around 环绕通知 ====================
    // 最强大的通知类型，可以完全控制目标方法的执行
    @Around("userServiceMethods()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArguments();

        System.out.println("[@Around-前] " + methodName + " 开始, 参数: "
                + Arrays.toString(args));

        long start = System.currentTimeMillis();
        Object result = null;
        try {
            // 必须调用proceed()才能执行目标方法，否则目标方法不会执行
            result = joinPoint.proceed();

            System.out.println("[@Around-返回] " + methodName
                    + " 正常返回, 结果: " + result);
            return result;

        } catch (Exception e) {
            System.out.println("[@Around-异常] " + methodName
                    + " 抛异常: " + e.getMessage());
            // 可以决定是否重新抛出或吞掉异常
            throw e;

        } finally {
            long end = System.currentTimeMillis();
            System.out.println("[@Around-最终] " + methodName
                    + " 总耗时: " + (end - start) + "ms");
        }
    }
}
```

**五种通知执行顺序验证**：

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id) {
        return userService.getUser(id); // 触发AOP
    }
}

/* 正常执行输出:
[@Around-前] getUser 开始, 参数: [1]
[@Before] 准备执行: getUser, 参数: [1]
【目标方法】查询用户 id=1
[@Around-返回] getUser 正常返回, 结果: User{id=1, name='张三'}
[@AfterReturning] getUser 正常返回, 返回值: User{id=1, name='张三'}
[@After] getUser 执行完毕（释放资源等）
[@Around-最终] getUser 总耗时: 5ms
*/
```

### 6.4 @Around环绕通知进阶 —— 性能监控

```java
@Aspect
@Component
@Slf4j
public class PerformanceMonitorAspect {

    // 监控所有Controller方法
    @Pointcut("execution(* com.example.controller..*(..))")
    public void controllerLayer() {}

    // 监控所有Service方法
    @Pointcut("execution(* com.example.service..*(..))")
    public void serviceLayer() {}

    @Around("controllerLayer() || serviceLayer()")
    public Object monitorPerformance(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.nanoTime();

        try {
            return joinPoint.proceed(); // 执行目标方法

        } finally {
            long end = System.nanoTime();
            long elapsedMs = (end - start) / 1_000_000;

            if (elapsedMs > 1000) { // 超过1秒记录警告
                String className = joinPoint.getTarget().getClass().getSimpleName();
                String methodName = joinPoint.getSignature().getName();
                log.warn("慢方法告警! {}.{} 执行耗时 {}ms",
                        className, methodName, elapsedMs);
            }
        }
    }
}
```

**@Around 使用注意事项**：

```java
@Around("somePointcut()")
public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
    // ✅ 正确：调用proceed()并返回其结果
    Object result = joinPoint.proceed();
    // 可以对result进行修改
    return result;

    // ✅ 正确：修改参数后再调用
    Object[] args = joinPoint.getArgs();
    args[0] = "修改后的参数";
    return joinPoint.proceed(args);

    // ✅ 正确：不调proceed()，完全拦截
    // 直接返回缓存结果，目标方法不执行

    // ❌ 错误：忘记调用proceed()
    // 目标方法不会执行，静默失败

    // ❌ 错误：忘记return
    // joinPoint.proceed(); ← 没有return，调用方拿不到结果
}
```

---

## 七、面试常见问题

### Q1：JDK动态代理和CGLIB代理的根本区别是什么？

**JDK代理基于接口**（`Proxy.newProxyInstance`），通过反射调用目标方法，代理对象必须实现目标类的接口。**CGLIB代理基于继承**（`Enhancer`），通过生成目标类的子类来拦截方法调用，使用FastClass索引避免反射。

### Q2：Spring Boot 3.x中为什么默认使用CGLIB代理？

1. **统一行为**：无论Bean是否实现接口，代理行为一致，减少因代理类型不同导致的"不可预知"bug
2. **性能更好**：CGLIB的FastClass机制比JDK反射性能高
3. **注入兼容性**：`instanceof`目标类为true，支持按具体类型注入
4. **自调用问题**：无论JDK还是CGLIB同样存在自调用不触发AOP的问题，但CGLIB的语义更直观

### Q3：@Transactional注解本质上是AOP，为什么同一个类内的方法互调不生效？

因为Spring AOP基于代理，**从外部调用时**先进代理对象→再调用目标对象。同一个类内`this.methodB()`直接调用了目标对象的`this`引用，绕过了代理对象，因此AOP拦截不生效。

**解决方案**：
1. 注入自身代理：`@Autowired private XxxService self; self.methodB();`
2. 拆分到不同的Bean中
3. 使用`AopContext.currentProxy()`
4. 使用AspectJ编译期织入（LTW）

### Q4：AOP的`@After`和`@AfterReturning`有什么区别？

- `@After`：类似finally，无论正常返回还是异常都执行，**无法获取返回值**
- `@AfterReturning`：仅在方法正常返回时执行，**可以获取返回值**
- `@AfterThrowing`：仅在方法抛出异常时执行，**可以获取异常信息**

### Q5：Spring AOP和AspectJ的区别是什么？什么时候该用AspectJ？

| 场景 | 选择 |
|------|------|
| 日志/事务/权限等横切关注点 | Spring AOP ✅ |
| 需要拦截private方法 | AspectJ ✅ |
| 需要拦截构造函数/字段访问 | AspectJ ✅ |
| 需要解决自调用问题 | AspectJ ✅ |
| 性能敏感（极致要求） | AspectJ ✅ |
| 简单快速开发 | Spring AOP ✅ |

### Q6：如何让多个切面按指定顺序执行？

```java
@Aspect
@Component
@Order(1)  // 数值越小，优先级越高（最先执行前置通知，最后执行后置通知）
public class FirstAspect { }

@Aspect
@Component
@Order(2)
public class SecondAspect { }

// 也可以实现Ordered接口
@Aspect
@Component
public class ThirdAspect implements Ordered {
    @Override
    public int getOrder() {
        return 3;
    }
}
```

### Q7：CGLIB代理无法代理哪些情况？

1. `final`类：无法继承 → ❌
2. `final`方法：无法重写 → ❌
3. `private`方法：子类不可见 → ❌
4. `static`方法 → ❌
5. 目标类没有无参构造器且不跳过构造器拦截 → ❌
6. `equals()`/`hashCode()`（Spring默认排除，但可配置）

---

## 八、LeetCode题目解析

### 8.1 [74. 搜索二维矩阵（Medium）](https://leetcode.cn/problems/search-a-2d-matrix/)

**题目描述**：编写一个高效的算法来判断 m x n 矩阵中，是否存在一个目标值。该矩阵具有如下特性：
- 每行中的整数从左到右按升序排列
- 每行的第一个整数大于前一行的最后一个整数

**思路分析**：矩阵整体升序，可以视为一维有序数组，使用**二分查找**。将一维索引 `mid` 映射到二维坐标：`row = mid / n`, `col = mid % n`。

```java
class Solution {
    public boolean searchMatrix(int[][] matrix, int target) {
        int m = matrix.length;
        int n = matrix[0].length;
        int left = 0, right = m * n - 1;

        while (left <= right) {
            int mid = left + (right - left) / 2;
            // 将一维索引映射到二维坐标
            int midValue = matrix[mid / n][mid % n];

            if (midValue == target) {
                return true;
            } else if (midValue < target) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
        return false;
    }
}
```

**复杂度分析**：
- 时间复杂度：O(log(m*n)) —— 标准二分查找
- 空间复杂度：O(1)

**另一种思路**：从右上角开始搜索，类似BST的搜索方式。值大则左移，值小则下移。

```java
class Solution2 {
    public boolean searchMatrix(int[][] matrix, int target) {
        int m = matrix.length, n = matrix[0].length;
        int row = 0, col = n - 1; // 从右上角开始

        while (row < m && col >= 0) {
            int cur = matrix[row][col];
            if (cur == target) {
                return true;
            } else if (cur > target) {
                col--; // 当前值太大，向左移动
            } else {
                row++; // 当前值太小，向下移动
            }
        }
        return false;
    }
}
```

### 8.2 [162. 寻找峰值（Medium）](https://leetcode.cn/problems/find-peak-element/)

**题目描述**：峰值元素是指其值严格大于左右相邻值的元素。给你一个整数数组 nums，找到峰值元素并返回其索引。数组可能包含多个峰值，返回任意一个峰值所在位置即可。假设 `nums[-1] = nums[n] = -∞`。

**思路分析**：题目要求 O(log n) 复杂度，自然想到**二分查找**。关键洞察：沿着上坡方向走，一定能找到峰值。

```java
class Solution {
    public int findPeakElement(int[] nums) {
        int left = 0, right = nums.length - 1;

        while (left < right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] > nums[mid + 1]) {
                // mid处于下坡，峰值在左边（包括mid）
                right = mid;
            } else {
                // mid处于上坡，峰值在右边（不包括mid）
                left = mid + 1;
            }
        }
        // left == right时就是峰值
        return left;
    }
}
```

**为什么一定能找到峰值（二分正确性证明）**：

```
情况1: nums[mid] > nums[mid+1]
  → mid处在下坡 → 左边必然存在峰值（因为nums[-1]=-∞，左端至少有一个上升段）

情况2: nums[mid] < nums[mid+1]
  → mid处在上升坡 → 右边必然存在峰值（因为nums[n]=-∞，右端至少有一个下降段）
```

**复杂度分析**：
- 时间复杂度：O(log n)
- 空间复杂度：O(1)

---

## 九、今日学习要点总结

### 核心要点归纳

| 序号 | 要点 | 说明 |
|------|------|------|
| 1 | **AOP = 切面（Aspect） = 切点 + 通知** | 切点决定"在哪里"，通知决定"干什么" |
| 2 | **JDK代理基于接口+反射** | 目标类必须实现接口，`$Proxy0`不是目标类子类 |
| 3 | **CGLIB代理基于继承+字节码** | 生成目标类子类，FastClass无反射调用更快 |
| 4 | **Spring Boot 3.x默认CGLIB** | `proxy-target-class=true`，不再回退JDK代理 |
| 5 | **五种通知类型** | @Before, @AfterReturning, @AfterThrowing, @After, @Around |
| 6 | **自调用陷阱** | 同一类内`this.method()`绕过代理，AOP无效 |
| 7 | **@Around最强大** | `ProceedingJoinPoint.proceed()`必须调用，可修改参数和返回值 |
| 8 | **Spring AOP仅支持方法级连接点** | 要拦截构造器/字段需要用AspectJ |
| 9 | **代理在BeanPostProcessor阶段创建** | `AbstractAutoProxyCreator.postProcessAfterInitialization()` |

### 验收清单

- [ ] 能画出AOP核心概念关系图（切面、切点、通知、连接点、织入）
- [ ] 能手写JDK动态代理（`Proxy.newProxyInstance` + `InvocationHandler`）
- [ ] 能手写CGLIB动态代理（`Enhancer` + `MethodInterceptor`）
- [ ] 能说出JDK代理和CGLIB代理的5个以上区别
- [ ] 能说明Spring Boot 3.x为何默认CGLIB代理
- [ ] 能解释"同一类内方法互调AOP不生效"的原因和解决方案
- [ ] 能使用@Aspect + 五种通知类型写完整示例
- [ ] 能写出`execution`切点表达式（含通配符）
- [ ] 能说明@Around与其他通知的区别和注意事项
- [ ] 理解CGLIB无法代理final类/final方法的原因
- [ ] 完成LeetCode 74（搜索二维矩阵）— 二分查找
- [ ] 完成LeetCode 162（寻找峰值）— 二分查找

---

## 十、练习建议

### 基础练习
1. **手写JDK动态代理**：创建一个接口和一个实现类，用JDK代理实现日志记录
2. **手写CGLIB代理**：对同一个功能用CGLIB重新实现，对比两者差异
3. **搭建Spring AOP Demo**：创建Spring Boot项目，写一个@Aspect切面，练习五种通知类型
4. **验证自调用问题**：故意在同一个Service内互调有@Transactional的方法，观察事务是否生效

### 进阶练习
5. **实现多切面顺序控制**：用`@Order`定义3个切面，观察执行顺序
6. **实现自定义注解+AOP**：自定义`@Loggable`注解，AOP拦截被该注解标记的方法
7. **对比JDK/CGLIB性能**：写Benchmark测试两种代理的创建耗时和调用耗时
8. **调试代理生成过程**：在`AbstractAutoProxyCreator.wrapIfNecessary()`打断点，观察Spring如何判断和创建代理

### 实战项目练习
9. **接口限流AOP**：利用@Around实现基于令牌桶的接口限流
10. **操作日志AOP**：自动记录用户的增删改操作日志（操作人、时间、参数、结果）
11. **Redis缓存AOP**：自定义`@Cacheable`注解，@Around中先查Redis，命中则直接返回（不执行目标方法）

---

## 十一、扩展阅读

- [Spring Framework AOP官方文档](https://docs.spring.io/spring-framework/reference/core/aop.html)
- [Spring Boot 3.x AOP文档](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#features.aop)
- [Spring AOP vs AspectJ 官方建议](https://docs.spring.io/spring-framework/reference/core/aop/ataspectj/limitations.html)
- [CGLIB GitHub仓库](https://github.com/cglib/cglib)
- [AspectJ开发者指南](https://eclipse.dev/aspectj/doc/released/devguide/)
- 书籍推荐：《Spring实战》第6版（Spring Boot 3）第4章
- LeetCode相关：二分查找专题（33, 34, 35, 74, 153, 162）

---

**学习时长：** 3小时理论 + 1小时LeetCode

**完成日期：** _______
