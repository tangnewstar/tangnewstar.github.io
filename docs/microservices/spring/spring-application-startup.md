# SpringApplication如何运行

入口方法

`SpringApplication.run()`

关键步骤：

- `createApplicationContext`：根据应用类型创建ApplicationContext实例

```java
ApplicationContextFactory DEFAULT = (webApplicationType) -> {
        try {
            switch (webApplicationType) {
                case SERVLET:
                    return new AnnotationConfigServletWebServerApplicationContext();
                case REACTIVE:
                    return new AnnotationConfigReactiveWebServerApplicationContext();
                default:
                    return new AnnotationConfigApplicationContext();
            }
        } catch (Exception var2) {
            Exception ex = var2;
            throw new IllegalStateException("Unable create a default ApplicationContext instance, you may need a custom ApplicationContextFactory", ex);
        }
    };
```

初始化上下文：设置基础属性（环境变量Environment，资源加载器ResourceLoader）

- `prepareContext` ：预准备上下文
    - 设置环境变量：将SpringApplication配置的环境（ConfigurableEnvironment）注入上下文，确保上下文能访问系统属性配置文件（`spring-application.properties`)
    - 注册BeanDefinition：将引入的配置类`@Bean` 定义注册到上下文中
    - **发布 `ApplicationPreparedEvent` 事件**：通知监听器（`SpringApplicationRunListener`）上下文已预准备完成。
    
- `refreshContext` ：刷新上下文，触发容器的刷新逻辑，完成Bean的实例化，依赖注入，生命周期管理，通过`AbstractApplicationContext.refresh()` 方法完成
    - 刷新Bean工厂：初始化 `BeanDefinitionRegistry`（用于管理 Bean 定义）
    - 加载BeanDefinition：从类路径、注解、XML（若启用）等来源加载所有 Bean 定义（如 `@Component`、`@Bean` 方法），并注册到 `BeanFactory`
    - 初始化 ****`BeanFactoryPostProcessor` (修改Bean定义）→ 初始化 `BeanPostProcessor` （后续实例化和依赖注入）
    - 实例化单例 Bean：按顺序实例化所有单例 Bean（非懒加载），完成依赖注入（`@Autowired`、`@Resource`）、生命周期回调（`InitializingBean`、`@PostConstruct`）等
    - 创建 `ApplicationEventMulticaster`用于后续事件发布 → 将 `ApplicationListener` 实现类注册到事件多播器
    
- `afterRefresh` 刷新后的钩子方法，在上下文刷新完成后，执行自定义的“后处理”逻辑，通常用于启动应用特有的服务（如 Web 服务器、定时任务）

### 一些思考

<aside>
💡

SpringBoot使用的默认Web服务器？

</aside>

自动配置项`ServletWebServerFactoryAutoConfiguration` 会导入`ServletWebServerFactoryConfiguration` ，代码如下

```java
@Bean
@ConditionalOnClass({Servlet.class, Tomcat.class})
public TomcatServletWebServerFactory tomcatServletWebServerFactory(
        ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
        ObjectProvider<TomcatContextCustomizer> contextCustomizers,
        ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    // ... 配置连接器、上下文等自定义逻辑
    return factory;
}
```

`spring-boot-starter-web` 引入了`tomcat-embed-core` ，所以当classpath包含Tomcat，会导入TomcatServletWebServerFactory。在refreshContext时会getWebServer，启动Tomcat。 

```java
SpringApplication.run() -> refreshContext -> 
ServletWebServerApplicationContext.refresh() -> onRefresh() -> createWebServer()
TomcatServletWebServerFactory.getWebServer() -> 
TomcatWebServer.initialize() -> this.tomcat.start();
// 通过BeanFactory加载ServletWebServerFactory的实现，默认是TomcatServletWebServerFactory。
// 所以创建按的WebServer就是TomcatWebServer，最后tomcat实例启动。
```