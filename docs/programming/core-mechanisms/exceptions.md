# 异常体系

异常用于表达程序“非正常”的执行路径。Java 用统一的 `Throwable` 体系区分 JVM 错误与业务异常，并用编译器约束受检异常的处理。

## Throwable / Error / Exception

```text
Throwable
├── Error              （JVM 或系统级严重问题，通常不捕获）
│   ├── VirtualMachineError：OutOfMemoryError、StackOverflowError
│   └── LinkageError：NoClassDefFoundError、ClassFormatError
└── Exception          （程序可以处理的异常）
    ├── RuntimeException（非受检异常：NPE、IllegalArgumentException…）
    └── 其他 Exception  （受检异常：IOException、SQLException…）
```

- **Error**：表示 JVM 自身或资源耗尽等不可恢复问题，捕获后通常也无法恢复，应让程序终止并保留现场。
- **Exception**：表示可预期的异常流程，业务代码主要处理这一支。

## 受检异常 vs 非受检异常

| 类型 | 继承关系 | 编译器约束 | 典型场景 |
| --- | --- | --- | --- |
| 受检异常 | `Exception` 但不继承 `RuntimeException` | 方法签名必须用 `throws` 声明，调用方必须捕获或继续声明 | I/O 失败、SQL 异常、类找不到 |
| 非受检异常 | `RuntimeException` 及其子类 | 不强制声明和捕获 | 空指针、参数非法、数组越界、状态错误 |

- 受检异常表达“调用者必须知道并应对的外部环境问题”；非受检异常多由程序逻辑错误引起，应通过修复代码避免，而不是到处 `catch`。
- 捕获 `InterruptedException` 后不要无声吞掉：若无法向上抛，应恢复中断状态 `Thread.currentThread().interrupt()`。

## 异常处理：try-catch-finally / throw / throws

- `try`：包裹可能抛异常的代码。
- `catch`：按**子类在前、父类在后**的顺序匹配；可用多异常合并 `catch (IOException | SQLException e)`。
- `finally`：无论是否发生异常都会执行（除非 `System.exit()`、JVM 崩溃或线程被强制终止），常用于释放资源。
- `throw`：抛出一个具体的异常对象。
- `throws`：在方法签名上声明可能抛出的异常类型，是方法契约的一部分。

```java
public String loadConfig(String path) {
    try {
        return Files.readString(Path.of(path));
    } catch (IOException e) {
        // 保留根因，转换为更有语义的业务异常
        throw new ConfigLoadException("加载配置失败: " + path, e);
    } finally {
        metrics.recordAttempt(); // 清理/统计逻辑
    }
}
```

<aside>
💡

`finally` 块中的 `return` / `throw` 会**覆盖** `try` 或 `catch` 中的返回和异常，导致异常丢失；应避免在 `finally` 中写 `return` 或抛异常。

</aside>

## try-with-resources 与 AutoCloseable

JDK 7 引入，用于自动关闭资源，本质上由编译器展开成嵌套的 `try-finally`：

- 资源类型需实现 `AutoCloseable`（`Closeable` 继承它）；
- 按资源**声明的逆序**调用 `close()`；
- 若主逻辑与 `close()` 都抛异常，`close()` 的异常会作为**被抑制异常**附加到主异常上，可通过 `getSuppressed()` 获取；
- JDK 9+ 允许把已在外部声明的、事实最终（`effectively final`）资源直接放进括号。

```java
try (InputStream in = new FileInputStream("a.txt");
     BufferedInputStream bin = new BufferedInputStream(in)) {
    return bin.readAllBytes();
} catch (IOException e) {
    for (Throwable suppressed : e.getSuppressed()) {
        log.warn("关闭资源失败", suppressed);
    }
    throw e;
}
```

## 异常设计原则

- **不吞异常**：不要写空的 `catch` 块。至少要记录日志并重新抛出，或明确说明为何可以忽略。
- **不掩盖根因**：包装异常时传入 `cause`（`new XxxException(msg, e)`），不要只取 `e.getMessage()` 重新构造新异常。
- **保留根因与上下文**：异常消息说明“做了什么、失败在哪”，堆栈保留“为什么失败”；跨越边界时抛出抽象层次匹配的异常。
- **只用于异常流程**：不要用异常做流程控制；构造异常会填充栈轨迹，高频路径开销明显。
- **异常只在边界记录一次**：统一在 Controller / 消费端等边界记录日志，避免层层重复打印同一堆栈。
- **定义业务异常体系**：按业务语义继承 `RuntimeException`，携带错误码；对外隐藏内部实现细节。
- **注意并发语义**：在 `finally` 或取消逻辑中处理中断时，恢复中断标志，而不是静默吞掉。

## 延伸阅读

- [Java 规范](java-specification.md)：`equals` / `hashCode` 等通用契约
