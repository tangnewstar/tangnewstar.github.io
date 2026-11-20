# JMM的可见性与原子性

- 可见性：一个线程对贡献变量的修改，能够被其他线程感知

JMM规定，所有变量存储在主内存中，而每个线程有自己的工作内存，用于缓存线程使用的变量副本。线程对变量的读写操作，均在工作内存中完成，仅在必要时和主内存同步。

### volatile的核心作用

1. 可见性：一个变量被volatile修饰时，**修改会立即同步到主内存**，且其**他线程会从主内存获取改变量的最新值**。
2. 有序性（禁止指令重排序）：JVM为了优化性能可能对指令进行重排序，volatile通过内存屏障（Memory Barrier）禁止特定类型的指令重排序。

例如对象初始化`instance=new Singleton()` 可能被拆成分配内存空间，对象初始化，赋值引用三步，重排可能导致获取到未初始化的对象。

*重排序是指在不改变单线程执行结果的前提下，调整指令执行顺序*

如何实现？

- 可见性保证：CPU 的**缓存一致性协议**
    - 写：JVM 会将该变量从线程工作内存刷新到主内存
    - 读：JVM 会 invalidate 线程工作内存中的缓存，强制从主内存重新加载最新值
- 有序性实现：
    - 写：写后插入写屏障StoreBarrier，确保之前的普通变量修改不能推迟到volatile变量后
    - 读：读前插入读屏障LoadBarrier，确保之后的普通变量不能提前到volatile变量读取之前

例如：写屏障保证了a=1先于flag=true执行，读屏障保证了a的值不能在flag前读取

```java
int a = 0;
boolean flag = false;

// 线程 1
a = 1;      // 操作 1
flag = true; // 操作 2

// 线程 2
if (flag) {  // 操作 3
    System.out.println(a); // 操作 4
}
```

适用场景：

状态标记量（如 `boolean isRunning` ）

双重检查锁定（DoubleCheckLock）中的对象引用（单例模式）

### **但是，不保证原子性：**

`volatile` 仅**支持基本的读写操作的原子**性，但**对于复合操作**（如 `i = i + 1` ，读→改→写三个步骤）**无法保证原子性**，因为这些操作会分解为多条指令。

<aside>
💡

 原子性：一个操作不可分割，要么全部执行完成，要么全部不执行。

</aside>

- 天然的原子操作
    - 基本类型：int、boolean、char等读写
    - volatile变量的简单读写
- 显示同步
    - synchronized同步块：进入同步前获取锁，退出时释放锁，期间操作对其他线程不可见。同时JVM保证同步块操作内是原子的。
    - 原子类：AtomicInteger、AtomicReference，基于CAS机制实现无锁原子操作。

Java 内存模型（JMM）通过**在语言层面对内存屏障的插入**，来实现 **Happens-Before** 的规则。

| Happens-Before 规则 | 对应的内存屏障 | 举例 |
| --- | --- | --- |
| `volatile` 变量的写操作 `happens-before` 读操作 | 插入 **Store Barrier** 和 **Load Barrier** | `volatile` 变量读写 |
| `synchronized` 的释放锁 `happens-before` 获取锁 | 插入 **Store Barrier** 和 **Load Barrier** | `synchronized` 块退出和进入 |
| `final` 字段的初始化 `happens-before` 任何对 final 字段的访问 | 无需屏障，JMM 保证 | `final` 字段访问 |
| 构造函数的执行 `happens-before` 对象的引用被赋予其他线程 | 无需屏障，由 JMM 保证 | 对象初始化并传给其他线程 |