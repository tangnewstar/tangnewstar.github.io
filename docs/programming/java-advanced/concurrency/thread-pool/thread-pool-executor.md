# ThreadPoolExecutor

class description：

The main pool control state, ctl, is an atomic integer packing two conceptual fields workerCount, indicating the effective number of threads runState, indicating whether running, shutting down etc 

In order to pack them into one int, we limit workerCount to (2^29)-1 (about 500 million) threads rather than (2^31)-1 (2 billion) otherwise representable. 

If this is ever an issue in the future, the variable can be changed to be an AtomicLong, and the shift/mask constants below adjusted. But until the need arises, this code is a bit faster and simpler using an int. 

The workerCount is the number of workers that have been permitted to start and not permitted to stop. 

The value may be transiently different from the actual number of live threads, for example when a ThreadFactory fails to create a thread when asked, and when exiting threads are still performing bookkeeping before terminating. The user-visible pool size is reported as the current size of the workers set. 

The runState provides the main lifecycle control, taking on values: 

- RUNNING: Accept new tasks and process queued tasks
- SHUTDOWN: Don't accept new tasks, but process queued tasks
- STOP: Don't accept new tasks, don't process queued tasks, and interrupt in-progress tasks
- TIDYING: All tasks have terminated, workerCount is zero, the thread transitioning to state TIDYING will run the terminated() hook method
- TERMINATED: terminated() has completed

The numerical order among these values matters, to allow ordered comparisons. 

The runState monotonically increases over time, but need not hit each state. 

The transitions are: 

- `RUNNING → SHUTDOWN` On invocation of `shutdown()`, perhaps implicitly in `finalize()`
- `(RUNNING or SHUTDOWN) → STOP` On invocation of `shutdownNow()`
- `SHUTDOWN → TIDYING` When both queue and pool are empty
- `STOP → TIDYING` When pool is empty
- `TIDYING → TERMINATED` When the `terminated()` hook method has completed Threads waiting in `awaitTermination()` will return when the state reaches TERMINATED. Detecting the transition from SHUTDOWN to TIDYING is less straightforward than you'd like because the queue may become empty after non-empty and vice versa during SHUTDOWN state, but we can only terminate if, after seeing that it is empty, we see that workerCount is 0 (which sometimes entails a recheck -- see below).

线程池控制状态的字段ctl，是AtomicInteger。打包了两个概念字段：

- workerCount：表明有效的线程数
- runState：表示是 running 还是shutdown等状态

为了将它们打包到一个 int 中，我们将 workerCount 限制为 (2^29)-1（约 5 亿）线程，而不是 (2^31)-1（20 亿），否则无法表示。

*这里Integer总共有32位，低29位用来存储workerCount，高3位用来存储runState。两个值互不影响且不需要占用锁资源去进行同步操作。*

*如果这样做将来有问题，可以将AtomicInteger改成AtomicLong，shift/mask常量也将响应调整，但在需求出现前，目前用int更简单也更快。*

workerCount 是已被允许启动且未被允许停止的工作线程数。此值可能与实际存活线程数暂时不同，例如当 ThreadFactory 在请求创建线程失败时，还有在退出线时程在终止前仍在bookkeeping 时。

用户可见的池大小为当前工作线程集的大小。

`runState`提供主要的生命周期控制，取值为：

- RUNNING：**接受新任务**并**处理排队任务**
- SHUTDOWN：**不接受新任务**，但**处理排队任务**
- STOP：**不接受新任务**，**不处理排队任务**，并**中断正在进行的任务**
- TIDYING：所有任务已终止，workerCount 为零，状态转换到 TIDYING 状态将运行 `terminated()` 钩子方法
- TERMINATED：`terminated()` 方法执行完成。

**runState 随时间单调递增，但不必达到每个状态**。

状态转换为：

`RUNNING → SHUTDOWN` 调用 `shutdown()` 时，也有可能在 `finalize()` 中隐式调用 

`(RUNNING or SHUTDOWN)  → STOP` 调用 `shutdownNow()` 时 

`SHUTDOWN → TIDYING` 当队列和池都为空时 

`STOP -> TIDYING` 当池为空时 

`TIDYING -> TERMINATED` 当 terminated() 钩子方法已完成时，在 `awaitTermination()` 中等待的线程将在状态到达 TERMINATED 时返回。

检测从 SHUTDOWN 到 TIDYING 的转换比预期的要复杂一些，因为队列在 SHUTDOWN 状态期间可能在非空后变为空，反之亦然，但只有在看到队列为空后，我们看到 workerCount 为 0 时才能终止

`ThreadPoolExecutor.execute`

