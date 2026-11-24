# 垃圾回收算法

从垃圾判定与回收算法出发，梳理常见收集器的实现与对比。收集器的演进脉络见 [拓展篇：GC演化历史](gc-evolution-history.md)，参数与调优见 [JVM参数调优](jvm-tuning.md)。

<aside>
💡

本文图示为外链社区/官方图片，版权归原作者，加载依赖外部站点；来源已在各图下注明。

</aside>

## 垃圾回收概览

垃圾回收（Garbage Collection, GC）自动回收堆中不再使用的对象，是 Java 自动内存管理的核心。

### 垃圾判定

- **引用计数法**：对象被引用时计数加 1，引用失效时减 1，计数为 0 即可回收；实现简单，但无法解决循环引用问题。
- **可达性分析**：从 GC Roots 出发沿引用链遍历，不可达的对象判定为可回收。GC Roots 包括虚拟机栈中的引用、方法区静态属性与常量引用、本地方法栈 JNI 引用等。

### 回收算法

- **标记 - 清除**：标记后统一清除；实现简单，但会产生内存碎片。
- **复制**：把内存分为两块，将存活对象复制到另一块后清空当前块；无碎片，但空间利用率低，适合存活率低的**新生代**。
- **标记 - 整理**：标记后把存活对象移向一端，再清理边界外内存；无碎片，适合存活率高的**老年代**。
- **分代收集**：基于“多数对象朝生暮死、经历多次 GC 的对象更难消亡”的经验，不同代采用不同算法（新生代复制、老年代标记 - 清除或标记 - 整理）。

分代模型下堆的划分与算法配合：

