## 1. 概念

间隙锁<font color="red">只在读重复读(RR)隔离级别下生效。间隙锁是解决InnoBD在可重读(RR)隔离级别下的幻读问题</font>。幻读的问题存在是因为新增或者更新操作，这时如果进行范围查询的时候（加锁查询），会出现不一致的问题，这时使用不同的行锁已经没有办法满足要求，需要对一定范围内的数据进行加锁，间隙锁就是解决这类问题的。<font color="red">在可重复读隔离级别下，数据库是通过行锁和间隙锁共同组成的（next-key lock），来实现的加锁规则</font>。

## 2. InnoDB存储引擎的锁分类

- **表级锁**：对整张表加锁，实现简单，加锁快，不会死锁。但是并发低。
- **行级锁**：对操作行加锁，并发高。但是加锁慢，开销大，会死锁。
	- **Record lock**：行锁。单个行索引记录上的锁，锁住的是索引，而非记录本身。
	- **Gap lock**：间隙锁。锁定一个范围，不包括记录本身。
	- **Next-key lock**：record+gap 锁定一个范围，包含记录本身。解决RR隔离级别的幻读问题。

## 3. InnoDB的加锁特性

1. RR的当前读(current read)通过间隙锁实现。快照读(snapshot read)通过MVCC实现。
2. 加锁的基本单位是（Next-key lock），左开右闭原则。
3. 索引上的查询范围含有等值查询（>=、<= 、 =）时，如果是唯一索引加锁且存在记录值，next-key lock退化为行锁(record lock)。
4. 索引上的查询范围含有等值查询（>=、<= 、 =）时，向左、右遍历；且是等值判断且右边值不满足等值条件时，next-key lock退化为间隙锁(gap lock)。
5. 查找过程中访问的对象才会加锁。
6. 唯一索引上的范围查询会访问到不满足条件的第一个值为止。
7. 使用普通索引检索时，不管是何种查询，只要加锁，都会产生间隙锁。
8. 同时使用唯一索引和普通索引时，由于数据行是优先根据普通索引排序，再根据唯一索引排序，所以也会产生间隙锁。


