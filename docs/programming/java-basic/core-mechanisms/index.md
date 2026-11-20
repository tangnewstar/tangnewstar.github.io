# Java基础 → Java核心机制

**JVM内存与GC**

[JVM内存布局及垃圾回收机制](jvm-memory-gc.md)

[JVM内存分布](jvm-memory-layout.md)

[GC](gc.md)

[CMS & G1](cms-g1.md)

**基本规范**

[hashCode and equals](hashcode-equals.md)

**数据结构**

[Java数据结构 → HashMap](hash-map/index.md)

**锁与同步机制**

<aside>
💡

synchronized和Reentrant Lock的区别？什么是AQS？

</aside>

synchronized和ReentrantLock都是Java中的同步机制。不同在于：

- 实现方式：synchronized是依赖操作系统的互斥锁来实现的。RentrantLock是基于AQS的同步机制来实现的。
- 获取和释放锁：synchronized自动获取和释放锁，RentrantLock需要手动释放。
- 功能灵活：synchronized只有基础功能，获取执行释放。RentrantLock支持可中断锁，锁超时，公平锁，唤醒条件Condition指定线程。
- 性能：JDK6之前synchronized依赖Mutex性能较差，JDK6及之后对锁进行了优化。（根据竞争激烈程度进行锁升级 偏向锁 → 轻量级锁（存在多线程CAS竞争，失败自旋）→ 重量级锁（自旋过多次数））

<aside>
💡

什么时候用synchronized？什么时候用ReentrantLock？

- 优先用 synchronized：场景简单（如普通同步方法/块）、追求代码简洁（无需手动管理锁）、避免人为操作失误（如忘记释放锁）。
- 用 ReentrantLock：需要灵活功能（如超时锁、可中断锁、公平锁、条件变量），或需要在复杂逻辑中精确控制锁的行为。
</aside>

[Lock](lock.md)