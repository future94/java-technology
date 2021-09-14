## 1. 集群

### 1.1 Redis主从复制

拓扑结构:

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/67539461-d1a26c00-f714-11e9-81ae-61fa89faf156.png)

### 1.2 同步方式

- 全量同步：一般发生在slave初始化的时候，slave连接master，发送SYNC命令，master接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令，主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令，slave收到快照之后删除旧数据，执行快照中的数据命令，然后接收数据命令，提供服务。
- 增量同步：slave初始化完成时候，master每执行完一个命令都会发送给slave进行同步，这个就是增量同步。

#### 问题点
- 同步故障
    - 复制数据延迟(不一致)
    - **读取过期数据** (Slave不会让key过期，不能删除数据，而是等接收到了master中的key过期之后发送给slave的del命令)
    - 从节点故障
    - 主节点故障
- 配置不一致
    - maxmemory 不一致:丢失数据
    - 优化参数不一致:内存不一致.
- 避免全量复制
    - 选择小主节点(分片)、低峰期间操作.
    - 如果节点运行 id 不匹配(如主节点重启、运行 id 发送变化)，此时要执行全量复制，应该配合哨兵和集群解决.
    - 主从复制挤压缓冲区不足产生的问题(网络中断，部分复制无法满足)，可增大复制缓冲区( rel_backlog_size 参数).
- 复制风暴

### 1.2 哨兵模式（sentinel）

主从模式很好，但是一般情况下之后主能写，master挂了咋么办呢，这时候就出现了sentinel对master进行监控，并在错误的时候能进行master与slave的切换。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/20170425225507386.png)

**quorum和majority**
- quorum：确认客观下线的最少的哨兵数量
- majority：授权进行主从切换的最少的哨兵数量

**主观下线（sdown）与客观下线（odown）** ：
- 主观下线
    - 即 Sentinel 节点对 Redis 节点失败的偏见，超出超时时间认为 Master 已经宕机。
    - Sentinel 集群的每一个 Sentinel 节点会定时对 Redis 集群的所有节点发心跳包检测节点是否正常。如果一个节点在 `down-after-milliseconds` 时间内没有回复 Sentinel 节点的心跳包，则该 Redis 节点被该 Sentinel 节点主观下线。
- 客观下线
    - 所有 Sentinel 节点对 Redis 节点失败要达成共识，即超过 `quorum` 个统一。
    - 当节点被一个 Sentinel 节点记为主观下线时，并不意味着该节点肯定故障了，还需要 Sentinel 集群的其他 Sentinel 节点共同判断为主观下线才行。
    - 该 Sentinel 节点会询问其它 Sentinel 节点，如果 Sentinel 集群中超过 `quorum` 数量的 Sentinel 节点认为该 Redis 节点主观下线，则该 Redis 客观下线。


**Leader选举**
选举出一个 Sentinel 作为 Leader：集群中至少有三个 Sentinel 节点，但只有其中一个节点可完成故障转移.通过以下命令可以进行失败判定或领导者选举。

步骤：（具体可看[Raft算法详解](../../algorithm/Raft算法详解.md)）
1. 每个主观下线的 Sentinel 节点向其他 Sentinel 节点发送命令，要求设置它为领导者.
2. 收到命令的 Sentinel 节点如果没有同意通过其他 Sentinel 节点发送的命令，则同意该请求，否则拒绝。
3. 如果该 Sentinel 节点发现自己的票数已经超过 Sentinel 集合半数且超过 quorum，则它成为领导者。
4. 如果此过程有多个 Sentinel 节点成为领导者，则等待一段时间再重新进行选举。

**故障转移（failover）**
- 转移流程
    1. Sentinel 选出一个合适的 Slave 作为新的 Master(slaveof no one 命令)。
    2. 向其余 Slave 发出通知，让它们成为新 Master 的 Slave( parallel-syncs 参数)。
    3. 等待旧 Master 复活，并使之称为新 Master 的 Slave。
    4. 向客户端通知 Master 变化。
- 从 Slave 中选择新 Master 节点的规则(slave 升级成 master 之后)
    1. 选择 slave-priority 最高的节点。
    2. 选择复制偏移量最大的节点(同步数据最多)。
    3. 选择 runId 最小的节点。

> Sentinel 集群运行过程中故障转移完成，所有 Sentinel 又会恢复平等。Leader 仅仅是故障转移操作出现的角色。

配置：
```conf

sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```
- **monitor** ：`sentinel monitor [master-group-name] [ip] [port] [quorum]`。quorum为票数，下面就是至少有2个slave认为master挂了才会触发选举
- **down-after-milliseconds** ：sentinel会向master发送心跳PING来确认master是否存活，如果master在“一定时间范围”内不回应PONG 或者是回复了一个错误消息，那么这个sentinel会主观地认为这个master已经不可用了。而这个down-after-milliseconds就是用来指定这个“一定时间范围”的，单位是毫秒。
- **parallel-syncs** ：在发生failover主从切换时，这个选项指定了最多可以有多少个slave同时对新的master进行同步，这个数字越小，完成主从故障转移所需的时间就越长，但是如果这个数字越大，就意味着越多的slave因为主从同步而不可用。可以通过将这个值设为1来保证每次只有一个slave处于不能处理命令请求的状态。

**配置版本号**