## 4. 案例分析

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(0,0,0),(5,5,5),
(10,10,10),(15,15,15),(20,20,20),(25,25,25);
```

| id(主键)	| c（普通索引）|	d（无索引）|
| :-: | :-: | :-: |
| 5 | 5	| 5 |
| 10 | 10 | 10 |
| 15 | 15 | 15 |
| 20 | 20 | 20 |
| 25 | 25 | 25 |


### 4.1 等值查询唯一索引存在记录值

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br /> select id from t where id = 5 lock in share mode; |  | |
| | begin;<br /> insert into t values(4,4,4); <br /> <font color="green">query ok</font><br /> insert into t values(6,6,6); <br /> <font color="green">query ok</font> | |
| | | begin;<br /> update t set d = d + 1 where id = 5;<br /> <font color="red">block</font> |
根据3.3原则，加锁单位是next-key lock 退化为行锁，所以Session A的范围是[5, 5]。  
所以Session B的id = 4与id = 5均query ok，Session C的id = 5 block。

### 4.2 等值查询唯一索引不存在记录值

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br /> update t set d = d + 1 where id = 7; |  | |
| | begin;<br /> insert into t values(8,8,8); <br /> <font color="red">block</font> | |
| | | begin;<br /> update t set d = d + 1 where id = 10;<br /> <font color="green">query ok</font> | 

根据3.2原则，加锁单位时next-key lock, 所以Session A的范围是(5, 10]。  
根据3.4原则，Session A等值查询(id=7)，而id = 10 不满足条件。next-key lock退化为(5, 10)。  
所以Session B的id = 8 block，而Session C的id=10 query ok。

### 4.3 等值查询普通索引

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br /> select id from t where c=5 lock in share mode; |  | |
| | begin;<br />update t set d=d+1 where id=5; <br /><font color="green">query ok</font><br /> update t set d=d+1 where c=5; <br /><font color="red">block</font>  | |
| | | begin;<br /> insert into t values (7,7,7);<br />  <font color="red">block</font> | 

根据3.2原则，加锁单位时next-key lock，所以Session A的范围是(0, 5]，而c = 5满足查询条件，所以向右遍历第一个不满足条件值为10，所以(5, 10]也要加锁。  
根据3.4原则，Session A的c = 5 ，而c=10 不满足条件，所以退化为间隙锁(5, 10)。整体加锁范围为(0, 5]、(5, 10)。  
根据3.5原则，返回到的对象才会加锁，Session A访问的是c，而Session用到的是id，所以第一个可以query ok，第二个在这个整体间隙锁范围内，所以block。但是Session C插入的是c = 7，在这个整体间隙锁范围内，所以block。

### 4.4 唯一索引范围加锁

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br /> select id from t where id>=10 and id <11 for update; |  | |
| | begin;<br />insert into t values (8,8,8); <br /><font color="green">query ok</font><br />insert into t values (13,13,13); <br /><font color="red">block</font>  | |
| | | begin;<br /> update t set d=d+1 where id=15;<br /> <font color="red">block</font> | 

根据3.2、3.4原则，next-key block为(5, 10]、(10, 15]。  
根据3.3原则，next-key block退化为行锁[10]、间隙锁(10, 15]。因为右边不是等值判断，所以不能退化为(10, 15)。所以最终锁范围为行锁[10]、间隙锁(10, 15]，即[10, 15]。
所以Session B第一个query ok，第二个block。Session C block。

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br />select id from t where c>10 and c<=15 for update;	 |  | |
| | begin;<br />update t set d=d+1 where id=20;<br /><font color="red">block</font>  | |
| | | begin;<br />insert into t values (16,16,16);<br /> <font color="red">block</font> | 

session A 是一个范围查询，按照原则 3.2 应该是索引 id 上只加 (10,15]这个 next-key lock，并且因为 id 是唯一键，所以循环判断到 id=15 这一行就应该停止了。但是实现上，InnoDB 会往后扫描到第一个不满足条件的行为止，也就是 id=20。而且由于这是个范围扫描，因此索引 id 上的 (15,20]这个 next-key lock 也会被锁上。

### 4.5 普通索引范围加锁

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br /> select id from t where id>=10 and id <11 for update; |  | |
| | begin;<br />insert into t values (8,8,8); <br /><font color="red">block</font>  | |
| | | begin;<br /> update t set d=d+1 where id=15;<br /> <font color="red">block</font> | 

在第一次用 c=10 定位记录的时候，索引 c 上加了 (5,10]这个 next-key lock 后，由于索引 c 是非唯一索引，没有优化规则，也就是说不会蜕变为行锁，因此最终 sesion A 加的锁是，索引 c 上的 (5,10] 和 (10,15] 这两个 next-key lock。

### 4.6 普通索引的相等值

**插入一条想等值数据：**

```sql
insert into t values(30, 10 ,30);
```

**表值如下：**

| id(主键)	| c（普通索引）|	d（无索引）|
| :-: | :-: | :-: |
| 5 | 5	| 5 |
| 10 | 10 | 10 |
| 30 | 10 | 30 |
| 15 | 15 | 15 |
| 20 | 20 | 20 |
| 25 | 25 | 25 |

**执行下面操作：**

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br /> delete from t where c = 10 |  | |
| | begin;<br />insert into values(12,12,12); <br /><font color="red">block</font>  | |
| | | begin;<br /> update t set d = d+ 1 where c = 15;<br /> <font color="green">query ok</font> | 

根据上述规则，锁区间为(5, 15)。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201020191201905.png)

### 4.7 普通索引的相等值中的limit

| Session A | Session B | Session C |
| -------- | -------- | -------- |
| begin;<br /> delete from t where c = 10 |  | |
| | begin;<br />insert into values(12,12,12); <br /><font color="green">query ok</font>  | |
| | | begin;<br /> update t set d = d+ 1 where c = 15;<br /> <font color="green">query ok</font> | 

去上面的区别是增加了limit 2，在对间隙锁(5, 10]加锁后发现已经对两条数据加锁了，所以加锁范围就是(5, 10]，而非上面的(5, 15)。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/20201020191320824.png)