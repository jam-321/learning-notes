# Spring IoC 学习笔记（进行中）

> 学习目标：理解 IoC 容器的结构、启动链路、依赖注入时序、实例化顺序与单例边界。对应学习路线主题 1（IoC 容器与 Bean 生命周期）前半段。
> 状态：2026-09-03 记录已学重点。循环依赖（三级缓存）、Bean 生命周期扩展点、非单例作用域未完。

---

## 一、IoC 解决什么问题

- 没有 IoC：类自己 `new` 依赖，依赖的依赖也要自己管，创建/装配逻辑散落各处。
- 有了 IoC：对象的创建、装配、生命周期统一交给容器管理；类只声明「我要什么」，不关心「从哪来」。
- 边界：只有注册进容器的 bean 归容器管。自己 `new` 出来的对象是「容器外的人」——注入不了依赖，切面也包不上。

## 二、容器结构：两张 Map（配方与产品分离）

| Map | key | value | 角色 |
|---|---|---|---|
| `beanDefinitionMap` | bean 名称 | `BeanDefinition`（类名/scope/lazy/构造器等） | 配方 |
| `singletonObjects` | bean 名称 | 单例实例 | 产品 |

- `BeanFactory` 是最基础容器接口；常用 `ApplicationContext` 是子接口，加了事件、资源加载、国际化等壳能力。
- 拆两步的意义：**先注册齐配方，容器才有全图**——按名/按类型解析依赖、懒加载、BeanPostProcessor 加工都建立在配方层。
- 注册 ≠ 实例化：扫描注册的是配方，`new` 出来的才是产品。

## 三、启动链路：「自动」是谁做的

```text
@SpringBootApplication（含 @ComponentScan，默认扫主类所在包）
  -> ClassPathBeanDefinitionScanner 读 classpath 字节码
  -> 发现 @Component（及派生 @Service/@Repository/@Controller/@Configuration）
  -> 注册 BeanDefinition 到 beanDefinitionMap
  -> 容器 refresh() -> preInstantiateSingletons() 按注册顺序实例化非懒加载单例
  -> 底层反射 new（默认无参构造器 / 唯一构造器 / @Autowired 构造器）
```

## 四、依赖注入的两种入口

| | 构造器注入 | 字段注入 |
|---|---|---|
| 写法 | 依赖是构造参数（Spring 4.3+ 单构造器免注解） | `@Autowired` 打在字段上 |
| 时序 | 实例化瞬间依赖就位，出生即完整 | 先无参 new（半成品，字段全 null）-> `AutowiredAnnotationBeanPostProcessor` 反射 `field.set` 赋值 |
| 可否 final | 可以 | 不可以 |
| 单测 | `new Service(mock)` 直接测 | 要依赖容器/反射工具 |
| 循环依赖 | 启动直接失败 | 可被三级缓存「救活」（见八） |
| 官方态度 | 推荐 | IDEA 黄色警告不推荐 |

- `new` 出来的对象成员变量是默认值（引用 null / 数值 0）；「赋值」就是给半成品填字段。
- 可选依赖 -> setter 注入 `@Autowired(required = false)`。
- 现代写法：构造器注入 + Lombok `@RequiredArgsConstructor`（自动生成 final 字段构造器）。
- 实践建议：存量项目可能大量使用字段注入，维护时跟随现有风格；新代码优先构造器注入，依赖明确且便于单测。

## 五、实例化顺序：依赖驱动递归，不是排序

- `preInstantiateSingletons()` 按 BeanDefinition **注册顺序**（= 扫描顺序，不稳定，别依赖）遍历；创建 A 时递归 `createBean(B)` -> `createBean(C)`，**最深的依赖先出生**。
- **@Order 不控制实例化顺序**（经典面试坑）：它控制的是集合注入排序——`List<Handler>`、过滤器、拦截器、切面、事件监听器。数值越小越靠前，默认 `Ordered.LOWEST_PRECEDENCE = Integer.MAX_VALUE` 排最后。

## 六、单例的边界与线程安全

- **单例 = 每容器一份，不是每 JVM 一份**。多实例（多 pod）部署时每台机器各一套容器和单例。
- 成员变量在堆上唯一实例里，**所有线程共享**；局部变量在栈上，线程私有。
- 无状态原则：bean 成员变量要么是无状态基础设施依赖（如共享的缓存客户端，每个实例各持有一个客户端，但客户端本身不含业务状态），要么干脆不放可变业务状态。

可变状态决策树（放状态前先问：谁读谁写？跨不跨线程？跨不跨 pod？）：

```text
只在方法内用              -> 局部变量（永远优先）
跨调用、单线程顺序访问     -> 实例字段（配合调度单线程保证）
多线程并发读写            -> AtomicReference / synchronized / 并发容器
跨实例（多 pod）要对齐     -> Redis / DB 共享存储
```

- 通用案例：多实例部署的定时任务要保存执行进度。
  - 如果只把书签放在单例字段，pod A 执行完只更新 A 的内存，下一轮可能调度到 pod B，B 会按旧书签重复消费。
  - 正确做法是把权威书签放到 Redis/DB 等共享存储；内存字段最多作为缓存，miss 时回源。
- `final` 的边界：只保证引用不换人 + 构造完成后安全发布；`final List` 内容照样能改；final ≠ 不可变 ≠ 线程安全。
- 非 `@Autowired` 字段为 null 当且仅当没人赋值（声明即初始化 / 构造器赋值 / `@PostConstruct` 都能填）。

