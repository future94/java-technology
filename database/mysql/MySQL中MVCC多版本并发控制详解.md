
前提：
- 对事务的ACID了解
- 对事务的隔离级别了解

如果不了解，请先看下[MySQL常见问题](MySQL常见问题.md)文章了解。

## 1. 多版本并发控制（MVCC）

MVCC全程Multi-Version Concurrency Control，<font color="red">用来处理读写冲突的手段，目的在于提高数据库高并发场景下的吞吐性能。</font>

如此一来不同的事务在并发过程中，SELECT 操作可以不加锁而是通过 MVCC 机制读取指定的版本历史记录，并通过一些手段保证保证读取的记录值符合事务所处的隔离级别，从而解决并发场景下的读写冲突。

### 1.1 多版本读写例子

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201012153623625.png)

上图执行的事务，rs1与rs2的值在不同事务隔离界别下的值都不一样。

1. 如果事务隔离级别是RU，rs1与rs2的值均为20。
2. 如果事务隔离级别是RC，rs1为10，rs2为20。
3. 如果事务隔离级别是RR或Serializable，rs1与rs2的值均为10。

### 1.2 实现隔离机制的方法主要有两种
1. 读写锁。
2. 一致性快照读，即MVCC。

本质上，隔离级别是一种在并发性能和并发产生的副作用间的妥协，通常数据库均倾向于采用 Weak Isolation。

## 2. InnoDB中的MVCC

### 说明
1. mysql中InnoDB支持MVCC。
2. 应对高并发事务，mvcc比单纯的行锁更有效，开销更小。
3. MVCC在RC与RR隔离级别下生效。
4. MVCC实现可以基于乐观锁或者悲观锁均可。
5. MVCC通过隐藏两个字段（没有主键时为3个）

## 3 InnoDB MVCC原理

InnoDB 中 MVCC 的实现方式为：每一行记录都有两个隐藏列DATA_TRX_ID、DATA_ROLL_PTR，如果没有主键，还会多一个隐藏主键DB_ROW_ID。

### 3.1 隐藏列说明

#### 3.1.1 DATA_TRX_ID

记录最近更新这行数据记录的事务ID，大小为6个字节。

#### 3.1.2 DATA_ROLL_PTR

表示指向该行回滚段的指针，大小为7个字节。InnoDB 便是通过这个指针找到之前版本的数据。该行记录上所有旧版本，在 undo log 中都通过链表的形式组织。

#### 3.1.3 DB_ROW_ID

行标识，大小为6个字节，如果表没有主键，InnoDB 会自动生成一个隐藏主键，即这个列。另外，每条记录的头信息（record header）里都有一个专门的 bit（deleted_flag）来表示当前记录是否已经被删除。

### 3.2 MVCC 例子

接着上面的例子（1.1 多版本读写例子）。假设该行之前x值为10，插入改行的事务ID为100，事务A的ID为200，该行的隐藏主键为1。

事务A的<font color="red">UPDATE操作</font>为：

1. 对 DB_ROW_ID = 1 的这行记录加排他锁。
2. 把该行原本的值拷贝到 undo log 中，DB_TRX_ID 和 DB_ROLL_PTR 都不动。
3. 修改该行的值这时产生一个新版本，更新 DATA_TRX_ID 为修改记录的事务 ID，将 DATA_ROLL_PTR 指向刚刚拷贝到 undo log 链中的旧版本记录，这样就能通过 DB_ROLL_PTR 找到这条记录的历史版本。如果对同一行记录执行连续的 UPDATE，Undo Log 会组成一个链表，遍历这个链表可以看到这条记录的变迁。
4. 记录 redo log，包括 undo log 中的修改。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201012161623612.png)

事务<font color="red">INSERT操作</font>是会产生一条新的记录，它的 DATA_TRX_ID 为当前插入记录的事务 ID，DATA_ROLL_PTR为null。

事务<font color="red">DELETE操作</font>其实是软删除，将标识位bit（deleted_flag）修改。当事务真正commit成功时才会删除。DATA_TRX_ID 则记录下删除该记录的事务 ID。

### 3.3 Read View 与 Snapshot 实现一致性读

#### 3.3.1 引入背景

