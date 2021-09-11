
## 1. 消息传递的保证性

- **最多一次** ：不过是否消息消费成功，最多投递一次。
- **至少一次** ：消息消费成功也可能出现多次投递的情况，需要保证不重复消费。
- **精准一次** ：消息被处理且只会被处理一次。

## 2. Kafka的消息传递

#### 2.1 Kafka的至多一次

当我们配置 acks = 0 时，在Producer发送了就代表成功。

#### 2.2 Kafka的至少一次

Kafka的Producer和Consumer默认是 `至少一次` ，在Procuder端如果没有接收到发送消息的响应，还会重新进行投递。在Consumer端，通过 `enable.auto.commit=false` 拉取到消息处理成功后在提交，可能消费完成还没有提交就死了，这时候还会推送。

#### 2.3 Kafka的精准一次

**Producer** ：  
kafka 0.11.0.0版本开始引入幂等的Producer，在这个机制中同一消息可能被producer发送多次，但是在broker端只会写入一次，他为每一条消息编号去重，而且对kafka开销影响不大。通过 `enable.idempotent=true` 配置。而多分区的情况，我们需要保证原子性的写入多个分区，即写入到多个分区的消息要么全部成功，要么全部回滚。这时候就需要使用事务，在producer端设置 transcational.id为一个指定字符串。

<font color="red">这样幂等producer只能保证单分区上无重复消息；事务可以保证多分区写入消息的完整性</font>。

**Consumer** ：
- 关闭自动提交的功能，处理成功后自动提交并结合业务进行幂等处理(唯一ID、Set去重复等)。
- 还有一个选择就是使用kafka自己的流处理引擎，也就是Kafka Streams，设置 `processing.guarantee=exactly_once` ，就可以轻松实现精准一次了。

参考文章：
- [Kafka如何保证百万级写入速度以及保证不丢失不重复消费](https://www.cnblogs.com/gxyandwmm/p/11432598.html)