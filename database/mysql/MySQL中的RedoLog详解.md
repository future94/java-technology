| | redo log | undo log | binlog
| -| - | - | - | - |
| 层面 | InnoBD层 | InnoBD层 | MySQL层 |
| 类型 | 物理日志 | 逻辑日志 | 逻辑日志
| 作用 | 1. 记录数据页物理修改<br />2. 数据恢复，保证持久性 | 1. 事务回滚<br />2. MVCC | 1. 主从复制同步<br />2. 数据恢复到某时间点 |

## 1. redo log概念

redo log（重做日志）

<font color="red">redo log是InnoDB存储引擎层的日志</font>，又称重做日志文件，用于记录事务操作的变化，记录的是数据修改之后的值，不管事务是否提交都会记录下来。在实例和介质失败（media failure）时，redo log文件就能派上用场，如数据库掉电，InnoDB存储引擎会使用redo log恢复到掉电前的时刻，以此来保证数据的完整性。

<font color="red">redo log大部分情况下是物理日志，记录的是数据页的物理变化，可以快速恢复数据页</font>。而逻辑redo log，不是记录页的实际修改，而是记录修改页的一类操作，比如新建数据页时，需要记录逻辑日志。关于逻辑Redo日志涉及更加底层的内容，这里我们只需要记住绝大数情况下，Redo是物理日志即可，DML（Data Manipulation Language：指数据的Delete、Insert、Update操作）对页的修改操作，均需要记录Redo.

<font color="red">redo log日志的大小是固定的，即记录满了以后就从头循环写</font>。

**redo log的两部分组成：**

1. <font color="red">重做日志缓冲（redo log buffer）</font>，容易丢失，在内存中。
2. <font color="red">重做日志文件（redo log file）</font>，持久的保存在文件中。

在概念上，innodb通过<font color="red">force log at commit</font>机制实现事务的持久性，即在事务提交的时候，必须先将该事务的所有事务日志写入到磁盘上的redo log file和undo log file中进行持久化。

为了确保每次日志都能写入到事务日志文件中，在每次将log buffer中的日志写入日志文件的过程中都会调用一次操作系统的fsync操作(即fsync()系统调用)。因为MariaDB/MySQL是工作在用户空间的，MariaDB/MySQL的log buffer处于用户空间的内存中。要写入到磁盘上的log file中(redo:ib_logfileN文件,undo:share tablespace或.ibd文件)，中间还要经过操作系统内核空间的 <font color="red">os buffer</font>（直接操走硬盘效率很低，所以mysql通过os buffer定期写入磁盘），调用fsync()的作用就是将OS buffer中的日志刷到磁盘上的log file中。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201013201938617.png)

**通过 <font color="red">innodb_flush_log_at_trx_commit</font> 变量自定义事务在 commit 时如何将 log buffer 中的日志刷到 log file 中**，注意：这个变量只是控制commit动作是否刷新log buffer到磁盘，取值0，1，2。
- 0：**事务提交到 log buffe时，不会将 log buffer 中日志立刻写入 os buffer 中，而是每秒写入 os buffer 并调用 fsync() 写入到 log file on dick**。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。
- 1：**事务提交到 log buffe时，都会将 log buffer 中的日志写入 os buffer 并调用 fsync() 刷到 log file on disk 中**。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。
- 2：**事务提交到 log buffe时，都会写入到 os buffer，然后每秒调用 fsync() 将 os buffer 中日志写入到 log file on disk**。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201013203200822.png)

**注意**：mysql的 `innodb_flush_log_at_timeout` 变量控制的是日志刷新的频率，与事务commit无关，不是控制 `innodb_flush_log_at_trx_commit` 值为0、2时的每秒写入频率。

**主从复制如果要保证事务的一致性**，就必须开启binlog的配置`sync_binlog=1`与innodb的og buffer配置`innodb_flush_log_at_trx_commit=1`，即每次事务提交都会写到磁盘中。

**写入磁盘性能测试**（innodb_flush_log_at_trx_commit分别为0、1、2）：

```sql
#创建测试表
drop table if exists test_flush_log;
create table test_flush_log(id int,name char(50))engine=innodb;

#创建插入指定行数的记录到测试表中的存储过程
drop procedure if exists proc;
delimiter $$
create procedure proc(i int)
begin
    declare s int default 1;
    declare c char(50) default repeat('a',50);
    while s<=i do
        start transaction;
        insert into test_flush_log values(null,c);
        commit;
        set s=s+1;
    end while;
end$$
delimiter ;
```