## 七、一句话速查

- 注册是存配方，实例化才是造产品；容器两张 Map，配方先行。
- 字段注入 = 无参出生 + 反射补依赖；构造器注入 = 出生即完整。
- 出生顺序靠依赖递归，不靠 @Order。
- 单例是每容器一份；跨 pod 状态放 Redis；可变成员变量先过决策树。
- 无状态的单例才是安全的单例。

## 八、循环依赖与三级缓存（2026-09-04 补）

### 定义与根源

- 定义：依赖图有环，如 A 的字段是 B、B 的字段是 A。
- 根源矛盾：容器的登记规则是「完整才入成品表、取 bean 只从成品表取」，这套规则撞上环就死锁：完成 A 要完整的 B，完成 B 要完整的 A，谁也完不成。
- JVM 视角：`new` 三步 = 分配（堆上定地址，一生不变）-> 清零（字段默认值）-> 构造（this 生效，字段赋值）。**对象存在 ≠ 对象完整**；引用就是地址值，赋值 = 复制地址。半成品完全可以被引用，卡住的是容器的交付规则。
- 朴素递归的后果不是干等：「0x100 正在创建中」无人记录，每次 getBean(A) 都重新分配新地址，无限制造重复半成品直到 StackOverflowError。

### 三级缓存

| 级 | 名字 | 存什么 |
|---|---|---|
| 一级 | `singletonObjects` | 成品 |
| 二级 | `earlySingletonObjects` | 提前暴露的半成品 |
| 三级 | `singletonFactories` | 生产早期引用的工厂（为 AOP 代理留口） |

- `beanDefinitionMap` 不在三环里，是独立注册表。
- 纯 IoC 两级就够；三级保证提前交出的引用在需要代理时是 AOP 代理，且多次领取拿到同一个。

### 本质速记：每级存什么、保证什么（2026-09-08 用户总结）

| 级 | 存什么 | 保证什么 |
|---|---|---|
| 一级 `singletonObjects` | 最终 Bean（定稿物，可能是代理） | 生命周期已完成，谁拿都能放心用 |
| 二级 `earlySingletonObjects` | 唯一 early reference（工厂只产一次的产物） | 提前暴露对象全局唯一：同窗口再要直接复用，不重复生产，保证单例 |
| 三级 `singletonFactories` | early reference 的生成器（Supplier/ObjectFactory） | 按需、延迟生产：没环就不执行；执行时才经 getEarlyBeanReference 决定发裸对象还是代理 |

- 一句话：一级 = 交付层；二级 = 产物缓存（“已生产”的事实 + 产物）；三级 = 生成器（“能生产”的能力）。
- 三级产的未必是代理：没有 AOP 命中时返回的就是裸 rawBean。代理与否由工厂执行时插件（AopPostProcessor.wrapIfNecessary）决定。
- 二级不可省：没有它，同一窗口第二次领取会重复执行工厂——AOP 场景会造出多个不同代理（getEarlyBeanReference 每次都会 wrapIfNecessary，不是幂等的），或用完即焚导致第二次索取直接失败；完工时也无法判断“以哪件货入一级”。把“只产一次”搬进插件自己记，本质仍是二级缓存。

### 正确时序（A 依赖 B，B 依赖 A，字段注入）

```text
createBean(A)
  1 实例化：A0x100 = new A()（半成品）
  2 标记 A「创建中」
  3 赋值 A.b = getBean(B)
      createBean(B)
        1 实例化 B0x200（半成品）
        2 标记 B「创建中」
        3 赋值 B.a = getBean(A)   <- 死锁点
            查 A：一级无 -> 二级无 -> 命中早期引用记录，返回 0x100
        4 B.a = 0x100
        5 B 完成 -> 入一级缓存，返回成品 B
  4 A.b = 成品 B
  5 A 完成 -> 入一级缓存
外层循环走到 B 的配方：一级命中，直接跳过
```

- 检测方式：`singletonsCurrentlyInCreation`「创建中」集合，getBean 重入即成环，不是引用链对比。
- 关键不变量：**B 先完成**（最深先落地）；成品表只进完整品；半成品只活在中转缓存；痊愈靠引用共享，无二次遍历。

### 构造器注入为什么救不了

- 整套解法的前提是「能无依赖地出生」（无参构造造半成品）。构造器注入把依赖变成出生硬前置：`new A(b)` 必须先有 b，连分配内存都轮不到，第一阶段即无限递归。
- Spring 也不会把半成品塞进构造器：构造器代码可能当场调用参数方法。

### 比喻与工程注脚

- 贴切的比喻是期房：分配内存 = 门牌号先定；半成品 = 未交付毛坯；早期引用 = 房号凭证先拿去办事；交付后凭证自动指向完整的房子。
- Spring Boot 2.6+ 默认禁止循环依赖（`spring.main.allow-circular-references=false`）：能救，不该用；遇到环优先重构依赖设计。

## 九、待展开（伏笔）

1. Bean 生命周期扩展点：BeanPostProcessor / BeanFactoryPostProcessor / Aware / InitializingBean（AOP 底座）。
2. 非单例作用域：prototype、request/session、`@RefreshScope`。
3. 三级缓存的工厂与 AOP 代理：为什么提前引用必须是代理，AOP 章展开。
