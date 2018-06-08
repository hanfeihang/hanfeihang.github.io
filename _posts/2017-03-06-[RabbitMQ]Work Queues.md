---
layout: post
title: '[RabbitMQ] Work Queues'
date: 2017-03-06
author: Feihang Han
tags: RabbitMQ
---

# 简介

在第一个教程中，我们实现了一个程序进行发送和接受消息，在这一章节，我们将创建一个工作队列，用于在多个Workers之间分配耗时的任务。

Work Queues的主要目的是避免立即执行一个资源密集型任务，并等待它完成。相反，我们安排这些任务稍后被处理。我们将任务封装为消息并将其发送给队列。后台运行的工作进程将消费任务并最终执行作业。这些Workers将一起完成这些任务。

![](https://www.rabbitmq.com/img/tutorials/python-two.png)

这一概念在Web应用程序是特别有用的，它不可能在一个短时间的HTTP请求内处理复杂的任务。

# 准备

与之前发送简单的消息不同，这次我们将发送消息用于复杂任务。通过使用Thread.sleep\(\)，我们假装工作线程处于忙碌状态。每一个dot代表一秒钟的忙碌，即Hello...表示这个任务将忙碌3秒。

```java
private static String getMessage(String[] strings) {
    if (strings.length < 1)
        return "Hello World!";
    return joinStrings(strings, " ");
}

private static String joinStrings(String[] strings, String delimiter) {
    int length = strings.length;
    if (length == 0)
        return "";
    StringBuilder words = new StringBuilder(strings[0]);
    for (int i = 1; i < length; i++) {
        words.append(delimiter).append(strings[i]);
    }
    return words.toString();
}
```

```java
final Consumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
        byte[] body) throws IOException {
        String message = new String(body, "UTF-8");

        System.out.println(" [x] Received '" + message + "'");
        try {
            doWork(message);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println(" [x] Done");
        }
    }
};

private static void doWork(String task) throws InterruptedException {
    for (char ch : task.toCharArray()) {
    if (ch == '.')
        Thread.sleep(1000);
    }
}
```

# 轮训分发

执行

```cmd
shell3$ java -cp $CP NewTask
First message.
shell3$ java -cp $CP NewTask
Second message..
shell3$ java -cp $CP NewTask
Third message...
shell3$ java -cp $CP NewTask
Fourth message....
shell3$ java -cp $CP NewTask
Fifth message.....
```

结果

```cmd
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'First message.'
 [x] Received 'Third message...'
 [x] Received 'Fifth message.....'
```

```cmd
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Second message..'
 [x] Received 'Fourth message....'
```

# 消息确认

如果有个花费较长时间的任务中途失败会发生什么？之前的代码中，一旦RabbitMQ分发完消息后，它立即删除该消息。在这种场景下，如果你杀死了一个worker，我们将丢失这个worker正在处理的消息。我们也将丢失所有被分发到这个worker，但是还没被处理的消息。

我们不希望这种事情发送。如果一个worker挂了，我们希望这个任务被重新分发给其他worker。

为了确保消息不丢失，RabbitMQ支持消息确认机制。消费者在成功消费某个消息后，发送回一个ACK消息，告诉RabbitMQ该消息已被成功消费，然后RabbitMQ将删除该消息。

如果消费者挂了（its channel is closed, connection is closed, or TCP connection is lost），没有发送回ACK消息，那么RabbitMQ会重新将该消息放到队列。如果此时有其他消费者在线，则RabbitMQ将把该消息立马投递给其他正常工作的消费者。

消息确认开关默认是打开的。在之前的例子里，我们通过`autoAck=true`显示的关闭了这个开关。是时候打开这个开关了。

```java
channel.basicQos(1); // accept only one unacked message at a time (see below)
final Consumer consumer = new DefaultConsumer(channel) {
    @Override
    public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
            byte[] body) throws IOException {
        String message = new String(body, "UTF-8");

        System.out.println(" [x] Received '" + message + "'");
        try {
            doWork(message);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println(" [x] Done");
            channel.basicAck(envelope.getDeliveryTag(), false);
        }
    }
};
boolean autoAck = false;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
```

# 消息持久化

当RabbitMQ退出或者宕机的时候，消息或者队列将被丢失。因此我们需要确保队列和消息都被持久化。

第一步，队列持久化。

```java
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
```

尽管代码很正确，但是由于之前一定定义过该队列了，RabbitMQ不允许你用不同的参数重新定义已存在的队列，因此会返回一个错误。我们可以选择一个新的队列名。

第二步，消息持久化。

```java
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
```

关于消息持久性的注记

将消息标记为持久性并不能完全保证消息不会丢失。虽然它告诉RabbitMQ保存信息到磁盘上，还有一个短的时间窗口时，RabbitMQ已经接受信息并没有保存它。另外，RabbitMQ不做fsync\(2\)每一个消息--它可能只是保存到缓存并没有真正写入到磁盘。持久性保证不强，但它是足够的简单的任务队列。如果您需要更强的担保，那么您可以使用[Publisher Confirms](https://www.rabbitmq.com/confirms.html)。

# 公平调度

你可能已经注意到，调度仍然没有如我们想要的那样工作。例如，在两个工人的情况下，奇数的消息是复杂的，偶数的消息是简单的，那么一个工人将不断忙碌，而另一个将几乎没有任何工作。RabbitMQ不知道发生了什么事，仍将消息发送均匀。

这是因为RabbitMQ只是在消息进入队列时调度消息。它不在意某个消费者未确认的消息数，盲目的发送第N条消息给第N个消费者。

![](https://www.rabbitmq.com/img/tutorials/prefetch-count.png)

通过设置`basicQos`的属性prefetchCount=1，RabbitMQ不会同时向一个消费者分发多余一条的消息，即RabbitMQ不会向消费者分发消息，直到前面一条消息被处理掉并收到ACK消息。相反，它会把消息分发给不繁忙的消费者。

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

# 代码

NewWork.java

```java
import java.util.concurrent.TimeoutException;

import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.MessageProperties;

public class NewTask {
    private final static String TASK_QUEUE_NAME = "task_queue";

    public static void main(String[] argv) throws java.io.IOException, TimeoutException {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

        String message = getMessage(argv);

        channel.basicPublish("", TASK_QUEUE_NAME, MessageProperties.PERSISTENT_TEXT_PLAIN, message.getBytes());
        System.out.println(" [x] Sent '" + message + "'");

        channel.close();
        connection.close();
    }

    private static String getMessage(String[] strings) {
        if (strings.length < 1)
            return "Hello World!";
        return joinStrings(strings, " ");
    }

    private static String joinStrings(String[] strings, String delimiter) {
        int length = strings.length;
        if (length == 0)
            return "";
        StringBuilder words = new StringBuilder(strings[0]);
        for (int i = 1; i < length; i++) {
            words.append(delimiter).append(strings[i]);
        }
        return words.toString();
    }
}
```

Worker.java

```java
import java.io.IOException;

import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.Envelope;

public class Worker {
    private final static String TASK_QUEUE_NAME = "task_queue";

    public static void main(String[] argv) throws Exception {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        final Channel channel = connection.createChannel();

        channel.queueDeclare(TASK_QUEUE_NAME, false, false, false, null);
        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        channel.basicQos(1); // accept only one unack-ed message at a time (see
                                // below)

        final Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
                String message = new String(body, "UTF-8");

                System.out.println(" [x] Received '" + message + "'");
                try {
                    doWork(message);
                } finally {
                    System.out.println(" [x] Done");
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        boolean autoAck = false;
        channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
    }

    private static void doWork(String task) {
        for (char ch : task.toCharArray()) {
            if (ch == '.') {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        }
    }
}
```

# 参考

[https://www.rabbitmq.com/tutorials/tutorial-two-java.html](https://www.rabbitmq.com/tutorials/tutorial-two-java.html)

