---
title: "Java线程池源码分析"
author: "爱吃芒果"
description:
date: 2025-04-26T09:47:55+08:00
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
  - "JVM"
  - "Java"
  - "JDK11"
  - "线程池"
categories:
  - "JVM"
  - "Java"
  - "JDK11"
  - "线程池"
---

## 阅前提示

参考文献中的文章非常的好，基本看完了就能理解很多东西，推荐阅读

源码中也提供了很多注释文本，推荐对照源码学习。

## 重要概念和接口

### Runnable 接口

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

线程可以接受一个实现 Runnable 接口的对象，并执行对应的逻辑。

### Callable 接口

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}

```

类似`Runnalbe`接口，但可以返回结果和抛出异常

### Future 接口

表示异步执行的结果，提供了获取结果以及取消计算执行等方法。

```java
public interface Future<V> {
   // 取消任务执行，mayInterruptIfRunning参数为true时将中断正在执行任务的线程，否则正在执行的任务将继续执行
    boolean cancel(boolean mayInterruptIfRunning);

   // 是否任务在执行完成前被取消
    boolean isCancelled();

   // 任务是否完成，不管任务正常结束、抛出异常还是被取消都认为任务完成
    boolean isDone();

   // 等待任务完成并获得结果
   // 计算被取消时抛出CalcellationException
   // 计算抛出异常时抛出ExecutionException
   // 当前线程等待时被中断抛出InteruptedException
    V get() throws InterruptedException, ExecutionException;

  // 超时版本的get，如果等待超时抛出TimeoutException
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

### Executor 接口

```java
public interface Executor {
    void execute(Runnable command);
}
```

线程池基础接口，提交一个任务到线程池执行。

> Memory consistency effects: Actions in a thread prior to submitting a Runnable object to an Executor happen-before its execution begins, perhaps in another thread.

### ExecutorSerivce 接口

`ExecutorService`继承了`Executor`接口，通常我们使用`ExecutorService`作为线程池接口，它提供了丰富的功能，一般能够满足需求。

```java
public interface ExecutorService extends Executor {

  // 已提交的任务继续执行，但不再接收新任务，等待正在执行任务终止请使用awaitTermination
    void shutdown();

  // 尝试停止所有正在执行的任务，终止所有等待任务的处理，返回等待任务列表，等待正在执行任务终止请使用awaitTermination
   // 无法保证所以任务都能终止，经典的实现会通过`interrupt`取消任务，但如果任务不响应中断，则可能永远都不会停止
    List<Runnable> shutdownNow();

   // 线程池是否关闭
    boolean isShutdown();

   // shutdown后所有任务是否已经结束
   // 这个方法只有在调用shutdown或者shutdownNow后才可能返回true
    boolean isTerminated();

  // 在shutdown后调用，阻塞直到所有任务执行完成或者超时发生或者当前线程被中断
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

  // 提交一个有返回值的任务，通过Future对象获取返回值
   // task不能为null，否则抛出NullPointerException
   // 如果任务不能被调用执行，抛出 RejectedExecutionException
    <T> Future<T> submit(Callable<T> task);

  // 类似 submit(Callable)，返回值对应传入的result参数
    <T> Future<T> submit(Runnable task, T result);

  // 类似 submit(Callable)，返回值为null
    Future<?> submit(Runnable task);

  // 执行给定的任务列表，当全部完成时返回Futrue列表，在操作执行时修改给定的集合会导致未定义行为
   // 在等待时发生中断，抛出InterruptedException，取消未完成的任务
   // 任务列表和其中的任务都不能为null，否则抛出NullPointerException
   // 如果任何任务不能被调度执行，抛出RejectedExecutionException
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

  // 超时版本的invokeAll，如果超时发生，取消未完成的任务
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

  // 类似invokeAll，执行给定的任务列表，返回一个成功执行任务的结果，指没有抛出异常，取消未执行完成的任务
   // tasks为空时抛出IllegalArgumentException
   // 如果没有任务成功完成，抛出ExecutionException
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

  // 超时版本的invokeAny
   // 超时发生抛出TimeoutException
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

> Memory consistency effects: Actions in a thread prior to the submission of a Runnable or Callable task to an ExecutorService happen-before any actions taken by that task, which in turn happen-before the result is retrieved via Future. get().

Doug Lea 给了一个终止线程池的例子，首先调用`shutdown`拒绝接受新任务，然后调用`shutdowNow`，取消逗留的任务，这里特别处理了当前线程遇到`interrupt`的情况。

```java
void shutdownAndAwaitTermination(ExecutorService pool) {
    pool.shutdown(); // Disable new tasks from being submitted
    try {
        // Wait a while for existing tasks to terminate
        if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
            pool.shutdownNow(); // Cancel currently executing tasks
            // Wait a while for tasks to respond to being cancelled
            if (!pool.awaitTermination(60, TimeUnit.SECONDS))
                System.err.println("Pool did not terminate");
        }
    } catch (InterruptedException ie) {
        // (Re-)Cancel if current thread also interrupted
        pool.shutdownNow();
        // Preserve interrupt status
        Thread.currentThread().interrupt();
    }
}
```

## FutureTask 类源码解析

接口`RunnableFuture`是`Runnalbe`的`Future`，`run`方法的成功执行对应`Future`的完成，并允许获取结果。

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

`FutureTask`类实现了`RunnableFuture`，`FutureTask`的具体实现原理留在后面再讲。(todo)

## AbstractExecutorService 源码解析

`AbstractExecutorService`抽象类派生自`ExecutorService`接口，然后在其基础上实现了几个实用的方法，这些方法提供给子类进行调用。

抽象类实现了 invokeAny 和 invokeAll 方法（这两个方法先不看 todo），方法`newTaskFor`用于将`Runnable`或者`Callable`包装成`FutureTask`。提交任务到线程池中有两类方法，`submit`用于需要返回值的场景，`execute`用于不需要返回值的场景，当然可以都只用`submit`方法，当不需要返回值时返回 null 即可。

```java
public abstract class AbstractExecutorService implements ExecutorService {

