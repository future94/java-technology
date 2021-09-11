
## 1. 产生的原因

![image](https://raw.githubusercontent.com/future94/java-technology/master/mq/kafka/images/12312312.png)

Kafka发送数据的时候是发送到Topic中，而一个Topic可以有多个不同的Partition，Kafka 中 Partition(分区)是真正保存消息的地方，我们发送的消息都被放在了这里。每次添加消息到 Partition(分区) 的时候都会采用尾加法，制定一个Offset值，但是Kafka只能保证同一个Partition的消息是顺序的。而一个Partition也可能会有多个线程消费者在消费消息，所以当我们依次发送A、B、C到一个Topic中，我们并不能保证消费的时候也是A、B、C。

<font color="red">所以Topic下的多个Partition和每个Partition下的多线程消费是产生消费无序的原因</font>。

## 2. 如何解决

### 2.1 一对一对一（不推荐）

通过上面的分析我们可以知道，我们可以通过一个Topic对应一个Partition对应一个消费线程来解决。但是违反了Kafka设计的初衷，性能很低。

### 2.2 相同信息的消息发送到相同Partition和相同的线程。

![image](https://raw.githubusercontent.com/future94/java-technology/master/mq/kafka/images/1631333678894.jpg)

Kafka 中发送 1 条消息的时候，可以指定 topic, partition, key,data（数据） 4 个参数。如果你发送消息的时候指定了 Partition 的话，所有消息都会被发送到指定的 Partition。并且，同一个 key 的消息可以保证只发送到同一个 partition，这个我们可以采用表/对象的 id 来作为 key 。如订单ID为1的都发送到相同的 Partition 。消费时候放入相同的内存队列中。


参考文章：
- [Kafka常见面试题总结](https://github.com/Snailclimb/JavaGuide/blob/master/docs/system-design/distributed-system/message-queue/Kafka%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93.md)
- [我该怎么保证Kafka消息的顺序](https://apppukyptrl1086.h5.xiaoeknow.com/v1/course/video/v_5d2f44a2e1149_CU5x6W2z?type=2&pro_id=p_5d3114935b4d7_CEcL8yMS)