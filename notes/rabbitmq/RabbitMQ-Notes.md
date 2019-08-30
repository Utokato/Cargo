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



**公平分发**是指，队列根据消费者的处理能力动态去调剂消息的发送，表现为能者多劳，所有的消费者共同在更多的时间内，将队列中的消息处理完毕。

解决了消费者和队列丢失数据的情况，此时需要考虑如何解决公平分发的问题。

![fair-dispatch](./imgs/fair-dispatch.png)

```java
int prefetchCount = 1;
channel.basicQos(prefetchCount);
```

通过配置`prefetchCount = 1`来告诉消息队列，每次只向消费者发送一条消息，也就是说，只有当消费者处理了前一个消息，队列才会给它发送新的消息，如果当前消费者一直忙于处理当前消息，队列就会向其他消费者发送消息。



### 发布订阅模式

![1567133652613](./imgs/pub-sub.png)

在上面的简单队列和工作队列中，队列中的消息只有一份，被一个消费者消费了，其它的消费者就不能消费了。这在某些场景下是不适用的，所有需要另外一种模式，即发布/订阅模式，这和我们去杂志社订阅杂志相似。

在发布/订阅模式中，可以有一个生产者和多个消费者，每个消费者都关联着自己的队列。生成者并没有直接将消息发送到队列中，而是发送到交换器(exchange)中，之后交换器将消息推送到每一个和它绑定(binding)的队列中。

值得注意的是，交换器本身是不具备存储数据的能力，当一个交换器没有绑定任何的队列时，生产者向交换器中发送数据，数据会全部丢失。

#### fanout exchange

fanout 交换器模式下，不处理路由键，直接进行匿名转发，将所有的消息全部推送到与之绑定的各个队列中。

#### direct exchange

direct 交换器模式下，处理路由键，根据不同的路由键将不同的消息推送到不同的队列中。

![1567135925438](./imgs/direct-exchange.png)



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



## Code in Spring





## Code in Springboot



