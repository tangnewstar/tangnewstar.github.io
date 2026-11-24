# 内存区域

JVM 把运行时内存划分为若干区域：堆与方法区线程共享，栈与程序计数器线程私有；直接内存虽不属于 JVM 规范定义的运行时数据区，却被 NIO 广泛使用。对象在堆中的具体布局见 [对象与内存](object-and-memory.md)。

## 区域划分

### 线程共享区域

- **堆（Heap）**
    - 存放对象实例与数组，是垃圾回收工作的主要区域。
    - 分代模型下分为**新生代**（Eden + 两个 Survivor）与**老年代**；G1 等收集器改为按 Region 划分，详见 [垃圾回收算法](gc-implementation-and-comparison.md)。
    - 可通过 `-Xms`（初始堆）与 `-Xmx`（最大堆）调整，生产环境通常设为相等以避免反复扩缩容。
- **方法区（Method Area）**
    - 存储类元信息、常量、静态变量、JIT 编译后的代码等。
    - JDK 8 之前用**永久代（PermGen）**实现，位于堆内；JDK 8 起改为**元空间（Metaspace）**，使用本地内存，降低溢出概率。
    - 无法满足分配需求时抛出 `OutOfMemoryError: Metaspace`。
- **直接内存（Direct Memory）**
    - 不属于 JVM 运行时数据区，由 NIO / `Unsafe` 在本地内存中分配。
    - 受 `-XX:MaxDirectMemorySize` 限制（默认与 `-Xmx` 一致），超限会抛出 `OutOfMemoryError: Direct buffer memory`。
    - 适合零拷贝 I/O，可减少堆与本地内存之间的数据复制，但分配和回收成本更高。

### 线程私有区域

- **虚拟机栈（VM Stack）**
    - 每个方法调用对应一个栈帧入栈，方法返回时出栈；栈帧保存局部变量表、操作数栈、动态链接、方法出口等。
    - 生命周期与线程相同；栈深度超过限制会抛出 `StackOverflowError`（受 `-Xss` 与栈帧大小影响）。
- **本地方法栈（Native Method Stack）**
    - 为 Native 方法服务；HotSpot 虚拟机中与虚拟机栈合二为一。
- **程序计数器（Program Counter Register）**
    - 记录当前线程正在执行的字节码指令地址；执行 Native 方法时为 `Undefined`。
    - 线程切换后靠它恢复到正确位置，是唯一不会抛出 `OutOfMemoryError` 的区域。

<aside>
💡

是否线程共享决定了关注点不同：堆和方法区中的共享数据需要保证线程安全；虚拟机栈和程序计数器天然线程隔离。

</aside>

## 延伸阅读

- [对象与内存](object-and-memory.md)：对象头、指针压缩与内存布局
- [垃圾回收算法](gc-implementation-and-comparison.md)：GC 判定、回收算法与收集器实现
- [拓展篇：GC演化历史](gc-evolution-history.md)：从 Serial 到 ZGC 的演进脉络
- [JVM参数调优](jvm-tuning.md)：关键指标、参数与问题排查
- [类加载机制](class-loading.md)、[字节码与执行引擎](bytecode-and-execution.md)
