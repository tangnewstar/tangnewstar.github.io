# Spring Bean

### **Spring Bean 的生命周期**

Spring Bean 的生命周期指的是从 Bean 被创建到最终被销毁的完整过程，主要包括以下阶段：

1. **实例化（Instantiation）**Spring 通过反射创建 Bean 的实例（调用无参构造方法）。
2. **属性注入（Population）**Spring 根据配置（XML / 注解）为 Bean 的属性赋值（依赖注入）。
3. **初始化（Initialization）**
    - 若 Bean 实现 `BeanNameAware`，调用 `setBeanName()` 传入 Bean 的 ID。
    - 若 Bean 实现 `BeanFactoryAware`，调用 `setBeanFactory()` 传入 BeanFactory 实例。
    - 若 Bean 实现 `ApplicationContextAware`，调用 `setApplicationContext()` 传入 ApplicationContext 实例。
    - 若存在 `BeanPostProcessor`，调用其 `postProcessBeforeInitialization()` 方法（初始化前置处理）。
    - 若 Bean 定义了 `init-method` 或使用 `@PostConstruct` 注解，执行初始化方法。
    - 若存在 `BeanPostProcessor`，调用其 `postProcessAfterInitialization()` 方法（初始化后置处理）。
4. **使用（In Use）**Bean 处于可用状态，可被应用程序调用。
5. **销毁（Destruction）**
    - 若 Bean 实现 `DisposableBean`，调用 `destroy()` 方法。
    - 若 Bean 定义了 `destroy-method` 或使用 `@PreDestroy` 注解，执行销毁方法。

### **BeanFactory 和 ApplicationContext 的区别**

| **对比维度** | **BeanFactory** | **ApplicationContext** |
| --- | --- | --- |
| **功能范围** | 基础容器，仅提供 Bean 的创建和获取功能。 | 继承 BeanFactory，扩展了更多企业级功能（如国际化、事件发布、AOP 集成等）。 |
| **初始化方式** | 懒加载（首次获取 Bean 时才实例化）。 | 预加载（容器启动时默认实例化所有单例 Bean）。 |
| **接口实现** | 最基础的实现是 `DefaultListableBeanFactory`。 | 常见实现：`ClassPathXmlApplicationContext`、`AnnotationConfigApplicationContext` 等。 |
| **附加功能** | 无额外功能。 | 支持国际化（`MessageSource`）、事件机制（`ApplicationEventPublisher`）、资源访问（`ResourceLoader`）等。 |
| **适用场景** | 资源受限环境（如移动设备）。 | 大多数企业级应用（功能更全面）。 |

**总结**：`ApplicationContext` 是 `BeanFactory` 的超集，提供了更丰富的功能，是实际开发中更常用的容器；而 `BeanFactory` 更轻量，适用于对资源占用有严格要求的场景。