# 通用日志框架设计

```jsx
public interface Logger {
    void info(String msg);
    void debug(String msg);
}

private LoggerService() {
        ServiceLoader<Logger> loader = ServiceLoader.load(Logger.class);
        List<Logger> list = new ArrayList<>();
        for (Logger log: loader) {
            list.add(log);
        }
        loggerList = list;
        logger = loggerList.stream().findFirst().orElse(null);
 }
```

参考 [‣](https://app.notion.com/p/197b23b0b5ae80e8bc5ad48b8a6f2b43?pvs=21) 

### **SLF4J (Simple Logging Facade for Java)**

SLF4J 是一个日志门面（Facade），它并不直接提供日志功能，而是作为其他日志框架的统一接口。通过 SLF4J，开发者可以在应用中使用统一的日志 API，而在部署时选择具体使用的日志框架（如 Logback、Log4j 等）。

**桥接器**：SLF4J 提供了多个桥接器（如 **slf4j-log4j12.jar**、**slf4j-jdk14.jar** 等），将 SLF4J 的 API 调用转发给实际的日志实现。

**适配器模式**：通过 SLF4J，不同日志框架之间的差异被隐藏起来，使得开发者只需关注日志接口而不用关心底层实现。

在使用时，引入`slf4j-api` 和相应是日志实现包，即可使用日志组件输出。

- Logback引入包 `**logback-classic**`
- Log4j 引入包 `log4j-api`