![HotSpot 堆结构与分代模型](https://oss.javaguide.cn/github/javaguide/java/jvm/hotspot-heap-structure.png)

> 图源：[JavaGuide · JVM垃圾回收详解](https://javaguide.cn/java/jvm/jvm-garbage-collection.html)（社区资料，版权归原作者）。

### 常见收集器

| 收集器 | 算法类型 | 定位 |
| --- | --- | --- |
| Serial | 新生代复制 + 老年代标记 - 整理 | 单线程收集，简单高效，适合单核 / 小内存场景 |
| Parallel | 新生代复制 + 老年代标记 - 整理 | 多线程并行，注重吞吐量，适合批处理 |
| CMS | 标记 - 清除 | 并发标记清除，降低停顿，但有内存碎片与浮动垃圾 |
| G1 | 整体标记 - 整理，Region 之间复制 | Region 化分代，按收益回收，兼顾吞吐与停顿，大堆默认 |
| ZGC | 并发标记 - 整理（染色指针） | 染色指针 + 读屏障，几乎全程并发，面向超大堆与极低延迟 |

## CMS 与 G1 的关键机制

### CMS

基于标记 - 清除算法，关键过程如下：

![CMS 收集器运行示意图](https://oss.javaguide.cn/github/javaguide/java/jvm/cms-garbage-collector.png)

> 图源：[JavaGuide · JVM垃圾回收详解](https://javaguide.cn/java/jvm/jvm-garbage-collection.html)（社区资料，版权归原作者）。

- 初始标记：标记 GC Roots 直接关联的对象。
- 并发标记：与用户线程一起做 GC Root Tracing。
- 重新标记：修正并发标记期间引用变化导致的对象标记变化。
- 优势：并发收集，停顿时间短，适合延迟敏感服务。
- 劣势：内存碎片、CPU 资源敏感、无法处理浮动垃圾。

### G1

基于标记 - 整理算法（Region 之间为复制），关键过程如下：

![G1 收集器运行示意图](https://oss.javaguide.cn/github/javaguide/java/jvm/g1-garbage-collector.png)

> 图源：[JavaGuide · JVM垃圾回收详解](https://javaguide.cn/java/jvm/jvm-garbage-collection.html)（社区资料，版权归原作者）。

- Region 化内存布局，只回收垃圾比例最高的 Region。
- 通过 `-XX:MaxGCPauseMillis` 设定软目标，优先回收收益高的 Region。

## 大堆场景：G1 与 ZGC

### G1：大堆上的增量回收

**为什么适合大堆**：CMS 回收老年代时需要扫描整个老年代空间，堆越大停顿越长；标记 - 清除产生的碎片积累到一定程度会触发单线程 Full GC，停顿不可接受。G1 不再以整个代为单位回收，每次只选择一部分 Region（Collection Set），单次回收工作量由 CSet 大小决定，与堆总大小解耦，从而把大堆上的回收拆成多次小回收。

- **Region 化布局**：不要求新生代、老年代物理连续，堆被切分为多个大小相等的 Region，可扮演 Eden、Survivor、Old、Humongous（大对象）等角色。
- **按收益回收**：跟踪各 Region 的回收收益（可回收空间与耗时），优先回收价值最高的 Region（Collection Set）。单次回收工作量由 CSet 大小决定，而非整个堆的大小。
- **Humongous Region**：超过 Region 容量一半的大对象直接进入 Humongous Region，避免大对象挤占老年代、频繁触发 Full GC。
- **Young GC + Mixed GC**：只回收年轻代 Region 为 Young GC；同时回收部分老年代 Region 为 Mixed GC，对象晋升更平滑，新生代可动态扩容。
- **停顿可预测**：通过 `-XX:MaxGCPauseMillis`（默认 200ms）设定软目标，G1 据此动态调整年轻代大小与每次回收的 Region 数量。

### ZGC：超大堆上的超低延迟

**为什么适合超大堆**：标记 - 整理中的对象移动阶段必须 STW，因为移动对象时要修正所有指向它的引用；堆越大、存活对象越多，停顿越长。ZGC 借助染色指针和读屏障，把标记、转移、重映射都做成并发，应用线程只在 Roots 扫描阶段短暂停顿（通常 < 1ms），与堆大小基本无关。

关键过程：

![ZGC GC 周期](https://dev.java/assets/images/garbage-collection/zgc-gc-cycle.png)

![ZGC 染色指针](https://dev.java/assets/images/garbage-collection/zgc-colored-pointers.png)

> 图源：[Oracle dev.java · Deep-dive into ZGC's Architecture](https://dev.java/learn/jvm/tool/garbage-collection/zgc-deepdive/)（官方资料，版权归 Oracle）。

- **STW 与堆大小解耦**：停顿主要发生在 GC Roots 扫描，通常 < 1ms，与堆容量、存活对象数量基本无关。
- **染色指针（Colored Pointer）**：把标记、重定位等元数据记录在指针本身而不是对象头，GC 线程据此判断对象状态。
- **读屏障（Load Barrier）**：应用线程读取引用时插入检查，发现引用指向已转移对象则动态修正，使对象移动也能并发进行。
- **全程并发**：标记、转移（移动对象）、重映射（修正指针）几乎全部并发，无需长时间 STW。
- **代价**：需要额外内存与读屏障开销，吞吐量可能略低于 G1；JDK 17+ 生产可用，适合百 GB 级堆与亚毫秒级延迟场景。

<aside>
💡

CMS 有两次 STW（初始标记、重新标记），G1 有三次 STW（初始标记、最终标记、筛选回收），ZGC 的 STW 只剩极短的 Roots 扫描。

</aside>

参数与调优实践见 [JVM参数调优](jvm-tuning.md)。

## 参考资料

- Oracle《Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide, Release 17》：<https://docs.oracle.com/en/java/javase/17/gctuning/>
- OpenJDK JEP 333《ZGC: A Scalable Low-Latency Garbage Collector》：<https://openjdk.org/jeps/333>

## 延伸阅读

- [内存区域](jvm-memory-and-gc.md)：运行时区域划分
- [对象与内存](object-and-memory.md)：对象布局与指针压缩
- [拓展篇：GC演化历史](gc-evolution-history.md)：从 Serial 到 ZGC 的演进脉络
- [JVM参数调优](jvm-tuning.md)：关键指标、参数与 OOM 排查
