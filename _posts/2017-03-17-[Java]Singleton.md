---
layout: post
title: 'Singleton模式'
date: 2017-03-17
author: Feihang Han
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: JAVA
---

## 简介

单例模式确保某个类只有一个由自身创建的单一实例，且自行实例化并提供访问入口。

## 实现

#### 懒汉式单例

```
public class LazySingleton {

    private LazySingleton() {
    }

    private static LazySingleton singleton = null;

    public static LazySingleton getInstance() {
        if (singleton == null) {
            singleton = new LazySingleton();
        }
        return singleton;
    }

}
```

LazySingleton通过把构造函数限定为private避免类在外部被实例化，来确保在同一虚拟机内只有一个实例。使用者可以通过getInstance\(\)方法来获取该单例。

* 事实上，Java反射机制可以实例化构造方法为private的类。

```
import java.lang.reflect.Constructor;

public class ReflectSingleton {

    public static void main(String[] args) throws Exception {
        Class<?> class1 = Class.forName("com.hanfeihang.singleton.LazySingleton");
        Constructor<?>[] cons = class1.getDeclaredConstructors();
        Constructor<?> con = cons[0];
        con.setAccessible(true);
        LazySingleton ls = (LazySingleton) con.newInstance();
        System.out.println(ls);
    }

}
```

上述懒汉式单例是线程不安全的，并发环境下可能有多个线程同时进行if \(singleton == null\)的判断，且都为true。这种情况下，多个线程都会new出LazySingleton实例。

使用synchronized关键字修饰getInstance\(\)，可以较为简单的使得单例类变成线程安全。

#### 双重检查锁定

使用synchronized关键字，锁定的是整个类对象，会对性能造成较大的影响。

```
public class DoubleCheckSingleton {

    private DoubleCheckSingleton() {
    }

    private volatile static DoubleCheckSingleton singleton = null;

    public static DoubleCheckSingleton getInstance() {
        if (singleton == null) {
            synchronized (DoubleCheckSingleton.class) {
                if (singleton == null) {
                    singleton = new DoubleCheckSingleton();
                }
            }

        }
        return singleton;
    }

}
```

改造getInstance\(\)函数，并增加两重检查，以此来减少在同步块内进行判断所浪费的时间：

* 第一重检查：先检查实例是否存在，如果不存在，就进入同步块。
* 第二重检查：同步块内再次检查实例是否存在，如果不存在，则进行实例化。

```
关键字volatile：被修饰的变量值不会在本地线程缓存，所有对该变量的读写都直接操作共享内存，从而确保多个线程能正确处理该变量。
在JAVA1.4及以前的版本，很多JVM对该关键字的实现有问题，慎用。建议在JAVA1.5之后使用。
```

#### 双检锁问题

由于instance = new Singleton\(\);这行代码在不同编译器上的行为是无法预知的。

一个优化编译器可以合法地如下实现：

1. instance = 给新的实体分配内存
2. 调用Singleton的构造函数来初始化instance的成员变量

现在想象一下有线程A和B在调用getInstance，线程A先进入，在执行到步骤1的时候被踢出了cpu。然后线程B进入，B看到的是instance已经不是null了（内存已经分配），于是它开始放心地使用instance，但这个是错误的，因为在这一时刻，instance的成员变量还都是缺省值，A还没有来得及执行步骤2来完成instance的初始化。

当然编译器也可以这样实现：

1. temp = 分配内存
2. 调用temp的构造函数
3. instance = temp

如果编译器的行为是这样的话我们似乎就没有问题了，但事实却不是那么简单，因为我们无法知道某个编译器具体是怎么做的，因为在Java的memory model里对这个问题没有定义。

#### 静态（类级）内部类

```
public class StaticInnerClassSingleton {

    private StaticInnerClassSingleton() {
    }

    private static class SingletonHolder {
        private static StaticInnerClassSingleton instance = new StaticInnerClassSingleton();
    }

    public static StaticInnerClassSingleton getInstance() {
        return SingletonHolder.instance;
    }

}
```

静态内部类即实现了线程安全，又避免同步带来的性能问题。

当getInstance\(\)第一次被调用的时候，JVM第一次读取SingletonHolder.instance类，导致类SingletonHolder被初始化；类SingletonHolder初始化时，JVM会初始化它的静态变量，从而创建实例。由于是静态域，JVM只会在装载类的时候初始化一次，保证了其线程安全性。

#### 饿汉式单例

```
public class EagerSingleton {

    private static EagerSingleton singleton = new EagerSingleton();

    private EagerSingleton() {
    }

    public static EagerSingleton getInstance() {
        return singleton;
    }

}
```

饿汉式单例在类创建时就初始化好了实例，天生是线程安全的。

#### 枚举

```
public enum SingletonEnum {

    INSTANCE;

}
```

JVM从根本上保障了枚举类，防止其多次实例化，并且提供了序列化机制，是更加简洁、高效、安全的实现单例的方式。

