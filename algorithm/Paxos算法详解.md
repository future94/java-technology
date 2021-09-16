
提到Paxos，很多文章都会提到`拜占庭将军问题`，其实对应分布式就是各个节点消息的传送会不会被篡改，Paxos的前提不会被篡改的。

## 1. Paxos中的角色
- 提议者（Proposer）
	- 提议一个值，用于投票表决。
	- 接入和协调，收到客户端的请求后，可以发起二阶段提交，进行共识协商。
- 接受者（Acceptor）
	- 对每个提议的值进行投票，并存储接受的值。
	- 投票协商和存储数据，对提议的值进行投票，并接受达成共识的值，存储保存。
	- 其实，集群中所有的节点都在扮演接受者的角色，参与共识协商，并接受和存储数据。
- 学习者（Learner）
	- 被告知投票的结果，接受达成共识的值，存储数据，
	- 不参与投票的过程，即不参与共识协商。

还有的时候会提及到客户端（Client），这个角色不属于Paxos，实际上就是代表请求发起方或者客户端的存在，如Client存储数据5，那么就是Client发送给Proposer、由Proposer发起一个提案。

规定一个提议包含两个字段：{n, v}，其中 n 为提议编号（具有唯一性），v 为提议值。

## 2. Paxos分类

Paxos算法有很多中，如Basic Paxos、Multi Paxos、Fast Paxos等等

## 3. Basic Paxos

Proposer接收Client的请求，然后发送给Acceptor，达成共识后传输改数据，并由learner备份。而且Accepter不关心Proposer提案内容，只关心提案号是否大于当前。

### 3.1 准备阶段

准备阶段不会发送具体值v，只传递提议编号n，即 `{n, 空}`。

- 第一阶段：Proposer接收到Client请求，生成提案编号n，将 Prepare(n) 发送给各个Accepter。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/hdiuh12390123.jpeg)

- 第二阶段：Accepter接收到这个提案n，会和本地接受过的最大提案m进行对比（没接受过为0）
	- 如果接收提案n大于等于本地提案m，回复给Proposer一个 Promised(n) ，该提议响应包含之前已经接收过的提议 [m, v]表示同意，并不会在接受小于n的提案。
	- 如果接收提案n小于本地提案，直接拒绝掉这个提案。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/jiuy87y21342.jpeg)

### 3.2 接收阶段

- 第一阶段：当Proposer发送的提案被大多数Accepter接收时，那么他就会向所有Accepter发送消息[n, v]。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/huiiy1u249jk123.jpeg)

- 第二阶段：Accepter接收到消息[n, v]，如果接收到v的这个n没有小于当前的n，那么就同步数据，并给Proposer发送一个响应，反之则拒绝掉。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/jij712320ihjnjkqw12.jpeg)

### 3.3 学习者

当Accepter同意一个提案时，那么就会通知给learner。当learner发现大多数Accepter都接受了这个方案时，那么learner也会通过这个提案，并记录该提案值。

### 3.4 问题

- **活锁** ：可以有多个提议者同时进行提议，由于提案编号的存在，同一时间段内可能有多个提案进行提议。如提案1进行提案的时候，会有提案2、3、4等等发送，由于提案大小关系的限制，自己的提案发出可能还没有被同意就过时了，又不得不生成一个新的提案重新进行提案。这个情况就是Basic Paxos的活锁问题。可以<font color="red">通过随机超时时间来解决活锁的问题</font>，[Raft算法](Raft算法详解.md)也有相关的思想来避免同时发起leader的竞争。
- **性能** ：每次发起提议都会有两次RPC的请求（发起提案一次，发送数据一次）。

## 4. Multi Paxos

针对Basic Paxos产生的问题，便提出了Multi Paxos。他只有一个Proposer能发起提案，这个Proposer就是 `leader`。每个Proposer都可以竞选为 leader 。同一时间只能有一个leader存在，负责接受Client请求和提案的发送。可以<font color="red">通过随机超时时间来解决初始化leader选举的问题</font>，[Raft算法](Raft算法详解.md)也有相关的思想来避免同时发起leader的竞争。

- 通过leader的模式解决活锁问题。
- 已经有leader了，所以不会进行提案的投票过程。

[Raft算法](Raft算法详解.md)就是Multi Paxos的变种实现。

## 5. Fast Paxos

// TODO 还没研究，欢迎大佬完善。

参考文章：
- [微信PaxosStore：深入浅出Paxos算法协议](https://segmentfault.com/a/1190000040635529)
- [Paxos共识算法详解](https://segmentfault.com/a/1190000018844326)
- [Paxos 图解 (秒懂)](https://www.cnblogs.com/crazymakercircle/p/14341015.html)