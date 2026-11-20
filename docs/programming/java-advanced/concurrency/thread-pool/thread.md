# Thread

### Lifecycle

- New：线程对象创建`new Thread()`，尚未调用`start()`
- RUNNABLE：调用了`start()`，处于就绪或运行中状态
    - 就绪：等待CPU调度（操作系统分配时间片）
    - 运行中：线程正在执行`run()`
- BLOCKED：竞争同步锁（synchronized）失败，等待获取锁
- WAITING：主动调用 `wait()` `join()` 或`LockSupport.park()` 等方法，进入无限等待。需要其他线程显示唤醒（`notify() notifyAll() unpark()`）
- TIMED_WAITING：调用带超时参数的等待（`sleep(long)` `wait(long)` `join(long)` `LockSupport.parkNanos()`），进入限时等待。超时后自动恢复运行。
- TERMINATED：线程的`run()` 执行完毕，或异常终止。

### interrupt

早期 Java 提供过 `Thread.stop()` 方法强制终止线程，但由于以下缺陷已被废弃：

- **不安全**：`stop()` 会直接终止线程，可能导致锁未释放、资源未清理（如文件未关闭）、事务未提交等。
- **不可控**：线程无法优雅地保存中间状态或执行必要的收尾操作。

因此，Java 推荐使用**协作式中断**（`interrupt`），让线程自主决定是否响应中断并退出。

每个线程对象（`Thread`）内部维护一个**中断标志位**（`interrupt status`），初始为 `false`。调用 `interrupt()` 方法会将此标志位设为 `true`（若线程被阻塞，可能直接触发异常）。线程通过检查该标志位或捕获特定异常来判断是否需要终止。

三种场景

- **RUNNABLE**：当线程正在执行 `run()` 方法中的逻辑时，调用 `interrupt()` 会**设置其中断标志位为 `true`**，但不会强制终止线程。
- **WAITING（如 `sleep()`、`wait()`、`join()`）**：

当线程因调用以下方法进入阻塞状态时，调用 `interrupt()` 会触发以下行为：**清除中断标志位**（设为 `false`）。**抛出 `InterruptedException`**（需显式捕获或声明抛出）。

- BLOCKED：当线程因竞争 `synchronized` 锁失败而进入 `BLOCKED` 状态时，调用 `interrupt()` **不会直接唤醒线程**，但会设置其中断标志位。线程需在获取锁后检查标志位，决定是否退出。

```
1. 除非当前线程正在中断自身，否则会调用该线程的 checkAccess 方法：
    •这意味着如果一个线程尝试中断另一个线程，而它没有权限这样做，那么 checkAccess 方法可能会抛出 SecurityException。不过，一个线程总是可以中断它自己。
2.如果线程被阻塞在 Object 类的 wait(), wait(long), 或 wait(long, int) 方法中，或者被阻塞在 Thread 类的 join(), join(long), join(long, int), sleep(long), 或 sleep(long, int) 方法中：
    •在这种情况下，线程的中断状态会被清除，并且它会接收到一个 InterruptedException 异常。
    •例如，如果线程在 Thread.sleep(1000) 中被中断，那么它会立即醒来并抛出 InterruptedException。
3.如果线程被阻塞在一个 InterruptibleChannel 的 I/O 操作上：
    •那么这个通道将会被关闭，线程的中断状态会被设置，并且线程会接收到一个 ClosedByInterruptException。
    •这种情况通常发生在 NIO（非阻塞 I/O）操作中，当一个线程在读写一个可中断的通道时被中断。
4.如果线程被阻塞在一个 Selector 上：
    •线程的中断状态会被设置，并且它会立即从选择操作中返回，就像调用了 selector.wakeup() 方法一样。
    •这通常发生在 NIO 选择器上，用于多路复用 I/O 操作。
5.如果以上条件都不满足：
    •那么线程的中断状态将被设置。
    •例如，如果线程正在执行某个计算任务而不是被阻塞，那么调用 interrupt() 只是简单地设置中断标志位，而不会立即抛出异常。
6.中断一个未存活的线程可能没有任何效果：
    •如果线程已经终止（即它的状态不是 RUNNABLE），那么调用 interrupt() 不会产生任何影响。
```