这个版本就相当于Raft里的term。每次failover都会附带有一个独一无二的版本号。sentinel集群都遵守一个规则：如果sentinel A推荐sentinel B去执行failover，B会等待一段时间后，自行再次去对同一个master执行failover，这个等待的时间是通过`failover-timeout`配置项去配置的。从这个规则可以看出，sentinel集群中的sentinel不会再同一时刻并发去failover同一个master，第一个进行failover的sentinel如果失败了，另外一个将会在一定时间内进行重新进行failover，以此类推。

例子：假设有一个名为mymaster的地址为192.168.1.50:6379。一开始，集群中所有的sentinel都知道这个地址，于是为mymaster的配置打上版本号1。一段时候后mymaster死了，有一个sentinel被授权用版本号2对其进行failover。如果failover成功了，假设地址改为了192.168.1.50:9000，此时配置的版本号为2，进行failover的sentinel会将新配置广播给其他的sentinel，由于其他sentinel维护的版本号为1，发现新配置的版本号为2时，版本号变大了，说明配置更新了，于是就会采用最新的版本号为2的配置。

**Slave选举与优先级** 
当一个sentinel准备好了要进行failover，并且收到了其他sentinel的授权，那么就需要选举出一个合适的slave来做为新的master。

slave的选举主要会评估slave的以下几个方面：
- 与master断开连接的次数
- Slave的优先级
- 数据复制的下标(用来评估slave当前拥有多少master的数据)
- 进程ID

如果一个slave与master失去联系超过10次，并且每次都超过了配置的最大失联时间(`down-after-milliseconds option`)，并且，如果sentinel在进行failover时发现slave失联，那么这个slave就会被sentinel认为不适合用来做新master的。

更严格的定义是，如果一个slave持续断开连接的时间超过
```
(down-after-milliseconds * 10) + milliseconds_since_master_is_in_SDOWN_state
```
就会被认为失去选举资格。符合上述条件的slave才会被列入master候选人列表，并根据以下顺序来进行排序：
- sentinel首先会根据slaves的优先级来进行排序，优先级越小排名越靠前（？）。
- 如果优先级相同，则查看复制的下标，哪个从master接收的复制数据多，哪个就靠前。
- 如果优先级和下标都相同，就选择进程ID较小的那个。

一个redis无论是master还是slave，都必须在配置中指定一个slave优先级。要注意到master也是有可能通过failover变成slave的。

如果一个redis的slave优先级配置为0，那么它将永远不会被选为master。但是它依然会从master哪里复制数据。

### 1.3 集群模式（Cluster）

前提，至少3主3从才能形成Cluster集群。集群相当于主从复制和哨兵模式的合成版本。

上面已经介绍了主从复制和哨兵模式，很强大，但是有一个问题，就是数据是共同存储的，如有1master和2slave，那么数据就会被存3份，非常的浪费。于是就有了cluster，多个master共同存储，每个master下面还有slave进行容灾备份。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/08afdecff95a.png)

**数据定位的算法**
那么多master，我们怎么知道最后使用的是哪个master呢？Redis使用的是[一致性Hash算法](../../algorithm/一致性Hash算法原理与实现.md)的变种形式。一致性hash算法采用的 `2^32` 取模，Redis是对 `2^14` （16384）取模，每个值称为`slot`。每个Redis实例会自己维护一份slot - Redis节点的映射关系。

数据具体通过 `CRC16(key)%16384` 来计算出来 slot 值，并找到与redis节点的对应关系。

Redis Cluster中保证集群高可用的思路和实现和Redis Sentinel如出一辙，就是cluster中的master挂了之后，选中其中的slave节点替换为master。动态扩容与缩减与[一致性Hash算法](../../algorithm/一致性Hash算法原理与实现.md)实现一致。

