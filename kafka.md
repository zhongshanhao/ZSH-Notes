消息队列的作用

- 异步
- 解耦
- 削峰

异步

点赞时生成一个事件，事件中有属性：主题，点赞的用户id，帖子id，将该事件发往指定主题，消费者监听对应主题，有数据就取，然后根据这个事件生成系统通知。

解耦

如果没有消息队列，要主动调用生成系统通知的模块，如果要该需求，就是在生成系统通知的时候要把通知同时发送到对方邮箱，这个时候要改动这个模块，如果用了消息队列，我这个点赞的业务就只需要将事件丢入到消息队列中，然后在消费者消费事件的时候在加一个发送邮件的模块就好。

削峰

双十一的订单，如果订单处理直接给到服务器，那么服务器可能承受不了而宕机了，先把订单放到消息队列中，然后根据其处理能力去处理订单。

队列模型

使用队列作为消息通信载体，满足生产者与消费者模式，一条消息只能被一个消费者消费。

队列模型存在的问题：如果我们需要将生产者产生的消息分发给多个消费者，但在队列模型中一条消息只能被一个消费者消费。

发布订阅模型：Kafka消息模型

![发布订阅模型](https://gitee.com/Krains/FigureBed/raw/master/img/%E5%8F%91%E5%B8%83%E8%AE%A2%E9%98%85%E6%A8%A1%E5%9E%8B.png)

发布订阅模型使用主题作为消息通信载体，类似广播模式，发布者发布一条消息，该消息通过主题传递给所有的订阅者。

Kafka重要概念

Kafka 将生产者发布的消息发送到 **Topic（主题）** 中，需要这些消息的消费者可以订阅这些 **Topic（主题）**，如下图所示：

<img src="https://gitee.com/Krains/FigureBed/raw/master/img/kafka%E9%9B%86%E7%BE%A4.png" alt="kafka集群" style="zoom: 80%;" />

- Producer：产生消息的一方
- Consumer：消费消息的一方
- Broker：可以看做是一个独立的Kafka实例，多个Kafka实例组成一个Kafka Cluster
- Topic：Producer将消息发送到特定的主题，Consumer通过订阅特定的Topic来消费消息
- Partition：Partition属于Topic的一部分，一个Topic可以有多个Partition，并且同一Topic下的Partition可以分布在不同的Broker上，这也就表明一个Topic可以横跨多个Broker

Kafka为Partition引入了多副本机制，分区中的多个副本之间会有一个leader，其他副本称为follower，我们发送的消息会被发送到leader副本，然后follower副本才能从leader副本中拉取消息进行同步

> 生产者和消费者只与leader交互，其他副本只是leader副本的拷贝，他们的存在只是为了保证消息存储的安全性，当leader副本发生故障时会从follower中选举出一个leader，但是如果follower中如果有和leader同步程度达不到要求的参加不了leader的竞选

Kafka如何保证消息的消费顺序？

Kafka中Partition是真正存放消息的地方，而Partition又存放在一个Topic下，一个Topic下有多个Partition，所以生产者往一个Topic放消息的时候可能会将两个消息放在不同的Partition中，那么就不能保证它们被消费的顺序性

![Partition](https://gitee.com/Krains/FigureBed/raw/master/img/Partition.png)

Kafka只能保证一个Partition中消息消费的顺序性，而不能保证一个Topic内消息消费的顺序性

解决方法

- 一个Topic只对应一个Partition
- 发送消息的时候指定Partition/Key

> kafka发送一条消息的时候，可以指定topic、partition、key、data 四个参数，如果发送消息的时候指定了Partition的话，那么所有消息都会被发送到指定的Partition。并且，同一个key的消息可以保证只发送到同一个partition。

