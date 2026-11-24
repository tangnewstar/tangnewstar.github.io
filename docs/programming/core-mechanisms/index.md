# Java 核心机制

JVM 运行时、异常体系、Java 规范、反射、动态代理与 SPI。

## JVM 运行时

- [内存区域](jvm-memory-and-gc.md)：堆 / 栈 / 方法区 / 程序计数器 / 直接内存的区域划分
- [对象与内存](object-and-memory.md)：对象头 / 指针压缩 / 内存布局 / 对象访问定位
- [垃圾回收算法](gc-implementation-and-comparison.md)：GC 判定、回收算法与常见收集器对比
- [JVM参数调优](jvm-tuning.md)：GC 关键指标、参数、日志与 OOM 排查
- [类加载机制](class-loading.md)：加载→验证→准备→解析→初始化、双亲委派与打破场景
- [字节码与执行引擎](bytecode-and-execution.md)：javap / JIT / 逃逸分析 / 栈上分配
- [拓展篇：GC演化历史](gc-evolution-history.md)：从 Serial 到 ZGC 的演进脉络与设计取舍

## 异常体系

- [异常体系](exceptions.md)：Throwable 层次、受检与非受检异常、try-with-resources 与异常设计原则

## Java 规范

- [Java 规范](java-specification.md)：equals 与 hashCode、toString / compareTo、clone、Serializable 与编码规约

## 反射

- [反射](reflection.md)：Class / Method / Field 与性能、安全限制

## 动态代理

- [动态代理](dynamic-proxy.md)：JDK 动态代理与 CGLIB

## SPI 机制

- [SPI 机制](spi.md)：ServiceLoader、双亲委派的打破与典型应用