innodb_flush_log_at_trx_commit为1时：

```txt
mysql> call proc(100000);
Query OK, 0 rows affected (15.48 sec)
```

innodb_flush_log_at_trx_commit为2时：

```txt
mysql> set @@global.innodb_flush_log_at_trx_commit=2;    
mysql> truncate test_flush_log;
mysql> call proc(100000);
Query OK, 0 rows affected (3.41 sec)
```

innodb_flush_log_at_trx_commit为0时：

```txt
mysql> set @@global.innodb_flush_log_at_trx_commit=0;
mysql> truncate test_flush_log;
mysql> call proc(100000);
Query OK, 0 rows affected (2.10 sec)
```

总结：<font color="red">每次将 log buffer 刷新到 os buffer 中并不会消耗特别多的时间，而 os buffer 到 log file on disk 则效率十分底下</font>。虽然为1时耗时增加，但是保证了数据的一致性。更好的做法，值为1，把每次写入替换为1次写入。

### 2. 日志块（log block）

innodb存储引擎中，redo log以块为单位进行存储的，每个块占512字节，这称为redo log block。所以不管是log buffer中还是os buffer中以及redo log file on disk中，都是这样以512字节的块存储的。

redo log block由3部分组成：<font color="red">日志块头(12字节)、日志块尾(8字节)和日志主体(492字节)</font>。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201013221353905.png)

**日志块头**：

- log_block_hdr_no：(4字节)该日志块在redo log buffer中的位置ID。
- log_block_hdr_data_len：(2字节)该log block中已记录的log大小。写满该log block时为0x200，表示512字节。
- log_block_first_rec_group：(2字节)该log block中第一个log的开始偏移位置。
- lock_block_checkpoint_no：(4字节)写入检查点信息的位置。

**日志主体**：

- redo_log_type：占用1个字节，表示redo log的日志类型。
- space：表示表空间的ID，采用压缩的方式后，占用的空间可能小于4字节。
- page_no：表示页的偏移量，同样是压缩过的。
- redo_log_body表示每个重做日志的数据部分，恢复时会调用相应的函数进行解析。例如insert语句和delete语句写入redo log的内容是不一样的

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201013232134394.png)

对于log block块头的第三部分 log_block_first_rec_group ，因为有时候一个数据页产生的日志量超出了一个日志块，这是需要用多个日志块来记录该页的相关日志。例如，某一数据页产生了552字节的日志量，那么需要占用两个日志块，第一个日志块占用492字节，第二个日志块需要占用60个字节，那么对于第二个日志块来说，它的第一个log的开始位置就是73字节(60+12)，因为前面的用来存储上一个日志块超出的数据。

如果该部分的值和 log_block_hdr_data_len 相等，则说明该log block中没有新开始的日志块，即表示该日志块所用的主体信息都是用户储存上个日志块超出的数据。

**日志块尾**：只含有一个 log_block_trl_no，与日志块头的 log_block_hdr_no 相等。

### 3. log group 与 redo log file

log group表示的是redo log group，一个组内由多个大小完全相同的redo log file组成。组内redo log file的数量由变量 `innodb_log_files_group` 决定，默认值为2，即两个redo log file。

查看innodb的log配置

```shell
mysql > show global variables like "innodb_log%";
+-----------------------------+----------+
| Variable_name               | Value    |
+-----------------------------+----------+
| innodb_log_buffer_size      | 8388608  |
| innodb_log_compressed_pages | ON       |
| innodb_log_file_size        | 5242880 |
| innodb_log_files_in_group   | 2        |
| innodb_log_group_home_dir   | /usr/local/mysql/var       |
| innodb_log_write_ahead_size | 8192 |
+-----------------------------+----------+
```

查看现象说明：由上面配置，两个分组文件为ib_logfile0、ib_logfile1，即log group 的redo log file，大小为5242880（innodb_log_file_size的值，且redo log为规定大小，循环写入）

