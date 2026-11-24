# GC演化历史

从 Serial 到 ZGC，梳理垃圾收集算法的演进脉络、每一代收集器要解决的问题及其设计取舍。实现细节与对比见 [垃圾回收算法](gc-implementation-and-comparison.md)。

## 不同版本 JDK 默认 GC

- Client VM 默认使用 Serial + Serial Old
- Server VM 默认使用 Parallel Scavenge + Serial Old（PS MarkSweep）
- JDK 9 起 Server VM 默认使用 G1；JDK 8 可通过 `-XX:+UseG1GC` 开启
- JDK 17 起 ZGC 可用于生产环境

## 从标记 - 清除到分代回收

- **标记 - 清除（Mark-Sweep）**：先标记存活对象再清除垃圾，思路直接，但会产生内存碎片并带来分配延迟。
- **标记 - 复制（Mark-Copy）**：把内存一分为二，每次只用一半，GC 时把存活对象复制到另一半后清空当前半区。无碎片、分配快，但内存利用率最高仅 50%。IBM 的研究把 Eden 与 Survivor 按 8:1 划分，将利用率提升到约 90%。
- **标记 - 整理（Mark-Compact）**：标记后把存活对象移向一端，再清理边界外内存，兼顾内存利用率与连续空间；代价是对象移动带来的二次遍历开销。
- **分代回收（Generational GC）**：基于弱分代假说（绝大多数对象朝生暮死）与强分代假说（经历多次 GC 的对象更难消亡），把堆划分为新生代与老年代，分别采用不同回收策略。为支持分代，引入了写屏障（Write Barrier）与卡表（Card Table，以 512 字节为粒度记录老年代对新生代的引用）等技术。

## 各阶段收集器

### 第一阶段：Serial GC - 奠基者

- **背景**：早期计算机多为单核，Java 需要自动化内存管理来避免手动管理的陷阱。
- **核心问题**：把程序员从易错的手动内存管理中解放出来。
- **实现**：新生代使用 Stop-Copy（Serial Copying），老年代使用标记 - 整理；回收时暂停所有应用线程（STW）。
- **局限**：STW 时间长，单线程无法利用多核 CPU。

### 第二阶段：Parallel GC - 效率追求者

- **背景**：多核服务器普及，批处理等场景关注单位时间完成的任务量，对单次停顿不敏感。
- **核心问题**：提升吞吐量。
- **实现**：Serial 的多线程版本。新生代 Parallel Scavenge 并行复制，老年代 Parallel Old 并行标记 - 整理。
- **局限**：总 GC 时间变短，但 STW 期间所有核心都在做 GC，停顿“毛刺”可能更明显。

### 第三阶段：CMS - 延迟挑战者

- **背景**：Web / B/S 系统对响应时间敏感，无法接受数百毫秒甚至数秒的服务中断。
- **核心问题**：让 GC 线程尽可能与用户线程并发，减少停顿。
- **实现**：初始标记（STW） → 并发标记 → 重新标记（STW） → 并发清除。
- **局限**：对 CPU 资源敏感；并发清理期间产生浮动垃圾；使用标记 - 清除不压缩内存，碎片积累后触发单线程 Full GC，停顿极长；复杂度高、调参困难。

### 第四阶段：G1 - 平衡与革新者

- **背景**：堆内存增长到几十 GB，CMS 的碎片化问题几乎无解。
- **核心问题**：解决碎片化，提供可预测的停顿模型（软实时）。
- **实现**：Region 化布局，跟踪各 Region 的“垃圾价值”，优先回收价值最高的 Region；整体看是标记 - 整理，Region 之间是复制，从根本上避免碎片。
- **局限**：Remembered Sets 等结构带来约 10%~20% 的额外内存开销；初始标记与最终标记仍需短暂 STW；小堆上管理开销可能不划算。

### 第五阶段：ZGC / Shenandoah - 超低延迟先锋

- **背景**：数百 GB 甚至 TB 级堆成为现实，金融交易、实时计算等场景无法接受超过 10ms 的停顿。
- **核心问题**：把停顿压到 10ms 以下甚至亚毫秒级，且停顿时间不随堆容量增长。
- **实现（以 ZGC 为例）**：染色指针把标记与重定位元数据放在指针中；读屏障在读取引用时检查并修正已转移对象的引用；标记、转移、重映射几乎全程并发，STW 只剩 Roots 扫描。
- **局限**：需要额外内存与读屏障开销，吞吐量可能略有下降。

## 总结

- **Serial → Parallel**：解决吞吐量，代价是更激烈的 STW。
- **Parallel → CMS**：解决延迟，但引入碎片与复杂度。
- **CMS → G1**：解决碎片化，提供可预测停顿，面向大堆。
- **G1 → ZGC / Shenandoah**：追求近乎无停顿，解决超大堆延迟，代价是设计更复杂、吞吐量略降。

## 参考资料

- Oracle《Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide, Release 17》：<https://docs.oracle.com/en/java/javase/17/gctuning/>
- OpenJDK JEP 333《ZGC: A Scalable Low-Latency Garbage Collector》：<https://openjdk.org/jeps/333>

## 延伸阅读

- [垃圾回收算法](gc-implementation-and-comparison.md)
- [JVM参数调优](jvm-tuning.md)
- [内存区域](jvm-memory-and-gc.md)
