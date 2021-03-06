
## MyISAM 和 InnoDB 的区别？

InnoDB 支持 事务、外键、聚集索引，通过MVCC来支持高并发，索引和数据存储在一起。
InnoDB 不保存表的具体行数，执行 `select count(*) from table` 时需要全表扫描。MyISAM 用一个变量保存了整个表的行数。
InnoDB 最小的锁粒度是行锁，MyISAM 最小的锁粒度是表锁，并发能力低。
MySQL 将默认存储引擎是 InnoDB.

## mysql 锁有哪些类型？

mysql锁分为共享锁( Slock ) 、排他锁 ( Xlock )，也叫做读锁和写锁。根据粒度，可以分为表锁、页锁、行锁。

## 什么是间隙锁?

间隙锁是可重复读级别下才会有的锁，mysql会帮我们生成了若干左开右闭的区间，结合MVCC和间隙锁可以解决幻读（事务1读取几行数据，然后事务2插入或删除几行数据，导致事务1在接下来的查询中发现了原本不存在的数据或者发现少了原本存在的数据）问题。

## 如何避免死锁?

死锁的四个必要条件：1、互斥；2、请求与保持； 3、环路等待； 4、不可剥夺。

- 合理的设计索引，区分度高的列放到组合索引前面，使业务 SQL 尽可能通过索引定位更少的行，减少锁竞争。
- 调整业务逻辑 SQL 执行顺序， 避免 update/delete 长时间持有锁的 SQL 在事务前面。
- 避免大事务，将大事务拆成多个小事务
- 以固定的顺序访问表和行。比如两个更新数据的事务，事务 A 更新数据的顺序为 1，2;事务 B 更新数据的顺序为 2，1。这样更可能会造成死锁。
- 在并发比较高的系统中，不要显式加锁，特别是是在事务里显式加锁。如 select … for update 语句，如果是在事务里（运行了 start transaction 或设置了autocommit 等于0）,那么就会锁定所查找到的记录。
- 尽量用主键/索引去查找记录
- 优化 SQL 和表设计，减少同时占用太多资源的情况。比如说，避免多个表join，将复杂 SQL 分解为多个简单的 SQL

## 事务的特性有哪些（ACID）？
1. **原子性(Atomicity)**：事务是最小的执行单元，不允许分割。原子性保证了动作要么全完成，要么全不完成。
2. **一致性(Consistency)**：事务执行成功后，数据库从一个正确的状态变化到另一个正确的状态。
3. **隔离性(Isolation)**：并发访问数据库时，一个事务不被其他事务所干扰，各并发事务之间数据独立。
4. **持久性(Durability)**：一个事务提交之后，对数据库的改变是持久的。


## 并发事务带来哪些问题?
- **赃读(Dirty read)：事务1读取了事务2修改但还未提交的数据**。当一个事务正在访问数据并且对数据进行了修改，而这种修改还没有提交到数据库中，这时另外一个事务也访问了这个数据，然后使用了这个数据。因为这个数据是还没有提交的数据，那么另外一个事务读到的这个数据是“脏数据”，依据“脏数据”所做的操作可能是不正确的。
- **丢失修改(Lost to modify)：两个事务同时对数据需要根据原始值操作**。指在一个事务读取一个数据时，另外一个事务也访问了该数据，那么在第一个事务中修改了这个数据后，第二个事务也修改了这个数据。这样第一个事务内的修改结果就被丢失，因此称为丢失修改。 例如：事务1读取某表中的数据A=20，事务2也读取A=20，事务1修改A=A-1，事务2也修改A=A-1，最终结果A=19，事务1的修改被丢失。
- **不可重复度(Unrepeatableread)：事务1查询某值，再次查询某值时可能已经被事务2所修改**。指在一个事务内多次读同一数据。在这个事务还没有结束时，另一个事务也访问该数据。那么，在第一个事务中的两次读数据之间，由于第二个事务的修改导致第一个事务两次读取的数据可能不太一样。这就发生了在一个事务内两次读到的数据是不一样的情况，因此称为不可重复读。
- **幻读(Phantom read)：事务1读取几行数据，然后事务2插入或删除几行数据，导致事务1在接下来的查询中发现了原本不存在的数据或者发现少了原本存在的数据**。它发生在一个事务（T1）读取了几行数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录，就好像发生了幻觉一样，所以称为幻读。

## 数据库的隔离级别?

- 读取未提交（RU）：最低的隔离级别，允许读取尚未提交的数据，<font color="red">可能导致赃读、幻读、不可重复读。</font>
- 读取已提交（RC）：允许读取已经提交的事务，<font color="red">可以阻止赃读，可能导致幻读和不可重复读</font>
- 可重复读（RR）：对同一字段多次的读取结果都是一致的，除非数据是当前事务本身修改的。<font color="red">可以阻止赃读、不可重复读，可能导致幻读</font>
- 串行（Serializable）：所有事务依次逐个执行。<font color="red">最高的隔离级别，完全服从ACID的隔离界别。</font>


|     隔离级别     | 脏读 | 不可重复读 | 幻读 |
| :--------------: | :--: | :--------: | :----: |
| RU |  √   |     √      |   √    |
|  RC  |  ×   |     √      |   √    |
| RR  |  ×   |     ×      |   √    |
|   SERIALIZABLE   |  ×   |     ×      |   ×    |


## Mysql有哪些类型的索引？

