# Spring 学习路线

> 2026-08-14 建立；2026-09-08 更新进度、补充「待补机制」清单。
> 目标：系统性深入 Spring，理解常见的 Spring Boot / Java 工程技术底座。
> 方法：AI 最简实现法 —— 每个核心机制先写最小实现（如手写迷你 IoC 容器），学透再对照 Spring 源码与生产用法。
> 笔记沉淀：主题笔记统一写在当前目录（Spring-IoC / Spring-AOP / 扩展点思想 / Supplier 等）。

## 路线
- [ ] 1. IoC 容器与 Bean 生命周期（手写迷你 IoC → 对照 Spring）——【进度】核心主线已完成：扫描注册、配方/产品分离、单例创建、依赖注入（字段）、循环依赖三级缓存、BeanPostProcessor、Aware、InitializingBean。剩余拆入「待补机制」（prototype / FactoryBean / @Configuration 配置类解析等）
- [ ] 2. AOP 原理（动态代理：JDK vs CGLIB → 框架中的切面/拦截器）——【进度】JDK 动态代理主线已完成（v2 BPP + v3 三级工厂 + invokeWithChain 洋葱链）；CGLIB 与 @Transactional 等应用拆入「待补机制」
- [ ] 3. 事务与传播机制（@Transactional 失效场景）——建议放 AOP 章收尾后，作为 AOP 实战应用读
- [ ] 4. Spring MVC 请求链路（DispatcherServlet → Controller）
- [ ] 5. Spring Boot 自动装配
- [ ] 6. 事件机制与容器扩展点（ApplicationEvent、BeanPostProcessor）——【进度】BeanPostProcessor / InitializingBean / 三级缓存扩展点已学透；事件机制未开始（见待补①）
- [ ] 7. Spring AI 框架底座（已学过 0618 笔记，查漏补缺）

## 待补机制（源码骨架增量，建议按此顺序）
> 每个增量做完后：补笔记到当前目录，并按需同步题库。

1. **事件机制**：ApplicationEvent / ApplicationListener / ApplicationEventMulticaster / @EventListener；对比同步发布与 @Async 异步；观察者模式闭环（小增量，建议下一个做）
2. **BeanFactoryPostProcessor + @Configuration/@Bean 简化配置类**：理解「改配方（BD）」与「改产品（BPP）」是两套不同层级的扩展点（经典区分点）
3. **prototype / 多作用域 + DisposableBean 销毁钩子**：与 InitializingBean 对称，补成容器管理的全生命周期闭环；顺手看清「单例缓存边界」
4. **FactoryBean / ObjectProvider / @Value / 类型转换**：各自独立的小机制，按需补
5. **CGLIB（无接口代理）**：动态代理的另一半；真实 Spring 里 @Configuration 类、无接口 @Transactional 都依赖它
6. **事务与传播机制**：TransactionInterceptor = AOP 实战应用；@Transactional 失效场景（自调用绕过代理、传播级别、回滚边界）
7. **Spring MVC 请求链路**：DispatcherServlet → HandlerMapping / HandlerAdapter / Interceptor / 参数解析——责任链 + 策略模式的套用现场
8. **Spring Boot 自动装配**：条件装配 / AutoConfigurationImportSelector

## mini-IoC 已建成范围（learning-lab，2026-09-08）
- 扫描注册 + 配方/产品分离；单例创建；字段注入；循环依赖三级缓存（含 AOP 提前暴露）
- BeanPostProcessor（before / after / getEarlyBeanReference）、ApplicationContextAware、InitializingBean
- AOP 织入主线：wrapIfNecessary → JDK Proxy.newProxyInstance → invokeWithChain → MethodInvocation 洋葱链；控制台常驻 demo
- 覆盖度口径：按「最难主干思想」≈60–70%；按 core 容器功能面积 ≈30–40%；按整个 Spring + Boot 源码面积 <20%

## 关联已有笔记
- `Spring-IoC学习笔记.md`（2026-09：容器两张 Map、生命周期、循环依赖与三级缓存、本质速记）
- `SpringAOP.md`（2026-09：AOP 两阶段、wrapIfNecessary、BPP vs InitializingBean、动态代理机制）
- `扩展点与插件机制思想.md`（2026-09：插件机制五要素、微内核思想）
- `Java-Supplier与Lambda理解.md`（2026-09：Supplier、lambda 底层 invokedynamic）
- 0618 Spring AI 框架（ChatModel/ToolCallback/Advisor 底座）
- Java多线程基础.md（线程池，Bean 生命周期外的运行时知识）
