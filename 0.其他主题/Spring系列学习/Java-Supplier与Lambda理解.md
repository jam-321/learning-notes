# Supplier 与 Lambda：把“获取过程”封装成对象

> 学习目标：搞懂三级缓存里 `singletonFactories` 为什么存 Supplier（而不是直接存对象），以及“把一段代码当参数传”在 Java 里到底是怎么实现的。
> 状态：2026-09-08 整理。源码实锤：`learning-lab` 的 MiniApplicationContext + javap 字节码。
> 关联阅读：Spring-IoC学习笔记.md（三级缓存）、Spring-AOP学习笔记.md（getEarlyBeanReference）。

---

## 一、Supplier 是什么

`Supplier<T>` 是 Java 8 的 `java.util.function` 函数式接口，只有一个抽象方法：`T get()`（无参、返回类型固定为 T）。

- 直接放对象 = 东西现在就做好了；
- 放 Supplier = 只放“怎么做的说明书”，**调 `get()` 才执行**。

用户总结（原话）：

> “它是将获取对象的过程封装成一个对象，和工厂的区别是，我要什么，工厂给什么对象；而 Supplier 是获取的类型固定的，我要，你再给。”

## 二、Supplier vs 工厂模式（本质区别）

| 维度 | 工厂模式 | Supplier |
|---|---|---|
| 是否带参数 | 通常带参数（type/config） | `get()` 无参 |
| 返回类型 | 多态：按参数返回多种产品 | 固定为 T |
| 何时创建/执行 | 调用工厂方法即创建 | **调 get() 才执行（延迟）** |
| 复杂程度 | 重：管理一组产品的创建逻辑 | 轻：一个接口一个方法 |
| 本质角色 | “产品创建的管理者”（选择+创建） | “获取过程的封装”（能力/取货券） |

一句话：**工厂解决“你要什么我给什么”，Supplier 解决“代码先打包，要的时候再跑”。** 所以 Supplier 是延迟执行（惰性求值）的天然载体，适合回调、按需生产、把决定权交给调用方。

## 三、为什么三级缓存要放 Supplier（应用点）

看 MiniApplicationContext#createBean 的登记代码：

```java
singletonFactories.put(name, () -> {
    Object earlyReference = rawBean;
    for (BeanPostProcessor processor : beanPostProcessors) {
        earlyReference = processor.getEarlyBeanReference(earlyReference, name);
    }
    return earlyReference;
});
```

- `put` 只执行一次，把“生产放行物”的能力登记进三级缓存；
- lambda **不执行**：没人来要，工厂永远不会被调（懒的价值）；
- 只有 `getBean` 撞上循环依赖、三级缓存命中时才 `factory.get()`——那一刻才跑 getEarlyBeanReference 链，决定发裸对象还是代理。

对比：如果 put 时直接放对象 / 立即调用 getEarlyBeanReference = “预造”，每个 bean 都会被提前处理，三级缓存的“按需、延迟决策”语义就没了。

## 四、lambda“当参数传递”的底层（源码 → 编译 → JVM 四层）

**源码层**：put 的第二参数类型是 `Supplier<Object>`（函数式接口）。lambda = 匿名实现 `get()` 的语法糖，等价于 `new Supplier<Object>() { public Object get() {...} }`。

**编译层（javac）**：
1. 把方法体抽成合成方法 `lambda$createBean$0(Object rawBean, String name)`（用了 this.beanPostProcessors，所以是实例方法）；
2. 调用点生成一条 `invokedynamic` 指令（不是 new 匿名类）；
3. 被捕获的局部变量（rawBean、name）成为合成方法的额外参数。

字节码实锤（javap -c 片段）：

```java
60: aload_0                     // this（被捕获）
61: aload 4                     // rawBean（被捕获）
63: aload_1                     // name（被捕获）
64: invokedynamic #100          // get:(MiniApplicationContext,Object,String)Supplier
69: invokeinterface Map.put
```

**JVM 层**：`invokedynamic` 首次执行 → BootstrapMethods 里的 `LambdaMetafactory.metafactory` 被调用，它持有实现方法句柄 `lambda$createBean$0`，在运行期生成一个实现 Supplier 的类；捕获变量成为该对象实例字段（闭包）；`get()` 转发到合成方法。可用 `-Djdk.internal.lambda.dumpProxyClasses` 导出生成的类。

**执行语义层**：map 里存的是一段“代码引用 + 捕获环境”的对象；`get()` 被调才执行 body。捕获的对象引用（rawBean）是地址值——工厂执行时看到的 rawBean 与容器正在填字段的是同一个堆对象。

## 五、一句话记忆

> **Supplier = 把“获取过程”打包成对象、要了才执行的取货券；lambda 是它的语法糖；JVM 用 invokedynamic + LambdaMetafactory 把它变成携带捕获变量的接口实现对象——传的不是“方法”，是“装着代码和环境的对象”。**