```shell
#> ll /usr/local/mysql/var
-rw-r----- 1 mysql mysql      334 2月  26 2020 ib_buffer_pool
-rw-r----- 1 mysql mysql 77594624 10月 13 22:25 ibdata1
-rw-r----- 1 mysql mysql  5242880 10月 13 22:25 ib_logfile0
-rw-r----- 1 mysql mysql  5242880 10月 13 22:25 ib_logfile1
```

**拓展**：ibdata1文件为`innodb_file_per_table`参数未开启时(innodb_file_per_table=OFF)，即<font color="red">共享表空间模式（某一个数据库的所有的表数据，索引文件全部放在一个文件中）</font>下的所有数据存储文件默认文件名为。`innodb_file_per_table`参数未开启时(innodb_file_per_table=ON)时，即<font color="red">独立表空间模式（每一个表都将会生成以独立的文件方式来进行存储，每一个表都有一个.frm表结构描述文件，还有一个.ibd文件。 其中这个ibd文件包括了单独一个表的数据内容以及索引内容）</font>下为frm与ibd文件。

```shell
-rw-r----- 1 mysql mysql  8876 10月  9 18:12 mt_advice.frm
-rw-r----- 1 mysql mysql 98304 10月 10 21:07 mt_advice.ibd
-rw-r----- 1 mysql mysql  9017 10月  8 22:37 mt_index_recommend.frm
-rw-r----- 1 mysql mysql 98304 10月  8 22:37 mt_index_recommend.ibd
```

在innodb将log buffer中的redo log block刷到这些log file中时，会以追加写入的方式循环轮训写入。即先在第一个log file（即ib_logfile0）的尾部追加写，直到满了之后向第二个log file（即ib_logfile1）写。当第二个log file满了会清空一部分第一个log file继续写入。

由于是将log buffer中的日志刷到log file，所以在log file中记录日志的方式也是log block的方式。

在每个组的第一个redo log file中，前2KB记录4个特定的部分，从2KB之后才开始记录log block。除了第一个redo log file中会记录，log group中的其他log file不会记录这2KB，但是却会腾出这2KB的空间。如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201013225419702.png)

redo log file的大小对innodb的性能影响非常大，设置的太大，恢复的时候就会时间较长，设置的太小，就会导致在写redo log的时候循环切换redo log file。

### 4. redo log 的格式

因为innodb存储引擎存储数据的单元是页(和SQL Server中一样)，所以redo log也是基于页的格式来记录的。默认情况下，<font color="red">innodb的页大小是16KB</font>(由 `innodb_page_size` 变量控制)，一个页内可以存放非常多的log block(每个512字节)，而log block中记录的又是数据页的变化。log block 在 1.2 日志块中已经介绍。

### 5. 日志刷盘规则

log buffer中未刷到磁盘的日志称为<font color="red">脏日志(dirty log)</font>。内存中(buffer pool)未刷到磁盘的数据称为<font color="red">脏数据(dirty data)</font>。由于数据和日志都以页的形式存在，所以<font color="red">脏页(dirty page)</font>表示脏数据和脏日志，即当内存数据页和磁盘数据页上的内容不一致时，这个内存页为赃页。

**刷日志的几种情况：**

1. 事务发出commit动作时，`innodb_flush_log_at_trx_commit` 参数如果为1会立刻刷日志。
2. `innodb_flush_log_at_timeout` 参数默认的刷日志频率。默认为1秒。
3. 当log buffer中内存使用超过一半时。
4. 当有checkpoint时，checkpoint在一定程度上代表了刷到磁盘时日志所处的LSN位置。

### 1.6 数据页刷盘的规则及checkpoint

checkpoint表示已经完整刷到磁盘上data page上的LSN。<font color="red">checkpoint目的是解决以下三个问题</font>：1、缩短数据库的恢复时间；2、缓冲池不够用时，将脏页刷新到磁盘；3、重做日志不可用时，刷新脏页。

由于刷脏页需要一定的时间来完成，所以记录检查点的位置是在每次刷盘结束之后才在redo log中标记的。

在<font color="red">innodb数据刷盘的规则只有一个：checkpoint</font>。但是触发checkpoint的情况却有几种。不管怎样，<font color="red">checkpoint触发后，会将buffer中脏数据页和脏日志页都刷到磁盘</font>。

**innodb存储引擎中checkpoint分为两种**：

- sharp checkpoint：在重用redo log文件（如：切换日志文件）的时候，将所有已记录到 redo log 中对应的赃数据刷到磁盘。
- fuzzy checkpoint：一次只刷新一小部分日志到磁盘。