```java
/*
 * Proceed in 3 steps:
 *
 * 1. If fewer than corePoolSize threads are running, try to
 * start a new thread with the given command as its first
 * task.  The call to addWorker atomically checks runState and
 * workerCount, and so prevents false alarms that would add
 * threads when it shouldn't, by returning false.
 *
 * 2. If a task can be successfully queued, then we still need
 * to double-check whether we should have added a thread
 * (because existing ones died since last checking) or that
 * the pool shut down since entry into this method. So we
 * recheck state and if necessary roll back the enqueuing if
 * stopped, or start a new thread if there are none.
 *
 * 3. If we cannot queue task, then we try to add a new
 * thread.  If it fails, we know we are shut down or saturated
 * and so reject the task.
 */
int c = ctl.get();
if (workerCountOf(c) < corePoolSize) {
		/*如果workerCount < corePoolSize
		添加worker创建线程执行任务command
		*/
    if (addWorker(command, true))
        return;
    c = ctl.get();
}
if (isRunning(c) && workQueue.offer(command)) {
		/*如果workCount >= corePoolSize
			且workQueue（阻塞队列）可以成功添加任务
		recheck线程池状态
			- 状态!RUNNING，移除并拒绝任务
			- wokerCount=0，添加woker创建线程	
		*/
    int recheck = ctl.get();
    if (! isRunning(recheck) && remove(command))
        reject(command);
    else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
}
else if (!addWorker(command, false))
		/*
				* scnario 1 *
		runState!=RUNNING,直接拒绝（addWorker返回false）
		
				*	scnario 2 *
		如果workerCount >= corePoolSize
			且 workQueue队列已满（workQueue.offer返回false）
				- workerCount < maximumPoolSize，添加worker创建线程执行任务
				- workerCount >= maximumPoolSize, 直接拒绝任务（addWorker返回false）			
		*/
    reject(command);
    
/*SUMMARY
1. runState!=RUNNING, addWorker(command, false)=false, 直接拒绝任务
2. workerCount < corePoolSize, 添加worker创建线程执行任务
3. workCount >= corePoolSize，阻塞队列未满添加任务到阻塞队列。成功了double check状态，线程池非运行中拒绝任务。workerCount=0，添加woker增加线程。
4. workerCount >= corePoolSize, 阻塞队列已满 & workerCount < maximumPoolSize，添加worker创建线程执行任务
5. workerCount >= corePoolSize, 阻塞队列已满 & workerCount >= maximumPoolSize，直接拒绝任务
*/
```

```java
/*
 * 分三步进行：
 *
 * 1. 如果运行的线程少于 corePoolSize，尝试使用给定命令作为其第一个任务启动新线程。
 *    调用 addWorker 会自动检查 runState 和 workerCount，从而防止意外增加线程，
 *    当不应该增加线程时返回 false。
 *
 * 2. 如果任务可以成功排队，那么我们仍然需要仔细检查是否应该添加线程（因为上次检查后现有线程已死亡）
 *    或者在进入此方法后池已关闭。因此我们重新检查状态，如有必要在停止时回滚排队，
 *    或者如果没有线程则启动新线程。
 *
 * 3. 如果我们无法将任务排队，则尝试添加新线程。如果失败，我们知道要么已关闭要么已饱和，因此拒绝任务。
 */
```

`*ThreadPoolExecutor.addWorker(Runnable firstTask, boolean core)*`

Checks if a new worker can be added with respect to current pool state and the given bound (either core or maximum). If so, the worker count is adjusted accordingly, and, if possible, a new worker is created and started, running firstTask as its first task. This method returns false if the pool is stopped or eligible to shut down. It also returns false if the thread factory fails to create a thread when asked. If the thread creation fails, either due to the thread factory returning null, or due to an exception (typically OutOfMemoryError in Thread.start()), we roll back cleanly.
Params:
firstTask – the task the new thread should run first (or null if none). Workers are created with an initial first task (in method execute()) to bypass queuing when there are fewer than corePoolSize threads (in which case we always start one), or when the queue is full (in which case we must bypass queue). Initially idle threads are usually created via prestartCoreThread or to replace other dying workers.
core – if true use corePoolSize as bound, else maximumPoolSize. (A boolean indicator is used here rather than a value to ensure reads of fresh values after checking other pool state).
Returns:
true if successful

```java
private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

检查在当前池状态和给定的界限（核心或最大）下是否可以添加新worker。如果可以，则相应调整worker count，并且如果可能的话，创建并启动一个新worker，将 firstTask 作为其第一个任务运行。如果池停止或符合关闭条件，该方法返回 false。如果thread factory在请求时未能创建线程，也返回 false。如果线程创建失败，无论是由于线程工厂返回 null，还是由于异常（通常是在 Thread.start() 中的 OutOfMemoryError），我们都会干净地回滚。
参数：
firstTask – 新线程应首先运行的任务（如果没有则为 null）。worker是通过初始的第一个任务创建的（在方法 execute() 中），以绕过排队，当线程少于 corePoolSize 时（在这种情况下我们总是启动一个），或当队列已满时（在这种情况下我们必须绕过队列）。初始空闲线程通常通过 prestartCoreThread 创建或替换其他即将终止的worker。
core – 如果为 true 使用 corePoolSize 作为界限，否则使用 maximumPoolSize。（这里使用布尔指示器而不是值，以确保在检查其他池状态后读取新值）。
返回：
成功则为 true

### 任务申请

Performs blocking or timed wait for a task, depending on current configuration settings, or returns null if this worker must exit because of any of: 1. There are more than maximumPoolSize workers (due to a call to setMaximumPoolSize). 2. The pool is stopped. 3. The pool is shutdown and the queue is empty. 4. This worker timed out waiting for a task, and timed-out workers are subject to termination (that is, allowCoreThreadTimeOut || workerCount > corePoolSize) both before and after the timed wait, and if the queue is non-empty, this worker is not the last thread in the pool.

根据当前配置设置执行阻塞或定时等待任务，如果该工作线程必须退出，则返回null。退出的原因可能是以下任一情况：1. 工作线程数量超过maximumPoolSize（由于调用了setMaximumPoolSize）。2. 线程池已停止。3. 线程池已关闭且队列为空。4. 该工作线程等待任务超时，并且超时的工作线程在定时等待前后都可能被终止（即allowCoreThreadTimeOut || workerCount > corePoolSize），且如果队列非空，该工作线程不是池中的最后一个线程。

```java

```

reference: [https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)