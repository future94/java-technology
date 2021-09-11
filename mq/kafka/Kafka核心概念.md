
## 1. 概览

Kafka采用订阅与发布模式，使用Topic作为消息通讯的载体。

![image](https://raw.githubusercontent.com/future94/java-technology/master/mq/kafka/images/20180902105920995.png)

## 2. 生产者(Producer)与消费者(Consumer)

- 生产者：生产消息的一方，生产消息，发送到MQ。
- 消费者：处理消息的一方，处理收到的MQ消息。

## 3. 代理(Broker)

Kafka是分布式的，一个Broker可以看作是一个独立的 Kafka 实例。多个 Kafka Broker 组成一个 Kafka Cluster。

### 3.1 主题(Topic)

Producer 将消息发送到特定的主题，Consumer 通过订阅特定的 Topic(主题) 来消费消息。

### 3.2 分区(Partition)

Partition 类似与 RabbitMQ 中的 Queue。Partition 属于 Topic 的一部分。一个 Topic 可以有多个 Partition ，并且同一 Topic 下的 Partition 可以分布在不同的 Broker 上，这也就表明一个 Topic 可以横跨多个 Broker 。如上图：Topic2有两个Partition，Partition1和Partition2，他们都对应一个Topic2，但是他们在不同的Broker上。

**多分区的好处**  
Kafka 通过给特定 Topic 指定多个 Partition, 而各个 Partition 可以分布在不同的 Broker 上, 这样便能提供比较好的并发能力（负载均衡）。

#### 3.2.1 分区的多副本机制(Replica)

Partition 中引入了多副本机制，多副本中有一个 `leader` 和 `follower`。我们发送的消息会发送到 Partition 的 leader 副本，然后会同步leader副本到 follower 中。<font color="red">生产者和消费者只与 leader 副本交互，我们发送的消息会被发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步</font>。当follower存在只是为了保证消息存储的安全性。当 leader 副本发生故障时会从 follower 中选举出一个 leader,但是 follower 中如果有和 leader 同步程度达不到要求的参加不了 leader 的竞选。

**多副本机制的好处**  
Partition 可以指定对应的 Replica 数, 这也极大地提高了消息存储的安全性, 提高了容灾能力，不过也相应的增加了所需要的存储空间。

#### 3.2.2 分区的偏移量机制(Offset)

![images](https://raw.githubusercontent.com/future94/java-technology/master/mq/kafka/images/719892-20180627205608005-146856169.png)

每个分区是一个有序的，不可变的消息序列，新的消息不断追加，同时分区会给每个消息记录分配一个顺序ID号 – 偏移量， 用于唯一标识该分区中的每个记录。

<font color="red">Kafka集群保留所有发布的记录，不管这个记录有没有被消费过，Kafka提供可配置的保留策略去删除旧数据(还有一种策略根据分区大小删除数据)</font>。例如，如果将保留策略设置为两天，在记录公布后两天，它可用于消费，之后它将被丢弃以腾出空间。Kafka的性能跟存储的数据量的大小无关， 所以将数据存储很长一段时间是没有问题的。


事实上，保留在每个消费者元数据中的最基础的数据就是消费者正在处理的当前记录的偏移量(offset)或位置(position)。这种偏移是由消费者控制：通常偏移会随着消费者读取记录线性前进，但事实上，因为其位置是由消费者进行控制，消费者可以在任何它喜欢的位置读取记录。例如，消费者可以恢复到旧的偏移量对过去的数据再加工或者直接跳到最新的记录，并消费从“现在”开始的新的记录。
数据日志的分区，有多个目的。首先，每个单独的分区的大小受到承载它的服务器的限制，但一个主题可能有很多分区，允许数据能够扩展到多个服务器，以便它能够支持海量的的数据。其次，更重要的意义是分区是进行并行处理的基础单元。
日志的分区会跨服务器的分布在Kafka集群中，每个服务器会共享分区进行数据请求的处理。每个分区可以配置一定数量的副本分区以提供容错能力。


参考文章：
- [Kafka常见面试题总结](https://github.com/Snailclimb/JavaGuide/blob/master/docs/system-design/distributed-system/message-queue/Kafka%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93.md)
- [kafka知识体系-基本概念](https://www.cnblogs.com/aidodoo/p/8873086.html)