**fuzzy checkpoint四种触发情况：**

1. <font color="red">Master Thread Checkpoint</font>：由master线程控制，每秒或每10秒刷入一定比例的脏页到磁盘。异步操作。
2. <font color="red">flush_lru_list Checkpoint</font>：因为InnoDB存储引擎需要保证LRU列表中需要有差不多100个空闲页可供使用。在InnoDB1.1.x版本之前，需要检查LRU列表中是否有足够的可用空间操作发生在用户查询线程中，显然这会阻塞用户的查询操作。**倘若没有100个可用空闲页，那么InnoDB存储引擎会将LRU列表尾端的页移除。如果这些页中有脏页，那么需要进行Checkpoint，而这些页是来自LRU列表的，因此称为FLUSH_LRU_LIST Checkpoint**。而从MySQL 5.6版本，也就是InnoDB1.2.x版本开始，这个检查被放在了一个单独的`Page Cleaner线程`中进行，该线程的目的是为了保证lru列表有可用的空闲页。并且用户可以通过参数`innodb_lru_scan_depth`控制LRU列表中可用页的数量，该值默认为1024。通过参数`innodb_page_cleaners`控制Page Cleaner线程的数量，该值默认为1。
3.  <font color="red">Async/Sync Flush Checkpoint</font>：指的是重做日志文件不可用的情况，这时需要强制将一些页刷新回磁盘，而此时脏页是从脏页列表中选取的。
4.  <font color="red">Dirty Page too much</font>：即脏页的数量太多，导致InnoDB存储引擎强制进行Checkpoint。其目的总的来说还是为了保证缓冲池中有足够可用的页。其可由参数`innodb_max_dirty_pages_pct`控制。默认值为75，表示当缓冲池中脏页的数量占据75%时，强制进行Checkpoint，刷新一部分的脏页到磁盘。

> MySQL停止时是否将脏数据和脏日志刷入磁盘，由变量innodb_fast_shutdown={ 0|1|2 }控制，默认值为1，即停止时只做一部分purge，忽略大多数flush操作(但至少会刷日志)，在下次启动的时候再flush剩余的内容，实现fast shutdown。

### 7. LSN逻辑序列号(log sequence number)

LSN称为日志的逻辑序列号(log sequence number)，在innodb存储引擎中，lsn占用8个字节。LSN的值会随着日志的写入而逐渐增大。

**作用：**

1. 数据页版本信息。
2. 写入的日志总量，通过LSN开始号码与结束号码可以计算出写入的日志量。
3. 可知道检查点（checkpoint）的位置。

LSN不仅存在于redo log中，还存在于数据页中，在每个数据页的头部，有一个`fil_page_lsn`记录了当前页最终的LSN值是多少。**通过数据页中的LSN值和redo log中的LSN值比较，如果页中的LSN值小于redo log中LSN值，则表示数据丢失了一部分**，这时候可以通过redo log的记录来恢复到redo log中记录的LSN值时的状态。

通过`show engine innodb status`查看 redo log 的LSN信息。

```txt
---
LOG
---
Log sequence number 93323119
Log flushed up to   93323119
Pages flushed up to 93323119
Last checkpoint at  93323110
0 pending log flushes, 0 pending chkp writes
396861 log i/o's done, 0.00 log i/o's/second
```

说明：

- log sequence number就是当前的redo log(in buffer)中的lsn；
- log flushed up to是刷到redo log file on disk中的lsn；
- pages flushed up to是已经刷到磁盘数据页上的LSN；
- last checkpoint at是上一次检查点所在位置的LSN。

### 8. redo log 相关变量

- innodb_flush_log_at_trx_commit={0|1|2} # 指定何时将事务日志刷到磁盘，默认为1。
- innodb_log_buffer_size：# log buffer的大小，默认8M
- innodb_log_file_size：#事务日志的大小，默认5M
- innodb_log_files_group =2：# 事务日志组中的事务日志文件个数，默认2个
- innodb_log_group_home_dir =./：# 事务日志组路径，当前目录表示数据目录
- innodb_mirrored_log_groups =1：# 指定事务日志组的镜像组个数，但镜像功能好像是强制关闭的，所以只有一个log group。在MySQL5.7中该变量已经移除。