Redis Cluster各个节点之间交换数据、通信所采用的一种协议，叫做[gossip协议](https://www.jianshu.com/p/8279d6fd65bb)。有兴趣可以点击查看。

## 2. 实战搭建

## 2.1 Redis主从复制

Redis集群不用安装多个Redis,只需复制多个配置文件,修改即可。

类型 | ip | 端口
---|---|---
Master | 127.0.0.1 | 6379
Slave | 127.0.0.1 | 6389
Slave | 127.0.0.1 | 6399

修改配置：

redis-6379.conf
```conf
# Redis使用后台模式
daemonize yes

# 关闭保护模式
protected-mode no

# 修改启动端口为6379
port 6379

# 密码
requirepass 123456
```

redis-6389.conf
```conf
# Redis使用后台模式
daemonize yes

# 关闭保护模式
protected-mode no

# 修改启动端口为6389
port 6389

# 密码
requirepass 123456

# 主库密码
masterauth 123456
```

redis-6399.conf
```conf
# Redis使用后台模式
daemonize yes

# 关闭保护模式
protected-mode no

# 修改启动端口为6399
port 6399

# 密码
requirepass 123456

# 主库密码
masterauth 123456
```
启动redis服务
```shell
redis-server /usr/local/etc/redis-6379.conf
redis-server /usr/local/etc/redis-6389.conf
redis-server /usr/local/etc/redis-6399.conf
```

**设置主从模式** 
- 在slave的redis.conf中设置，格式为：`slaveof [masterip] [masterport]` ，永久有效。
- 在slave客户端执行命令：`slaveof [masterip] [masterport]` ，重启失效。

**slave启动输出**：
```txt
21297:S 13 Sep 2021 22:59:08.756 # Server initialized
21297:S 13 Sep 2021 22:59:08.756 * Loading RDB produced by version 6.2.5
21297:S 13 Sep 2021 22:59:08.756 * RDB age 33 seconds
21297:S 13 Sep 2021 22:59:08.756 * RDB memory usage when created 0.98 Mb
21297:S 13 Sep 2021 22:59:08.756 * DB loaded from disk: 0.000 seconds
21297:S 13 Sep 2021 22:59:08.756 * Ready to accept connections
21297:S 13 Sep 2021 22:59:08.756 * Connecting to MASTER 127.0.0.1:6379
21297:S 13 Sep 2021 22:59:08.756 * MASTER <-> REPLICA sync started
21297:S 13 Sep 2021 22:59:08.756 * Non blocking connect for SYNC fired the event.
21297:S 13 Sep 2021 22:59:08.756 * Master replied to PING, replication can continue...
21297:S 13 Sep 2021 22:59:08.756 * Partial resynchronization not possible (no cached master)
21297:S 13 Sep 2021 22:59:08.757 * Full resync from master: 1104fd20699b35e2cbeb0747e255f528f0737daf:0
21297:S 13 Sep 2021 22:59:08.830 * MASTER <-> REPLICA sync: receiving 175 bytes from master to disk
21297:S 13 Sep 2021 22:59:08.830 * MASTER <-> REPLICA sync: Flushing old data
21297:S 13 Sep 2021 22:59:08.830 * MASTER <-> REPLICA sync: Loading DB in memory
21297:S 13 Sep 2021 22:59:08.832 * Loading RDB produced by version 6.2.5
21297:S 13 Sep 2021 22:59:08.832 * RDB age 0 seconds
21297:S 13 Sep 2021 22:59:08.832 * RDB memory usage when created 2.06 Mb
21297:S 13 Sep 2021 22:59:08.832 * MASTER <-> REPLICA sync: Finished with success
```

**查看主从信息**
- master

```txt
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:2
slave0:ip=127.0.0.1,port=6389,state=online,offset=1707,lag=0
slave1:ip=127.0.0.1,port=6399,state=online,offset=1707,lag=1
master_failover_state:no-failover
master_replid:1104fd20699b35e2cbeb0747e255f528f0737daf
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1707
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1707
```

- slave

```txt
127.0.0.1:6389> info replication
# Replication
role:slave
master_host:127.0.0.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:2
master_sync_in_progress:0
slave_repl_offset:1805
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:1104fd20699b35e2cbeb0747e255f528f0737daf
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:1805
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:1805
```

测试主从：

```txt
127.0.0.1:6379> set weilai cool
OK
127.0.0.1:6389> get weilai
"cool"
127.0.0.1:6399> get weilai
"cool"
127.0.0.1:6379> del weilai
(integer) 1
127.0.0.1:6389> get weilai
(nil)
127.0.0.1:6399> get weilai
(nil)
127.0.0.1:6389> set la 123
(error) READONLY You can't write against a read only replica.
```

slave写入 `set la 123` 时候会报错，默认从库指定读，可以添加配置关闭。
```conf
slave-read-only no
```

## 2.2 哨兵模式

使用哨兵模式的前提：主从模式已经搭建完成并正确运行。

服务类型 | 主从类型 | ip | 端口
---|---|---
Redis | Master | 127.0.0.1 | 6379
Redis | Slave | 127.0.0.1 | 6389
Redis | Slave | 127.0.0.1 | 6399
Redis-Sentinel | - | 127.0.0.1 | 26379
Redis-Sentinel | - | 127.0.0.1 | 26389
Redis-Sentinel | - | 127.0.0.1 | 26399

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/3f40b17z412116c.png)

修改配置redis-sentinel-26379.conf
```
# 设置端口
port 26379

# 监控主库 mymaster是给主库起的名字，2是有两个人同意就会成功选为master（因为我们有3个服务器）
sentinel monitor mymaster 127.0.0.1 6379 2
```

修改配置redis-sentinel-26389.conf
```
# 设置端口
port 26389

# 监控主库 mymaster是给主库起的名字，2是有两个人同意就会成功选为master（因为我们有3个服务器）
sentinel monitor mymaster 127.0.0.1 6379 2
```

修改配置redis-sentinel-26399.conf
```
# 设置端口
port 26399

# 监控主库 mymaster是给主库起的名字，2是有两个人同意就会成功选为master（因为我们有3个服务器）
sentinel monitor mymaster 127.0.0.1 6379 2
```

启动哨兵（先启动master、在启动slave、在启动sentinel）
```
redis-sentinel /usr/local/etc/redis-sentinel-26379.conf
redis-sentinel /usr/local/etc/redis-sentinel-26389.conf
redis-sentinel /usr/local/etc/redis-sentinel-26399.conf
```

sentinel-26379输出：
```txt
22856:X 14 Sep 2021 11:44:02.874 # Sentinel ID is 942f2ec1dc44c86bd26ef4b6a84a6dcec18ba243
22856:X 14 Sep 2021 11:44:02.874 # +monitor master mymaster 127.0.0.1 6379 quorum 2
22856:X 14 Sep 2021 11:44:02.875 * +slave slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6379
22856:X 14 Sep 2021 11:44:02.876 * +slave slave 127.0.0.1:6399 127.0.0.1 6399 @ mymaster 127.0.0.1 6379
22856:X 14 Sep 2021 11:44:16.809 * +sentinel sentinel 1c13c227bccdaf6a362ae8df0213c36c195ca621 127.0.0.1 26389 @ mymaster 127.0.0.1 6379
22856:X 14 Sep 2021 11:44:23.353 * +sentinel sentinel a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 127.0.0.1 26399 @ mymaster 127.0.0.1 6379
```