在 RU 隔离级别下，直接读取版本的最新记录就 OK，对于 SERIALIZABLE 隔离级别，则是通过加锁互斥来访问数据，因此不需要 MVCC 的帮助。因此 MVCC 运行在 RC 和 RR 这两个隔离级别下，当 InnoDB 隔离级别设置为二者其一时，在 SELECT 数据时就会用到版本链。<font color="red">解决的核心问题是版本链中哪些版本对当前事务可见，为了解决这个问题设计了 Read View （刻度视图）的概念</font>。

#### 3.3.2 什么是 Read View

**ReadView 中是当前活跃的事务 ID 列表，称之为 m_ids。最小值为up_limit_id，最大值为 low_limit_id。<font color="red">这个low_limit_id=未开启的事务id=当前最大事务id+1</font>。事务ID是在事务开启时 InnoDB 分配的，其值的大小决定了事务的开启先后顺序，因此我们可以通过 ID 的大小关系来决定版本记录的可见性。**

注意：<font color="red">low_limit_id是随着新事务生成而增加的。如：事务A生成的m_ids为[200]，此时事务B的事务ID为300，则事务A的m_ids为[200]，但是low_limit_id为301。</font>下面的误区（6.5 并非所有情况都能套用 MVCC 的可见性判断误区）就是例子。

#### 3.3.3 RR 级别下的 ReadView 生成

事务<font color="red">在begin/start transaction之后的第一条select读操作后, 会创建一个快照(read view), 将当前系统中活跃的其他事务记录记录起来</font>。

在RR隔离界别下，每个事务的第一条Select时会将当前系统中的所有活跃事务拷贝到一个列表中生成ReadView，后续的所有Select操作都是复用这个ReadView（CUD操作与快照的建立没有关系）。

下图说明即使事务B提交了事务修改，但是事务A依然无法读取到事务B的修改（x = 20）。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201012163817314.png)

**但是需要注意**，虽然事务A在事务B之前开启，但是如果第一条查询在事务B提交之后进行，那么事务A可以读取到事务B修改的值。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201012164410054.png)

因为<font color="red">事务ID在第一条操作语句的时候才生成，并不是begin语句。</font>

#### 3.3.4 RC 级别下的 ReadView 生成

事务中<font color="red">每条select语句都会创建一个快照</font>。

在 RC 隔离级别下，每个 SELECT 语句开始时，都会重新将当前系统中的所有的活跃事务拷贝到一个列表生成 ReadView。二者的区别就在于生成 ReadView 的时间点不同，一个是事务之后第一个 SELECT 语句开始、一个是事务中每条 SELECT 语句开始。


### 3.4  可见性比较算法

由上面我们知道（3.3.2 什么是 Read View），我们知道ReadView 中是当前活跃的事务 ID 列表，而事务ID大小决定了事务的开启先后顺序。

#### 3.4.1 可见性判断流程

1. 如果被访问数据版本的DATA_TRX_ID小于ReadView的最小值up_limit_id，则说明该版本的事务已经在 ReadView生成前就已经提交过了，所以是可以被当前事务访问的。
2. 如果被访问数据版本的DATA_TRX_ID大于ReadView的最大值low_limit_id，则说明该版本的事务是在生成ReadView之后才生成的，所以不能被访问，需要根据 Undo Log 链找到前版本的 DATA_TRX_ID 重新判断可见性。
3. 如果被访问数据版本的DATA_TRX_ID在 m_ids 列表之前（包含最大值最小值），则判断 DATA_TRX_ID 是否在 m_ids 列表中，如果在，说明创建 ReadView时该事务依然处于活跃状态，则该版本数据不能被访问，需要根据 Undo Log 链找到前版本的 DATA_TRX_ID 重新判断可见性；如果不再，说明创建 ReadView时该事务已经被提交了，所以可以访问。
4. 此时经过一系列判断我们已经得到了这条记录相对 ReadView 来说的可见性结果，还需继续判断下bit（delete_flag）标志位是否被删除，如果为true则表示被删除，如果为false则可以访问。

#### 3.4.2  RR 级别下的 MVCC 判断流程演示

上面例子（1.1 多版本读写例子）我们知道，在RR情况下事务B的rs1、rs2值均为10。

**重点信息回顾：**
- 不要忘了看4.1的图，即事务A、B的查询时机
- RR 下 ReadView 是在第一条select时生成，后面不会变化
- 之前数据的事务ID为100
- 事务A的ID为200
- 事务B的ID为300

