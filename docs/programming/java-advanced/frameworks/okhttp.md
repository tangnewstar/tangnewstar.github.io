# OkHttp

线程池：OkHttp请求有Dispatcher来分配，内部创建了一个 **核心线程为0** 最大线程为Integer.Max的线程池，空闲线程回收60秒，适合大量的耗时短任务。

**Dispatcher**通过双端队列（**readyAsyncCalls**和**runningAsyncCalls**）控制并发请求数（默认每主机5个、全局64个），避免线程池过载[⁠⁣](https://juejin.cn/post/7005541726794285092)

```java

private int maxRequests = 64;
private int maxRequestsPerHost = 5;
public synchronized ExecutorService executorService() {
        if (this.executorService == null) {
            this.executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue(), Util.threadFactory("OkHttp Dispatcher", false));
        }

        return this.executorService;
    }
```