sentinel-26389输出：
```txt
22863:X 14 Sep 2021 11:44:14.744 # Sentinel ID is 1c13c227bccdaf6a362ae8df0213c36c195ca621
22863:X 14 Sep 2021 11:44:14.744 # +monitor master mymaster 127.0.0.1 6379 quorum 2
22863:X 14 Sep 2021 11:44:14.745 * +slave slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6379
22863:X 14 Sep 2021 11:44:14.746 * +slave slave 127.0.0.1:6399 127.0.0.1 6399 @ mymaster 127.0.0.1 6379
22863:X 14 Sep 2021 11:44:15.186 * +sentinel sentinel 942f2ec1dc44c86bd26ef4b6a84a6dcec18ba243 127.0.0.1 26379 @ mymaster 127.0.0.1 6379
22863:X 14 Sep 2021 11:44:23.354 * +sentinel sentinel a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 127.0.0.1 26399 @ mymaster 127.0.0.1 6379
```

sentinel-26399输出：
```txt
22870:X 14 Sep 2021 11:44:21.300 # Sentinel ID is a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0
22870:X 14 Sep 2021 11:44:21.300 # +monitor master mymaster 127.0.0.1 6379 quorum 2
22870:X 14 Sep 2021 11:44:21.300 * +slave slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:44:21.302 * +slave slave 127.0.0.1:6399 127.0.0.1 6399 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:44:21.316 * +sentinel sentinel 942f2ec1dc44c86bd26ef4b6a84a6dcec18ba243 127.0.0.1 26379 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:44:22.860 * +sentinel sentinel 1c13c227bccdaf6a362ae8df0213c36c195ca621 127.0.0.1 26389 @ mymaster 127.0.0.1 6379
```

redis-sentinel-(26379、26389、26399).conf中追加了下面配置：
```conf
sentinel myid a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0
sentinel config-epoch mymaster 0
sentinel leader-epoch mymaster 0
sentinel current-epoch 0
sentinel known-replica mymaster 127.0.0.1 6389
sentinel known-replica mymaster 127.0.0.1 6399
sentinel known-sentinel mymaster 127.0.0.1 26389 1c13c227bccdaf6a362ae8df0213c36c195ca621
sentinel known-sentinel mymaster 127.0.0.1 26379 942f2ec1dc44c86bd26ef4b6a84a6dcec18ba243
```

哨兵模式的代码使用（Jedis为例子，redisTemplate一样的）
```java
import redis.clients.jedis.Client;
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPoolConfig;
import redis.clients.jedis.JedisSentinelPool;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

/**
 * @author weilai
 */
public class RedisTest {

    public static void main(String[] args) {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMaxTotal(10);
        jedisPoolConfig.setMaxIdle(5);
        jedisPoolConfig.setMinIdle(5);
        // 哨兵信息
        Set<String> sentinels = new HashSet<>(Arrays.asList("127.0.0.1:26379",
                "127.0.0.1:26389","127.0.0.1:26399"));
        // 创建连接池
        JedisSentinelPool pool = new JedisSentinelPool("mymaster", sentinels, jedisPoolConfig);
        System.out.println("当前线程池master：" + pool.getCurrentHostMaster());
        // 获取客户端
        Jedis jedis = pool.getResource();
        Client client = jedis.getClient();
        System.out.println("获取客户端：" + client.getHost() + ":" + client.getPort());
        // 执行两个命令
        jedis.set("mykey", "myvalue");
        String value = jedis.get("mykey");
        System.out.println("读取的值:" + value);
        jedis.close();
        pool.destroy();
    }
}
```

运行结果：
```txt
当前线程池master：127.0.0.1:6379
获取客户端：127.0.0.1:6379
读取的值:myvalue
```

当master节点（6379）挂了时（这里我们主动kill掉），哨兵会监控到超时，重新选举。


redis-6389输出：
```txt
22620:S 14 Sep 2021 11:59:37.637 * Connecting to MASTER 127.0.0.1:6379
22620:S 14 Sep 2021 11:59:37.637 * MASTER <-> REPLICA sync started
22620:S 14 Sep 2021 11:59:37.637 # Error condition on socket for SYNC: Connection refused
22620:S 14 Sep 2021 11:59:37.695 * Connecting to MASTER 127.0.0.1:6399
22620:S 14 Sep 2021 11:59:37.695 * MASTER <-> REPLICA sync started
22620:S 14 Sep 2021 11:59:37.696 * REPLICAOF 127.0.0.1:6399 enabled (user request from 'id=16 addr=127.0.0.1:49462 laddr=127.0.0.1:6389 fd=14 name=sentinel-a89b1e0a-cmd age=916 idle=0 flags=x db=0 sub=0 psub=0 multi=4 qbuf=329 qbuf-free=65201 argv-mem=4 obl=45 oll=0 omem=0 tot-mem=82980 events=r cmd=exec user=default redir=-1')
22620:S 14 Sep 2021 11:59:37.698 # CONFIG REWRITE executed with success.
22620:S 14 Sep 2021 11:59:37.698 * Non blocking connect for SYNC fired the event.
22620:S 14 Sep 2021 11:59:37.698 * Master replied to PING, replication can continue...
22620:S 14 Sep 2021 11:59:37.698 * Trying a partial resynchronization (request b4cb53908c982860c19d3bff2a3fa43e0c62ffc2:177315).
22620:S 14 Sep 2021 11:59:37.698 * Successful partial resynchronization with master.
22620:S 14 Sep 2021 11:59:37.698 # Master replication ID changed to 9113174427d7d0e919487079765f2c7f26122101
22620:S 14 Sep 2021 11:59:37.698 * MASTER <-> REPLICA sync: Master accepted a Partial Resynchronization.
```

