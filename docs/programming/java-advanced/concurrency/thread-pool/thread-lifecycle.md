# 线程分配回收策略

核心参数：

- corePoolSize：核心线程数，通常为CPU核心数
- maxPoolSize：最大线程数
- queueCapacity：缓冲队列大小
- KeepAliveTime：空闲线程存活

分配策略：

1. 当`workerCount ≤ corePoolSize`，直接创建核心线程处理任务
2. 当`workerCount > corePoolSize`，workerQueue未满，任务进入缓冲队列
3. 当`workerCount < maxPoolSize`且workerQueue已满，直接创建非核心线程执行任务
4. 当`workerCount > maxPoolSize`且workerQueue已满，拒绝任务

拒绝策略：

- CallerRunsPolicy：有调用线程来处理任务
- AbortPolicy：丢弃任务并抛出异常
- DiscardPolicy：直接丢弃任务
- DiscardOldestPolicy：丢弃最早未处理的任务，并获取队列任务执行

回收策略：

1. 核心线程一旦创建始终存在，不会被销毁；
2. 新任务如果有空闲的核心线程直接使用，如果没有进入缓冲队列
3. 空闲线程主动去poll队列任务然后执行，但是超出空闲时间（KeepAliveTime）则会销毁

*notice：如果allowCoreThreadTimeout设置为true，核心线程也会被回收。*