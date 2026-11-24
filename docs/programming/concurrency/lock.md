# 锁与同步

从 synchronized / ReentrantLock 到 AQS 的锁与同步机制。

## synchronized 与 ReentrantLock

synchronized 和 ReentrantLock 都是 Java 中实现线程同步的机制，但设计和功能有明显差异，核心区别如下：

1. 锁的实现方式
    - synchronized：
    是 JVM 内置关键字，底层通过 JVM 字节码指令（monitorenter/monitorexit）实现，依赖操作系统的互斥锁（Mutex）机制。
    - ReentrantLock：
    是 Java 类库（java.util.concurrent.locks）中的类，基于 AQS（AbstractQueuedSynchronizer）框架实现，通过代码逻辑控制锁的获取和释放（见本文「AQS 抽象队列同步器」一节）。
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


## AQS 抽象队列同步器

AQS（AbstractQueuedSynchronizer，抽象队列同步器）是 Java 并发包（java.util.concurrent）的核心基础类，是实现各种锁（如 ReentrantLock）和同步工具（如 CountDownLatch、Semaphore）的“底层骨架”。

核心作用

它的核心作用**是解决多线程竞争共享资源时的同步问题**，通过“状态管理 + 等待队列”的机制，统一处理线程的竞争、等待和唤醒逻辑。

AQS 封装了“同步状态管理”和“线程排队等待”的通用逻辑，让开发者无需重复实现这些复杂细节，只需专注于具体同步规则（如“锁是否可获取”“资源是否足够”）。

核心设计

AQS 的核心是 “状态变量 + 等待队列”：

1. 同步状态（State）
    1. 用一个 volatile int 变量 state 存储同步状态（如锁的持有计数、可用资源数量）。
    2. 提供 getState()、setState()、compareAndSetState()（CAS 原子操作）方法操作状态，确保线程安全。
    

示例：

- ReentrantLock 中，state=0 表示锁未被持有，state>0 表示被持有（数值为重入次数
- Semaphore 中，state 表示可用许可证数量。

1. 等待队列（CLH 队列）
    1. 当线程获取同步状态失败时，AQS 会将线程封装成 节点（Node），加入一个双向链表（等待队列），让线程阻塞等待。
    2. 队列遵循“FIFO（先进先出）”原则，确保线程按顺序等待资源。
    3. 当持有资源的线程释放状态后，会唤醒队列头部的线程，让其重新尝试获取状态。
    

使用方式（开发者视角）

AQS 是抽象类，需通过 子类继承 实现，核心是重写两个模板方法：

```
•	tryAcquire(int arg)：尝试获取同步状态（如“锁是否能被当前线程获取”）。
•	tryRelease(int arg)：尝试释放同步状态（如“锁是否能被当前线程释放”）。
```

AQS 会在内部调用这些方法，完成“获取→排队→释放→唤醒”的完整流程，子类无需关心队列和阻塞的细节。

1. **独占 / 共享模式**：
    - **独占模式**：资源只能被一个线程持有（如 `ReentrantLock`）
    - **共享模式**：资源可被多个线程同时持有（如 `CountDownLatch`、`Semaphore`）
2. **模板方法**：
    - AQS 定义了获取 / 释放资源的模板方法（如 `acquire()`、`release()`）
    - 子类需重写核心方法（如 `tryAcquire()`、`tryRelease()`）自定义同步逻辑

### **AQS 工作流程（以独占锁为例）：**

1. 线程尝试通过 `tryAcquire()` 获取资源（自定义逻辑）
2. 成功则直接返回；失败则将线程封装为 Node 加入等待队列
3. 线程进入阻塞状态（通过 `LockSupport.park()`）
4. 持有资源的线程释放资源时，唤醒队列中的后继线程
5. 被唤醒的线程重新尝试获取资源，重复步骤 1

AQS 通过标准化的队列管理和状态控制，简化了并发工具的实现，是 Java 并发体系的基础骨架。


## AQS 代码注释

```jsx
java. util. concurrent. locks
public abstract class AbstractQueuedSynchronizer
extends java. util. concurrent. locks. AbstractOwnableSynchronizer
implements java. io. Serializable
```

Provides a framework for implementing blocking locks and related synchronizers (semaphores, events, etc) that rely on first-in-first-out (FIFO) wait queues. This class is designed to be a useful basis for most kinds of synchronizers that rely on a single atomic int value to represent state. 

Subclasses must define the protected methods that change this state, and which define what that state means in terms of this object being acquired or released. Given these, the other methods in this class carry out all queuing and blocking mechanics. Subclasses can maintain other state fields, but only the atomically updated int value manipulated using methods getState, setState and compareAndSetState is tracked with respect to synchronization.

Subclasses should be defined as non-public internal helper classes that are used to implement the synchronization properties of their enclosing class. Class AbstractQueuedSynchronizer does not implement any synchronization interface. Instead it defines methods such as acquireInterruptibly that can be invoked as appropriate by concrete locks and related synchronizers to implement their public methods.