redis-6399输出：
```txt
22622:S 14 Sep 2021 11:59:34.909 # Error condition on socket for SYNC: Connection refused
22622:S 14 Sep 2021 11:59:35.937 * Connecting to MASTER 127.0.0.1:6379
22622:S 14 Sep 2021 11:59:35.937 * MASTER <-> REPLICA sync started
22622:S 14 Sep 2021 11:59:35.937 # Error condition on socket for SYNC: Connection refused
22622:S 14 Sep 2021 11:59:36.974 * Connecting to MASTER 127.0.0.1:6379
22622:S 14 Sep 2021 11:59:36.974 * MASTER <-> REPLICA sync started
22622:S 14 Sep 2021 11:59:36.974 # Error condition on socket for SYNC: Connection refused
22622:M 14 Sep 2021 11:59:36.995 * Discarding previously cached master state.
22622:M 14 Sep 2021 11:59:36.995 # Setting secondary replication ID to b4cb53908c982860c19d3bff2a3fa43e0c62ffc2, valid up to offset: 177315. New replication ID is 9113174427d7d0e919487079765f2c7f26122101
22622:M 14 Sep 2021 11:59:36.995 * MASTER MODE enabled (user request from 'id=9 addr=127.0.0.1:49464 laddr=127.0.0.1:6399 fd=14 name=sentinel-a89b1e0a-cmd age=915 idle=0 flags=x db=0 sub=0 psub=0 multi=4 qbuf=188 qbuf-free=65342 argv-mem=4 obl=45 oll=0 omem=0 tot-mem=82980 events=r cmd=exec user=default redir=-1')
22622:M 14 Sep 2021 11:59:36.998 # CONFIG REWRITE executed with success.
22622:M 14 Sep 2021 11:59:37.698 * Replica 127.0.0.1:6389 asks for synchronization
22622:M 14 Sep 2021 11:59:37.698 * Partial resynchronization request from 127.0.0.1:6389 accepted. Sending 422 bytes of backlog starting from offset 177315.
```

redis-sentinel-26379输出：
```txt
22856:X 14 Sep 2021 11:59:36.739 # +sdown master mymaster 127.0.0.1 6379
22856:X 14 Sep 2021 11:59:36.779 # +new-epoch 1
22856:X 14 Sep 2021 11:59:36.780 # +vote-for-leader a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 1
22856:X 14 Sep 2021 11:59:36.833 # +odown master mymaster 127.0.0.1 6379 #quorum 3/2
22856:X 14 Sep 2021 11:59:36.833 # Next failover delay: I will not start a failover before Tue Sep 14 12:05:37 2021
22856:X 14 Sep 2021 11:59:37.695 # +config-update-from sentinel a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 127.0.0.1 26399 @ mymaster 127.0.0.1 6379
22856:X 14 Sep 2021 11:59:37.695 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6399
22856:X 14 Sep 2021 11:59:37.696 * +slave slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6399
22856:X 14 Sep 2021 11:59:37.696 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
22856:X 14 Sep 2021 12:00:07.716 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
```

redis-sentinel-26389输出：
```txt
22863:X 14 Sep 2021 11:59:36.704 # +sdown master mymaster 127.0.0.1 6379
22863:X 14 Sep 2021 11:59:36.779 # +new-epoch 1
22863:X 14 Sep 2021 11:59:36.780 # +vote-for-leader a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 1
22863:X 14 Sep 2021 11:59:37.695 # +config-update-from sentinel a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 127.0.0.1 26399 @ mymaster 127.0.0.1 6379
22863:X 14 Sep 2021 11:59:37.695 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6399
22863:X 14 Sep 2021 11:59:37.696 * +slave slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6399
22863:X 14 Sep 2021 11:59:37.696 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
22863:X 14 Sep 2021 12:00:07.704 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
```

redis-sentinel-26399输出：
```txt
22870:X 14 Sep 2021 11:59:36.722 # +sdown master mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:36.776 # +odown master mymaster 127.0.0.1 6379 #quorum 2/2
22870:X 14 Sep 2021 11:59:36.776 # +new-epoch 1
22870:X 14 Sep 2021 11:59:36.776 # +try-failover master mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:36.778 # +vote-for-leader a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 1
22870:X 14 Sep 2021 11:59:36.780 # 942f2ec1dc44c86bd26ef4b6a84a6dcec18ba243 voted for a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 1
22870:X 14 Sep 2021 11:59:36.780 # 1c13c227bccdaf6a362ae8df0213c36c195ca621 voted for a89b1e0a14fd308bc0bf2d7ba830f42c4b4bb6e0 1
22870:X 14 Sep 2021 11:59:36.881 # +elected-leader master mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:36.881 # +failover-state-select-slave master mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:36.937 # +selected-slave slave 127.0.0.1:6399 127.0.0.1 6399 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:36.937 * +failover-state-send-slaveof-noone slave 127.0.0.1:6399 127.0.0.1 6399 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:36.995 * +failover-state-wait-promotion slave 127.0.0.1:6399 127.0.0.1 6399 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:37.642 # +promoted-slave slave 127.0.0.1:6399 127.0.0.1 6399 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:37.642 # +failover-state-reconf-slaves master mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:37.695 * +slave-reconf-sent slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:37.863 # -odown master mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:38.654 * +slave-reconf-inprog slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:38.654 * +slave-reconf-done slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:38.727 # +failover-end master mymaster 127.0.0.1 6379
22870:X 14 Sep 2021 11:59:38.727 # +switch-master mymaster 127.0.0.1 6379 127.0.0.1 6399
22870:X 14 Sep 2021 11:59:38.728 * +slave slave 127.0.0.1:6389 127.0.0.1 6389 @ mymaster 127.0.0.1 6399
22870:X 14 Sep 2021 11:59:38.728 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
22870:X 14 Sep 2021 12:00:08.770 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
```

