# HashMap

HashMap 的底层数据结构、线程安全性与关键参数。

<aside>
💡

HashMap底层数据结构？为什么不是线程安全的？链表转红黑树的阈值？

</aside>

HashMap底层是 数组 + 链表 / 红黑树

由于HashMap没有同步机制，所以是线程不安全的。主要体现在：

- 扩容时节点会丢失或循环引用：多线程同时扩容没发保证链表节点的有序
- 并发修改时设置的值会被覆盖（例如hash值相同但是key不同，会先后设置同一个初始节点）
- 迭代时HashMap被修改，会抛出ConcurrentModificationException

链表转红黑树的阈值：`TREEIFY_THRESHOLD = 8` `MIN_TREEIFY_CAPACITY = 64`

- 当链表长度到8且数组容量大于64时会进行链表转红黑树，提高查询效率
- 如果容量小于64则扩容实现节点分布均匀

[HashMap源码深度分析](hashmap-source-analysis.md)

[HashMap底层原理](hashmap.md)

<aside>
💡

ConcurrentHashMap如何保证线程安全？分段锁和CAS在JDK8中的演进是什么？

</aside>

ConcurrentHashMap是通过JDK7中的Segment分段锁和JDK8中的CAS+synchronized机制实现的

- Segment分段锁：将整个哈希表用Segment数组来表示（数组长度初始化固定，默认16），每个Segment持有一个ReentrantLock。写入时定位到具体的Segment，获取锁写入完再释放锁。

优劣势：可以避免全表锁，Segment数组无法动态调整，占用内存

- CAS + synchronized：
    - volatile：table数组和链表节点的val和next都由volatile修饰，确多线程下的可见性
    - CAS：table数组初始化，扩容，插入头节点时，使用compareAndSwap保证原子操作
    - synchronized：对链表头节点加锁，锁粒度更细基于哈希桶
    
    写入数据时，如果链表的头节点不存在则使用CAS操作创建，如果头节点存在则使用synchronized来添加或修改元素
    
    读取操作依赖volatile实现可见性，无锁
    

[ConcurrentHashMap 线程安全](concurrent-hashmap.md)


## 底层实现原理

### **HashMap 底层实现原理**

HashMap 是 Java 中常用的键值对存储结构，其底层实现在 JDK8 及之后进行了优化，主要特点如下：

1. **数据结构**：采用数组（哈希表）+ 链表 + 红黑树的组合结构
    - 数组（Node [] table）：作为哈希桶，每个元素是链表或红黑树的头节点
    - 链表：解决哈希冲突（不同 key 计算出相同哈希值），相同哈希桶的元素以链表形式存储
    - 红黑树：当链表长度超过阈值时转换为红黑树，提高查询效率（从 O (n) 优化到 O (log n)）
2. **核心原理**：
    - 通过`key.hashCode()`计算哈希值，经过扰动函数处理后得到数组索引
    - 若索引位置为空，直接存放键值对
    - 若发生哈希冲突，判断 key 是否相同：
        - 相同则覆盖 value
        - 不同则插入链表尾部（JDK8 之前是头部）
    - 当数组容量不足（负载因子默认 0.75）时，进行扩容（容量翻倍）
3. **关键参数**：
    - 初始容量：默认 16（必须是 2 的幂）
    - 负载因子：默认 0.75（扩容阈值 = 容量 × 负载因子）
    - 扩容机制：当元素数量超过阈值时，容量翻倍并重新哈希（rehash）

### **为什么 HashMap 线程不安全？**

HashMap 的线程不安全主要体现在多线程并发操作时：

1. **扩容期间的数据丢失**：
    - 多线程同时触发扩容时，链表迁移过程中可能出现循环引用或节点丢失（JDK7 及之前的头插法问题）
2. **数据覆盖**：
    - 多线程同时执行 put 操作时，可能导致后插入的元素覆盖先插入的元素
    - 例如：线程 A 和线程 B 同时判断某个位置为空，先后执行插入操作
3. **迭代期间的 ConcurrentModificationException**：
    - 迭代过程中若其他线程修改了 HashMap 结构（增删元素），会触发快速失败机制

### **JDK8 中链表转红黑树的阈值**

在 JDK8 中，HashMap 的链表转为红黑树需满足两个条件：

1. **链表长度阈值**：当链表中元素数量达到**8**时，可能触发转换
2. **数组容量阈值**：同时要求数组容量不小于**64**（若容量不足会先扩容而非转树）

红黑树转回链表的阈值是**6**（当删除元素后树节点数量减少到 6 时），这样设计是为了避免频繁在链表和树之间转换。

如果需要线程安全的哈希表实现，可以使用`ConcurrentHashMap`（JDK1.5+）或`Collections.synchronizedMap()`。
