---
layout: post
title: '[RabbitMQ] RPC'
date: 2017-03-10
author: Feihang Han
tags: RabbitMQ
---

# 简介

在RPC模式下，我们发送一个消息，并等待远程的服务的响应。

在这章节下，我们将打造一个PRC系统：一个客户端和一个可伸缩的服务端。其中服务端返回斐波那契数列。

# 客户端接口

RPC服务暴露一个叫call\(\)的方法。

```java
FibonacciRpcClient fibonacciRpc = new FibonacciRpcClient();   
String result = fibonacciRpc.call("4");
System.out.println( "fib(4) is " + result);
```

# 回调队列

使用RabbtiMQ进行RPC调用是相当简单的。客户端发送一个请求消息，服务端返回一个响应消息。为了能收到响应消息，我们需要在发送请求时携带一个callback队列地址。

```java
callbackQueueName = channel.queueDeclare().getQueue();

BasicProperties props = new BasicProperties
                            .Builder()
                            .replyTo(callbackQueueName)
                            .build();

channel.basicPublish("", "rpc_queue", props, message.getBytes());

// ... then code to read a response message from the callback_queue ...
```

```
消息属性
AMQP协议预定义了14个属性，其中大部分很少被使用到，除了以下几个比较常用：
deliveryMode：标记一个消息为persistent或transient
contentType：描述编码的媒体类型，比如application/json
replyTo：描述一个回调队列
correlationId：标记该RPC响应与哪个请求对应的唯一ID
```

# Correlation Id

在上面的场景中，一个客户端使用同一个回调队列进行RPC调用时，在收到响应时无法区分哪个响应对应哪个请求。correlationId属性可以很好的解决这个问题。我们可以根据回调消息中的correlationId属性进行请求与响应的匹配。如果我们收到一个未知的correlationId，那么我们可以很放心的丢弃这条消息。

你可能会问，为什么我们应该忽略回调队列中的未知信息，而不是直接失败？这是由于在服务器端存在竞争条件的可能性。虽然概率极低，但是RPC服务器很有可能在发送完结果后，还没发送确认消息前宕机了。如果这种情况发生，再次重启的RPC服务器会再次处理之前那个请求。这就是为什么在客户端必须合理的处理重复的响应。RPC服务应该是幂等。

# 总结

![](https://www.rabbitmq.com/img/tutorials/python-six.png)

RPC模式流程：

1. 客户端启动时，创建一个匿名排他的回调队列。
2. 客户端在发送RPC请求时，需要设置两个属性：replyTo表示回调队列；correlationId表示请求的唯一ID。
3. 请求发往RPC队列。
4. RPC服务等待消费RPC队列上的消息。当消息出现时，PRC服务进行工作，并且把结果发往replyTo指定的队列。
5. 客户端在回调队列等待响应消息。当消息出现时，它检查correlationId属性。如果这个属性值符合请求时携带的那个值，程序将消费这个响应。

# 代码

斐波那契数列函数

```java
private static int fib(int n) {
    if (n == 0) return 0;
    if (n == 1) return 1;
    return fib(n-1) + fib(n-2);
}
```

RPCServer.java

```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Consumer;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Envelope;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

public class RPCServer {

    private static final String RPC_QUEUE_NAME = "rpc_queue";

    private static int fib(int n) {
        if (n == 0)
            return 0;
        if (n == 1)
            return 1;
        return fib(n - 1) + fib(n - 2);
    }

    public static void main(String[] argv) {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        Connection connection = null;
        try {
            connection = factory.newConnection();
            final Channel channel = connection.createChannel();

            channel.queueDeclare(RPC_QUEUE_NAME, false, false, false, null);

            channel.basicQos(1);

            System.out.println(" [x] Awaiting RPC requests");

            Consumer consumer = new DefaultConsumer(channel) {
                @Override
                public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                        byte[] body) throws IOException {
                    AMQP.BasicProperties replyProps = new AMQP.BasicProperties.Builder()
                            .correlationId(properties.getCorrelationId()).build();

                    String response = "";

                    try {
                        String message = new String(body, "UTF-8");
                        int n = Integer.parseInt(message);

                        System.out.println(" [.] fib(" + message + ")");
                        response += fib(n);
                    } catch (RuntimeException e) {
                        System.out.println(" [.] " + e.toString());
                    } finally {
                        channel.basicPublish("", properties.getReplyTo(), replyProps, response.getBytes("UTF-8"));

                        channel.basicAck(envelope.getDeliveryTag(), false);
                    }
                }
            };

            channel.basicConsume(RPC_QUEUE_NAME, false, consumer);

            // loop to prevent reaching finally block
            while (true) {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException _ignore) {
                }
            }
        } catch (IOException | TimeoutException e) {
            e.printStackTrace();
        } finally {
            if (connection != null)
                try {
                    connection.close();
                } catch (IOException _ignore) {
                }
        }
    }
}
```

RPCClient.java

```java
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.DefaultConsumer;
import com.rabbitmq.client.AMQP;
import com.rabbitmq.client.Envelope;

import java.io.IOException;
import java.util.UUID;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.TimeoutException;

public class RPCClient {

    private Connection connection;
    private Channel channel;
    private String requestQueueName = "rpc_queue";
    private String replyQueueName;

    public RPCClient() throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");

        connection = factory.newConnection();
        channel = connection.createChannel();

        replyQueueName = channel.queueDeclare().getQueue();
    }

    public String call(String message) throws IOException, InterruptedException {
        final String corrId = UUID.randomUUID().toString();

        AMQP.BasicProperties props = new AMQP.BasicProperties.Builder().correlationId(corrId).replyTo(replyQueueName)
                .build();

        channel.basicPublish("", requestQueueName, props, message.getBytes("UTF-8"));

        final BlockingQueue<String> response = new ArrayBlockingQueue<String>(1);

        channel.basicConsume(replyQueueName, true, new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
                if (properties.getCorrelationId().equals(corrId)) {
                    response.offer(new String(body, "UTF-8"));
                }
            }
        });

        return response.take();
    }

    public void close() throws IOException {
        connection.close();
    }

    public static void main(String[] argv) {
        RPCClient fibonacciRpc = null;
        String response = null;
        try {
            fibonacciRpc = new RPCClient();

            System.out.println(" [x] Requesting fib(30)");
            response = fibonacciRpc.call("30");
            System.out.println(" [.] Got '" + response + "'");
        } catch (IOException | TimeoutException | InterruptedException e) {
            e.printStackTrace();
        } finally {
            if (fibonacciRpc != null) {
                try {
                    fibonacciRpc.close();
                } catch (IOException _ignore) {
                }
            }
        }
    }
}
```

# 参考

[https://www.rabbitmq.com/tutorials/tutorial-six-java.html](https://www.rabbitmq.com/tutorials/tutorial-six-java.html)

