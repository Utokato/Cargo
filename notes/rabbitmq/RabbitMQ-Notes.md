# RabbitMQ Notes

## JMS VS AMQP

> The **Advanced Message Queuing Protocol** (AMQP) is an open standard application layer protocol for message-oriented middleware. The defining features of AMQP are message orientation, queuing, routing (including point-to-point and publish-and-subscribe), reliability and security.
>
> AMQP mandates the behavior of the messaging provider and client to the extent that implementations from different vendors are interoperable, in the same way as SMTP, HTTP, FTP, etc. have created interoperable systems. Previous standardizations of middleware have happened at the API level (e.g. JMS) and were focused on standardizing programmer interaction with different middleware implementations, rather than on providing interoperability between multiple implementations. Unlike JMS, which defines an API and a set of behaviors that a messaging implementation must provide, AMQP is a wire-level protocol. A wire-level protocol is a description of the format of the data that is sent across the network as a stream of bytes. Consequently, any tool that can create and interpret messages that conform to this data format can interoperate with any other compliant tool irrespective of implementation language.

上面的这两段话，摘自维基百科对于 AMQP 的解释，其中将 AMQP 和 JMS 进行了对比。最大的不同在于，JMS 是定义了 API 层面的一系列规范，在 Java 中根据这些规范就可以进行消息的处理，与 JDBC 相似，这些规范都是存在于 Java 的生态体系中，每一个产品都必须去实现所有的 JMS 规范，当然离开了 Java 这些产品也就会失效。著名的 ActiveMQ 就是这么一款产品，当然 ActiveMQ 不仅实现了 JMS 规范，也实现了 AMQP 协议，这意味这 ActiveMQ 也可以在其他的语言中大展身手。

