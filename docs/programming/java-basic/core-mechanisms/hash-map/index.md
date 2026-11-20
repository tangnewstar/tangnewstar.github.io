# Java数据结构 → HashMap

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

[HashMap底层原理](hashmap-underlying.md)

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