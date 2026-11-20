# Spring自动配置机制

Spring Boot的自动配置机制，本质上是一套**“基于条件判断的智能装配流水线”**。

从工程师的视角来看，它的核心逻辑可以概括为：**引入依赖 -> 触发配置 -> 条件过滤 -> 注册Bean**。

以下是图示与核心机制拆解：

### 一、 核心流程图示

```markdown
┌───────────────────────────────────────────────────────────────┐
│ 1. 启动入口                                                     │
│    @SpringBootApplication                                      │
│      └─> @EnableAutoConfiguration (开启自动配置开关)           │
└───────────────────────────┬───────────────────────────────────┘
                            ↓
┌───────────────────────────────────────────────────────────────┐
│ 2. 加载候选配置类 (SPI机制)                                     │
│    AutoConfigurationImportSelector 扫描所有Jar包中的路径：      │
│    META-INF/spring.factories (Spring Boot 2.x)                │
│    META-INF/spring/org.springframework.boot.autoconfigure...  │
│    ...AutoConfiguration.imports (Spring Boot 3.x)             │
│    【结果】：拿到几百个候选的 XxxAutoConfiguration 类全路径    │
└───────────────────────────┬───────────────────────────────────┘
                            ↓
┌───────────────────────────────────────────────────────────────┐
│ 3. 条件过滤 (@Conditional 核心逻辑，即"IF-THEN"判断)           │
│    ┌─────────────────────┐                                     │
│    │ @ConditionalOnClass │ -> classpath中有没有相关的类？(引依赖没)│
│    ├─────────────────────┤                                     │
│    │ @ConditionalOnBean  │ -> 容器中有没有指定的Bean？          │
│    ├─────────────────────┤                                     │
│    │@ConditionalOnProperty│-> application.yml中配置了特定参数没？│
│    ├─────────────────────┤                                     │
│    │@ConditionalOnMissingBean│-> 用户自己有没有定义这个Bean？   │
│    └─────────────────────┘                                     │
│    【结果】：只有条件全满足的 AutoConfiguration 类才会生效      │
└───────────────────────────┬───────────────────────────────────┘
                            ↓
┌───────────────────────────────────────────────────────────────┐
│ 4. 属性绑定 (外部化配置)                                        │
│    通过 @EnableConfigurationProperties 将 application.yml      │
│    中的前缀属性，绑定到 XxxProperties 类中。                   │
└───────────────────────────┬───────────────────────────────────┘
                            ↓
┌───────────────────────────────────────────────────────────────┐
│ 5. 实例化注册                                                   │
│    满足条件的 AutoConfiguration 类通过 @Bean 方法，             │
│    结合 Properties 属性值，向 Spring IoC 容器注册组件。         │
└───────────────────────────────────────────────────────────────┘
```

---

### 二、 核心机制拆解（工程师视角）

为了方便记忆和理解，自动配置其实只做了三件事：**找配置、看条件、拿参数**。

#### 1. 找配置（怎么知道有哪些配置？）

- **机制**：基于 Java SPI 机制的思想（Spring 自己扩展了一版）。
- **动作**：启动时，Spring 扫描所有依赖包下的特定配置文件，把里面声明的几百个 `AutoConfiguration` 类加载进内存作为“候选名单”。
- **工程师思考**：这是一种**解耦和可插拔设计**。Spring Boot 官方把所有常见组件的配置类都写好放在那里，用不用取决于你的工程引没引依赖。

#### 2. 看条件（怎么决定用哪个配置？）

- **机制**：`@Conditional` 及其派生注解。
- **动作**：对候选名单进行“裁剪”。这是自动配置最精髓的地方。
- **工程师思考**：典型的 **防御性编程** 和 **开闭原则（OCP）**。
    - 你引了 `mysql-connector-java`，`@ConditionalOnClass` 发现 Driver 类存在，数据库相关配置生效。
    - 你自己在代码里写了一个 `@Bean RestTemplate`，自动配置里的 `@ConditionalOnMissingBean` 发现容器里已经有了，就不再自动创建，**把控制权交还给开发者**。

#### 3. 拿参数（怎么自定义配置细节？）

- **机制**：`@ConfigurationProperties`。
- **动作**：将 `application.yml` 中的配置（如 `spring.datasource.url`）通过 setter 方法注入到专门的 Properties 类中，自动配置类再拿这些参数去 `new` 对象。
- **工程师思考**：**约定大于配置**。底层组件需要参数，但我不强制写死，我提供默认值，并留出外部配置的口子。

---

### 三、 一段代码看懂本质

以 `RedisAutoConfiguration` 为例，剥离掉复杂逻辑后，它的骨架是这样的：

```java
// 1. 标记为配置类
@Configuration
// 2. 条件1：classpath里必须存在 RedisOperations 类（即你引入了 redis 的依赖）
@ConditionalOnClass(RedisOperations.class)
// 3. 读取 application.yml 中的 spring.redis.* 配置
@EnableConfigurationProperties(RedisProperties.class)
public class RedisAutoConfiguration {

    // 4. 条件2：容器中没有 RedisTemplate 时，才执行此方法
    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(factory); // 使用注入的参数
        return template;
    }
}
```

### 四、 总结

Spring自动配置机制，对工程师而言就是一套**“-if-else-的自动化工厂”**：

1. **启动触发** (`@EnableAutoConfiguration`)
2. **加载候选** (读取 `imports` 文件)
3. **条件判定** (`@ConditionalOnXxx` 判 classpath、判 Bean、判配置)
4. **参数注入** (`@ConfigurationProperties` 绑定 yml)
5. **实例化** (`@Bean` 注册到容器，且不覆盖用户自定义的 Bean)

这种设计让框架既做到了**“开箱即用”**，又保留了**“灵活定制”**的能力。