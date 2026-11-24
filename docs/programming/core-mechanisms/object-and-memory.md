# 对象与内存

对象在堆上如何布局、如何定位，以及 64 位虚拟机的指针压缩为何能省下大量引用内存。

<aside>
💡

本文图示为外链社区图片，版权归原作者，加载依赖外部站点；来源已在各图下注明。

</aside>

## 对象创建过程

1. **类加载检查**：`new` 指令先检查常量池中能否定位到该类的符号引用，以及该类是否已被加载、解析和初始化；否则先执行类加载。
2. **分配内存**：对象所需大小在类加载完成后即可确定。根据堆是否规整选择分配方式：
   - **指针碰撞（Bump the Pointer）**：Serial、Parallel 等带整理能力的收集器，已用/空闲内存边界清晰，移动指针即可分配。
   - **空闲列表（Free List）**：CMS 等基于标记-清除的收集器，需维护可用内存块列表。
   - 并发分配通过 **CAS + 失败重试** 保证原子性，或通过 **TLAB（Thread Local Allocation Buffer）** 为每个线程预先划分一小块 Eden 区，线程内分配无需同步。
3. **初始化零值**：实例字段设为 `0` / `false` / `null`，保证即使未显式赋值也能安全访问。
4. **设置对象头**：写入 Mark Word（哈希码、分代年龄、锁标志等）与 Klass Pointer（类型元数据指针）。
5. **执行 `<init>`**：按源码顺序执行字段赋值、实例代码块与构造方法，此时对象才真正可用。

## 对象内存布局

对象在堆中的存储可分为三块：

| 区域 | 内容 |
| --- | --- |
| **对象头（Header）** | `Mark Word`（哈希码、GC 分代年龄、锁状态、偏向线程 ID 等）+ `Klass Pointer`（指向方法区中的类元数据）；数组对象还额外包含**数组长度** |
| **实例数据（Instance Data）** | 父类字段在前、子类字段在后；相同宽度字段常被排在一起；`long` / `double` 占 8 字节，引用占 4 或 8 字节（取决于是否开启指针压缩） |
| **对齐填充（Padding）** | HotSpot 要求对象起始地址按 **8 字节**对齐，不足时补齐，便于指针压缩与内存访问 |

对象在堆中的连续布局（64 位、开启指针压缩）：

![Java 对象内存布局示意图](https://cdn.jsdelivr.net/gh/doocs/jvm@main/images/object-memory-layout.png)

> 图源：[doocs/jvm · 对象的内存布局](https://github.com/doocs/jvm/blob/main/docs/02-hotspot-jvm-object.md)，CC BY-SA 4.0。图中为结构示意（按 32 位视角展示各 4 字节字段）；64 位 HotSpot 下 Mark Word 为 8 字节、Klass Pointer 在指针压缩后为 4 字节，具体见下文与 [JOL 示例](#对象大小估算)。

<aside>
💡

Mark Word 在 32 位虚拟机固定 4 字节，64 位固定 8 字节；锁升级（偏向锁 → 轻量级锁 → 重量级锁）的信息就记录在这里，而不是额外的锁对象。

</aside>

## 指针压缩（Compressed Oops）

- **背景**：64 位虚拟机中每个对象引用占 8 字节，堆中大量引用会带来显著内存开销。
- **原理**：启用 `-XX:+UseCompressedOops` 后，引用被压缩为 **4 字节偏移量**；访问时以基址 + 偏移 × 8 还原真实地址。因为对象按 8 字节对齐，低 3 位恒为 0，可用来放大寻址范围。
- **效果**：引用与 Klass Pointer 从 8 字节降到 4 字节，通常可节省可观的堆内存；64 位 JVM 在堆小于约 32GB 时默认开启。
- **相关参数**：
  - `-XX:+UseCompressedOops`：压缩普通对象指针（默认开启，堆过大时自动失效）。
  - `-XX:+UseCompressedClassPointers`：压缩 Klass Pointer（默认开启，依赖 `UseCompressedOops`）。
  - 堆超过约 32GB 时压缩失效，引用回到 8 字节，这也是“堆从 31G 加到 35G 反而更慢”的常见原因；可通过 `-XX:ObjectAlignmentInBytes=16` 提升对齐粒度来扩大压缩上限。

压缩引用的还原过程：

![HotSpot 压缩指针位布局](https://oss.javaguide.cn/github/javaguide/java/jvm/hotspot-markOop.hpp..png)

> 图源同上；图中 `narrowOop:32`、`COOPs && normal object` 等行展示了压缩后的窄指针布局。

还原公式：

```text
真实地址 = 堆基地址 + 4 字节偏移量 × 8
```

## 对象大小估算

以 64 位 HotSpot（开启指针压缩）为例，可借助 JOL 观察真实布局：

```java
// 依赖：org.openjdk.jol:jol-core
System.out.println(ClassLayout.parseInstance(new Object()).toPrintable());
// Object: 16 bytes
//   8 bytes (Mark Word) + 4 bytes (Klass Pointer) + 4 bytes (alignment padding)

class Point {
    int x;      // 4 bytes
    int y;      // 4 bytes
    Object ref; // 4 bytes（压缩后）
}
// 12 bytes 对象头 + 4 + 4 + 4 字段 = 24 bytes（已是 8 的倍数）
```

估算公式：**对象头 + 实例字段（按 8 字节对齐排列） + 对齐填充**，数组对象再加 4 字节长度字段。

## 对象的访问定位

- **句柄访问**：堆中维护句柄池，引用指向句柄，句柄再分别指向对象实例数据与类型数据。优点是对象移动时只需改句柄，引用本身稳定；缺点是多一次指针跳转。
- **直接指针（HotSpot 采用）**：引用直接指向对象实例，对象头中的 Klass Pointer 指向类型数据。优点是访问速度更快，但对象移动时需要修正所有引用。

两种定位方式的引用链路：

句柄访问：

![句柄访问定位](https://oss.javaguide.cn/github/javaguide/java/jvm/access-location-of-object-handle.png)

直接指针（HotSpot 采用）：

![直接指针定位](https://oss.javaguide.cn/github/javaguide/java/jvm/access-location-of-object-handle-direct-pointer.png)

> 图源：[JavaGuide · Java内存区域详解](https://javaguide.cn/java/jvm/memory-area.html)（社区资料，版权归原作者）。

## 延伸阅读

- [内存区域](jvm-memory-and-gc.md)
- [字节码与执行引擎](bytecode-and-execution.md)：逃逸分析、标量替换与栈上分配
