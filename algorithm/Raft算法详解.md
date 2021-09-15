
## 1. Raft核心概念

Raft是一种管理复制日志的一致性算法。通过一切以领导者为准的方式，实现一系列值的共识和各节点日志的一致。

- **leader** ：领导者。负责处理请求，管理日志，发送与其他人发送心跳请求。
- **follower** ：跟随者。接收和同步leader发送的信息，与leader保持心跳，当心跳断开时，将自己变为candidate发起选举并投自己一票。
- **candidate** ：候选者。像其他节点请求投票信息，让他们投自己一票，绝大部分人通过，就成为leader。
- **term** ：任期。记录每个leader的任期，成功选举一次，任期会加1。

## 2. 初始状态

刚开始创建的节点都是follower，term都是0。每个server启动的时候都有一个选举超时延迟，这个选举超时延迟是随机生成的处于某一范围内的值，因此该延迟短的server先发起竞选——变为candidate,term+1,向其他server发送请求投票RPC，因为其最先发起竞选，term最大，因此能被选为leader。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/312312123.png)

比如 A 节点等待超时的时间间隔 150 ms，B 节点 200 ms，C 节点 300 ms。那么 a 先超时，最先因为没有等到领导者的心跳信息，发生超时。如下图所示，三个节点的超时计时器开始运行。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/2312312312321.gif)

## 3. 发起投票

当A最超时超时时间到了后，A把自己从follower状态变为candidate状态，term加1并投自己一票，并让其他节点B、C也投自己一票。

**流程如下** ：
- 触发超时后，A把自己从follower变为candidate，term加1并投自己一票，并让其他节点给自己投票（B、C）。
- B、C收到投票消息后
	- 如果B、C的term比A的term小。
		- 如果自己是follower。内自己还没有投票，那就投A一票。并改term为A的term。
		- 如果自己是candidate或者leader。收到投票消息后，将自己恢复为follower状态。
	- 如果B、C的term与A的term相同。则说明已经投票过了，会忽略请求。
	- 如果B、C的term比A的term大。则说明已经是上一届的投票了，直接解决这个投票。
- A处理结果
	- 如果自己得到了大多数节点的投票，那么自己将从candidate变为leader。并向其他节点发送消息告诉自己成为leader，其他节点与其发送心跳信息。
	- 如果自己没有得到大多数节点的投票，那么自己将从candidate变为follower。
	- 如果一段时间内没有人为leader（无人投票过半）一段时间后重新发起投票。
	- 如果A收到收到其他节点为leader生命消息的term比自己还小，自己还会保持candidate；否则转为follower。


![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/4312321321.gif)

## 4. 任期

- term初始化的时候都是0，当自己与leader断开超时时候，自己升级了candidate并把term加1发起投票。
- 如果发现比自己大的term发来投票，会将自己的term更新为大的term。
- 如果发现比自己小的term发来投票，会直接拒绝投票。
- 如果发现与自己相同的term发来投票，会直接忽略投票。
- 由于上面的机制，保证了一个节点在一个term期间只有一张选票。

## 5. 如何方式多节点同时发起投票

上面我们已经提到了，每个节点会设置随机的超时时间，可以有效的避免在同一时间内发起选举，从而减少了因选票瓜分导致选举失败的情况。

## 6. 节点挂了

每次rpcs挂了都会进行重试。

- leader挂了：当领导者A挂了时候，B、C会触发新一轮选举。
- follower挂了：无法参与投票和变为candidate。
- candidate挂了：此次选举就搁置了。

**leader挂了流程如下** ：
- 如触发C先超时后，C把自己从follower变为candidate，term加1（ 1 + 1 = 2）并投自己一票，并让其他节点给自己投票（A、B）。
- B把自己票投给了C（B的term为1，C的term为2，没投过，投给B并term改为2）。
- A服务器挂了，不能投票，所以B一共收到了2票（自己一票，B一票）。自己变为leader。
- 节点 C 向节点 A 和 B 发送心跳信息，节点 B 响应心跳信息，节点 A 不响应心跳信息。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/3123213151257.gif)

## 7. Raft日志复制

每个leader收到请求后，执行指令操作，并且携带日志索引和term并行发送所有的followers，每次follower收到消息后，如果可以和leader的对应上，则追加到后面，<font color="red">如果不一致，raft是通过跟随者强制复制领导者的日志来保证的</font>。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/789423412.png)

如下面不一致的情况。a、b会同步日志；c、d会丢弃日志；e、f会覆盖修正日志（如何覆盖下面会说）。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/31231255.png)

## 8. leader如何覆盖follower日志

如上图的情况，节点刚成为leader时候，会初始化一个数组Arr。这个Arr的初始只为最后一个数加1。上面的情况leader的index为11。而同步c、d、f时候发现11处已经出现重复（和自己的term不一致，因为刚升级为leader的时候term会加1）。这时候以leader为准，覆盖掉日志。

## 9. 当同步很差日志的follower被选为leader时

如果同步很差的follower被选为leader，根据一切以leader为准的原则，会造成正常同步的follower被覆盖的情况。

