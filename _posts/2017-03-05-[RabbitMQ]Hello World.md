---
layout: post
title: 'RabbitMQ Hello World'
date: 2017-03-05
author: Feihang Han
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: RabbitMQ
---

# 简介

RabbitMQ是一个消息队列，主要目的是接收和发送消息。你可以将它简单的想象成一个邮局：当你将一个新建投递到一个邮箱时，你可以确保邮递员最终会将邮件送达你的收件人。RabbitMQ类似于邮箱、邮局和邮递员。

RabbitMQ和邮局主要的区别在于它不处理纸质邮件，而是接受、存储和发送二进制数据-消息。

RabbitMQ和通用消息中间件一样，有如下组件：

* 生产者
* 消息队列
* 消费者

在下面的例子里，我们写了两个JAVA程序。一个生产单个消息的生产者；一个接受并打印消息的消费者。

在下图中，P代表生产者，C代表消费者，中间部分代表消息队列。

![](https://www.rabbitmq.com/img/tutorials/python-one.png)

```
RabbitMQ支持很多协议，在这里我们将使用AMQP 0-9-1
```

# 发送端

下面的connection抽象了socket连接，并为我们处理了协议转换、鉴权等。我们连接到在本地消息代理（localhost），如果你想连接到其他消息代理，可以简单的指定其他IP地址。

```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class Send {
    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        String message = "Hello World!";
        channel.basicPublish("", QUEUE_NAME, null, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");
        channel.close();
    }
}
```

接下来创建一个通道，它是一个很多API开始工作的起点。然后声明一个队列，然后发送一个消息。

声明队列是幂等的，它只有当不存在的时候才会被创建。消息内容是一个byte数据，因此你可以在任何时候进行编码。

最后，关闭通道和连接。

# 接受端

`DefaultConsumer`是接口`Consumer`的默认实现，它帮助缓存了从服务端发来的消息。

```
import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class Recv {
    private final static String QUEUE_NAME = "hello";

    public static void main(String[] argv) throws Exception {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");
            }
        };
        channel.basicConsume(QUEUE_NAME, true, consumer);
    }
}
```

接收端也声明了队列。我们可能启动接收端早于发送端，因此需要确保队列已存在。

我们告诉服务器从队列分发消息到这里。由于是同步消费，我们提供了一个callback，即DefaultConsumer。这个类缓存了消息直到我们使用为止。

# 参考

[https://www.rabbitmq.com/tutorials/tutorial-one-java.html](https://www.rabbitmq.com/tutorials/tutorial-one-java.html)

