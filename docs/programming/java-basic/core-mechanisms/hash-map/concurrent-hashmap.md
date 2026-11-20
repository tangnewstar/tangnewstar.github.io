# ConcurrentHashMap 线程安全

### **ConcurrentHashMap 的线程安全实现机制**

ConcurrentHashMap 是 Java 中线程安全的哈希表实现，相比 Hashtable（全表锁）具有更高的并发性能，其核心是通过**精细化的锁机制**和**无锁操作**结合实现线程安全。

### **一、JDK7 中的实现：分段锁（Segment）**

JDK7 中 ConcurrentHashMap 采用 "分段锁" 机制，核心设计如下：

1. **数据结构**：整体由一个`Segment`数组组成，每个`Segment`本质是一个小的 HashMap（包含 HashEntry 数组）
2. **锁机制**：每个 Segment 独立加锁（ReentrantLock），不同 Segment 上的操作互不干扰
3. **并发度**：默认 Segment 数组长度为 16，理论上支持 16 个线程同时写入（每个线程操作不同 Segment）
4. **操作逻辑**：
    - 计算 key 的哈希值，先定位到具体 Segment
    - 对该 Segment 加锁后执行 put/remove 等操作
    - 解锁后返回结果

**优点**：避免全表锁，提高并发写入效率**缺点**：Segment 数组一旦初始化大小固定，无法动态调整；内存占用较大

### **二、JDK8 中的演进：CAS + synchronized + 红黑树**

JDK8 彻底重构了 ConcurrentHashMap 的实现，废弃了 Segment，采用更高效的机制：

1. **数据结构**：与 HashMap 类似，采用数组 + 链表 + 红黑树，但节点为`Node`（val 和 next 用 volatile 修饰）
2. **核心同步机制**：
    - **synchronized 锁**：对链表头节点或红黑树根节点加锁，粒度更细（锁单个哈希桶而非分段）
    - **CAS 无锁操作**：对于数组初始化、扩容、插入头节点等场景，使用 CAS（Compare-And-Swap）保证原子性
    - **volatile 可见性**：数组`table`和节点`val/next`用 volatile 修饰，确保多线程间的内存可见性
3. **关键操作实现**：
    - **put 操作**：
        1. 计算哈希值定位数组索引
        2. 若数组未初始化，用 CAS 初始化
        3. 若当前桶为空，用 CAS 插入新节点
        4. 若桶不为空，对桶头节点加 synchronized 锁后插入元素
        5. 链表长度达标时转为红黑树
    - **扩容操作**：支持多线程协同扩容（通过`sizeCtl`控制状态）
    - **get 操作**：全程无锁，依赖 volatile 保证节点值可见性

**优点**：锁粒度更细（单个哈希桶）、支持动态扩容、内存效率更高、读写并发性能更好

### **三、分段锁到 CAS+synchronized 的演进意义**

1. **锁粒度优化**：从 "段级锁" 细化到 "桶级锁"，减少锁竞争范围
2. **无锁化提升**：CAS 避免了部分场景的锁开销，适合读多写少的场景
3. **内存效率提升**：去掉 Segment 层级，减少内存占用
4. **并发扩展性更好**：不再受固定 Segment 数量限制，理论并发度更高
5. **与 HashMap 结构对齐**：简化实现，同时继承红黑树带来的查询性能优化

JDK8 的 ConcurrentHashMap 通过结合`synchronized`（JVM 对其有持续优化）和 CAS，在保证线程安全的同时，实现了接近 HashMap 的性能，是高并发场景下的首选哈希表实现。