我们可以看出，redis-6399被选为了master。6389和6399配置文件中slaveof不见了，6389多了一行 `replicaof 127.0.0.1 6399` 。

再次运行上面代码，master已经变为了6399.
```txt
当前线程池master：127.0.0.1:6399
获取客户端：127.0.0.1:6399
读取的值:myvalue
```

#### 这时候我们启动redis6379，6379转为6399的slave，6399并会同步数据给6379。

redis6379输出如下：
```txt
23016:M 14 Sep 2021 12:05:16.584 * Ready to accept connections
23016:S 14 Sep 2021 12:05:26.873 * Before turning into a replica, using my own master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.
23016:S 14 Sep 2021 12:05:26.873 * Connecting to MASTER 127.0.0.1:6399
23016:S 14 Sep 2021 12:05:26.873 * MASTER <-> REPLICA sync started
23016:S 14 Sep 2021 12:05:26.873 * REPLICAOF 127.0.0.1:6399 enabled (user request from 'id=3 addr=127.0.0.1:52180 laddr=127.0.0.1:6379 fd=8 name=sentinel-1c13c227-cmd age=10 idle=0 flags=x db=0 sub=0 psub=0 multi=4 qbuf=196 qbuf-free=65334 argv-mem=4 obl=45 oll=0 omem=0 tot-mem=82980 events=r cmd=exec user=default redir=-1')
23016:S 14 Sep 2021 12:05:26.876 # CONFIG REWRITE executed with success.
23016:S 14 Sep 2021 12:05:26.876 * Non blocking connect for SYNC fired the event.
23016:S 14 Sep 2021 12:05:26.876 * Master replied to PING, replication can continue...
23016:S 14 Sep 2021 12:05:26.876 * Trying a partial resynchronization (request d07f4b53a49f3ff40c8087f0f20a789d75afb411:1).
23016:S 14 Sep 2021 12:05:26.876 * Full resync from master: 9113174427d7d0e919487079765f2c7f26122101:246744
23016:S 14 Sep 2021 12:05:26.876 * Discarding previously cached master state.
23016:S 14 Sep 2021 12:05:26.903 * MASTER <-> REPLICA sync: receiving 205 bytes from master to disk
23016:S 14 Sep 2021 12:05:26.903 * MASTER <-> REPLICA sync: Flushing old data
23016:S 14 Sep 2021 12:05:26.903 * MASTER <-> REPLICA sync: Loading DB in memory
23016:S 14 Sep 2021 12:05:26.903 * Loading RDB produced by version 6.2.5
23016:S 14 Sep 2021 12:05:26.903 * RDB age 0 seconds
23016:S 14 Sep 2021 12:05:26.903 * RDB memory usage when created 2.18 Mb
23016:S 14 Sep 2021 12:05:26.903 * MASTER <-> REPLICA sync: Finished with success
```

redis6399输出如下：
```txt
22622:M 14 Sep 2021 12:05:26.876 * Replica 127.0.0.1:6379 asks for synchronization
22622:M 14 Sep 2021 12:05:26.876 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'd07f4b53a49f3ff40c8087f0f20a789d75afb411', my replication IDs are '9113174427d7d0e919487079765f2c7f26122101' and 'b4cb53908c982860c19d3bff2a3fa43e0c62ffc2')
22622:M 14 Sep 2021 12:05:26.876 * Starting BGSAVE for SYNC with target: disk
22622:M 14 Sep 2021 12:05:26.876 * Background saving started by pid 23018
23018:C 14 Sep 2021 12:05:26.877 * DB saved on disk
22622:M 14 Sep 2021 12:05:26.903 * Background saving terminated with success
22622:M 14 Sep 2021 12:05:26.903 * Synchronization with replica 127.0.0.1:6379 succeeded
```

redis-sentinel-26379输出：
```txt
22856:X 14 Sep 2021 12:05:17.070 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
```

redis-sentinel-26389输出：
```txt
22863:X 14 Sep 2021 12:05:16.916 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
22863:X 14 Sep 2021 12:05:26.873 * +convert-to-slave slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
```

redis-sentinel-26399输出：
```txt
22870:X 14 Sep 2021 12:05:17.104 # -sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ mymaster 127.0.0.1 6399
```

