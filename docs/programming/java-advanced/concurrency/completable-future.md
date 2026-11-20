# CompletableFuture

**Future (JDK5新增)**

*描述一个异步计算的结果*

缺点：

- 获取结果需要阻塞主线程：同步阻塞 *get()* 或者轮询 *Future.isDone()*
- 异步任务需要相互依赖的场景，主线程需要等待子线程执行完毕，比如CountdownLatch

CompletableFuture

*任务异步化*
`CompletableFuture.runAsync()` : 执行任务不返回值，支持自定义线程池。默认使用ForkJoin
`CompletableFuture.supplyAsync()` : 执行任务并返回值，支持自定义线程池。默认ForkJoin

*任务依赖*

supplyAsync - 第一个调用结果, 可通过链式调用来传递结果

- thenRun(Runnable r): 接着上一个异步任务，对任务结果不处理
- thenAccept(Consumer c): 对上一个任务结果进行消费
- thenApply(Function f)：对上个任务的结果进行改写，返回新值作为任务结果

```java
CompletableFuture future2 = CompletableFuture.supplyAsync(() -> 5)
            .thenAccept(result -> {
                System.out.println("任务执行结果 = " + result * 3);
            }).thenAccept(result -> {
                System.out.println("任务执行完成");
            });
```

### 任务并行 - 第一个

可以通过 anyOf() 方法来进行实现。需要注意的是，如果**最快完成**的任务出现异常，则返回的CompletableFuture 执行 get() 方法时会抛出异常。

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "朋友小王去你家送药");
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "朋友小张去你家送药");

        CompletableFuture<Object> anyFutures = CompletableFuture.anyOf(future1, future2);
        
打印日志如下：

朋友小王去你家送药       
```

### 任务并行 - 全部

可以通过 allOf() 方法来进行实现。 需要注意的是，如果**任意一个**任务出现异常，则返回的CompletableFuture 执行 get() 方法时会抛出异常。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 15);
        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 10);

        CompletableFuture<Integer> allFutures = CompletableFuture.allOf(future1, future2).thenApply(res -> {
            return future1.join()  + future2.join();
        });
打印日志如下：

25
```

### 任务回调

可以通过 whenComplete()、handle() 方法实现任务回调。两者区别在于：

- whenComplete() 方法没有返回值，且返回的 CompletableFuture 为任务结果，而非回调结果。
- handle() 方法有返回值，且返回的 CompletableFuture 为回调结果。

```java
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            return "返回任务结果";
        });

        CompletableFuture<String> future1Callback = future1.whenComplete((result, throwable) -> {
            System.out.println("执行回调任务");
        });

        System.out.println(future1Callback.join());

        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            return "返回任务结果";
        });

        CompletableFuture<String> future2Callback = future2.handle((result, throwable) -> {
            return "返回回调结果";
        });

        System.out.println(future2Callback.join());
        
打印日志如下：

执行回调任务
返回任务结果
返回回调结果        
```