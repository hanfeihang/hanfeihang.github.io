---
layout: post
title: 'ThreadPoolExecutor简介'
date: 2017-04-14
author: Feihang Han
tags: JAVA
---

# 线程池优缺点

与直接New Thread\(\)相比，线程池有如下优点：

* 重用存在的线程，减少对象创建、消亡的开销，性能佳。
* 可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。
* 提供定时执行、定期执行、单线程、并发数控制等功能。

# 实现

Java线程池的底层都是基于ThreadPoolExecutor实现的。

```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

参数介绍

```
@param corePoolSize the number of threads to keep in the pool, even
       if they are idle, unless {@code allowCoreThreadTimeOut} is set
@param maximumPoolSize the maximum number of threads to allow in the
       pool
@param keepAliveTime when the number of threads is greater than
       the core, this is the maximum time that excess idle threads
       will wait for new tasks before terminating.
@param unit the time unit for the {@code keepAliveTime} argument
@param workQueue the queue to use for holding tasks before they are
       executed.  This queue will hold only the {@code Runnable}
       tasks submitted by the {@code execute} method.
@param threadFactory the factory to use when the executor
       creates a new thread
@param handler the handler to use when execution is blocked
       because the thread bounds and queue capacities are reached
```

# 关键属性

* corePoolSize/maximumPoolSize

> 当一个任务委托给线程池时，如果池中线程数量低于corePoolSize，一个新的线程将被创建，即使池中可能尚有空闲线程。
>
> 如果内部任务队列已满，而且有至少corePoolSize 正在运行，但是运行线程的数量低于 maximumPoolSize，一个新的线程将被创建去执行该任务。

* workQueue

> 超过corePoolSize的线程会先放入workQueue等待执行。

* allowCoreThreadTimeOut

> 如果为false（default），core threads即使处于空闲状态也会保留；
>
> 如果为true，core threads在空闲keepAliveTime后被回收。

* RejectedExecutionHandler

> 当线程池满了后，拒绝新线程的处理器。

# 拒绝策略

目前JDK7自带实现了以下几种拒绝策略的RejectedExecutionHandler

* AbortPolicy：抛出一个异常RejectedExecutionException来拒绝线程。**线程池的默认拒绝策略。**
* AbortPolicyWithReport：在AbortPolicy基础上，增加打印日志的功能。
* CallerRunsPolicy：直接在execute函数中执行当前线程的run方法。如果线程池被shut down，在这种情况下，该线程会被丢弃。
* DiscardOldestPolicy：丢弃workQueue中最老的一个线程，并execute该线程。如果线程池被shut down，在这种情况下，该线程会被丢弃。
* DiscardPolicy：直接丢弃该线程。

# 线程池

* newFixedThreadPool

创建一个固定大小的线程池。当线程数达到nThreads时，其余线程会被丢入LinkedBlockingQueue。该队列默认大小是Integer.MAX\_VALUE。

```
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                0L, TimeUnit.MILLISECONDS,
                new LinkedBlockingQueue<Runnable>());
}
```

* newSingleThreadExecutor

创建一个单线程的线程池。当线程数达到1时，其余线程会被丢入LinkedBlockingQueue。该队列默认大小是Integer.MAX\_VALUE。

```
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>()));
}
```

* newCachedThreadPool

创建一个可缓存线程池。如果线程池的大小超过了处理任务所需要的线程，那么就会回收部分空闲的线程。当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小

```
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```

* newScheduledThreadPool

创建一个定长线程池，支持定时及周期性的执行任务。在ScheduledThreadPoolExecutor中工作队列类型是它的内部类DelayedWorkQueue，而DelayedWorkQueue的Task容器是DelayQueue类型，而ScheduledFutureTask作为Delay的实现类作为Runnable的封装后的Task类。也就是说ScheduledThreadPoolExecutor是通过DelayQueue优先级判定规则来执行任务的。

```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE, 0, TimeUnit.NANOSECONDS,
              new DelayedWorkQueue());
}
```