我们还可以通过链接sentinel用命令查询或者操作
- `PING` ：返回PONG
- `INFO` ：返回该sentinel各个section的信息
- `INFO <section>` ：返回该sentile某个section的信息，例如 Server，Clients，CPU，Stats，Sentinel
- `role` ：返回sentinel监视的所有的master name
- `SENTINEL master <master name>` ：获取sentinel 监视的某个 master信息
- `SENTINEL masters` ：列出所有被监视的主服务器，以及这些主服务器的当前状态。
- `SENTINEL slaves <master name>` ：列出给定主服务器的所有从服务器，以及这些从服务器的当前状态。
- `SENTINEL sentinels <master-name>` ：获取sentinel监视的某个master的sentinel 信息
- `SENTINEL is-master-down-by-addr <ip> <port> <current-epoch> <runid>` ：询问该sentinel，该 ip，port的master是否为down状态，如果该sentinel为tilt模式，会不理会这个询问，不去判断该master是否为主观下线状态，直接回复正常状态。
- `SENTINEL get-master-addr-by-name <master name>` ： 返回给定名字的主服务器的 IP 地址和端口号。 如果这个主服务器正在执行故障转移操作， 或者针对这个主服务器的故障转移操作已经完成， 那么这个命令返回新的主服务器的 IP 地址和端口号。
- `SENTINEL reset <pattern>` ： 重置所有名字和给定模式 pattern 相匹配的主服务器。 pattern 参数是一个 Glob 风格的模式。 重置操作清除主服务器目前的所有状态， 包括正在执行中的故障转移， 并移除目前已经发现和关联的， 主服务器的所有从服务器和 Sentinel 。首先删除关于这个cluster的状态，再重新发现这个集群的状态。例如节点挂掉之后，sentinel reset之后，就不会在sentinel slaves中看到这个节点，但是挂掉节点起来之后，挂掉节点还是会加入集群，因为挂掉节点的磁盘配置文件中还是有原来的信息。
- `SENTINEL failover <master name>` ： 当主服务器失效时， 在不询问其他 Sentinel 意见的情况下， 强制开始一次自动故障迁移 （不过发起故障转移的 Sentinel 会向其他 Sentinel 发送一个新的配置，其他 Sentinel 会根据这个配置进行相应的更新）。
- `SENTINEL moniotr <name> <ip> <port> <quorum>` ：添加监视的master
- `SENTINEL auth-pass <master-name> <password>`：如果sentinel监控的主节点设置了密码，sentinel通过以上命令添加主节点的密码
- `SENTINEL remove <name>` ：将监视的为name的master移除监视
- `SENTINEL set <mastername> [<option> <value>]` ：修改监视的master的一些属性
	- `down-after-milliseconds` ：过了这个时间考虑master go down
	- `failover-timeout` ：刷新故障转移状态的最大时间
	- `parallel-syncs` ：slave同时reconfigure的个数
	- `notification-script` ：设置通知脚本
	- `client-reconfig-script` ：设置通知脚本
	- `auth-pass` ：执行auth的密码
	- `quorum` ：修改master的quorum
- `client list`：列出服务器所有的client的相关信息
- `client kill <ip:port>` ：杀死client

## 2.3 集群模式

版本为6.2.5。6.0+已经不需要ruby了。至少需要6台。修改配置文件：
```conf
# 6001～6007配置类似
port 6001
# 注意同台机器的时候dir目录要分开，要不会Node XXX is not empty。
dir /usr/local/etc/6001
cluster-enabled yes
# 下面的文件自己会生成
cluster-config-file /usr/local/etc/node-6001.conf
```

启动6个服务.
```
redis-server /usr/local/etc/redis-6001.conf
redis-server /usr/local/etc/redis-6002.conf
redis-server /usr/local/etc/redis-6003.conf
redis-server /usr/local/etc/redis-6004.conf
redis-server /usr/local/etc/redis-6005.conf
redis-server /usr/local/etc/redis-6006.conf
```

发生下面问题，将每个节点下aof、rdb、nodes.conf本地备份文件删除即可。运行：`flushdb`。
```txt
redis-cli --cluster create 127.0.0.1:6001 127.0.0.1:6002 127.0.0.1:6003 127.0.0.1:6004 127.0.0.1:6005 127.0.0.1:6006 --cluster-replicas 1
[ERR] Node 127.0.0.1:6002 is not empty. Either the node already knows other nodes (check with CLUSTER NODES) or contains some key in database 0.
```

搭建集群：
```shell
# 1位master和slave的比例为1:1
redis-cli --cluster create 127.0.0.1:6001 127.0.0.1:6002 127.0.0.1:6003 127.0.0.1:6004 127.0.0.1:6005 127.0.0.1:6006 --cluster-replicas 1
```

