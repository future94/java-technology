典型应用场景：[zookeeper](https://zookeeper.apache.org/)

总结：

| 机制 | 作用 |
---|---
初始化选主时候，如果ZXID相同，ServerId大的优先成为leader | 同时间竞选冲突
epoch机制 | 同一时间只有一个leader，在数据同步和选举时有过滤作用
只有同步数据良好的follower才能竞选leader | 保证数据的一致性
超过半数的follower进行了ACK之后才会提交数据 | 保证数据的一致性
不会给比自己同步日志还差的follower进行投票 | 保证数据的一致性
维护数组的ZXID中的事务ID | 保证消息的顺序性
广播时，leader为每个follower创建一个FIFO异步广播消息 | 提高并发性能

## 1. ZK中的概念

### 1.1 Zookeeper中的角色
- **leader ：领导者** 。负责投票的发起和决议。
- **followr ：跟随者** 。接收客户端的请求，可以参与选举leader的投票。
- **observer ：观察者** 。可以链接客户端，将写请求转发给leader。不参与投票，只同步leader的状态。起备份作用，提高系统的稳定性。

Observer：随着Server原来越多，投票阶段延迟增大，影响性能。又为了保证稳定性，机器很多，所以引入Observer，只同步数据，不参与投票。

### 1.2 ZK Server的状态
- LOOKING ：还不知道leader是谁，在搜索阶段。或者leader挂了之后的正处于选主状态。
- FOLLOWING ：leader和follower正在同步数据。
- LEADING ：集群中已经有了一个leader。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/20160927103234652.png)

### 1.3 Zab协议

Zk Client会连接到Zk Server集群中的一个节点，Zk Client会对Zk Server发起请求，即读请求和写请求。

- 读请求：直接在当前节点进行数据获取。
- 写请求：会重定向到leader，并向leader提交事务，而leader会广播该事务，超过半数写入成功，那么就提交该事务。

Zookeeper的核心是原子广播，这个机制保证了各个节点之间数据的同步。实现这个机制的协议就叫Zab协议（Zookeeper Atomic Broadcast），而Zookeeper也是通过Zab协议实现的数据强一致性。

## 2. Zab协议中的概念

- epoch：年号。相当于Raft中的term任期。
- history：历史记录。节点收到log的记录。
- ZXID：由两部分组成，是一个64位long类型。低32位是事务ID，高32位是epoch的ID。
- SID：serverID。当前服务器的编号。
- status：looking、following、leading。

与Raft算法类似，Raft叫term，而Zab叫epoch。心跳方向，Raft是leader发给folloer，Zab是follower给leader。

## 3. Zk的集群模式

- **崩溃恢复** ：
- **消息广播** ：

### 3.1 崩溃恢复模式

一旦Leader服务器出现崩溃，或者说网络原因导致Leader服务器失去了与过半的Follower的联系，那么就会进入崩溃恢复模式，并选出新的leader。zab需要保证下面两个情况：

1. 已经在leader中提交的事务最终被所有人提交。
2. 丢弃只有在leader中提交的事务。

所以<font color="red">保证新选举leader的epoch是其他机器中最大的epoch即可</font>。

### 3.2 消息广播模式

当选出新的leader，并且大半数机器已经同步完成之后，Zk集群退出崩溃恢复模式，进入消息广播模式。这时候如果有一台新的机器加入集群中，他先进入崩溃恢复模式，从leader中同步数据，然后在加入消息广播流程中。

之前我们提过，客户端接收到写请求之后，会先重定向到leader进行事务提交，并广播给其他节点，大半数同步成功之后，提交事务。

Zab与2PC不同点在于<font color="red">Zab在广播消息时不需要等待所有节点同ACK，只要有半数节点ACK即可提交事务</font>。而当leader挂了之后，通过崩溃恢复模式解决数据中的不一致问题。

## 4. Zab协议选主

### 4.1 启动时选主

流程如下：
- 先投自己一票，epoch+1 并让其节点投自己一票。推举的服务器的serverID、ZXID。先看ZXID，如果ZXID相同，则serverID大的优先成为leader。
- 然后统计票，如果自己票数过半，自己成为leader。
- 告诉其他节点自己已经为leader。

### 4.2 运行时选主

- 当follower运行时与leader断开链接，那么他会开始重新选举。
- 如果选举时候，发现其他节点告诉他leader还活着，并且超过半数时候，那么他会将为follwer。
- 如果选举的时候，leader挂了，那么就出发正常的选举。

### 4.3 选举的四个阶段

#### 选举阶段

当节点处于looking阶段的时候，会发起投票，超过半数之后，自己变为准leader。

#### 发现阶段

成为准leader之后，与其他follower进行通信，告诉他们我已经是leader了。

#### 同步阶段

当准leader告诉其他follower我是leader并且半数已经ACK了之后，leader开始与follower同步数据，当大半数follower同步完成之后，准leader成为真正的leader，并进入广播消息阶段。

#### 广播阶段

当Zk集群进入广播阶段时候，才能真正的接受Client消息。leader接收数据之后对其他follower进行广播，超过半数ACK了之后就提交事务，不会等待所有follower都ACK。而且新follower加入集群要先从leader同步数据，在加入广播中。

### 5. Zab协议的事务

当Client发出写请求时，leader接收到并同步到其他follower的过程（消息广播模式）为Zab协议的事务。

事务的一致性：半数通过才会写入。
事务的顺序性：通过ZXID的高32ID一次增大保证事务的顺序。

Leader 服务器与每一个 Follower 服务器之间都维护了一个单独的 FIFO 消息队列进行收发消息，使用队列消息可以做到异步解耦。 Leader 和 Follower 之间只需要往队列中发消息即可。如果使用同步的方式会引起阻塞，性能要下降很多。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/1053629-447433fdf7a1d7d6.png)


参考文章：
- [Zab协议 (史上最全)](https://www.cnblogs.com/crazymakercircle/p/14339702.html)
- [ZAB协议概述与选主流程详解](https://blog.csdn.net/a724888/article/details/80757503)