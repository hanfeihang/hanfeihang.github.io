---
layout: post
title: 'Java volatile'
date: 2017-06-29
author: Feihang Han
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: JAVA
---

#### 特性

关键字volatile是Java提供的最轻量级的同步机制。当一个变量被volatile修饰后，它具备以下两个特性：

* 线程可见性：当一个线程修改了被volatile修饰的变量后，无论是否加锁，其他线程都可以立即看到最新的修改，而普通变量却做不到这点。
* 禁止指令重排序优化：普通的变量仅仅保证在该方法的执行过程中所有依赖赋值结果的地方都能获得正确的结果，而不能保证变量赋值操作的顺序与程序代码执行顺序一致。

#### 例子

以下例子可说明这问题。

```
public class Demo {
    private static boolean stop;

    public static void main(String[] args) {
        Thread workThread = new Thread(new Runnable() {
            public void run() {
                int i = 0;
                while (!stop) {
                    System.out.println(i++);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        workThread.start();
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        stop = true;
    }
}
```

我们预期程序会在3s后停止，但是实际上它会一直执行下去（不一定，实际上和虚拟机的实现有关），原因是虚拟机对代码进行了指令重排序和优化，优化后如下：

```
if (!stop)
while(true) {
...
}
```

#### 解决

如果我们用volatile修饰stop变量，再次运行，会发现程序3s后退出。使用volatile解决了如下两个问题。

* main线程对stop的修改在worThread线程中可见。
* 禁止指令重排序，防止因为重排序导致的并发访问逻辑混乱。

#### 经验

一些人认为使用volatile可以代替传统锁，提升并发性能，这个认识是错误的。volatile仅仅解决了可见性的问题，但是它并不能保证互斥性，也就是说多个线程并发修改某个变量时，依旧会产生多线程问题。因此，不能依靠volatile来完全代替传统的锁。

根据经验总结，volatile最适用的是一个线程写，其他线程读的场景，在这种场景下用volatile来代替synchronized关键字可以提升并发访问的性能；如果有多线程并发写操作，仍然需要使用锁或者线程安全的容器或者原子量来代替。