- 普通索引：一个索引只包含一个列，一个表可以有多个单列索引。
- 唯一索引：索引列的值必须唯一，但允许有空值
- 复合索引：多列值组成一个索引，专门用于组合搜索，其效率大于索引合并
- 聚簇索引：也称为主键索引，是一种数据存储方式。B+Tree结构，非叶子节点包含健值和指针，叶子节点包含索引列和行数据。一张表只能有一个聚簇索引。
- 非聚簇索引：不是聚簇索引，就是非聚簇索引。叶子节点只是存索引列和主键id。如果sql还要返回除了索引列的其他字段信息，需要回表，第一次索引一般是顺序IO，回表的操作属于随机IO。回表的次数越多，性能越差。此时我们推荐覆盖索引

## 什么是覆盖索引和回表？

- 覆盖索引：指的是在一次查询中，一个索引包含所有需要查询的字段的值，可能是返回值或where条件(select name from user where name="test")，这时候不需要进行回表操作了。
- 回表：指查询时一些字段值拿不到，需要到主键索引B+树再查一次。

## 索引优化方向

- slow_query_log 日志中收集到的慢 SQL ，结合 explain 分析是否命中索引。
- 减少索引扫描行数，有针对性的优化慢 SQL。
- 建立合理的复合索引，查询时候尽量减少回表。
- 还可以使用虚拟列和联合索引来提升复杂查询的执行效率。

## 官方为什么建议采用自增id作为主键？

因为自增id连续的，每次插入的都是最大值，会放在后面，减少了数据页分裂，有效减少数据的移动。

## 索引为什么采用B+树，而不用B-树，红黑树？

总体原因：提升查询速度，首先要减少磁盘IO次数，也就是要降低树的高度。

- 平衡二叉树、红黑树，都属于二叉树。时间复杂度为O(n)，当表的数据量上千万时，树的深度很深，mysql读取时消耗大量 IO。另外，InnoDB引擎采用页为单位读取，每个节点一页，但是二叉树每个节点储存一个关键词，导致空间浪费。
- B-树，非叶子节点存储数据，占用较多空间，导致每个节点的指针少很多，无形增加了树的深度。
- B+树数据都存储在叶子节点，非叶子节点只存储健值+指针，索引树更加扁平，三层深度可以支持千万级表存储。同时叶子节点之间通过链表关联，范围查找更快。B+树所有的数据在叶子节点，一般来说都会进行一个优化，就是将所有的叶子节点用指针串起来。这样遍历叶子节点就能获得全部数据，这样就能进行区间访问啦。在数据库中基于范围的查询是非常频繁的，而 B- 树不支持这样的遍历操作。

## mysql 主从同步具体过程？

1. master主库，有数据更新，将此次更新的事件类型写入到主库的binlog文件中
2. 主库会创建`log dump 线程`通知slave有数据更新
3. slave，向master节点的 `log dump线程` 请求一份指定binlog文件位置的副本，并将请求回来的binlog存到本地的 `Relay log` 中继日志中
4. slave 再开启一个`SQL 线程`读取Relay log事件，并在本地执行redo操作。将发生在主库的事件在本地重新执行一遍，从而保证主从数据同步.

## 什么是主从延迟？

主库执行完SQL，到写入Bin Log，再到通知从库，从库拉取到对应的日志并保存到 `Relay Log` 并执行成功有时差。

## 主从延迟的排查方法？

通过 `show slave status` 命令输出的 `Seconds_Behind_Master` 参数的值来判断
- 0：表示无延迟。
- 正数：越高表示延迟越严重。

## 主从延迟解决方案?

- 看业务的接受程度。如果不能接受延迟，那么建议强制走主库查询
- 可以考虑引入缓存，更新主库后同步写入缓存，保证缓存的及时性
- 提升从库的机器配置，提高从库binlog的同步效率
- 缩短主、从库的网络距离，减少binlog的网络传输时间
- 一主多从，每个从库都启一个线程从主库同步 binlog，导致主库压力过大，可以采用canal 增量订阅&消费组件，缓解主库压力。
- 减少大事务的执行，因为数据库必须要等到事务完成之后才会写入binlog，尽量控制数量，分批执行。
- 5.6版本之前，从库是单线程复制，当遇到执行慢的sql时，就会阻塞后面的同步。5.7 版本后支持多线程复制，可以在从服务上设置slave_parallel_workers为一个大于0的数，然后把slave_parallel_type参数设置为LOGICAL_CLOCK
- 为从库增加浮动IP，并通过脚本检测从库的延迟，延迟大于指定阈值时，将浮动IP切换至Master库，追平后再切换回从库。

## 数据量太大解决方案？

[海量数据业务有哪些优化手段](https://mp.weixin.qq.com/s/plI3wkLqtremcIkjz8hPaQ)

- 缓存加速
- 读写分离
- 垂直拆分
- 分库分表
- 冷热数据分离
- ES助力复杂搜索
- NoSQL
- NewSQL

## 分表分库带来的问题?怎么解决？

分表后，查询需要根据 `sharding_key` 路由规则进行查询。分库后，查询共同的数据比较麻烦。

- 分买家库和卖家库，将买家库做为写库，保存完整的数据关系。同时将数据异构同步一份到卖家库，卖家库可以只存储seller_id，order_id，buyer_id 等几个简单关系字段即可，以seller_id作为分表键
- 多线程扫描，分段查找，然后再聚合结果
- 另外也可以存到ES中，支持多维度复杂搜索
















