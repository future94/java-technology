
## 1. Broker选举

所谓控制器就是一个Borker，在一个kafka集群中，有多个broker节点，但是它们之间需要选举出一个leader，其他的broker充当follower角色。

### 1.1 启动broker时
集群中第一个启动的broker会通过在zookeeper中创建 `临时节点/controller` 来让自己成为控制器，其他broker启动时也会在zookeeper中创建临时节点，但是发现节点已经存在，所以它们会收到一个异常，意识到控制器已经存在，那么就会在zookeeper中创建watch对象，便于它们收到控制器变更的通知。

### 1.2 leader断开
那么如果控制器由于网络原因与zookeeper断开连接或者异常退出，那么其他broker通过watch收到控制器变更的通知，就会去尝试创建 `临时节点/controller` ，如果有一个broker创建成功，那么其他broker就会收到创建异常通知，也就意味着集群中已经有了控制器，其他broker只需创建watch对象即可。<font color="red">集群中每选举一次控制器，就会通过zookeeper创建一个controller epoch，每一个选举都会创建一个更大，包含最新信息的epoch，如果有broker收到比这个epoch旧的数据，就会忽略它们，kafka也通过这个epoch来防止集群产生“脑裂”</font>。

### 1.3 follwer断开
如果集群中有一个broker发生异常退出了，那么控制器就会检查这个broker是否有分区的副本leader，如果有那么这个分区就需要一个新的leader，此时控制器就会去遍历其他副本，决定哪一个成为新的leader，同时更新分区的ISR集合。

### 1.4 新broker加入
如果有一个broker加入集群中，那么控制器就会通过Broker ID去判断新加入的broker中是否含有现有分区的副本，如果有，就会从分区副本中去同步数据。


## 2. 分区副本选举机制

在kafka的集群中，会存在着多个主题topic，在每一个topic中，又被划分为多个partition，为了防止数据不丢失，每一个partition又有多个副本，在整个集群中，总共有三种副本角色：

- **leader副本** ：每个分区都有一个leader副本，为了保证数据一致性，所有的生产者与消费者的请求都会经过该副本来处理。
- **follower副本** ：除了首领副本外的其他所有副本都是跟随者副本，跟随者副本不处理来自客户端的任何请求，只负责从首领副本同步数据，保证与首领保持一致。如果首领副本发生崩溃，就会从这其中选举出一个leader。

follower需要从leader中同步数据，但是由于网络或者其他原因，导致数据阻塞，出现不一致的情况，为了避免这种情况，follower会向leader发送请求信息，这些请求信息中包含了follower需要数据的偏移量offset，而且这些offset是有序的。

如果有follower向leader发送了请求1，接着发送请求2，请求3，那么再发送请求4，这时就意味着follower已经同步了前三条数据，否则不会发送请求4。leader通过跟踪 每一个follower的offset来判断它们的复制进度。

默认的，如果follower与leader之间超过10s内没有发送请求，或者说没有收到请求数据，此时该follower就会被认为“不同步副本”。而持续请求的副本就是 `同步副本` ，当leader发生故障时，只有 `同步副本` 才可以被选举为leader。其中的请求超时时间可以通过参数 `replica.lag.time.max.ms` 参数来配置。

### leader的均衡机制

当一个broker停止或者crashes时，所有本来将它作为leader的分区将会把leader转移到其他broker上去，极端情况下，会导致同一个leader管理多个分区，导致负载不均衡，同时当这个broker重启时，如果这个broker不再是任何分区的leader,kafka的client也不会从这个broker来读取消息，从而导致资源的浪费。我们希望每个分区的leader可以分布到不同的broker中，尽可能的达到负载均衡。

kafka中有一个被称为优先副本（preferred replicas）的概念。如果一个分区有3个副本，且这3个副本的优先级别分别为0,1,2，根据优先副本的概念，0会作为leader 。当0节点的broker挂掉时，会启动1这个节点broker当做leader。当0节点的broker再次启动后，会自动恢复为此partition的leader。不会导致负载不均衡和资源浪费，这就是leader的均衡机制。

我们设置参数 `auto.leader.rebalance.enable=true` 开启。

## 3. 消费组选主

在kafka的消费端，会有一个消费者协调器以及消费组，组协调器 `GroupCoordinator` 需要为消费组内的消费者选举出一个消费组的leader。

- 如果消费组内还没有leader，那么第一个加入消费组的消费者即为消费组的leader.
- 如果某一个时刻leader消费者由于某些原因退出了消费组，那么就会重新选举leader

```java
private val members = new mutable.HashMap[String, MemberMetadata]
leaderId = members.keys.headOption
```

上面代码是kafka源码中的部分代码，member是一个hashmap的数据结构，key为消费者的member_id，value是元数据信息，那么它会将leaderId选举为Hashmap中的第一个键值对，它和随机基本没啥区别。

参考文章：  
- [浅谈Kafka选举机制](https://blog.csdn.net/qq_37142346/article/details/91349100)
- [kafka的原理](https://blog.csdn.net/wyqwilliam/article/details/82022987)