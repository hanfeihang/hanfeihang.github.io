---
layout: post
title: 'Java Agent'
date: 2017-10-30
author: Feihang Han
tags: JAVA
---

JavaAgent是JDK1.5以后引入的，也可以叫做Java代理。
JavaAgent运行在main方法之前的拦截器，它内定的方法名叫 premain ，也就是说先执行premain方法然后再执行main方法。

# 实现

1、实现premain的类

```java
package com.github.hfh.agent;

import java.lang.instrument.Instrumentation;

public class MyAgent {

    /**
     * 该方法在main方法之前运行，与main方法运行在同一个JVM中
     * 并被同一个System ClassLoader装载
     * 被统一的安全策略(security policy)和上下文(context)管理
     */
    public static void premain(String agentOps, Instrumentation inst) {
        System.out.println("===pre main===");
        System.out.println(agentOps);
    }

    /**
     * 如果不存在 premain(String agentOps, Instrumentation inst)
     * 则会执行 premain(String agentOps)
     */
    public static void premain(String agentOps) {
        System.out.println("===pre main2===");
        System.out.println(agentOps);
    }
}

```

2、在 src 目录下添加 META-INF/MANIFEST.MF 文件，内容如下：

```
Manifest-Version: 1.0
Premain-Class: com.github.hfh.agent.MyAgent
Can-Redefine-Classes: true

```

3、最终项目路径为

```bash
MyAgent
└── src
    ├── META-INF
    │   └── MANIFEST.MF
    └── com
        └── github
            └── hfh
                └── agent
                    └── MyAgent.java
```

4、打包代码为 MyAgent.jar。注意打包的时候选择我们自己定义的 MANIFEST.MF。

# 检验

1、新建项目Test，编写main类

```java
package com.github.hfh.test;

import java.util.Arrays;

public class Test {
    
    public static void main(String[] args) {
        System.out.println("===main class===");
        Arrays.asList(args).stream().forEach(s -> System.out.println(s));
    }
    
}
```

2、MANIFEST.MF

```
Manifest-Version: 1.0
Main-Class: com.github.hfh.test.Test

```

3、最终项目路径如下

```bash
Test
├── Test.iml
└── src
    ├── META-INF
    │   └── MANIFEST.MF
    └── com
        └── github
            └── hfh
                └── test
                    └── Test.java
```

4、打包成Test.jar

5、将MyAgent.jar和Test.jar放到一个目录，执行

```shell
java -javaagent:./MyAgent.jar=Hello -javaagent:./MyAgent.jar=World -jar Test.jar xyz
```

6、显示

```bash
===pre main===
Hello
===pre main===
World
===main class===
xyz
```

# 现实应用

基于 JavaAgent 的 spring-loaded 实现 jar 包的热更新。



