---
layout: post
title: '[RabbitMQ] Consumer Acknowledgements and Publisher Confirms'
date: 2017-03-11
author: Feihang Han
tags: RabbitMQ
---

# 简介

使用RabbitMQ这类消息中间件的系统往往是分布式的。由于被传递的消息不能确保达到节点或成功被处理，所以publicsher和consumer需要一个机制来保证投递和处理确认。RabbitMQ支持的一些消息协议提供了这些特性。本指导涵盖了AMQP 0-9-1中的特性，而其他协议（STOMP, MQTT等）也有相似的特性。

在AMQP 0-9-1中，从consumer到RabbtiMQ的投递处理确认被认为是consumer acknowledgements；从broker到publisher的确认则是一种协议的扩展，被称为puhlisher confirms。

# 投递确认（Consumer）

当RabbitMQ投递一个消息给consumer，它需要知道什么可以认为该消息被发送成功。而这个判断往往只有应用系统才能做出合理的决定。在AMQP 0-9-1中，当consumer注册后并使用basic.consume方法，或通过命令行使用basic.get方法，消息才被认为投递成功。

如果你更倾向于一个实例，或者一步一步的教程，你可以参考文章\[RabbitMQ\]Work Queues。

# 投递标识：Delivery Tags

当一个consumer注册后，RabbitMQ会使用basic.deliver方法进行投递消息。这个方法带上delivery tag，来唯一标识一个通道上的一次投递行为。因此delivery tag作用于每个通道。

Delivery tag是一个单调递增的整数，并由客户端库呈现。客户端库中确认投递的方法将Delivery tag作为一个参数。

# acknowledgement模型

当使用不同的acknowledgement模型时，RabbtiMQ确认一个消息被成功投递的时机也不同，可能在消息立马被发出后，或收到一个明确的（手动）acknowledgement。手动发送acknowledgement可以是正向的或负向的，并使用下列协议方法中的某一个：

* basic.ack被用于positive acknowledgement（ack）
* basic.nack被用于negative acknowledgement（nack）（这是RabbitMQ基于AMQP 0-9-1额外扩展的）
* basic.reject被用于negative acknowledgement，但是比basic.nack多一个限制

ack简单的通知RabbitMQ一个消息被投递了。使用basic.reject进行nack也有同样的效果，区别是ack认为消息被成功处理，而nack则认为消息没被成功处理，但是依旧应该被RabbitMQ删除。

# 批量acknowledgement

可以进行批量的手动ack来减少网络流量。把ack方法的multiple域的值设置为true即可。注意到basic.reject一直没有这个multiple域，那正是RabbitMQ引入basic.nack作为协议扩展的原因。

当multiple被设置成true时，RabbitMQ会确认所有小于等于该delivery tags的消息，该机制作用于每一个通道。例如，如果通道ch上有5/6/7/8四个未确认的消息，当一个delivery\_tag为8，且mutiple是true的确认到来时，所有tags从5到8的消息都会被确认。相反，如果multiple被设置成false，则5/6/7三个消息仍旧是未确认状态。

# QoS

由于消息投递是异步的，一个通道内往往会有多条消息处于未确认状态（in flight）。开发者需要限制这类消息数量，来避免消费者端的无界缓冲问题。使用basic.qos方法，设置一个prefecth count的值，可以解决这个问题。这个值，定义了一个通道内未确认消息数目的最大值。一旦未确认消息数达到了这个值，RabbitMQ将停止投递消息直到某个未确认消息被确认。

例如，某通道ch上有4个未确认消息，其delivery tags为5/6/7/8，该通道的prefetch被设置为4，RabbitMQ将不再忘该通道投递任何消息，直到有某个未确认消息被确认。当上述四个消息中的某一个被确认后，RabbitMq会投递下一个消息。

投递和消费者确认流程异步是很重要的一点。因此，当通道内以存在有未确认消息时，这时修改了prefetch的值，会引起一个竞争条件，导致短时间内该通道内存在超过prefecth数目的未确认消息。

# 消费者错误：双倍确认和未知标签

如果消费者确认同一条delivery tag多余一次，RabbitMQ会返回一个通道错误，比如PRECONDITION\_FAILED - unknown delivery tag 100。如果确认了一条未知delivery tag的消息也会有同样的通道异常产生。

# Publisher Confirms

