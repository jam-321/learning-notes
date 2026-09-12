# Spring AOP 学习笔记（v2/v3 落地后整理）

> 学习目标：把 AOP 挂回 bean 生命周期里理解——织入期（创建 bean）与运行期（调用方法）两阶段各自做了什么，核心类/方法是什么。
> 状态：2026-09-08 整理。基于 `learning-lab` 手写 mini-IoC（v2 BeanPostProcessor + v3 三级工厂 + AOP 插件）与 Spring 5.3.31 源码对照。

---

## 一、一句话心智模型

AOP 不是容器里的特殊机制，而是 **Spring 官方实现的一个插件**：它实现 `BeanPostProcessor`（教学版 = `AopPostProcessor`），在 bean 创建完成的 after 钩子里决定“这个 bean 要不要换成代理”。容器不认识 AOP，只认识 BeanPostProcessor。

## 二、Bean 创建生命周期全时序（教学版 MiniApplicationContext#createBean）

```text
1 实例化 instantiate                 new 裸对象（字段全 null）
2 登记三级工厂 singletonFactories     循环依赖“期房凭证”（被要才生产）
3 populateBean 属性填充               依赖注入：字段要谁 → 递归 getBean(谁)
4 before 钩子                         容器级插件第一次插手（教学版暂无插件用）
5 ★ init 回调 afterPropertiesSet      bean 自己声明的初始化（教学版补上，对应 Spring invokeInitMethods）
6 after 钩子                         容器级插件第二次插手 ← AOP 代理在这里诞生
7 singletonObjects.put                代理作为成品入表；裸对象退场
```

- init 回调跑在**裸对象**上：此刻还没代理（demo 日志：`afterPropertiesSet … 还是裸对象（AOP 代理未生成）`）。
- after 钩子之后进成品表的可能是**另一个对象（代理）**，这就是 AOP 能“换掉”bean 的机制来源。

## 三、BeanPostProcessor vs InitializingBean：本质区别

| 维度 | BeanPostProcessor（before/after） | InitializingBean（afterPropertiesSet） |
|---|---|---|
| 谁实现 | 扩展方，在 bean 之外 | bean 自己，写在自己类里 |
| 作用范围 | **全局**：注册一个，作用于所有受管 bean | **只对自己生效** |
| 返回值 | 可以返回替换对象（代理/包装） | void，只能自己做完初始化 |
| 时机 | before/after 包络 init 阶段 | 就是 init 本身 |

用户总结（原话）：**“InitializingBean 只对自己生效，BeanPostProcessor 实现的方法，是对所有 spring 管理的类生效。”**

为什么 init 不并入 BeanPostProcessor：

- 角色错位：业务 bean 若实现 BPP 会变成“全局插件”，会被提前实例化并作用于所有 bean——只想初始化自己，却变成加工全世界。
- 模式不同：InitializingBean 是 bean 自身生命周期回调（模板方法），BPP 是容器级插件（插件/策略）；before/after 的命名本身就建立在“中间有个 init 阶段”之上。
- 真实 Spring 里它们是配对存在的：InitializingBean / DisposableBean 管 bean 自己的出生与死亡，BeanPostProcessor 管容器在出生与死亡点插一脚。

放哪的判断标准：给自己做出厂准备（预热缓存、初始化连接池、校验配置）→ InitializingBean；给所有/某类 bean 做横切加工（注入容器引用、解析 @PostConstruct、AOP 代理）→ BeanPostProcessor。想换掉/包装对象，只能靠 BPP 的返回值。

真实 Spring 用 before 钩子的例子：ApplicationContextAwareProcessor（注入容器引用/Aware）、CommonAnnotationBeanPostProcessor（触发 @PostConstruct）。

## 四、AOP 织入期（创建期）：核心类与核心方法

核心类有且仅有一个（教学版织入侧）：**AopPostProcessor implements BeanPostProcessor**。

核心方法：**wrapIfNecessary**（判断 + 执行一体）。它有两个入口：

- 常规入口：`postProcessAfterInitialization`（after 钩子）→ wrapIfNecessary——每个 bean 创建完成都走；
- 特殊入口：`getEarlyBeanReference`（三级缓存工厂回调，循环依赖提前暴露时）→ 也调 wrapIfNecessary——保证提前发出去的是代理而不是裸对象。

```java
wrapIfNecessary(裸 bean):
  若是 @Aspect 类                → 原样返回（切面不代理）
  若没有任何 Advisor 命中该 bean  → 原样返回（按需代理）
  否则 → Proxy.newProxyInstance(...)   // 命中，生成 JDK 代理
```

一个 bean 变成代理的**三个条件**（教学版）：

1. 不是 `@Aspect` 切面类（否则切面套切面）；
2. 至少一条 Advisor（切点+通知，由 getAdvisors 懒构建扫描 @Aspect bean 而来）命中该 bean 的类——任一方法匹配即可；
3. 实现了接口（JDK 动态代理前提）。没接口教学版抛异常；真实 Spring 会用 CGLIB 生成子类代理。

对应真实 Spring：`AbstractAutoProxyCreator#wrapIfNecessary`（判断）+ `getAdvicesAndAdvisorsForBean`（找候选通知）+ `createProxy`；JDK 代理有接口、CGLIB 无接口。

## 五、AOP 运行期（调用期）：代理是类级，拦截是方法级