This class supports either or both a default exclusive mode and a shared mode. When acquired in exclusive mode, attempted acquires by other threads cannot succeed. Shared mode acquires by multiple threads may (but need not) succeed. This class does not "understand" these differences except in the mechanical sense that when a shared mode acquire succeeds, the next waiting thread (if one exists) must also determine whether it can acquire as well. Threads waiting in the different modes share the same FIFO queue. Usually, implementation subclasses support only one of these modes, but both can come into play for example in a ReadWriteLock. Subclasses that support only exclusive or only shared modes need not define the methods supporting the unused mode.
This class defines a nested AbstractQueuedSynchronizer. ConditionObject class that can be used as a Condition implementation by subclasses supporting exclusive mode for which method isHeldExclusively reports whether synchronization is exclusively held with respect to the current thread, method release invoked with the current getState value fully releases this object, and acquire, given this saved state value, eventually restores this object to its previous acquired state. No AbstractQueuedSynchronizer method otherwise creates such a condition, so if this constraint cannot be met, do not use it. The behavior of AbstractQueuedSynchronizer. ConditionObject depends of course on the semantics of its synchronizer implementation.
This class provides inspection, instrumentation, and monitoring methods for the internal queue, as well as similar methods for condition objects. These can be exported as desired into classes using an AbstractQueuedSynchronizer for their synchronization mechanics.
Serialization of this class stores only the underlying atomic integer maintaining state, so deserialized objects have empty thread queues. Typical subclasses requiring serializability will define a readObject method that restores this to a known initial state upon deserialization.

**Usage**
To use this class as the basis of a synchronizer, redefine the following methods, as applicable, by inspecting and/ or modifying the synchronization state using getState, setState and/ or — —- compareAndSetState:

- `tryAcquire`
- `tryRelease`
- `tryAcquireShared`
- `tryReleaseShared`
- `isHeldExclusively`

Each of these methods by default throws `UnsupportedOperationException`. Implementations of these methods must be internally thread-safe, and should in general be short and not block. Defining these methods is the only supported means of using this class. All other methods are declared final because they cannot be independently varied.
You may also find the inherited methods from AbstractOwnableSynchronizer useful to keep track of the thread owning an exclusive synchronizer. You are encouraged to use them -- this enables monitoring and diagnostic tools to assist users in determining which threads hold locks.
Even though this class is based on an internal FIFO queue, it does not automatically enforce FIFO acquisition policies. The core of exclusive synchronization takes the form:

```jsx
Acquire:
while (!tryAcquire(arg)) {
enqueue thread if it is not already queued;
possibly block current thread;
}

Release:
if (tryRelease(arg))
unblock the first queued thread;
```

(Shared mode is similar but may involve cascading signals.)
Because checks in acquire are invoked before enqueuing, a newly acquiring thread may barge ahead of others that are blocked and queued. However, you can, if desired, define tryAcquire and/ or tryAcquireShared to disable barging by internally invoking one or more of the inspection methods, thereby providing a fair FIFO acquisition order. In particular, most fair synchronizers can define tryAcquire to return false if hasQueuedPredecessors (a method specifically designed to be used by fair synchronizers) returns true. Other variations are possible.
Throughput and scalability are generally highest for the default barging (also known as greedy, renouncement, and convoy-avoidance) strategy. While this is not guaranteed to be fair or starvation-free, earlier queued threads are allowed to recontend before later queued threads, and each recontention has an unbiased chance to succeed against incoming threads. Also, while acquires do not "spin" in the usual sense, they may perform multiple invocations of tryAcquire interspersed with other computations before blocking. This gives most of the benefits of spins when exclusive synchronization is only briefly held, without most of the liabilities when it isn't. If so desired, you can augment this by preceding calls to acquire methods with "fast-path" checks, possibly prechecking hasContended and/ or hasQueuedThreads to only do so if the synchronizer is likely not to be contended.
This class provides an efficient and scalable basis for synchronization in part by specializing its range of use to synchronizers that can rely on int state, acquire, and release parameters, and an internal FIFO wait queue. When this does not suffice, you can build synchronizers from a lower level using atomic classes, your own custom java. util. Queue classes, and LockSupport blocking support.
Usage Examples
Here is a non-reentrant mutual exclusion lock class that uses the value zero to represent the unlocked state, and one to represent the locked state. While a non-reentrant lock does not strictly require recording of the current owner thread, this class does so anyway to make usage easier to monitor. It also supports conditions and exposes one of the instrumentation methods:

```jsx

class Mutex implements Lock, java.io.Serializable {
    // Our internal helper class
    private static class Sync extends AbstractQueuedSynchronizer {
        // Reports whether in locked state
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        // Acquires the lock if state is zero
        public boolean tryAcquire(int acquires) {
            assert acquires == 1; // Otherwise unused
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        // Releases the lock by setting state to zero
        protected boolean tryRelease(int releases) {
            assert releases == 1; // Otherwise unused
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        // Provides a Condition
        Condition newCondition() {
            return new ConditionObject();
        }

        // Deserializes properly
        private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

    // The sync object does all the hard work. We just forward to it.
    private final Sync sync = new Sync();

    public void lock() { sync.acquire(1); }

    public boolean tryLock() { return sync.tryAcquire(1); }

    public void unlock() { sync.release(1); }

    public Condition newCondition() { return sync.newCondition(); }

    public boolean isLocked() { return sync.isHeldExclusively(); }

    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }

    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
}
```

Here is a latch class that is like a CountDownLatch except that it only requires a single signal to fire. Because a latch is non-exclusive, it uses the shared acquire and release methods.

```jsx
import java.util.concurrent.locks.AbstractQueuedSynchronizer;

class BooleanLatch {
    private static class Sync extends AbstractQueuedSynchronizer {
        boolean isSignalled() {
            return getState() != 0;
        }

        protected int tryAcquireShared(int ignore) {
            return isSignalled() ? 1 : -1;
        }

        protected boolean tryReleaseShared(int ignore) {
            setState(1);
            return true;
        }
    }

    private final Sync sync = new Sync();

    public boolean isSignalled() {
        return sync.isSignalled();
    }

    public void signal() {
        sync.releaseShared(1);
    }

    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
}
```
