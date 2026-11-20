# Lock

synchronized 和 ReentrantLock 都是 Java 中实现线程同步的机制，但设计和功能有明显差异，核心区别如下：

1. 锁的实现方式
    - synchronized：
    是 JVM 内置关键字，底层通过 JVM 字节码指令（monitorenter/monitorexit）实现，依赖操作系统的互斥锁（Mutex）机制。
    - ReentrantLock：
    是 Java 类库（java.util.concurrent.locks）中的类，基于 AQS [AQS](../../java-advanced/concurrency/lock-sync/lock/aqs.md) （AbstractQueuedSynchronizer）框架实现，通过代码逻辑控制锁的获取和释放。
2. 锁的获取与释放
    - synchronized：
        
        **自动获取和释放**：进入同步块时自动加锁，退出同步块（包括正常结束、异常退出）时自动释放锁，无需手动操作，避免“忘记释放锁”导致死锁。
        
    - ReentrantLock：
        
        **手动获取和释放**：必须通过 lock() 方法加锁，unlock() 方法释放锁，且 unlock() 需放在 finally 块中（否则异常时可能导致锁无法释放）。
        
3. 功能灵活性
    - synchronized：
    功能简单，仅支持“获取锁→执行→释放锁”的基础逻辑，不支持额外控制。
    - ReentrantLock：
    提供更多灵活功能：
        - 可中断锁：支持 `lockInterruptibly()`，在等待锁时可被其他线程中断（避免无限等待）。
        - 超时获取锁：支持 `tryLock(long timeout, TimeUnit unit)`，超时未获取到锁则返回 false（避免死锁）。
        - 公平锁选择：可通过构造函数 new ReentrantLock(true) 创建公平锁（按线程等待顺序分配锁），默认是非公平锁；而 synchronized 只能是非公平锁。
        - 条件变量（Condition）：通过 `newCondition()` 获取 Condition 对象，支持更精细的线程唤醒（如唤醒特定线程），而 synchronized 只能通过 `wait()/notify()/notifyAll()` 唤醒，且 `notifyAll()` 会唤醒所有等待线程。
4. 性能差异
    - 早期（JDK 5/6 前）：synchronized 性能较差（因为依赖操作系统互斥锁，切换成本高），ReentrantLock 性能更优。
    - 后期（JDK 6 及以后）：JVM 对 synchronized 做了大量优化（如偏向锁、轻量级锁、锁消除、锁粗化），两者性能差异极小；在简单同步场景下，synchronized 甚至更优（因为是 JVM 内置优化）。

总结：如何选择？

```
•	优先用 synchronized：场景简单（如普通同步方法/块）、追求代码简洁（无需手动管理锁）、避免人为操作失误（如忘记释放锁）。

•	用 ReentrantLock：需要灵活功能（如超时锁、可中断锁、公平锁、条件变量），或需要在复杂逻辑中精确控制锁的行为。

```

两者核心都是“保证线程安全”，但 synchronized 更简单易用，ReentrantLock 更灵活可控。

<aside>
💡

synchronized 锁定的必须是对象吗

</aside>

是的。synchronized通过对象监视器Monitor实现锁机制。无论是实例方法（实例对象）静态方法（class对象）还是代码块（自定义对象）

synchronized性能优化

通过对象头的MarkWord存储锁状态

根据线程访问该同步块的激烈程度，

- 分别有偏向锁（线程id，单个线程访问）
- 轻量级锁（持有锁线程栈中的锁记录，多线程cas竞争，失败则自旋等待）
- 自旋次数过多会升级为重量级锁（mutex，失败线程会挂起进入阻塞队列，释放锁会通知等待队列的线程）

**锁撤销**：偏向锁遇到竞争时，偏向锁失效，按轻量级锁的逻辑处理。如果持有偏向锁的线程已退出，则重偏向给T2，如果还持有就彻底撤销偏向状态。