输出下面日志，可以查看到slot的信息和master、slave信息：
```txt
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 127.0.0.1:6005 to 127.0.0.1:6001
Adding replica 127.0.0.1:6006 to 127.0.0.1:6002
Adding replica 127.0.0.1:6004 to 127.0.0.1:6003
>>> Trying to optimize slaves allocation for anti-affinity
[WARNING] Some slaves are in the same host as their master
M: 492acfacd78f2431356daca9c77f95a3fb5b6a91 127.0.0.1:6001
   slots:[0-5460] (5461 slots) master
M: 01522e32df886ccf78051031aba6a746245a7d2a 127.0.0.1:6002
   slots:[5461-10922] (5462 slots) master
M: 76705208f466ac1921fff5dac774650cd954e636 127.0.0.1:6003
   slots:[10923-16383] (5461 slots) master
S: c1b0b35caeaeb98049fc788b2cb2a0ff2fd126b9 127.0.0.1:6004
   replicates 01522e32df886ccf78051031aba6a746245a7d2a
S: 01abfac179d6914beca3ca186c8093accb06f3e5 127.0.0.1:6005
   replicates 76705208f466ac1921fff5dac774650cd954e636
S: 26aa30c2ad8470b0bae88330b79d34aad2a82b18 127.0.0.1:6006
   replicates 492acfacd78f2431356daca9c77f95a3fb5b6a91
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
..
>>> Performing Cluster Check (using node 127.0.0.1:6001)
M: 492acfacd78f2431356daca9c77f95a3fb5b6a91 127.0.0.1:6001
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
S: c1b0b35caeaeb98049fc788b2cb2a0ff2fd126b9 127.0.0.1:6004
   slots: (0 slots) slave
   replicates 01522e32df886ccf78051031aba6a746245a7d2a
M: 76705208f466ac1921fff5dac774650cd954e636 127.0.0.1:6003
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 26aa30c2ad8470b0bae88330b79d34aad2a82b18 127.0.0.1:6006
   slots: (0 slots) slave
   replicates 492acfacd78f2431356daca9c77f95a3fb5b6a91
M: 01522e32df886ccf78051031aba6a746245a7d2a 127.0.0.1:6002
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 01abfac179d6914beca3ca186c8093accb06f3e5 127.0.0.1:6005
   slots: (0 slots) slave
   replicates 76705208f466ac1921fff5dac774650cd954e636
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

通过redis-cli的`-c`参数指定链接集群模式，我们看到，设置mykey通过slot定位到了127.0.0.1:6003，获取wei定位到了127.0.0.1:6001。
```txt
redis-cli -c -p 6001

127.0.0.1:6001> set mykey myvalue
-> Redirected to slot [14687] located at 127.0.0.1:6003
OK
127.0.0.1:6003> get wei
-> Redirected to slot [5320] located at 127.0.0.1:6001
"lai"
```

如果我们不通过-c命令链接，可以看到下面结果，在6003 get时候返回的值，在6004获取时候，提示在6003机器。
```txt
127.0.0.1:6003> get mykey
"myvalue"
127.0.0.1:6004> get mykey
(error) MOVED 14687 127.0.0.1:6003
```

代码使用，以Jedis为例。
```java
import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.JedisPoolConfig;

import java.util.HashSet;
import java.util.Set;

/**
 * @author weilai
 */
public class RedisTest {

    public static void main(String[] args) {
        JedisPoolConfig jedisPoolConfig = new JedisPoolConfig();
        jedisPoolConfig.setMaxTotal(10);
        jedisPoolConfig.setMaxIdle(5);
        jedisPoolConfig.setMinIdle(5);
        Set<HostAndPort> config = new HashSet<>();
        config.add(HostAndPort.parseString("127.0.0.1:6001"));
        config.add(HostAndPort.parseString("127.0.0.1:6002"));
        config.add(HostAndPort.parseString("127.0.0.1:6003"));
        config.add(HostAndPort.parseString("127.0.0.1:6004"));
        config.add(HostAndPort.parseString("127.0.0.1:6005"));
        config.add(HostAndPort.parseString("127.0.0.1:6006"));
        // 创建连接池
        JedisCluster jedisCluster = new JedisCluster(config, jedisPoolConfig);
        jedisCluster.set("mykey", "myvalue");
        System.out.println(jedisCluster.get("mykey"));
    }
}
```

输出:
```txt
myvalue
```

**cluster常用命令** :

集群:

- `cluster info` ：打印集群的信息
- `cluster nodes` ：列出集群当前已知的所有节点（ node），以及这些节点的相关信息。

节点:

- `cluster meet <ip> <port>` ：将 ip 和 port 所指定的节点添加到集群当中，让它成为集群的一份子。
- `cluster forget <node_id>` ：从集群中移除 node_id 指定的节点。
- `cluster replicate <master_node_id>` ：将当前从节点设置为 node_id 指定的master节点的slave节点。只能针对slave节点操作。
- `cluster saveconfig` ：将节点的配置文件保存到硬盘里面。

槽(slot):

- `cluster addslots <slot> [slot ...]` ：将一个或多个槽（ slot）指派（ assign）给当前节点。
- `cluster delslots <slot> [slot ...]` ：移除一个或多个槽对当前节点的指派。
- `cluster flushslots` ：移除指派给当前节点的所有槽，让当前节点变成一个没有指派任何槽的节点。
- `cluster setslot <slot> node <node_id>` ：将槽 slot 指派给 node_id 指定的节点，如果槽已经指派给
另一个节点，那么先让另一个节点删除该槽>，然后再进行指派。
- `cluster setslot <slot> migrating <node_id>` ：将本节点的槽 slot 迁移到 node_id 指定的节点中。
- `cluster setslot <slot> importing <node_id>` ：从 node_id 指定的节点中导入槽 slot 到本节点。
- `cluster setslot <slot> stable` ：取消对槽 slot 的导入（ import）或者迁移（ migrate）。
键
- `cluster keyslot <key>` ：计算键 key 应该被放置在哪个槽上。
- `cluster countkeysinslot <slot>` ：返回槽 slot 目前包含的键值对数量。
- `cluster getkeysinslot <slot> <count>` ：返回 count 个 slot 槽中的键 。

参考文章：
- [sentinel 可以接收的命令](https://www.jianshu.com/p/a29050278b71)
- [redis主从、集群、哨兵](https://www.cnblogs.com/xuwc/p/8900717.html)
- [Redis Sentinel实现的机制与原理详解](https://www.cnblogs.com/knowledgesea/p/6567718.html)
- [Redis Cluster日常操作命令梳理](https://www.cnblogs.com/kevingrace/p/7910692.html)