   // 将runnable包装成FutureTask
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

  // 将Callable包装成FutureTask
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

   // 包装成FutureTask，并交给底层execute方法执行
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);
        execute(ftask);
        return ftask;
    }

    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }

```

## ThreadPoolExecutor

`ThreadPoolExecutor`是 JDK 中的线程池实现，实现了任务提交、线程管理、监控等方法。

通过构造函数，介绍一些重要的属性：

- corePoolSize 核心线程数，注意有时将核心线程数内的线程称为核心线程，但核心线程本身和其他线程一样
- maximumPoolSize 最大线程数
- workQueue 任务队列，BlockingQueue 接口的某个实现（常用 ArrayBlockingQueue 和 LinkedBlockingQueue）
- keepAliveTime 空闲线程的保活线程，默认只对非核心线程生效，可以通过设置`allowCoreThreadTimeout(true)`使核心线程数内的线程可以被回收
- threadFactory 用于生成线程，比如设置线程的名字
- handler 设置线程池的拒绝策略

Doug Lea 采用一个 32 为的整数来存放线程池状态和线程池中的线程数，其中高 3 为用于存放线程池状态，低 29 位表示线程数。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }

/*
 * Bit field accessors that don't require unpacking ctl.
 * These depend on the bit layout and on workerCount being never negative.
 */

private static boolean runStateLessThan(int c, int s) {
    return c < s;
}

private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```

线程池各种状态的介绍：

- RUNNING: 接受新的任务，处理等待队列中的任务
- SHUTDWON: 不接受新的任务，但会继续处理等待队列中的任务
- STOP: 不接受新的任务提交，不再处理等待队列中的任务，中断正在执行的线程
- TIDYING: 所有的任务都销毁了，workCount 为 0，执行钩子方法 terminated()
- TERMINATED: terminated()方法调用结束后，线程池的状态切换为此

RUNNING 定义为-1，SHUTDOWN 定义为 0，其他都比 0 大，所以等于 0 时不能提交任务，大于 0 的话，连正在执行的任务也要中断

状态迁移过程：

- RUNNING -> SHUTDOWN，调用 shutdown()
- (RUNNING or SHUTDOWN) -> STOP: 调用 shutdownNow()
- SHUTDOWN -> TIDYING: 当任务队列和线程池都清空后，有 SHUTDOWN 转换为 TIDYING
- STOP -> TIDYING: 任务队列清空后
- TIDYING -> TERMINATED: terminated()方法结束后

Doug Lea 将线程池中的线程包装成内部类 Worker，所以任务是 Runnable （内部变量名叫 task 或者 command)，线程是 worker

AQS: todo worker 的实现包含复杂的并发控制，这些暂时不考虑

```java
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }
```