- 创建期只决定“这个 bean 是不是代理”（类级，一个 bean 一个代理）；
- 运行期每次调用，代理的 InvocationHandler 走 `invokeWithChain`：先把接口 Method 具体化到实现类（对应 Spring `getMostSpecificMethod`，否则接口上找不到实现类打的注解），再按**当前方法**匹配 Advisor、现拼拦截器链；
- `MethodInvocation.proceed()` 递归推进：链上通知按序执行，链尾反射调用业务，返回值沿调用栈逐层出栈（洋葱模型）。

典型例证：`orderServiceImpl` 因 `createOrder` 命中 `@Log` 整个 bean 被代理；但 `queryOrder` 没打 `@Log`，调用时链为空、直通目标——**代理是类级，拦截是方法级**。

## 六、动态代理知识（机制底座，独立于 AOP 范畴）

### 定位：AOP 是“插件/决策层”，动态代理是“机制/实现层”

- AOP = Spring 在 BPP 阶段判断“要不要代理、何时生成”，教学版就是 `AopPostProcessor`（after 钩子 / 提前暴露时调 `wrapIfNecessary`）；
- JDK 动态代理 = Java 自带机制，解决“怎么造一个代理类、代理如何把调用转给 handler”；它本身与 AOP 无关，任何场景都能单独用；
- 两者是“决策”与“执行”的分层：AOP 决定并调用 `Proxy.newProxyInstance`，动态代理负责把代理对象造出来。

### 核心 API：Proxy.newProxyInstance 是静态工厂

```java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
```

内部三步：① `getProxyClass0` 查缓存，未命中则 `ProxyClassFactory → ProxyGenerator.generateProxyClass` 现场生成字节码 → `defineClass0` 加载成 Class；② 找生成类的唯一构造器（参数就是 `InvocationHandler.class`）；③ `cons.newInstance(h)` 构造代理实例。

### 生成的代理类长什么样（javap 实锤：jdk/proxy2/$Proxy8，JDK 17 dump）

```java
public final class jdk.proxy2.$Proxy8 extends java.lang.reflect.Proxy
        implements OrderService {
    private static final Method m0;   // 每个接口方法 + equals/hashCode/toString 各一个
    ...
    public $Proxy8(InvocationHandler h) { super(h); }   // handler 存父类字段 Proxy.h

    @Override
    public String createOrder(String user, int amount) {  // 接口方法被重写成“转发壳”
        return (String) super.h.invoke(this, mCreateOrder,
                new Object[]{ user, amount });            // 业务逻辑不在代理里
    }
}
```

要点：代理类 **implements 接口并重写每个接口方法**（同名方法来自 ProxyGenerator 按接口现场生成），每个方法体都只是“取 `Proxy.h` → `h.invoke(proxy, method, args)`”；`InvocationHandler` 是总机，真正逻辑在 handler 里。

### handler 绑定与调用时机

- **绑定发生在构造**：`newProxyInstance` 把 h 传给生成类唯一构造器，h 存进父类字段 `Proxy.h`；
- **每次调用才进 `h.invoke`**，`method`/`args` 每次不同；拦不拦、拼什么链由 handler 决定（mini = `invokeWithChain` 方法级匹配现拼链）。

### 为什么叫“动态”：相对于编译期，不是“调用时才创建”

- 静态代理：你自己手写一个代理类，javac 编译期就产出 `.class`；
- JDK 动态代理：代理类在**程序运行后**才由 `ProxyGenerator` 现场生成（javac 从没编译过它），生成时机通常是容器启动（`wrapIfNecessary` → `newProxyInstance`）；
- **“生成后类型固定”与“动态”不矛盾**：“动态”指类型是运行时才被造出来，不是指方法调用那一刻才创建；每次调用真正“变”的是 handler 收到的 method/args 决定的拦截行为，而不是代理类本身。

### AOP 与动态代理的分工时间轴

```text
① 编译期：        只有 UserService / UserServiceImpl / AopPostProcessor，不存在 $ProxyN
② 运行期·启动：   wrapIfNecessary → Proxy.newProxyInstance → 生成 $ProxyN + 绑定 handler
                  → 代理作为成品入 singletonObjects（AOP 织入的落点）
③ 运行期·每次调用：$ProxyN 转发壳 → h.invoke → invokeWithChain → 链尾反射真身
```

### 边界与坑

- 调用入口在代理，业务代码在真身执行；
- bean 内部用 `this` 自调用会**绕过代理**，切面不生效（self-invocation）；
- 复现 dump：`java -Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true ...`（JDK 8 用 `-Dsun.misc.ProxyGenerator.saveGeneratedFiles=true`），再用 `javap -p -c` 看生成类的方法体。

## 七、代码阅读主线（learning-lab，防止再看迷路）

1. `MiniApplicationContext.java` 类头注释 → 构造方法 → getBean → createBean（155/202 行一带）；
2. `BeanPostProcessor.java`（3 个 default 钩子）；
3. `AopPostProcessor.java`：43/52/61/77 行（getEarlyBeanReference → postProcessAfterInitialization → wrapIfNecessary → Proxy.newProxyInstance）；
4. 运行期再看 `invokeWithChain`（102 行）与 `MethodInvocation.proceed()`。

## 八、复现验证

```bash
cd apps/learning-lab
java -Dfile.encoding=UTF-8 -cp target/classes com.guojiang.miniioc.demo.aop.MiniAopDemo
```

关键日志顺序：`实例化 → afterPropertiesSet（裸对象）→ 注册表构建 → wrapIfNecessary 命中 → 完成入成品表（代理）`；验证段打印 `$Proxy8`、两次 getBean 同一代理、审计→日志→业务洋葱链。
