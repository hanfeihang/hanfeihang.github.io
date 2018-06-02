---
layout: post
title: 'ZooKeeper简介'
date: 2017-03-23
author: Feihang Han
tags: ZooKeeper
---

ZooKeeper是一种用于分布式应用的分布式开源协调服务。它公开了一组简单的原语，分布式应用程序可以在此基础上实现更高级别的服务，例如同步，配置维护，组和命名。它的目的是易于编程，并使用一种数据模型风格类似于文件系统的目录树结构。它是Java语言编写的，且提供Java和C的客户端。

总所周知，协调服务很难取得正确的值。它们特别容易出现竞争条件和死锁。ZooKeeper的目的是减少开发分布式应用的责任，避免其从零开始发开一套协调服务。

# Design Goals

**ZooKeeper是简单的。**ZooKeeper允许分布式进程相互协调，通过共享一个类似于标准的文件系统的层次命名空间。命名空间包含数据寄存器，称为znodes。这些znodes类似于文件和目录。不像一个典型的文件系统，这是专为存储而设计的。ZooKeeper的数据是保存在内存中，也就是说，可以实现高吞吐量和低延迟。

ZooKeeper的实现注重于高性能、高可用性、严格有序的访问。在性能方面的优势使它可以用于大型的、分布式系统。可靠性方面避免它成为一个单点故障。严格的排序意味着可以在客户端实现复杂的同步原语。

ZooKeeper是可复制的。像它协调的分布式进程，ZooKeeper本身也是被设计成跨一组主机进行复制备份。

![](http://zookeeper.apache.org/doc/current/images/zkservice.jpg)

组成ZooKeeper的服务器都必须知道彼此。它们在内存中保持状态的镜像，以及持久存储事务日志和快照。只要大部分的服务器是可用的，ZooKeeper将正常提供服务。

客户端连接到一台ZooKeeper服务器。客户端维护一个TCP连接，通过连接发送请求，得到响应，得到watch事件，并发送心跳。如果TCP连接中断，客户端将尝试连接到不同的服务器。

ZooKeeper是有序的。ZooKeeper通过一个反映ZooKeeper事务顺序的数字，给每个更新进行标记。随后的操作可以使用顺序来实现更高级别的抽象，如同步原语。

ZooKeeper是很快的。这主要体现在“读取主导”的工作负载。ZooKeeper应用程序运行在数千台机器，它表现的最好的场景是读写比在10:1左右。

# Data model and the hierarchical namespace

ZooKeeper提供的命名空间与标准文件系统极其相似。名字是路径元素组成的序列，以斜杠"/"分隔。每一个ZooKeeper命名空间中的节点由一个路径进行标识。

![](http://zookeeper.apache.org/doc/current/images/zknamespace.jpg)

# Nodes and ephemeral nodes

不同于标准文件系统，ZooKeeper命名空间中的每个节点都有与它相关的数据以及子节点。这就像有一个文件系统，允许一个文件也可以是一个目录。（ZooKeeper是用来储存协调数据：状态信息、配置、位置信息等，使数据存储在每个节点通常是很小的，在字节范围。）我们使用术语znode来代表我们谈论的是ZooKeeper数据节点。

znodes保持数据结构（包括数据修改、ACL变化、时间戳的版本号）来允许缓存验证和协调更新。每次znode的数据变化，版本号的增加。例如，每当客户端检索数据时，它也接收数据的版本信息。

数据存储在一个命名空间中的每个znode，且读写都是原子性的。读行为可以得到znode关联的所有的数据字节；写行为会替换所有的数据。每个节点有一个访问控制列表（ACL），限制谁可以做什么。

ZooKeeper也有临时节点的概念。这些znodes与创建它们的会话共存亡。当会话结束，znode会被删除。

# Conditional updates and watches

ZooKeeper支持watch的概念。客户端可以针对一个znodes设置watch。当znode的变化时，watch会被触发并删除。当watch被触发，客户端接收到一个信息说znode已改变。如果客户端和一个ZooKeeper服务器之间的连接被破坏，客户端将收到一个本地通知。

# Guarantees

ZooKeeper非常的快且简单。由于它的目标是成为一个用于建设更复杂服务的基础，如同步，它提供了一套保证。如下：

* 顺序一致性：保证更新将按客户端发送的顺序来应用。
* 原子性：更新要么成功，要么失败。
* 单系统镜像：客户端将会看到相同的服务器视图，不管它连的是哪个服务器。
* 可靠性：一旦更新被应用，它将被持久化，直到客户端来修改它。
* 及时性：系统的客户端视图在一定的时间范围内保证是最新的。

# Simple API

ZooKeeper的设计目的之一是提供一个非常简单的编程接口。它提供如下操作：

* create：创建一个znode
* delete：删除一个znode
* exists：检测znode是否存在
* get data：从znode查询数据
* set data：向znode写入数据
* get children：查询一个节点的子节点
* sync：等待数据被传播

# Implementation

下图显示了ZooKeeper服务的高层组件。除了Request Processor ，其他组成ZooKeeper的每个服务器都会复制其每个组件的副本。

![](http://zookeeper.apache.org/doc/current/images/zkcomponents.jpg)

备份的数据库是包含整个数据树的内存数据库。将更新记录到磁盘以获取可恢复性，写操作会被序列化到磁盘，然后才应用到内存数据库。

每个ZooKeeper服务器都为客户端服务。客户端连接到一个服务器以提交请求。读请求从每个服务器数据库的本地副本提供数据。更改服务状态、写请求由一致性协议进行处理。


作为一致性协议的一部分，客户端的所有写请求都将转发到单个服务器，称为leader。其余服务器被称为follower，它们从leader处接受消息提议，并同意消息传递。消息层负责失败leader的更新换代，并使得follower同步新的leader。


ZooKeeper使用自定义的原子消息协议。由于消息层是原子的，所以ZooKeeper可以保证本地副本不会发散。当leader收到写请求时，它会计算要应用写入时系统的状态，并将其转换为可描述此新状态的事务。

# Uses

ZooKeeper的编程接口是故意简单的。有了它，你可以实现更高层次的操作，如同步原语，组成员，所有权等。

# Performance

ZooKeeper是专为高性能设计的。 ZooKeeper在Yahoo! Research的开发团队的结果表明它确实如此。在读取超出数量写入的应用程序中，这是特别高的性能，因为写入涉及同步所有服务器的状态。（协调服务通常情况下读取数量超过写入次数。）

# 参考

[http://zookeeper.apache.org/doc/current/zookeeperOver.html](http://zookeeper.apache.org/doc/current/zookeeperOver.html)