此时事务B在select rs1时生成，m_ids 值为 [200, 300]，后面不会发生变化。由于事务A将数据DATA_TRX_ID更新为200，DATA_TRX_ID 为200在事务B的 m_ids 列表中，所以不能访问，根据 Undo Log 找到上一条X值为10、DATA_TRX_ID 为100，100小于 m_ids 最小值200，所以可以访问。则 rs1 值为 10，当事务B在select rs2时 m_ids 列表不会发生变化，所以 rs2 同值为 20。

#### 3.4.2  RC 级别下的 MVCC 判断流程演示

上面例子（1.1 多版本读写例子）我们知道，在RC情况下事务B的 rs1 值为10，rs2 值为20。

**重点信息回顾：**
- 不要忘了看4.1的图，即事务A、B的查询时机
- RC 下 ReadView 是在每条select时生成
- 之前数据的事务ID为100
- 事务A的ID为200
- 事务B的ID为300

此时事务B在select rs1时生成，m_ids 值为 [200, 300]，由于事务A将数据DATA_TRX_ID更新为200，DATA_TRX_ID 为200在事务B的 m_ids 列表中，所以不能访问，根据 Undo Log 找到上一条X值为10、DATA_TRX_ID 为100，100小于 m_ids 最小值200，所以可以访问。则 rs1 值为 10，当事务B在select rs2时事务A已经提交，则重新生成的 m_ids 列表 [300]，数据DATA_TRX_ID为200小于 m_ids 列表最小值300，所以可以访问，rs2 值为20。

### 3.5 错误：并非所有情况都能套用 MVCC 的可见性判断

#### 3.5.1 错误说法

其实并非所有的情况都能套用 MVCC 读的判断流程，特别是针对在事务进行过程中，另一个事务已经提交修改的情况下，这时不论是 RC 还是 RR，直接套用 MVCC 判断都会有问题。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201012175601167.png)

事务 A 的 trx_id = 200，事务 B 的 trx_id = 300，且事务 B 修改了数据之后在事务 A 之前提交，此时 RC 下事务 A 读到的数据为事务 B 修改后的值，这是很显然的。

**RC 隔离级别：**

下面我们套用下 MVCC 的判断流程，考虑到事务 A 第二次 SELECT 时，m_ids 应该为 [200]，此时该行数据最新的版本 DATA_TRX_ID = 300 比 200 大，照理应该不能被访问，但实际上事务 A 选取了这条记录返回。

**RR 隔离级别：**

下面我们套用下 MVCC 的判断流程，事务 B 的 trx_id = 300 比事务 A 200 小，且事务 B 先于事务 A 提交，按照 MVCC 的判断流程，事务 A 生成的 ReadView 为 [200]，最新版本的行记录 DATA_TRX_ID = 300 比 200 大，照理不能访问到，但是事务 A 实际上读到了事务 B 已经提交的修改。

#### 3.5.2 正确解答

注意：<font color="red">上述说法是错误的：low_limit_id是随着新事务生成而增加的。如事：务A生成的m_ids为[200]，此时事务B的事务ID为300，则事务A的m_ids为[200]，但是 low_limit_id 为301。</font>

**mysql源码地址：**
- [mysql5.6源码1](https://github.com/facebook/mysql-5.6/blob/42a5444d52f264682c7805bf8117dd884095c476/storage/innobase/include/trx0sys.h#L628)
- [mysql5.6源码2](https://github.com/facebook/mysql-5.6/blob/42a5444d52f264682c7805bf8117dd884095c476/storage/innobase/read/read0read.cc#L349)

注视已经明确说明，low_limit_id为至今未分配的最小的事物ID，即<font color="red">这个low_limit_id=未开启的事务id=当前最大事务id+1</font>
> The smallest number not yet assigned as a transaction id or transaction number

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201012190200048.png)

**RC 隔离级别：**

事务 A 第二次 SELECT 时，m_ids 应该为 [200]，此时该行数据最新的版本 DATA_TRX_ID = 300 比 low_limit_id = 301 小，并不再 m_ids 其中，所以可以访问。

**RR 隔离级别：**

事务 B 的 trx_id = 300 比事务 A 200 小，且事务 B 先于事务 A 提交，按照 MVCC 的判断流程，事务 A 生成的 ReadView 为 [200]，最新版本的行记录 DATA_TRX_ID = 300 比 low_limit_id = 301 小，并不再 m_ids 其中，所以可以访问。