
## Kafka 如何保证消息不丢失

### 1. 生产者消息丢失情况

**解决** ：异步Callback接收发送结果，在合理的时间间隔范围内进行合理的重试。

我们在 Producer 发送消息的时候（如SpringBoot的kafkaTemplate.send方法），发送消息时候是异步投递的，我们不能认为发送了调用了之后就是发送成功。但是我们不能调用get方法来获取结果，因为这样就转化为同步等待结果降低了性能。

同步等待发送结果。

```java
SendResult<String, Object> sendResult = kafkaTemplate.send(topic, o).get();
if (sendResult.getRecordMetadata() != null) {
  logger.info("生产者成功发送消息到" + sendResult.getProducerRecord().topic() + "-> " + sendRe
              sult.getProducerRecord().value().toString());
}
```

一般都是异步Callback接收，如果消息发送失败的话，我们检查失败的原因之后重新发送即可！

```java
ListenableFuture<SendResult<String, Object>> future = kafkaTemplate.send(topic, o);
future.addCallback(
	result -> logger.info("生产者成功发送消息到topic:{} partition:{}的消息", result.getRecordMetadata().topic(), result.getRecordMetadata().partition()),
    ex -> logger.error("生产者发送消失败，原因：{}", ex.getMessage())
);
```

另外这里推荐为 Producer 的retries （重试次数）设置一个比较合理的值，一般是 3 ，但是为了保证消息不丢失的话一般会设置比较大一点。设置完成之后，当出现网络问题之后能够自动重试消息发送，避免消息丢失。另外，建议还要设置重试间隔，因为间隔太小的话重试的效果就不明显了，网络波动一次你3次一下子就重试完了

### 2. 消费者消息丢失情况

**解决** ：关闭拉去消息自动提交Offset功能，消费完成后手动提交Offset，并防止重复消费。

我们知道消息在被追加到 Partition(分区)的时候都会分配一个特定的偏移量（offset）。偏移量（offset)表示 Consumer 当前消费到的 Partition(分区)的所在的位置。Kafka 通过偏移量（offset）可以保证消息在分区内的顺序性。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/mq/kafka/images/719892-20180627205608005-146856169.png)

当消费者获取到消息的时候，会自动提交Offset(`enable.auto.commit=true`)，如果这时候消费者还没有来的及消息就挂了，那么就提交了Offset但是还没有消费。所以我们可以简单粗暴的关闭自动提交Offset功能，当消息完成消费后在手动提交Offset。这样也会有问题，当我们消息处理完成，还没有提交Offset的时候挂了，这样就会出现重复消费的情况，所以我们要做幂等处理，防止重复消费。[Kafka如何保证消息不重复消费](Kafka如何保证消息不重复消费.md)


### 3. Kafka内部消息丢失情况

**解决** ：四个设置
> 设置 `acks = all` ：所有副本接收到消息之后才算成功。
> 设置 `replication.factor >= 3` ：设置大于等于3个副本存储消息。
> 设置 `min.insync.replicas >= 2` ：设置至少成功写入2个副本才算成功。
> 设置 `unclean.leader.election.enable = false` ：leader副本选举的时候去掉同步程度达不到要求的副本。

知道 Kafka 为分区（Partition）引入了多副本（Replica）机制。分区（Partition）中的多个副本之间会有一个叫做 leader 的家伙，其他副本称为 follower。我们发送的消息会被发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步。生产者和消费者只与 leader 副本交互。你可以理解为其他副本只是 leader 副本的拷贝，它们的存在只是为了保证消息存储的安全性。

试想一种情况：假如 leader 副本所在的 broker 突然挂掉，那么就要从 follower 副本重新选出一个 leader ，但是 leader 的数据还有一些没有被 follower 副本的同步的话，就会造成消息丢失。

> 设置 `acks = all`

解决办法就是我们设置 acks = all。acks 是 Kafka 生产者(Producer) 很重要的一个参数。

acks 的默认值即为1，代表我们的消息被leader副本接收之后就算被成功发送。当我们配置 acks = all 代表则所有副本都要接收到该消息之后该消息才算真正成功被发送。当我们配置 acks = 0 时，发送了就代表成功，可以实现Kafka的至多一次。

> 设置 `replication.factor >= 3`

为了保证 leader 副本能有 follower 副本能同步消息，我们一般会为 topic 设置 replication.factor >= 3。这样就可以保证每个 分区(partition) 至少有 3 个副本。虽然造成了数据冗余，但是带来了数据的安全性。

> 设置 `min.insync.replicas >= 2`

一般情况下我们还需要设置 min.insync.replicas >= 2 ，这样配置代表消息至少要被写入到 2 个副本才算是被成功发送。min.insync.replicas 的默认值为 1 ，I9

> 设置 `unclean.leader.election.enable = false`

Kafka 0.11.0.0版本开始 unclean.leader.election.enable 参数的默认值由原来的true 改为false

我们最开始也说了我们发送的消息会被发送到 leader 副本，然后 follower 副本才能从 leader 副本中拉取消息进行同步。多个 follower 副本之间的消息同步情况不一样，当我们配置了 unclean.leader.election.enable = false 的话，当 leader 副本发生故障时就不会从 follower 副本中和 leader 同步程度达不到要求的副本中选择出 leader ，这样降低了消息丢失的可能性。



参考文章：
- [Kafka常见面试题总结](https://github.com/Snailclimb/JavaGuide/blob/master/docs/system-design/distributed-system/message-queue/Kafka%E5%B8%B8%E8%A7%81%E9%9D%A2%E8%AF%95%E9%A2%98%E6%80%BB%E7%BB%93.md)
- [Kafka如何保证百万级写入速度以及保证不丢失不重复消费](https://www.cnblogs.com/gxyandwmm/p/11432598.html)

