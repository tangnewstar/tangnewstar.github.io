# 反射

反射允许程序在运行时检查类、构造对象、调用方法和读写字段。框架之所以能在编译期不知道具体实现的情况下工作，反射是基础能力之一。

## Class 对象：反射的入口

获取 `Class` 对象的三种常见方式：

```java
Class<?> c1 = Class.forName("com.example.User"); // 默认触发类初始化
Class<?> c2 = user.getClass();                    // 通过实例获取，已初始化
Class<?> c3 = User.class;                         // 类字面量，不触发初始化
```

- 一个类在 JVM 中由“**加载它的类加载器 + 全限定名**”唯一标识；不同加载器加载的同名类是两个不同的 `Class`。
- `Class.forName(name, initialize, loader)` 可显式指定类加载器和是否初始化。

## 常用反射 API

```java
Class<?> clazz = Class.forName("com.example.User");

// 构造器
Constructor<?> ctor = clazz.getDeclaredConstructor(String.class);
Object obj = ctor.newInstance("Tom");

// 字段：getField 只能取 public（含父类），getDeclaredField 可取本类所有（含 private）
Field nameField = clazz.getDeclaredField("name");
nameField.setAccessible(true);        // 绕过访问控制检查
nameField.set(obj, "Jerry");
System.out.println(nameField.get(obj));

// 方法
Method method = clazz.getMethod("getName");
System.out.println(method.invoke(obj));
```

常用方法对照：

| 目标 | 只取 public | 取本类声明的所有成员 |
| --- | --- | --- |
| 字段 | `getField` / `getFields` | `getDeclaredField` / `getDeclaredFields` |
| 方法 | `getMethod` / `getMethods` | `getDeclaredMethod` / `getDeclaredMethods` |
| 构造器 | `getConstructor` | `getDeclaredConstructor` |

## 性能与安全限制

**性能开销来源：**

- 方法/字段查找、参数数组封装、访问权限检查、自动装箱拆箱；
- 反射调用阻碍 JIT 内联，热点路径上差距更明显。

**优化手段：**

- 缓存 `Class`、`Method`、`Field`、`Constructor`，避免重复查找；
- 对重复调用的成员提前 `setAccessible(true)` 关闭访问检查；
- 高频调用改用 `MethodHandle`（JDK 7+）、`VarHandle`（JDK 9+）或 ASM/CGLIB 等字节码生成方案（框架常用）。

**安全与封装限制：**

- 反射能访问 private 成员，破坏封装；同时泛型因类型擦除，运行时可被绕过（如向 `List<String>` 塞入 `Integer`）。
- 传统 `SecurityManager` 可限制反射权限，但在现代 JDK 中已废弃。
- JPMS 模块系统（JDK 9+）引入强封装：对未 `exports` / `opens` 的包调用 `setAccessible(true)` 会抛 `InaccessibleObjectException`，可通过 `--add-opens` 显式开放。

## 典型应用

- Spring 的依赖注入与 Bean 实例化、MyBatis 的字段映射、JSON 库的序列化/反序列化；
- JDK 动态代理中的 `InvocationHandler`（见 [动态代理](dynamic-proxy.md)）；
- `ServiceLoader` 通过反射调用 provider 的 public 无参构造器（见 [SPI 机制](spi.md)）。

## 延伸阅读

- [类加载机制](class-loading.md)：`Class` 对象是如何被加载与初始化的