所以<font color="red">我们让同步好的节点才有可能成为leader</font>。Raft中节点在投票的时候，会判断被投票的候选者对应的日志是否至少和自己一样新。如果不是，则不会给该候选者投票。

**日志比较的方法** ：
- 最后一条日志的任期号。如果大说明新。如果小，说明不新。如果相等。看索引长度
- 判断索引长度。大的更新。

## 10. 日志流程推演示例

### 10.1 正常流程

一开始，Leader 和 两个 Follower 都没有任何数据。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/63424123.png)

客户端发送请求给 Leader，储存数据 “sally”，Leader 先将数据写在本地日志，这时候数据还是 Uncommitted (还没最终确认，红色表示)

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/g3412412.png)

Leader 给两个 Follower 发送 AppendEntries 请求，数据在 Follower 上没有冲突，则将数据暂时写在本地日志，Follower 的数据也还是 Uncommitted。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/g3r2123213.png)

Follower 将数据写到本地后，返回 OK。Leader 收到后成功返回，只要收到的成功的返回数量超过半数 (包含Leader)，Leader 将数据 “sally” 的状态改成 Committed。( 这个时候 Leader 就可以返回给客户端了)

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/fw24231.png)

Leader 再次给 Follower 发送 AppendEntries 请求，收到请求后，Follower 将本地日志里 Uncommitted 数据改成 Committed。这样就完成了一整个复制日志的过程，三个节点的数据是一致的.

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/d21ijio312.png)

### 10.2 分区异常情况

部分节点之间没办法互相通信，Raft 也能保证在这种情况下数据的一致性。

一开始有 5 个节点处于同一网络状态下。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/dqwhui123.png)

将节点分成两边，一边有两个节点，一边三个节点。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/dsadhauhi213.png)

两个节点这边已经有 Leader 了，来自客户端的数据 “bob” 通过 Leader 同步到 Follower。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/dash123739.png)

因为只有两个节点，少于3个节点，所以 “bob” 的状态仍是 Uncommitted。所以在这里，服务器会返回错误给客户端

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/su2j5ks.png)

另外一个 Partition 有三个节点，进行重新选主。客户端数据 “tom” 发到新的 Leader，通过和上节网络状态下相似的过程，同步到另外两个 Follower。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/dajjio123.png)

因为这个 Partition 有3个节点，超过半数，所以数据 “tom” 都 Commit 了。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/3123121fsdf.jpg)

网络状态恢复，5个节点再次处于同一个网络状态下。但是这里出现了数据冲突 “bob" 和 “tom"

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/1dasdjui123.png)

三个节点的 Leader 广播 AppendEntries

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/y78hnjl.png)

两个节点 Partition 的 Leader 自动降级为 Follower，因为这个 Partition 的数据 “bob” 没有 Commit，返回给客户端的是错误，客户端知道请求没有成功，所以 Follower 在收到 AppendEntries 请求时，可以把 “bob“ 删除，然后同步 ”tom”，通过这么一个过程，就完成了在 Network Partition 情况下的复制日志，保证了数据的一致性。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/miojou98j.png)

最终状态

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/mklm23.png)


### 10.3 未知状态

写入的状态：成功、失败、未知。

**未知但最后成功的情况：**

如果有5个节点，其中挂了3个。S5是leader，S1是Slave，S2、S3、S4挂掉了，这时候term为3，向leaderS5写入数据3。这时候S1、S5的数据3状态都是uncommitted，因为没有达到多数派。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/vjiu1239jai2.png)

这时候S2、S3、S4上线，leader S5发送未提交的数据3给S2、S3、S4会写入成功，最终数据会写入成功。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/j9821ehjb.png)

**未知但最后失败的情况：**

如果有5个节点，其中挂了3个。S5是leader，S1是Slave，S2、S3、S4挂掉了，这时候term为3，再次向leaderS5写入数据3。这时候S1、S5的数据3状态都是uncommitted，因为没有达到多数派。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/jo872312jklws.png)

这时候如果S5、S1挂了，S2、S3、S4好了，会重新出发选举。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/dasjoi2429310ujo.png)

假设S4先超时，触发选举并当选term4号leader。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/djaijo1230gfc8.png)

这时候我们向S4 leader写入数据4，因为达到多数派同步，所以数据4最后写入成功。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/dasml21329iods.png)

这时候我们S1、S5恢复上线，他们的term为3，因为这时候term最大已经为4了，S5发送的消息会被其他拒绝（因为term太小），而S4告诉S5自己已经是term4的leader了，所以S5变为slave。由于数据强制以leader为准，所以S5、S1的uncommitted数据3最终会被清除，如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/daji122390jjnxkm.png)


参考文章：
- [raft动画掩饰](http://thesecretlivesofdata.com/raft/)
- [raft官网各种情况动画](https://raft.github.io/)
- [共识算法：Raft](https://www.jianshu.com/p/8e4bbe7e276c)
- [raft图解(秒懂)](https://www.cnblogs.com/crazymakercircle/p/14343154.htm)





