# SPI 机制

### SPI

**SPI 是 Java 提供的一种服务提供与发现机制**，它允许开发者在运行时动态地加载和替换实现类。通过 SPI，Java 应用可以支持插件式的架构设计，使得框架与具体的实现解耦

- **定义接口**：首先，需要定义一个接口或抽象类。
- **配置文件**：在 **META-INF/services** 目录下创建一个以接口全限定名为文件名的文件（例如：**com.example.MyInterface**），并在该文件中指定实现类的全限定名。
- **加载实现**：使用 **java.util.ServiceLoader** 类来加载所有可用的服务实现。

spi，简单来说，就是 `service provider interface` ，说白了是什么意思呢，比如你有个接口，现在这个接口有 3 个实现类，那么在系统运行的时候对这个接口到底选择哪个实现类呢？这就需要 spi 了，需要**根据指定的配置**或者是**默认的配置**，去**找到对应的实现类**加载进来，然后用这个实现类的实例对象。

spi 机制一般用在哪儿？**插件扩展的场景**，比如说你开发了一个给别人使用的开源框架，如果你想让别人自己写个插件，插到你的开源框架里面，从而扩展某个功能，这个时候 spi 思想就用上了。

spi 经典的思想体现，大家平时都在用，比如说 jdbc。

<aside>
💡

SPI 与反射、类加载紧密相关：服务实现通过反射实例化，而 `ServiceLoader` 借助线程上下文类加载器发现实现类。

</aside>

## ServiceLoader 与 META-INF/services 约定

标准 JDK SPI 的使用步骤：

1. 定义服务接口，例如 `com.example.spi.Greeting`。
2. 提供实现类，例如 `com.example.spi.impl.EnglishGreeting`。
3. 在资源目录 `META-INF/services/` 下创建以**接口全限定名**命名的文件 `com.example.spi.Greeting`，每行写一个实现类全限定名：
   ```text
   com.example.spi.impl.EnglishGreeting
   # 注释以 # 开头
   ```
4. 通过 `ServiceLoader` 加载并遍历服务实现：
   ```java
   ServiceLoader<Greeting> loader = ServiceLoader.load(Greeting.class);
   for (Greeting greeting : loader) {
       greeting.sayHello();
   }
   ```

约定与限制：

- 实现类必须是 public 类，并提供 **public 无参构造器**（由反射实例化）。
- `ServiceLoader` 扫描 classpath / module path 中所有 jar 的 `META-INF/services`，按需实例化（懒加载），可调用 `reload()` 重新加载。
- 只能遍历所有实现，无法按名称直接获取、没有优先级与依赖注入；配置错误通常在运行时才发现。Dubbo、Spring 等框架在此基础上扩展了自己的 SPI。

## 与双亲委派的关系

`ServiceLoader.load()` 默认使用 `Thread.currentThread().getContextClassLoader()`（TCCL）作为类加载器。这相当于让**父加载器中的代码反过来使用子加载器**去加载服务实现，是打破双亲委派的典型手段。

- **JDBC**：`java.sql.DriverManager` 位于启动类路径，而 `com.mysql.cj.jdbc.Driver` 等驱动实现位于应用 classpath。`DriverManager` 通过 `ServiceLoader.load(Driver.class)` + TCCL 加载驱动，这就是父加载器“委托”子加载器的过程。
- **Tomcat**：每个 WebApp 使用独立的 `WebappClassLoader`，优先加载 `WEB-INF/classes` 和 `WEB-INF/lib` 中的类，再委派父加载器，实现应用间隔离；JRE 核心类仍由父加载器加载。容器会在请求入口切换 TCCL，使服务发现定位到当前 WebApp 的 provider。
- **其他**：JNDI、JAXP、OSGi 等也有类似借助 TCCL 或自定义加载器绕过双亲委派的做法。详见 [类加载机制](class-loading.md)。

## 典型应用

| 场景 | 机制 | 说明 |
| --- | --- | --- |
| JDBC Driver | `META-INF/services/java.sql.Driver` | `DriverManager` 通过 `ServiceLoader` 发现各数据库驱动 |
| 日志门面 | SLF4J / commons-logging | 门面定义接口，运行时绑定具体日志实现；桥接器把旧 API 调用转到门面 |
| Dubbo SPI | `@SPI` + `META-INF/services` | 扩展 `ExtensionLoader`，支持按名称获取、IoC 注入与自适应扩展 |
| Spring Boot 自动装配 | `spring.factories`（2.7 及以前）/ `AutoConfiguration.imports`（3.x） | 通过配置发现自动配置类，再结合 `@ConditionalOnClass` 等条件注解决定生效与否 |

<aside>
💡

Spring Boot 的自动装配并不是 JDK `ServiceLoader` 的直接实现，而是同源的“基于配置发现实现”思想；JDK SPI 面向接口扩展，Spring 的机制还叠加了条件判断与 Bean 生命周期。

</aside>

## 延伸阅读

- [反射](reflection.md)：provider 实例的反射创建与性能注意事项
- [动态代理](dynamic-proxy.md)：另一种面向接口的运行时扩展机制
