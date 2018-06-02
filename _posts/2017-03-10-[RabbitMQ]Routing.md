---
layout: post
title: 'RabbitMQ Routing'
date: 2017-03-10
author: Feihang Han
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: RabbitMQ
---

# 简介

在这个章节中，我们新增一个特性——支持订阅消息的某个子集。比如我们可以把严重错误的日志记录到日志文件，而同时在屏幕上打印出所有日志。

# 绑定

绑定是指交换机和队列之间的联系。简而言之，就是指队列对某个交换机的消息感兴趣。

```
channel.queueBind(queueName, EXCHANGE_NAME, "");
```

我们可以对绑定设置第三个参数`routingKey`。

```
channel.queueBind(queueName, EXCHANGE_NAME, "black");
```

请注意，routingKey的作用受限于交换机的类型，如果是fanout类型的交换机，则routingKey不会生效。

# Direct exchange

现在我们需要新增一个特性，允许日志系统根据日志级别进行过滤消息。fanout类型交换机缺少灵活性，它只能进行盲目的广播。

使用direct类型交换机时，消息会传递给队列，其绑定键与消息的路由键完全匹配。

![](https://www.rabbitmq.com/img/tutorials/direct-exchange.png)

从上图可以看到，X交换机有两个队列与之绑定。队列Q1的绑定键是orange，队列Q2的绑定建是black和green。

在这种配置下，一个routingKey为orange的消息会被路由到Q1，而routingKey为black或green的消息会被路由到Q2。

# 多重绑定

![](https://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)

多个队列使用同一个的绑定键与一个交换机进行绑定也是可行的。在这个例子里，Q1和Q2都使用black作为绑定建进行绑定，那么这个direct交换机将会表现的类似于fanout交换机。一个routingKey为black的消息将会同时分发给Q1和Q2。

# 发布日志

我们使用direct交换机来实现日志系统，把日志级别作为routingKey，那么消费者可以有选择的进行日志的接受。

```
channel.exchangeDeclare(EXCHANGE_NAME, "direct");
channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes());
```

其中，`severity`可以是info、warning、error等。

# 订阅

我们将基于日志级别severity进行队列的绑定。

```
String queueName = channel.queueDeclare().getQueue();

for(String severity : argv){    
  channel.queueBind(queueName, EXCHANGE_NAME, severity);
}
```

# 代码

![](https://www.rabbitmq.com/img/tutorials/python-four.png)

EmitLogDirect.java

```
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

public class EmitLogDirect {

    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] argv) throws Exception {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "direct");

        String severity = getSeverity(argv);
        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, severity, null, message.getBytes("UTF-8"));
        System.out.println(" [x] Sent '" + severity + "':'" + message + "'");

        channel.close();
        connection.close();
    }

    private static String getSeverity(String[] strings) {
        if (strings.length < 1)
            return "info";
        return strings[0];
    }

    private static String getMessage(String[] strings) {
        if (strings.length < 2)
            return "Hello World!";
        return joinStrings(strings, " ", 1);
    }

    private static String joinStrings(String[] strings, String delimiter, int startIndex) {
        int length = strings.length;
        if (length == 0)
            return "";
        if (length < startIndex)
            return "";
        StringBuilder words = new StringBuilder(strings[startIndex]);
        for (int i = startIndex + 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}
```

ReceiveLogsDirect.java

```
import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogsDirect {

    private static final String EXCHANGE_NAME = "direct_logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "direct");
        String queueName = channel.queueDeclare().getQueue();

        if (argv.length < 1) {
            System.err.println("Usage: ReceiveLogsDirect [info] [warning] [error]");
            System.exit(1);
        }

        for (String severity : argv) {
            channel.queueBind(queueName, EXCHANGE_NAME, severity);
        }
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + envelope.getRoutingKey() + "':'" + message + "'");
            }
        };
        channel.basicConsume(queueName, true, consumer);
    }
}
```

# 参考

[https://www.rabbitmq.com/tutorials/tutorial-four-java.html](https://www.rabbitmq.com/tutorials/tutorial-four-java.html)

