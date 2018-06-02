# 简介

在这个章节，我们将把一个消息发送给多个消费者，即发布订阅模型。

为了更好的说明这个模型，我们将打造一个简易日志系统。它包括两部分，第一个部分发出日志，第二部分接受并打印日志。

在这个日志系统中，有多个消费者程序都将接受日志消息，其中一个消费者把日志保存到磁盘，另外一个消费者把日志打印到屏幕。因此，被发布的日志消息将被广播给所有的消费者。

# 交换机

交换机负责从生产者接受消息，并根据一定的规则把消息推送到指定队列。

![](https://www.rabbitmq.com/img/tutorials/exchanges.png)

交换机支持多种模式：

* direct
* topic
* headers
* fanout

我们先讨论fanout模式。通过以下代码申明一个fanout模式的交换机。

```
channel.exchangeDeclare("logs", "fanout");
```

fanout模式交换机会把收到的消息广播到所有它知道的队列。

RabbtiMQ会默认创建很多amq.\*的交换机和默认（unnamed）交换机。这些交换机一般不会被使用到。

在之前的章节中，我们在发送消息时，使用了默认的交换机，即空字符串。消息通过指定的routekey被路由到相应的队列。

```
channel.basicPublish("", "hello", null, message.getBytes());
```

现在我们可以通过指定交换机名字来发送消息了。

```
channel.basicPublish( "logs", "", null, message.getBytes());
```

# 临时队列

也许你还记得以前我们使用的队列所指定的名称（hello和task\_queue）。对我们来说，能够列出一个队列是至关重要的。我们需要把Worker指向同一个队列。如果要在生产者和消费者之间共享队列，给队列命名是很重要的。

但这并不是我们的记录情况。我们想听到所有的日志消息，而不仅仅是其中的一个子集。我们也只对当前流动的信息感兴趣，而不是在旧的。为了解决这些问题，我们需要两件事。

首先，每当我们连接到RabbitMQ，我们需要一个新的空队列。要做到这一点，我们可以创建一个队列的随机名称，或者，甚至更好-让服务器为我们选择一个随机队列名称。

其次，一旦我们断开消费者的队列应该自动删除。

在java客户端，当我们使用无参函数queuedeclare\(\)，我们创建一个非持久的，独享的，自动删除的队列：

```
String queueName = channel.queueDeclare().getQueue();
```

这样，队列名是一个随机值，例如`amq.gen-JzTY20BRgKO-HjmUJj0wLg`。

# 绑定

![](https://www.rabbitmq.com/img/tutorials/bindings.png)

当我们已经创建了一个交换机和队列后，我们需要告诉交换机向哪个队列发送消息。交换机和队列之间的关系称为绑定。

```
channel.queueBind(queueName, "logs", "");
```

# 代码

![](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)

发送端把消息发送到了logs的交换机，而不是之前的默认交换机。由于交换机是fanout模式的，即使我们提供了routeKey，该值也会被直接忽略掉。

```
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.ConnectionFactory;

public class EmitLog {

    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] argv) throws Exception {

        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");

        String message = getMessage(argv);

        channel.basicPublish(EXCHANGE_NAME, "", null, message.getBytes());
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

接收端仍需要声明交换机，因为向一个不存在的交换机发送消息是被禁止的。如果没有队列被绑定到指定交换机，发往该交换机的消息会被丢弃掉，但是这正好满足了日志系统的需求。

```
import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogs {
    private static final String EXCHANGE_NAME = "logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
        String queueName = channel.queueDeclare().getQueue();
        channel.queueBind(queueName, EXCHANGE_NAME, "");

        System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties,
                    byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");
            }
        };
        channel.basicConsume(queueName, true, consumer);
    }
}
```

# 参考

[https://www.rabbitmq.com/tutorials/tutorial-three-java.html](https://www.rabbitmq.com/tutorials/tutorial-three-java.html)

