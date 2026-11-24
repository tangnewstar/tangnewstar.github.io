# 类加载机制

一个 `.class` 文件从字节流变成可执行类型，要经历加载、连接与初始化；双亲委派决定了“由谁来加载”，而 SPI、容器等场景又会主动打破它。

## 类加载生命周期

```text
加载(Loading) → 验证(Verification) → 准备(Preparation) → 解析(Resolution) → 初始化(Initialization) → 使用(Using) → 卸载(Unloading)
```

其中验证、准备、解析合称**连接（Linking）**。

### 1. 加载

- 通过类的全限定名获取二进制字节流（classpath、jar、网络、动态生成等）。
- 将字节流中的静态存储结构转换为方法区的运行时数据结构。
- 在堆中生成一个 `java.lang.Class` 对象，作为访问方法区类型数据的入口。
- 数组类由 JVM 直接创建，其元素类型仍由类加载器加载。

### 2. 验证

确保 `.class` 不破坏虚拟机安全：文件格式、元数据（如父类是否合法）、字节码（操作数栈类型是否匹配）、符号引用（能否解析到目标）等。

### 3. 准备

为**类变量（static 字段）**分配内存并设置为**零值**（`0`、`false`、`null`），而不是代码中写的初始值。

<aside>
💡

`public static final int VALUE = 1;` 在编译期会写入 `ConstantValue` 属性，准备阶段就被赋值为 1；普通 `static int value = 1` 此时仍是 0，要到初始化阶段才变成 1。

</aside>

### 4. 解析

将常量池中的**符号引用**替换为**直接引用**（指向目标的内存地址或句柄）。解析目标包括类、接口、字段、方法等。

### 5. 初始化

执行编译器合成的 `<clinit>()`：按源码顺序合并静态变量赋值与 `static {}` 代码块，父类先于子类执行。JVM 通过加锁保证同一类只被初始化一次。

**主动触发初始化的常见场景：**

- `new`、`getstatic`、`putstatic`、`invokestatic` 四条字节码指令；
- `Class.forName(name)`（默认 `initialize = true`）、反射调用；
- 初始化子类时父类先初始化；
- 作为启动类（含 `main` 方法）；
- JDK 7+ 的 `MethodHandle`、接口默认方法实现类的初始化。

被动引用不会触发初始化：通过子类引用父类静态字段、通过数组定义引用类、访问编译期常量。

## 类加载器与双亲委派

HotSpot 内置加载器层次：

| 加载器 | 加载范围 | 说明 |
| --- | --- | --- |
| Bootstrap ClassLoader | `JAVA_HOME/lib` 中的核心类库 | C++ 实现，`getParent()` 返回 `null` |
| Platform ClassLoader（JDK 9+） | 平台模块 | 替代旧版 Extension ClassLoader |
| Application ClassLoader | `classpath` | 默认的系统类加载器 |
| Custom ClassLoader | 自定义路径/来源 | 继承 `ClassLoader`，重写 `findClass` |

**双亲委派流程**：`loadClass()` 先查缓存 → 委托父加载器 → 父加载不到再调用自己的 `findClass()`。

```java
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    synchronized (getClassLoadingLock(name)) {
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) c = parent.loadClass(name, false);
                else c = findBootstrapClassOrNull(name);
            } catch (ClassNotFoundException ignored) { }
            if (c == null) c = findClass(name);
        }
        if (resolve) resolveClass(c);
        return c;
    }
}
```

**意义**：

- 安全：核心类库（如 `java.lang.String`）由 Bootstrap 加载，不会被应用自定义的同名类替换。
- 避免重复加载：一个类由“加载器 + 全限定名”唯一标识，父加载器加载过的类子加载器直接使用。

## 双亲委派的打破场景

双亲委派不是强制规范，而是推荐模型。以下场景需要“子加载器反过来影响父加载器”或“优先自行加载”：

1. **SPI / JDBC**：`DriverManager` 位于启动类路径，但 JDBC 驱动实现由应用提供。通过**线程上下文类加载器（TCCL）**把父加载器“委托”回应用类加载器：`ServiceLoader.load(Driver.class)` 默认使用 `Thread.currentThread().getContextClassLoader()`。详见 [SPI 机制](spi.md)。
2. **Tomcat 等 Web 容器**：每个 WebApp 拥有独立的 `WebappClassLoader`，**先加载 `WEB-INF/classes` 与 `WEB-INF/lib`，再委派父加载器**（顺序与双亲委派相反），实现应用间类隔离；JRE 核心类仍交给父加载器；共享依赖由 `CommonClassLoader` 加载。
3. **OSGi / 热部署**：模块或插件各自使用独立加载器；替换类时直接丢弃整个 ClassLoader，使旧类可被卸载。
4. **历史兼容**：JNDI、JAXP 等需要在核心库中加载服务实现，同样借助 TCCL 或上下文加载器切换。

## 自定义类加载器要点

- 继承 `ClassLoader`，**重写 `findClass`**（不要轻易重写 `loadClass`，以免破坏双亲委派）；
- 在 `findClass` 中读取字节数组，调用 `defineClass(name, bytes, 0, len)` 生成 `Class` 对象；
- 需要打破委派时可覆盖 `loadClass` 调整“先父后己”的顺序，但必须保证核心类仍由父加载器加载；
- 类卸载条件：该类的所有实例不可达、加载它的 ClassLoader 不可达、对应的 `Class` 对象不可达。

## 延伸阅读

- [反射](reflection.md)：`Class.forName` 与 `Class` 对象的获取方式
- [SPI 机制](spi.md)：TCCL 与 JDBC 驱动加载
