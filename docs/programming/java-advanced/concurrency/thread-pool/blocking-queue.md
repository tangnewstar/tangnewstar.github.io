# thread pool blocking queue

在线程池使用BlockingQueue，利用的是阻塞队列的生产和消费功能。

- LinkedBlockingQueue：基于链表动态扩容，入队/出队比ArrayBlockingQueue效率高
- ArrayBlockingQueue：基于数组固定大小，需要预先分配连续内存，入队出队肯呢个有锁竞争
- SynchronousQueue:

为什么选择LinkedBlockingQueue?

- 无界队列的可应对高并发下的任务缓冲，避免重复拒绝任务

ArrayBlockingQueue是有界的需要考虑边界不然容易触发拒绝策略，SynchronousQueue不存储任务直接传递适合高吞吐但拒绝风险高