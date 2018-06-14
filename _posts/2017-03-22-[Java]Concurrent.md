---
layout: post
title: 'Concurrent包简介'
date: 2017-03-22
author: Feihang Han
tags: JAVA
---

Java 5新增了一个concurrent包。它包含了一系列类，能够让Java的并发（多线程）编程变得更加容易。在Java 5之前，你需要自己动手去实现相关工具类。

# BlockingQueue

BlockingQueue通常用于一个线程生产对象，而另外一个线程消费这些对象的场景。

![](http://tutorials.jenkov.com/images/java-concurrency-utils/blocking-queue.png)

特点：

* 队列有界。
* 队列达到上限时，生产线程如果尝试插入对象，会被阻塞，直到消费线程从队列拿走一个对象。
* 队列为空时，消费线程如果尝试去获取对象，会被阻塞。
* 无法插入null元素。

实现：

* ArrayBlockingQueue

有界阻塞队列。
初始化需要传入大小。基于数组实现，具有数组的特性：一旦初始化，无法修改大小。
FIFO

* DelayQueue

无界阻塞队列。
对元素进行持有直到一个特定的延迟到期。注入其中的元素必须实现 java.util.concurrent.Delayed 接口。
只有在延迟期满时才能从中提取元素。

* LinkedBlockingQueue

链式结构。
可定义大小；也可不定义，此时使用 Integer.MAX_VALUE 作为上限。
FIFO

* PriorityBlockingQueue

带优先级的队列，元素按优先级消费；如果优先级相等，队列不做任何特定行为。
无界。
元素需要实现java.lang.Comparable 接口。
基于堆数据结构。

* SynchronousQueue

同时只能够容纳单个元素。
如果该队列已有一元素的话，试图向队列中插入一个新元素的线程将会阻塞，直到另一个线程将该元素从队列中抽走。
同样，如果该队列为空，试图向队列中抽取一个元素的线程将会阻塞，直到另一个线程向队列中插入了一条新的元素。

# BlockingDeque

deque\(双端队列\) 是 "Double Ended Queue" 的缩写。线程在双端队列的两端都可以插入和提取元素。

![](http://tutorials.jenkov.com/images/java-concurrency-utils/blocking-deque.png)

BlockingDeque继承BlockingQueque接口。这就意味着你可以像使用一个 BlockingQueue 那样使用 BlockingDeque。

实现：

* LinkedBlockingDeque

双端队列。
在它为空的时候，一个试图从中抽取数据的线程将会阻塞，无论该线程是试图从哪一端抽取数据。

# ConcurrentMap

ConcurrentMap接口表示了一个能够对别人的访问\(插入和提取\)进行并发处理的 java.util.Map。ConcurrentMap 除了从其父接口 java.util.Map 继承来的方法之外还有一些额外的原子性方法。

实现：

* ConcurrentHashMap

基于锁分离技术，整个ConcurrentHashMap被分成多个seg。当写入Map时，只会锁住当前key所在的seg。
ConcurrentHashMap中的每一个seg和HashTable类很相似。

* ConcurrentNavigableMap

一个支持并发访问的 java.util.NavigableMap。
它还能让它的子map具备并发访问的能力。所谓的 "子map" 指的是诸如 headMap()，subMap()，tailMap() 之类的方法返回的 map。

# CountDownLatch

CountDownLatch是一个并发构造，它允许一个或多个线程等待一系列指定操作的完成。CountDownLatch 以一个给定的数量初始化。countDown\(\) 每被调用一次，这一数量就减一。通过调用 await\(\) 方法之一，线程可以阻塞等待这一数量到达零。

# CyclicBarrier

CyclicBarrier 类是一种同步机制，它能够对处理一些算法的线程实现同步。换句话讲，它就是一个所有线程必须等待的一个栅栏，直到所有线程都到达这里，然后所有线程才可以继续做其他事情。

满足以下任何条件都可以让等待 CyclicBarrier 的线程释放：

* 最后一个线程也到达 CyclicBarrier\(调用 await\(\)\)
* 当前线程被其他线程打断\(其他线程调用了这个线程的 interrupt\(\) 方法\)
* 其他等待栅栏的线程被打断
* 其他等待栅栏的线程因超时而被释放
* 外部线程调用了栅栏的 CyclicBarrier.reset\(\) 方法

CyclicBarrier 支持一个栅栏行动，栅栏行动是一个 Runnable 实例，一旦最后等待栅栏的线程抵达，该实例将被执行。你可以在 CyclicBarrier 的构造方法中将 Runnable 栅栏行动传给它：

```
Runnable      barrierAction = ... ;  
CyclicBarrier barrier       = new CyclicBarrier(2, barrierAction);  
```

# Exchanger

java.util.concurrent.Exchanger 类表示一种两个线程可以进行互相交换对象的会和点。

![](http://tutorials.jenkov.com/images/java-concurrency-utils/exchanger.png)

# Semaphore

java.util.concurrent.Semaphore 类是一个计数信号量。它具备两个主要方法：

* acquire\(\)
* release\(\)

计数信号量由一个指定数量的 "许可" 初始化。每调用一次 acquire\(\)，一个许可会被调用线程取走。每调用一次 release\(\)，一个许可会被返还给信号量。因此，在没有任何 release\(\) 调用时，最多有 N 个线程能够通过 acquire\(\) 方法，N 是该信号量初始化时的许可的指定数量。这些许可只是一个简单的计数器。

信号量主要有两种用途：

1. 保护一个重要\(代码\)部分防止一次超过 N 个线程进入。
2. 在两个线程之间发送信号。

没有办法保证线程能够公平地可从信号量中获得许可。也就是说，无法担保掉第一个调用 acquire\(\) 的线程会是第一个获得一个许可的线程。如果第一个线程在等待一个许可时发生阻塞，而第二个线程前来索要一个许可的时候刚好有一个许可被释放出来，那么它就可能会在第一个线程之前获得许可。

如果你想要强制公平，Semaphore 类有一个具有一个布尔类型的参数的构造子，通过这个参数以告知 Semaphore 是否要强制公平。强制公平会影响到并发性能，所以除非你确实需要它否则不要启用它。以下是如何在公平模式创建一个 Semaphore 的示例：

```
Semaphore semaphore = new Semaphore(1, true); 
```

# ExecutorService

java.util.concurrent.ExecutorService 接口表示一个异步执行机制，使我们能够在后台执行任务。因此一个 ExecutorService 很类似于一个线程池。实际上，存在于 java.util.concurrent 包里的 ExecutorService 实现就是一个线程池实现。

![](http://tutorials.jenkov.com/images/java-concurrency-utils/executor-service.png)

ExecutorService接口定义了以下几个方法：

* execute\(Runnable\)：异步执行Runnable。
* submit\(Runnable\)：异步执行Runnable，并返回Futrue对象，用来检查Runnable是否已经执行完毕。
* submit\(Callable\)：异步执行Callable，并返回Callable的结果Futrue对象。
* invokeAny\(...\)：返回其中一个 Callable 对象的结果。无法保证返回的是哪个 Callable 的结果 - 只能表明其中一个已执行结束。如果其中一个任务执行结束\(或者抛了一个异常\)，其他 Callable 将被取消。
* invokeAll\(...\)：异步执行所有Callable对象。返回一系列Future对象。
* shutdown\(\)：关闭线程池。ExecutorService 并不会立即关闭，但它将不再接受新的任务，而且一旦所有线程都完成了当前任务的时候，ExecutorService 将会关闭。在 shutdown\(\) 被调用之前所有提交给 ExecutorService 的任务都被执行。
* shutdownNow\(\)：立即尝试停止所有执行中的任务，并忽略掉那些已提交但尚未开始处理的任务。无法担保执行任务的正确执行。可能它们被停止了，也可能已经执行结束。

实现：

* ThreadPoolExecutor

关键属性corePoolSize/maxPoolSize
当一个任务委托给线程池时，如果池中线程数量低于 corePoolSize，一个新的线程将被创建，即使池中可能尚有空闲线程。
如果内部任务队列已满，而且有至少 corePoolSize 正在运行，但是运行线程的数量低于 maximumPoolSize，一个新的线程将被创建去执行该任务。

# ScheduledExecutorService

ScheduledExecutorService接口继承于ExecutorService接口。能够将任务延后执行，或者间隔固定时间多次执行。 

实现：

* ScheduledThreadPoolExecutor

# ForkJoinPool

ForkJoinPool 在 Java 7 中被引入。它和ExecutorService很相似，除了一点不同。ForkJoinPool 让我们可以很方便地把任务分裂成几个更小的任务，这些分裂出来的任务也将会提交给 ForkJoinPool。

一个使用了分叉和合并原理的任务可以将自己分叉\(分割\)为更小的子任务，这些子任务可以被并发执行。图示如下：

![](http://tutorials.jenkov.com/images/java-concurrency-utils/java-fork-and-join-1.png)

当一个任务将自己分割成若干子任务之后，该任务将进入等待所有子任务的结束之中。一旦子任务执行结束，该任务可以把所有结果合并到同一个结果。图示如下：

![](http://tutorials.jenkov.com/images/java-concurrency-utils/java-fork-and-join-2.png)

# Lock

java.util.concurrent.locks.Lock 是一个类似于 synchronized 块的线程同步机制。但是 Lock 比 synchronized 块更加灵活、精细。

Lock 和 synchronized 代码块的主要不同点：

* synchronized 代码块不能够保证进入访问等待的线程的先后顺序。
* 你不能够传递任何参数给一个 synchronized 代码块的入口。因此，对于 synchronized 代码块的访问等待设置超时时间是不可能的事情。
* synchronized 块必须被完整地包含在单个方法里。而一个 Lock 对象可以把它的 lock\(\) 和 unlock\(\) 方法的调用放在不同的方法里。

实现：

* ReentrantLock：可重入锁
* ReadWriteLock：读写锁

加读锁前提：如果没有任何写操作线程锁定 ReadWriteLock，并且没有任何写操作线程要求一个写锁(但还没有获得该锁)。
           因此，可以有多个读操作线程对该锁进行锁定。
加写锁前提：如果没有任何读操作或者写操作。因此，在写操作的时候，只能有一个线程对该锁进行锁定。

# 原子量

乐观锁的一种实现，通过compareAndSet来实现。

```java
for (;;) {
    boolean current = get();
    if (compareAndSet(current, newValue))
        return current;
}
```

有以下几个原子类

* AtomicBoolean
* AtomicInteger
* AtomicLong
* AtomicReference

# 参考

[http://tutorials.jenkov.com/java-util-concurrent/index.html](http://tutorials.jenkov.com/java-util-concurrent/index.html)





