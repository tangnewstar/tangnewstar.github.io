# JVM参数调优

JVM 参数调优围绕吞吐量、延迟与内存占用三个目标展开：先明确目标，再选择收集器与参数，最后用日志和工具验证。三者互相制约，通常只能优先保证其中两项。

## GC 关键指标

- **吞吐量（Throughput）**：应用程序运行时间占总时间（应用运行时间 + GC 时间）的比例。
- **延迟（Latency）**：GC 导致的应用程序停顿时间（STW），通常是 Web 服务最敏感的指标。
- **内存开销（Footprint）**：GC 自身数据结构与空间开销，例如 G1 的 Remembered Sets。

## 调优目标与收集器选型

| 场景 | 推荐收集器 | 说明 |
| --- | --- | --- |
| 客户端 / 小内存应用 | Serial | 单线程、简单，内存开销最小 |
| 后台批处理、吞吐优先 | Parallel | 多线程并行回收，追求单位时间完成更多工作 |
| JDK 8 低延迟 Web 服务 | CMS | 并发标记清除，但存在内存碎片与浮动垃圾 |
| JDK 9+ 大堆通用场景 | G1 | Region 化分代，可预测停顿，默认收集器 |
| JDK 17+ 超大堆、极低延迟 | ZGC | 染色指针 + 读屏障，STW 与堆大小基本解耦 |

各收集器的实现细节见 [垃圾回收算法](gc-implementation-and-comparison.md)。

## 常用参数

| 参数 | 作用 |
| --- | --- |
| `-Xms` / `-Xmx` | 堆初始 / 最大大小，生产环境常设为相等，避免动态扩缩容抖动 |
| `-Xss` | 单个线程的栈大小，影响可支持线程数与 `StackOverflowError` 出现时机 |
| `-Xmn` / `-XX:NewRatio` | 新生代大小或比例；使用 G1 时一般不显式设置，交由其动态调整 |
| `-XX:MaxGCPauseMillis` | G1 的目标停顿时间（默认 200ms），是软目标而非硬保证 |
| `-XX:G1NewSizePercent` / `-XX:G1MaxNewSizePercent` | G1 年轻代占比的上下限（默认 5% / 60%） |
| `-XX:InitiatingHeapOccupancyPercent` | G1 触发并发标记的堆占用阈值（默认 45%） |
| `-XX:+UseG1GC` / `-XX:+UseZGC` | 显式选择 G1 / ZGC |
| `-XX:ParallelGCThreads` / `-XX:ConcGCThreads` | 并行 / 并发 GC 线程数 |
| `-XX:MaxDirectMemorySize` | 直接内存上限 |
| `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=...` | OOM 时自动转储堆快照 |

## 常见调优问题

**什么时候触发 Young GC？**

新生代空间不足以分配新对象时触发。G1 下若 Young GC 耗时远小于 `MaxGCPauseMillis`，会增大年轻代 Region 数量以提升吞吐量；反之则减少，以缩短停顿。

**G1 堆内存 2G 时，年轻代最大能分配多少？**

由 `-XX:G1MaxNewSizePercent`（默认 60%）决定，最大值约为 `2G × 60% = 1.2G`。实际大小通常介于 `G1NewSizePercent`（默认 5%）与 `G1MaxNewSizePercent`（默认 60%）之间，G1 会根据目标停顿时间动态调整。

<aside>
💡

**避免显式设置年轻代大小**：通过 `-Xmn` 或 `-XX:NewSize` 强制指定会禁用 G1 的动态调整机制，可能导致性能下降。另外，老年代 Region 与 Humongous Region 也会占用堆空间，年轻代实际可用空间会略小于计算值。

</aside>

## GC 日志与排查思路

1. **先开日志**：JDK 9+ 使用 `-Xlog:gc*:file=gc.log:time,uptime,level,tags`；JDK 8 使用 `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log`。
2. **看趋势**：用 `jstat -gcutil <pid> 1000` 观察 YGC / FGC 频率与耗时、老年代占用变化；用 `jcmd <pid> GC.heap_info` 查看堆分布。
3. **定位问题**：
   - Young GC 频繁：新生代过小或短命对象过多，可增大新生代或降低对象分配速率；
   - Full GC 频繁：老年代不足、内存泄漏，或 CMS 出现 `concurrent mode failure` / 晋升失败；
   - 单次停顿过长：检查大对象、收集器选型与堆规模。
4. **必要时做堆分析**：OOM 转储后用 MAT / GCViewer 等工具定位支配对象与引用链。

## OOM 常见类型

| **OOM 类型** | **错误信息关键词** | **常见原因** |
| --- | --- | --- |
| 堆内存溢出（最常见） | `java.lang.OutOfMemoryError: Java heap space` | 对象创建过多且未被 GC 回收（内存泄漏或大对象） |
| 方法区 / 元空间溢出 | `java.lang.OutOfMemoryError: Metaspace` | 类加载过多（如动态生成类、依赖冲突导致类重复加载） |
| 直接内存溢出 | `java.lang.OutOfMemoryError: Direct buffer memory` | NIO 直接内存使用超限（未及时释放） |
| 栈内存溢出 | `java.lang.StackOverflowError` | 递归调用过深或线程栈设置过小 |

## OOM 排查

不同区域的 OOM 对应不同参数与排查方向：

- **Java heap space**：检查是否存在内存泄漏（对象被长期引用无法回收）或一次性加载过多大对象；结合堆转储分析引用链。
- **Metaspace**：检查是否动态生成大量类、依赖冲突导致类重复加载，必要时调整 `-XX:MaxMetaspaceSize`。
- **Direct buffer memory**：检查 NIO `DirectByteBuffer` 是否未及时释放，调整 `-XX:MaxDirectMemorySize`。
- **StackOverflowError**：检查递归深度与 `-Xss` 设置；线程数过多导致无法创建线程时，需减小线程栈或线程池规模。

## 参考资料

- Oracle《Java Platform, Standard Edition HotSpot Virtual Machine Garbage Collection Tuning Guide, Release 17》：<https://docs.oracle.com/en/java/javase/17/gctuning/>
- Oracle《java - Java application launcher》参数说明：<https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html>

## 延伸阅读

- [内存区域](jvm-memory-and-gc.md)：运行时区域划分
- [垃圾回收算法](gc-implementation-and-comparison.md)：GC 判定、回收算法与收集器实现
- [拓展篇：GC演化历史](gc-evolution-history.md)：从 Serial 到 ZGC 的演进脉络
- [对象与内存](object-and-memory.md)：对象布局与指针压缩