`execute`是一个非常重要的方法，所有`submit`方法底层都会调用`execute`方法提交任务。可以看到尽管这段代码非常短小，但由于并发问题实现逻辑比较绕。

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
   // 如果当前线程数少于核心线程数，直接添加一个worker来执行任务，将当前任务作为它的第一个任务
   // addWorker调用会原子的检查runState和workerCount，避免错误添加新的线程
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
   // 如果线程池处于RUNNING状态，将这个任务添加到任务队列workQueue中
    if (isRunning(c) && workQueue.offer(command)) {
       // double-check
        int recheck = ctl.get();
       // 如果线程不处于RUNNING状态，移除已经入队的任务，并执行拒绝策略
        if (! isRunning(recheck) && remove(command))
            reject(command);
       // 如果线程池还是RUNNING状态，并且线程数为0，那么开启新的线程
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
   // 如果队列满了，尝试创建新的线程，如果已经达到最大线程数，执行拒绝策略
    else if (!addWorker(command, false))
        reject(command);
}
```

addWorker 方法用来创建新的线程

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
       // 线程池非RUNNINKG状态，则关闭
       // 需要排除一种特殊情况，线程池处于SHUTDOWN状态，且等待队列非空，这种情况下应该允许进一步判断是否创建新的worker
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
           // 原子操作，增加worker计数
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
           // 如果线程池自身状态发生改变，则回到外层for循环
           // 如果仅仅是cas失败，在内层循环重试
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
       // Worker的构造方法会调用ThreadFactory创建新的线程
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
           // 这是整个线程池的全局锁，因为关闭线程池需要这个锁，所以持有锁期间，线程池不会被关闭
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

               // RUNNING状态或者处于SHUTDOWN，这时会继续执行等待队列中的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                   // 添加worker到workers HashSet中
                    workers.add(w);
                    int s = workers.size();
                   // largestPoolSize用来记录worker数量的最大值
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
           // 添加成功的话，启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
       // 如果线程没有启动，需要做一些清理工作，比如前面workCount加一，将其回退
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
       // 移除创建的worker
        if (w != null)
            workers.remove(w);
       // 减低worker计数
        decrementWorkerCount();
       // 如果当前worker阻碍线程池终止，重新检查
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

线程池中的线程实际会执行`runWorker`方法

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
      // 处理第一个任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
          // 循环获取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
              // 如果线程状态大于等于STOP，需要中断线程
              // 否则清理线程中断状态并且检查此时线程池状态是否大于等于STOP，如果是，则中断线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                  // 执行前操作
                beforeExecute(wt, task);
                try {
                    task.run();
                    afterExecute(task, null);
                } catch (Throwable ex) {
                      // 执行后操作
                    afterExecute(task, ex);
                    throw ex;
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
          // 执行线程关闭
          // 1. 说明getTask返回null，队列中没有任务需要执行了，执行关闭
          // 2. completdAbruptly
        processWorkerExit(w, completedAbruptly);
    }
}
```



```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();

        // Check if queue empty only if necessary.
          // 1. 线程池处于SHUTDOWN状态，并且workQueue为空
          // 2. 线程池状态>= STOP
        if (runStateAtLeast(c, SHUTDOWN)
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
          // 允许核心线程数内的线程回收，或者当前线程数超过了核心线程数，那么有可能发生超时关闭
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

          // 如果线程数量超过最大线程数量，比如开发者调用setMaximumPoolSize调小了线程池，那么多余的worker就需要被关闭
          // 如果允许超时，并且获取任务发生超时，则关闭线程
          // 在队列不为空时，不允许关闭最后一个线程
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
              // 如果此 worker 发生了中断，采取的方案是重试
            // 解释下为什么会发生中断，这个读者要去看 setMaximumPoolSize 方法，中断空闲的worker
            timedOut = false;
        }
    }
}
```

拒绝策略：

- CallerRunsPolicy: 只要线程池没有关闭，由调用者线程执行任务
- AbortPolicy: 默认策略，直接抛出 RejectedExecutionException
- DiscardPolicy: 直接丢弃任务
- DiscardOldestPolicy: 将队列对头的任务（等待时间最久的任务）扔掉，提交这个任务到等待队列中

## Executors工具类

- 生成一个固定大小的线程池

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

最大线程数设置为和核心线程数相等，此时keepAliveTime设置为0（线程池默认不会不会corePoolSize内的线程），任务队列采用LinkedBlockingQueue，无界队列。

- 单线程线程池，类似上面，核心线程数为1

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

- 缓存线程池

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

核心线程数为0，最大线程数为Integer.MAX_VALUE，keepAliveTime为60s，任务队列采用SynchronousQueue

线程数不设上限，任务队列为同步队列，60s超时后空闲线程会被回收



## 参考文献

1. [深度解读 java 线程池设计思想及源码实现 javadoop](https://www.javadoop.com/post/java-thread-pool)
