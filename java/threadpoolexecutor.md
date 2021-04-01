# ThreadPoolExecutor

### 线程池状态

ctl类型为AtomicInteger，包含两部分：运行状态 ＋ 任务数量，其中运行状态占3位，任务数量占29位。


**运行状态**

- `RUNNING=-1`：Accept new tasks and process queued tasks
- `SHUTDOWN=0`：Don't accept new tasks, but process queued tasks
- `STOP=1`：Don't accept new tasks, don't process queued tasks, and interrupt in-progress tasks
- `TIDYING=2`：All tasks have terminated, workerCount is zero, the thread transitioning to state TIDYING will run the terminated() hook method
- `TERMINATED=3`：terminated() has completed

**状态转换**

状态转换只能是递增转换，从值较低的状态转换到值较高的状态，但是不一定是每个状态都会经历，可以从一个状态直接跳跃到下一个值较高的状态。

- `RUNNING -> SHUTDOWN`: On invocation of shutdown(), perhaps implicitly in finalize()
- `(RUNNING or SHUTDOWN) -> STOP`: On invocation of shutdownNow()
- `SHUTDOWN -> TIDYING`: When both queue and pool are empty
- `STOP -> TIDYING`: When pool is empty
- `TIDYING -> TERMINATED`: When the terminated() hook method has completed

### RejectedExecutionHandler

> This may occur when no more threads or queue slots are
 available because their bounds would be exceeded, or upon shutdown of the Executor.

> In the absence of other alternatives, the method may throw an unchecked `RejectedExecutionException`, which will be propagated to the caller of `execute()`.

实现类如下：

- DiscardPolicy：直接丢弃，do nothing，线程池默认选项
- AbortPolicy：抛出异常RejectedExecutionException
- DiscardOldestPolicy：丢弃队列里最老的任务(最先进入队列)，然后再尝试将当前任务加入线程池
- CallerRunsPolicy：在调用者当前线程直接执行

下面我们看看代码就能更清晰地理解。

**DiscardPolicy**

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    //直接丢弃，do nothing
}
```

**AbortPolicy**

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    //抛出异常
    throw new RejectedExecutionException(
        "Task " + r.toString() + " rejected from " + e.toString());
}
```

**DiscardOldestPolicy**

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    //丢弃队列里最老的任务(最先进入队列)，然后再尝试将当前任务加入线程池
    if (!e.isShutdown()) {
        e.getQueue().poll();
        e.execute(r);
    }
}
```

**CallerRunsPolicy**

```java
public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    //在调用者当前线程直接执行
    if (!e.isShutdown()) {
        r.run();
    }
}
```

### 配置参数

- boolean allowCoreThreadTimeOut
- int corePoolSize
- int maximumPoolSize
- long keepAliveTime
- ThreadFactory threadFactory
- RejectedExecutionHandler handler
- BlockingQueue<Runnable> workQueue