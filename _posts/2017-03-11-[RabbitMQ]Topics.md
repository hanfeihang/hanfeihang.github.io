# 简介

虽然我们通过使用direct交换机提升了日志系统，但它仍旧有限制，无法基于多个条件进行路由。

消费者可能需要基于日志级别和日志来源进行订阅日志。日志系统将变得十分灵活。

# Topic交换机

topic交换机使用的routingKey由一组字母组成，由逗号分隔。字母可以是任意的，但是一般表明消息的某种特性，比如"stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit"。routingKey长度为255字节。

类似于direct交换机，使用topics类型交换机时，消息会传递给队列，其绑定键与消息的路由键进行匹配，但是有两个重要的差异：

* \* 号代表一个单词
* \# 号代表零个或多个单词

![](https://www.rabbitmq.com/img/tutorials/python-five.png)

在这个例子里，我们将发送代表动物的消息。消息的routingKey由三部分组成"&lt;speed&gt;.&lt;colour&gt;.&lt;species&gt;"。

Q1的绑定键是"\*.orange.\*"，Q2的绑定键是"\*.\*.rabbit" and "lazy.\#"。

我们可以把上述绑定关系理解为：

* Q1对黄色动物感兴趣
* Q2对兔子或者懒懒的动物感兴趣

绑定键为"quick.orange.rabbit"或"lazy.orange.elephant"的消息将会路由给两个队列；绑定键为"quick.orange.fox"的消息将会路由给Q1；绑定键为"lazy.brown.fox"的消息将会路由给Q2；绑定键为"lazy.pink.rabbit"的消息将会路由给Q2，有且只有一次，虽然它同时匹配了Q2的两条绑定键；绑定键为"quick.brown.fox"的消息将会被丢弃。

如果我们不按常理出牌，发送绑定键为"orange"或"quick.orange.male.rabbit"的消息，那么这些消息会被丢弃掉；而绑定建为"lazy.orange.male.rabbit"的消息会被路由到Q2，即使它有四个单词。

```
topic交换机可以表现的类似于其他类型的交换机。
当一个队列使用绑定建"#"进行绑定时，那么它将收到所有消息，类似于fanout交换机。
当一个队列不使用通配符("#"/"*")进行绑定时，那么它将表现的类似于direct交换机。
```

# 代码

我们设计日志系统的绑定建为"&lt;facility&gt;.&lt;severity&gt;"。

EmitLogTopic.java

```
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;

public class EmitLogTopic {

    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) {
        Connection connection = null;
        Channel channel = null;
        try {
            ConnectionFactory factory = new ConnectionFactory();
            factory.setHost("localhost");

            connection = factory.newConnection();
            channel = connection.createChannel();

            channel.exchangeDeclare(EXCHANGE_NAME, "topic");

            String routingKey = getRouting(argv);
            String message = getMessage(argv);

            channel.basicPublish(EXCHANGE_NAME, routingKey, null, message.getBytes("UTF-8"));
            System.out.println(" [x] Sent '" + routingKey + "':'" + message + "'");

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (connection != null) {
                try {
                    connection.close();
                } catch (Exception ignore) {
                }
            }
        }
    }

    private static String getRouting(String[] strings) {
        if (strings.length < 1)
            return "anonymous.info";
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

ReceiveLogsTopic.java

```
import com.rabbitmq.client.*;

import java.io.IOException;

public class ReceiveLogsTopic {

    private static final String EXCHANGE_NAME = "topic_logs";

    public static void main(String[] argv) throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost("localhost");
        Connection connection = factory.newConnection();
        Channel channel = connection.createChannel();

        channel.exchangeDeclare(EXCHANGE_NAME, "topic");
        String queueName = channel.queueDeclare().getQueue();

        if (argv.length < 1) {
            System.err.println("Usage: ReceiveLogsTopic [binding_key]...");
            System.exit(1);
        }

        for (String bindingKey : argv) {
            channel.queueBind(queueName, EXCHANGE_NAME, bindingKey);
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

[https://www.rabbitmq.com/tutorials/tutorial-five-java.html](https://www.rabbitmq.com/tutorials/tutorial-five-java.html)

