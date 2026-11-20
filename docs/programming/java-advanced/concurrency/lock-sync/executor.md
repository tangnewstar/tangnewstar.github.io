# Executor

Executor: This interface has a single method execute(Runnable command) which is used to execute submitted tasks.

ExecutorService: This interface extends Executor and adds methods like submit, invokeAll, invokeAny, shutdown, and awaitTermination to manage task execution and the executor's lifecycle.