使用标准的AMQP 0-9-1，保证消息不丢失的唯一方式即为使用事务 – 将channel设置为事务的，发布消息，然后提交。这种情况下，事务会变成没必要的重量级，且会将吞吐量降低到原来的1/250。生产者确认机制便是为了补救这类问题而引入的。它类似于在协议中已存在消费者确认机制。

为开启确认，生产者需要发送confirm.select方法。取决于no-wait是否被设置，broker将返回confirm.select-ok。一旦客户端在某个通道上使用了comfirm.select方法，那么这个通道处于confirm模式。一个事务通道不能被修改成confirm模式，反之同理。

一旦通道处于confirm模式下，broker和客户端都会对消息计数（在第一个confirm.select调用后，从1开始计数）。broker在处理消息后，通过在同一个channel中发送basic.ack来确认消息。basic.ack的delivery-tag域包含了此次确认的消息的序列号。broker也可以设置basic.ack中的multiple域，表明到这个序列号为止的消息（包含此序列号）都已经被处理。

```
Java中发送confirm.select的方法是channel.confirmSelect()，等待ack返回的方法是：channel.waitForConfirmsOrDie();
```

Java例子参考[这里](http://hg.rabbitmq.com/rabbitmq-java-client/file/default/test/src/com/rabbitmq/examples/ConfirmDontLoseMessages.java)。

# Negative Acknowledgment

异常情况下，如果broker无法成功处理消息，则不会返回basic.ack，而是返回basic.nack。再这个场景下，basic.nack里面各个域的含义和对应的basic.ack是一样的，除了requeue域需要被忽略。通过nack一条或多条消息，broker表明它无法处理这些消息，并且拒绝承担相应的责任；此时，客户端可以选择重发这些消息。

在channel被设置成confirm模式之后，接下来所有被发布的消息都会被ack或者nack一次。但是一条消息多久之后被确认是没有保证的。不会有消息被同时ack和nack。

只有在负责queue的Erlang进程发生内部错误发生时，basic.nack才会被发布出来。

# 消息何时被确认?

对于无法路由的消息，在exchange确认消息无法被路由到任何queue时（即返回空的queue列表时）， broker会发送一条confirm。如果发送消息时指定了mandatory，在basic.ack之前，basic.return会先被发送到客户端。basic.nack也是一样的。  
对于可被路由的消息，在消息被所有适合的queue接收后，basic.ack会被返回。对于路由到持久化队列中的持久化消息，接收意味着持久化到硬盘；对于镜像队列（mirrored queues），接收意味着所有镜像都已经接收了消息。

# 持久化消息的Ack 时延

持久化消息被路由到持久化队列时，只有在消息被持久化到硬盘之后，basic.ack才会被返回。当队列空闲，或者每隔一段特定时间（数百毫秒）（为了降低fsync\(2\)的调用次数），RabbitMQ才会将消息批量存储到硬盘。这意味着，在恒定负载下，basic.ack的延时会达到数百毫秒。为了提高吞吐量，强烈建议应用程序使用异步处理ack（当做stream）；或者批量发布消息，然后等待响应。具体的API因不同的客户端而异。

# 确认与确保送达

在消息被写入disk前，如果broker挂了，则持久化消息会被丢失。在特定情况下，这会导致broker的异常表现。  
例如，考虑如下情景（不开启confirm mode时）：

1. 某客户端发布一条持久化消息到持久化queue中
2. 某另一个客户端消费了queue中的这条消息（注意，消息和queue都是持久化的），但是还没有发送ack。
3. broker挂掉然后重启了（由于存在批量保存机制，此时消息还未来得及保存到磁盘）。
4. 消费客户端重新连接，并开始消费消息。

此时，消费客户端认为这条消息会被broker再次分发。但事实并非如此：重启导致broker丢失了消息。为了保证持久化，客户端应该使用确认机制。如果发送端的channel设置为确认模式，那么发送端就不会收到被丢失消息对应的ack（因为消息还没有被写入disk，ack不会被发送）。

# 限制

Delivery tag是个64位的long值，因此最大值为9223372036854775807。由于delivery tags作用域是通道，生产者或消费者在生产环境中很难溢出这个值。

# 参考

[https://www.rabbitmq.com/confirms.html](https://www.rabbitmq.com/confirms.html)