与 JMS 不同，AMQP 是一个协议，与大名鼎鼎的 HTTP 相似，都属于应用层协议，可以通过 [OSI model](https://en.wikipedia.org/wiki/OSI_model) 来详细了解网络通信协议的分层。既然是一个通用协议，那么就不会局限于某一种语言的生态中，基于协议实现的消息队列产品，具有天然性的跨语言支持，如 RabbitMQ。

同时，在 AMQP 中定义一些规范的术语，这些术语之间共同作用，描绘了整个消息的声明周期，如：消息如何进入队列，消费者如何绑定队列等等。这些过程与编程语言是没有关系的。



> 什么是协议？协议，是一个官方的称呼，本质上就是一套标准，约束着特定领域的活动准则，如 HTTP 协议，描述了应用层文本传输的标准动作，TCP 协议描述了如何可靠的保证内容的传输。协议也是由人、组织去制定的，是满足特定时期的一些规范，可能时过境迁，有些协议中制定的规范不能满足当前的需求，就会重新去制定或修改完善。



### AMQP 术语解释

- Message：消息是不具名的，它由消息头消息体组成。消息体是不透明的，而消息头则由 一系列可选属性组成，这些属性包括：routing-key (路由键)、priority (相对于其他消息的优先 权)、delivery-mode (指出消息可能持久性存储)等；
- Publisher / Producer / Provider：消息的生产者。也是一个向交换器(或队列)发布消息的客户端应用程序；
- Consumer：消息的消费者。表示一个从消息队列中取得消息的客户端应用程序；
- Exchange：交换器。用来接收生产者发送的消息并将这些消息路由(根据路由键进行分发)给服务器中的队列，交换器本身不具备存储消息的能力；
- Binding：绑定。用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器和消息 队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表；
- Routing-key：路由键。根据路由键决定消息该投递到哪个队列， 队列通过路由键绑定到交换器。 消息发送到 MQ 服务器时，消息将拥有一个路由键，即便是空的，也会将其和绑定使用的路由键进行匹配。 如果相匹配，消息将会投递到该队列；如果不匹配，消息将会进入黑洞；
- Queue：消息队列。用来保存消息直到发送给消费者。它是消息的容器，也是**消息的终点**。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者链接到这个队列将其取 走；
- Connection：链接。生产者或消费者与服务器之间构建起的 TCP 链接；
- Channel：信道。既然已经通过 TCP 构建起了链接，为什么还需要 Channel ？复用，建立和断开 TCP 通道是需要消耗一些资源，Channel 复用了 TCP 通道，更加轻量级，减少了资源的损耗。

还有一些其他的术语，可能不是通用的，但是在 RabbitMQ 中却是很重要的：

- Virtual Host：虚拟主机，标识了一批交换器、消息队列和相关对象。每个 vhost 本质上就是一个 mini 版的 RabbitMQ 服务器，拥有 自己的队列、交换器、绑定和权限机制。如果把 RabbitMQ 和 MySQL 类别，虚拟主机就好比是一个 MySQL 中的数据库；
- Borker：消息队列服务器实例。




## Code in Java client

### 简单队列

![simple-queue](./imgs/simple-queue.png)

简单队列中，生产者与消费者一一对应，如果此时想有多个消费者共同消费队列中的消息，就不能实现。

```java
/**
*  获取 rabbitmq 连接
*  工具类，后面获取 rabbitmq 的连接都使用该方法
*/
public class ConnectionUtils {
    public static Connection getConnection() throws IOException, TimeoutException {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost(""); // 设置rabbitmq服务器的ip
        factory.setPort(5672);// 设置rabbitmq服务器的服务端口
        factory.setVirtualHost(""); // 设置rabbitmq服务器的虚拟host，类似于SQL中的数据库
        factory.setUsername(""); // 设置用户名
        factory.setPassword(""); // 设置密码
        return factory.newConnection();
    }
}
```

```java
/**
*  简单队列，发送消息
*/
public class SendMsg {
    private static final String QUEUE_NAME = "test_simple_queue";
    public static void main(String[] args) throws IOException, TimeoutException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
         	/**
         	 * 声明队列，rabbitmq 中的操作大部分都是基于 channel 来完成的
         	 * 
         	 * @param queue			String 队列名
             * @param durable 		boolean 是否持久化 
             * @param exclusive 	boolean 是否独占
             * @param autoDelete	boolean 是否自动删除
             * @param arguments 	Map 其他参数
         	 */
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            String msg = "Hello, simple queue!";
            /**
         	 * @param exchange 交换器 - 简单队列和工作队列中都没有使用交换器
			 * @param routingKey 路由键 - 路由键指向了队列的队名
			 * @param props 参数
			 * @param body 真正的消息，以byte[]方式进行传递
         	 */
            channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
            System.out.println("-- send msg: " + msg);
        }
    }
}
```

```java
/**
*  简单队列，接受消息
*/
public class ReceiveMsg {
    private static final String QUEUE_NAME = "test_simple_queue";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection conn = ConnectionUtils.getConnection();
        Channel channel = conn.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) 
                throws IOException {
                String msg = new String(body, StandardCharsets.UTF_8);
                System.out.println("-- receive msg: " + msg);
            }
        };
        channel.basicConsume(QUEUE_NAME, true, consumer); // 监听队列
    }
}
```



### 工作队列

![work-queue](./imgs/work-queue.png)

一般而言，生产者产生数据可能是毫不费力的，而消费者可能由于业务复杂性的诸多原因，导致消费数据的能力不足，如果采用简单队列的模式，会出现大量消息积压在消费队列中。为了解决简单队列的不足，出现了工作队列。

多个消费者共同消费一个队列中的数据，队列向消费者发送数据时所采取的策略有两种：

- 轮询分发
- 公平分发

**轮询分发**是指，队列并不会根据消费者的处理能力去动态的调节消息的发送，消息总是你一个我一个的方式发送给多个消费者。如果其中有的消费者处理能力强，而有的消费者处理能力弱，就会出现能力弱的消费者疲于处理消息，而能力强的消费者处于空闲的状态。

在消息分发的过程中，队列只会一股脑的分发，并不知道消费者对消息处理的状态，如果此时有一个消费者异常了，队列还是继续发送消息，这种情况下，就会出现消息的丢失。为什么会出现这种状况？这是由于队列与消费者之间是没有通信的，消费者通过自动应答(`autoAck=true`)来通知队列，这种情况下可能出现消息的丢失。

如何解决消息的丢失问题？改变消息的应答模式，即`autoAck=false`，使用手动应答的机制。如果有消费者异常，队列就会将消息发送给其他消费者去处理。在手动应答的机制下，消息正确抵达消费者时，会向队列返回一个应答，队列接受到手动返回的应答消息时，才会从内存中删除消息。

保证了消费者在异常时不丢失数据的状况下，当队列服务异常时，也会出现消息的丢失，此时需要设置消息队列的持久化机制。需要通过`durable = true`来开启消息的持久化。

```java
/**
 * 轮询分发 round-robin
 */
public class SendMsg {
    private static final String QUEUE_NAME = "test_work_queue";
    public static void main(String[] args) throws IOException, TimeoutException, 
    	InterruptedException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            for (int i = 0; i < 50; i++) {
                String msg = "Hello, msg" + i;
                System.out.println("[WQ] send msg: " + msg);
                channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
                TimeUnit.MILLISECONDS.sleep(i * 20);
            }
        }
    }
}
```

```java
/**
 * 需要多启动几份消费者，就可以看到工作队列轮询地将消息发送给这几个消费者
 */
public class ReceiveMsg {
    private static final String QUEUE_NAME = "test_work_queue";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection conn = ConnectionUtils.getConnection();
        Channel channel = conn.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) {
                String msg = new String(body, StandardCharsets.UTF_8);
                System.out.println("[1] receive msg: " + msg);
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    // swallow
                } finally {
                    System.out.println("[1] finished!");
                }
            }
        };
        boolean autoAck = true; // 开启自动应答
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);
    }
}
```



**公平分发**是指，队列根据消费者的处理能力动态去调剂消息的发送，表现为能者多劳，所有的消费者共同在更多的时间内，将队列中的消息处理完毕。

解决了消费者和队列丢失数据的情况，此时需要考虑如何解决公平分发的问题。

![fair-dispatch](./imgs/fair-dispatch.png)

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

通过配置`prefetchCount = 1`来告诉消息队列，每次只向消费者发送一条消息，也就是说，只有当消费者处理了前一个消息，队列才会给它发送新的消息，如果当前消费者一直忙于处理当前消息，队列就会向其他消费者发送消息。

```java
/**
 * 公平分发
 */
public class SendMsg {
    private static final String QUEUE_NAME = "test_work_queue";
    public static void main(String[] args) throws IOException, TimeoutException, 
    	InterruptedException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            // 队列每次只向每个消费者发送prefetchSize条消息
            // 如果消费者不给队列发送一个返回的确认信号，队列将不会继续向消费者发送消息
            int prefetchSize = 1;
            channel.basicQos(prefetchSize);
            for (int i = 0; i < 50; i++) {
                String msg = "Hello, msg" + i;
                System.out.println("[WQ] send msg: " + msg);
                channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
                TimeUnit.MILLISECONDS.sleep(i * 20);
            }
        }
    }
}
```

```java
/**
 * 需要多启动几份消费者，就可以看到工作队列根据消费者的处理能力，将消息发送给这几个消费者
 */
public class ReceiveMsg {
    private static final String QUEUE_NAME = "test_work_queue";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection conn = ConnectionUtils.getConnection();
        Channel channel = conn.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        int prefetchSize = 1;
        channel.basicQos(prefetchSize);
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) 
                throws IOException {
                String msg = new String(body, StandardCharsets.UTF_8);
                System.out.println("[1] receive msg: " + msg);
                try {
                    TimeUnit.SECONDS.sleep(2); // 通过休眠来模拟消费者的处理能力
                } catch (InterruptedException e) {
                    // swallow
                } finally {
                    System.out.println("[1] finished!");
                    // 手动应答
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        // 关闭自动应答，改为手动应答
        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);
    }
}
```



### 发布订阅模式

![1567133652613](./imgs/pub-sub.png)

在上面的简单队列和工作队列中，队列中的消息只有一份，被一个消费者消费了，其它的消费者就不能消费了。这在某些场景下是不适用的，所有需要另外一种模式，即发布/订阅模式，这和我们去杂志社订阅杂志相似。

在发布/订阅模式中，可以有一个生产者和多个消费者，每个消费者都关联着自己的队列。生成者并没有直接将消息发送到队列中，而是发送到交换器(exchange)中，之后交换器将消息推送到每一个和它绑定(binding)的队列中。

值得注意的是，交换器本身是不具备存储数据的能力，当一个交换器没有绑定任何的队列时，生产者向交换器中发送数据，数据会全部丢失。

#### fanout exchange

fanout 交换器模式下，不处理路由键，直接进行匿名转发，将所有的消息全部推送到与之绑定的各个队列中。

```java
/**
 * 发布订阅模式 - fanout
 */
public class SendMsg {
    private final static String EXCHANGE_NAME = "text_exchange_fanout";
    public static void main(String[] args) throws IOException, TimeoutException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            // 声明一个交换机(转发器) - fanout 分发模式
            // exchangeDeclare 是一个重载方法，还有附带更多参数的方法，其中可以定义更加详细的交换器
            // 如设置交换器的持久化等
            channel.exchangeDeclare(EXCHANGE_NAME, "fanout");
            String msg = "hello, publish-subscribe, fanout";
            /**
         	 * @param exchange 交换器 
			 * @param routingKey 路由键 
			 *                   fanout模式下，交换器将消息发送给所有和它绑定的队列，路由键为空
			 * @param props 参数
			 * @param body 真正的消息，以byte[]方式进行传递
         	 */
            channel.basicPublish(EXCHANGE_NAME, "", null, msg.getBytes());
            System.out.println("-- send msg: " + msg);
        }
    }
}
```

```java
/**
 * 需要多启动几份消费者
 * 声明不同的队列，然后将这些队列都绑定到一个交换机中
 * 每个队列的消费者都可以获得一份生产者产生的数据
 */
public class ReceiveMsg {
    private static final String QUEUE_NAME = "test_queue_fanout_1";
    private final static String EXCHANGE_NAME = "text_exchange_fanout";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection conn = ConnectionUtils.getConnection();
        Channel channel = conn.createChannel(); // 从连接中获取一个通道
        // 队列声明
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 绑定队列到 交换机(转发器)
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "");
        int prefetchSize = 1;
        channel.basicQos(prefetchSize);
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) 
                					  throws IOException {
                String msg = new String(body, StandardCharsets.UTF_8);
                System.out.println("[1] receive msg: " + msg);
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    // swallow
                } finally {
                    System.out.println("[1] finished!");
                    // 手动应答
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        // 关闭自动应答，改为手动应答
        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);
    }
}
```



#### direct exchange

direct 交换器模式下，处理路由键，根据不同的路由键将不同的消息推送到不同的队列中。

![1567135925438](./imgs/direct-exchange.png)

```java
/**
 * 发布订阅模式 - direct
 * 生成者设定了路由键为 "info" 
 * 通过路由键，能将消息按照不同的路由规则发送给不同的队列
 */
public class SendMsg {
    private final static String EXCHANGE_NAME = "text_exchange_direct";
    public static void main(String[] args) throws IOException, TimeoutException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            // 声明一个交换机(转发器) - direct 分发模式
            channel.exchangeDeclare(EXCHANGE_NAME, "direct");
            String msg = "hello, publish-subscribe, direct";
            String routingKey = "info"; // 设置路由键 
            channel.basicPublish(EXCHANGE_NAME, routingKey, null, msg.getBytes());
            System.out.println("--send msg: " + msg);
        }
    }
}
```

```java
/**
 * 只有生产者绑定的队列，也将路由键设置为 "info"，才能正确与交换机绑定
 */
public class ReceiveMsg {
    private static final String QUEUE_NAME = "test_queue_direct_1";
    private final static String EXCHANGE_NAME = "text_exchange_direct";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection conn = ConnectionUtils.getConnection();
        Channel channel = conn.createChannel();
        // 队列声明
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 绑定队列到 交换机(转发器)
        String routingKey = "info"; // 设置路由键
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, routingKey);

        int prefetchSize = 1;
        channel.basicQos(prefetchSize);

        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) 
                		throws IOException {
                String msg = new String(body, StandardCharsets.UTF_8);
                System.out.println("[1] receive msg: " + msg);
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    //
                } finally {
                    System.out.println("[1] finished!");
                    // 手动应答
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        // 关闭自动应答，改为手动应答
        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);
    }
}
```



#### topic exchange

topic 交换器模式下，由路由键和关键字共同生效。

![1567136137477](./imgs/topic-exchange.png)

其中，`#` 代表可以匹配一个或多个字符串，`*` 代表可以匹配一个字符串，所以按照这个规则，我们可以判断以下的消息会进入哪个队列中。

``` 
quick.orange.rabbit: 同时满足了 *.orange.* 和 *.*rabbit 
lazy.orange.elephant：同时满足了 *.orange.* 和 lazy.#
quick.orange.fox：只满足了 *.orange.*
lazy.brown.fox：只满足了 lazy.#
lazy.pink.rabbit：同时满足了 *.*rabbit 和 lazy.#，但只会进入队列中一次
```

```java
/**
 * 发布订阅模式 - topic
 */
public class SendMsg {
    private final static String EXCHANGE_NAME = "text_exchange_topic";
    public static void main(String[] args) throws IOException, TimeoutException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            // 声明一个交换机(转发器) - topic 分发模式
            channel.exchangeDeclare(EXCHANGE_NAME, "topic");
            String msg = "hello, publish-subscribe, topic";
            channel.basicPublish(EXCHANGE_NAME, "goods.find", null, msg.getBytes());
            System.out.println("-- send msg: " + msg);
        }
    }
}
```

```java
/**
 * 通过 topic 来确定路由规则，其中使用到了 # 和 * 通配符
 */
public class ReceiveMsg {
    private static final String QUEUE_NAME = "test_queue_topic_2";
    private final static String EXCHANGE_NAME = "text_exchange_topic";

    public static void main(String[] args) throws IOException, TimeoutException {
        Connection conn = ConnectionUtils.getConnection();
        Channel channel = conn.createChannel();
        // 队列声明
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        // 绑定队列到 交换机(转发器)
        // goods.# 表示 goods 主题 topic 下的所有消息都可以接受到
        // 如：goods.add, goods.del, goods.find ...
        // # 是通配符，可以匹配一个或多个字符
        channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, "goods.#");

        int prefetchSize = 1;
        channel.basicQos(prefetchSize);
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) 
                throws IOException {
                String msg = new String(body, StandardCharsets.UTF_8);
                System.out.println("[2] receive msg: " + msg);
                try {
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    //
                } finally {
                    System.out.println("[2] finished!");
                    // 手动应答
                    channel.basicAck(envelope.getDeliveryTag(), false);
                }
            }
        };
        // 关闭自动应答，改为手动应答
        boolean autoAck = false;
        channel.basicConsume(QUEUE_NAME, autoAck, consumer);
    }
}
```



## 消息发送确认机制

在上面的描述了，更多地保障了消费者正确消费数据时的情形，即使用消费者与队列之间关闭了自动应答，而采用了手动应答的机制。当消费者正确消费数据时，手动给队列一个应答，此时队列才会删除这个消息。

但是，消息的生产者端，只进行了消息的发送，但是没有确认消息是否成功的进入了队列。为了保障消息入队的可靠性，需要开启 RabbitMQ 的发送确认。

RabbitMQ 提供了两种方式：

1. 事务模式
2. Confirm 模式

值得注意的是，当开启了消息发送确认后，多了一些额外的服务，所以会在一定程度上降低 RabbitMQ 的消息吞吐量。

### 事务模式

通过 `txSelect` 来开启事务模式，正常情况下通过 `txCommit` 提交本次事务，异常情况下通过 `txRollback` 来回滚本次操作。

通过一个简单队列的示例来演示上面的描述

```java
/**
 * 消息生产端 - 开启事务机制
 */
public class TxSendMsg {
    private static final String QUEUE_NAME = "test_simple_queue_tx";
    public static void main(String[] args) throws IOException, TimeoutException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            String msg = "hello, tx with simple queue";
            try {
                channel.txSelect();
                channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
                // int x = 1/0;  // 模拟异常，在事务支持下，发生异常，会回滚消息
                System.out.println("-- send msg is: " + msg);
                channel.txCommit();
            } catch (Exception e) {
                channel.txRollback();
                System.out.println("-- send msg fail, prompt is: " + e.getMessage());
            }
        }
    }
}
```

```java
/**
 * 消息消费端 - 正常操作
 */
public class TxReceiveMsg {
    private static final String QUEUE_NAME = "test_simple_queue_tx";
    public static void main(String[] args) throws IOException, TimeoutException {
        Connection conn = ConnectionUtils.getConnection();
        Channel channel = conn.createChannel();
        channel.queueDeclare(QUEUE_NAME, false, false, false, null);
        DefaultConsumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) 
                throws IOException {
                String msg = new String(body, StandardCharsets.UTF_8);
                System.out.println("-- receive msg: " + msg);
            }
        };
        channel.basicConsume(QUEUE_NAME, true, consumer); // 监听队列
    }
}
```



### Confirm 模式

生产者将信道设置为 confirm 模式，一旦信道进入了 confirm 模式，所有在该信道上发布的消息都会被指派一个唯一的 id (从1开始)，当消息被投递到所有匹配的队列后，broker 就会发送一个确认消息给生产者，这时生产者就知道消息已经正确的抵达了队列。

- **每次发送一条数据**

```java
/**
 * confirm 模式一： 发送一条消息就进行一次确认
 */
public class ConfirmSendMsg1 {
    private static final String QUEUE_NAME = "test_simple_queue_confirm_1";
    public static void main(String[] args) throws IOException, TimeoutException, 
    	InterruptedException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            // 生产者调用confirmSelect，将channel设置为confirm模式
            // 一个队列不能同时设置为 tx 模式和 confirm模式
            channel.confirmSelect();
            String msg = "Hello, simple queue with confirm!";
            channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
            // waitForConfirms
            if (!channel.waitForConfirms()) {
                System.err.println("-- send msg fail");
            } else {
                System.out.println("-- send msg ok");
                System.out.println("-- send msg: " + msg);
            }
        }
    }
}
```

消费者与普通的消费者没有什么区别，所以就不展示了。时刻注意的是，这一节考虑的消息发送的可靠性，所以和消费者没有关系。

- **每次发送一批数据**

```java
/**
 * confirm 模式二： 发送一批消息后，才进行一次确认
 */
public class ConfirmSendMsg2 {
    private static final String QUEUE_NAME = "test_simple_queue_confirm_2";
    public static void main(String[] args) throws IOException, TimeoutException, 
    		InterruptedException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null); 
            // 生产者调用confirmSelect，将channel设置为confirm模式
            channel.confirmSelect();
            // 批量发送消息
            for (int i = 0; i < 10; i++) {
                String msg = "Hello,simple queue with confirm in batch, num is: " + i;
                channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
            }
            // waitForConfirms
            // 批量发送完消息后，进行这批消息发送的确认
            if (!channel.waitForConfirms()) {
                System.err.println("-- send msg fail");
            } else {
                System.out.println("-- send msg ok");
            }
        }
    }
}
```



- **基于异步回调的方式**

```java
/**
 * confirm 模式三： 基于异步回调的模式
 */
public class ConfirmSendMsg3 {

    private static final String QUEUE_NAME = "test_simple_queue_confirm_3";

    public static void main(String[] args) throws IOException, TimeoutException, InterruptedException {
        try (Connection conn = ConnectionUtils.getConnection();
             Channel channel = conn.createChannel()) {
            channel.queueDeclare(QUEUE_NAME, false, false, false, null);
            // 生产者开始消息的confirm模式
            channel.confirmSelect();
            // 声明一个集合，用于存放 已经发送，但还 没有确认的消息 的标识
            // 每发送一个消息，未确认，set集合就会增加一个数据
            // 每收到一个确认消息，就从 set 集合中将该条消息的标识去掉
            final SortedSet<Long> confirmSet = Collections
                						.synchronizedSortedSet(new TreeSet<>());
            // confirm 确认的到底是什么的? 确认的是，生产者发送的数据，有没有正确到达指定的队列
            // 至于队列中的数据，消费者是何时进行消费的。 confirm 并不保证这些问题
            // 这里就存在另外一个问题了： 生产者大量的发送数据到了队列中，并且正确送达队列
            // 队列中积压了大量的数据，消费者还没有来得及处理数据，此时队列宕机了，数据应该如何保证?
            // rabbitMQ 如果开启了队列的持久化，就可以保证队列宕机时，数据的本地化
            channel.addConfirmListener(new ConfirmListener() {
                // Acks represent messages handled successfully
                // 消息确认 成功后，由handleAck处理
                @Override
                public void handleAck(long deliveryTag, boolean multiple) {
                    if (multiple) {
                        // multiple，代表了可以传递多个确认消息标识回来
                        // 依次将多个确认消息的标识从set集合中移除
                        System.out.println("handleAck and multiple is true");
                        confirmSet.headSet(deliveryTag + 1).clear();

                    } else {
                        // 只传递回来一个确认消息的标识，仅将这个标识从set结合中移除
                        System.out.println("handleAck and multiple is false");
                        confirmSet.remove(deliveryTag);
                    }
                    System.err.println("The item of set is: " + confirmSet.size());
                }

                // Nacks represent messages lost by the broker
                // 消息确认 失败后，由handleNack处理
                @Override
                public void handleNack(long deliveryTag, boolean multiple) {
                    if (multiple) {
                        System.out.println("handleNack and multiple is true");
                        confirmSet.headSet(deliveryTag + 1).clear();
                    } else {
                        System.out.println("handleNack and multiple is false");
                        confirmSet.remove(deliveryTag);
                    }
                    System.err.println("The item of set is: " + confirmSet.size());
                }
            });

            String msg = "Hello, simple queue with confirm and add a listener";

            while (true) {  // 模拟不断地发送数据
                long seqNo = channel.getNextPublishSeqNo();
                channel.basicPublish("", QUEUE_NAME, null, msg.getBytes());
                confirmSet.add(seqNo);
                TimeUnit.MILLISECONDS.sleep(200);
            }
        }
    }
}
```





## Code in Spring





## Code in Springboot



