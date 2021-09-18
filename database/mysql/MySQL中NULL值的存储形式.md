
## 1. NULL值是怎么在记录中储存的

在MySQL中每一条记录都有他固定的格式，下面在 InnoDB 存储引擎的 Compact 行格式查看 NULL 值的存储形式。Compact 行格式中一条记录有如下构成方式：

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/4d25699a1c679b91a69eee6313.png)

我们新建一个称之为 record_format_demo 的表：
```sql
CREATE TABLE record_format_demo (
     c1 VARCHAR(10),
     c2 VARCHAR(10) NOT NULL,
     c3 CHAR(10),
     c4 VARCHAR(10)
 ) CHARSET=ascii ROW_FORMAT=COMPACT;
 ```

因为我们的重点是NULL值是如何存储在记录中的，所以重点唠叨一下行格式的NULL值列表部分，其他的部分可以到小册中查看。

**存储NULL值的过程如下：**
1、<font color="red">首先统计表中允许存储NULL的列有哪些</font>。
我们前边说过，主键列、被NOT NULL修饰的列都是不可以存储NULL值的，所以在统计的时候不会把这些列算进去。比方说表record_format_demo的3个列c1、c3、c4都是允许存储NULL值的，而c2列是被NOT NULL修饰，不允许存储NULL值。

2、 <font color="red">如果表中没有允许存储NULL的列，则NULL值列表也不存在了，否则将每个允许存储NULL的列对应一个二进制位，二进制位按照列的顺序逆序排列</font>，二进制位表示的意义如下：
- 二进制位的值为1时，代表该列的值为NULL。
- 二进制位的值为0时，代表该列的值不为NULL。

因为表record_format_demo有3个值允许为NULL的列，所以这3个列和二进制位的对应关系就是这样：

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/8cbf83ac2a3953c582b6e120ebd.png)

再一次强调，二进制位按照列的顺序逆序排列，所以第一个列c1和最后一个二进制位对应。

3、<font color="red">设计InnoDB的大叔规定NULL值列表必须用整数个字节的位表示，如果使用的二进制位个数不是整数个字节，则在字节的高位补0</font>。

表record_format_demo只有3个值允许为NULL的列，对应3个二进制位，不足一个字节，所以在字节的高位补0，效果就是这样：

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/7ba7bc95aa2625672ebc54df3a.png)

以此类推，如果一个表中有9个允许为NULL，那这个记录的NULL值列表部分就需要2个字节来表示了。

假设我们现在向record_format_demo表中插入一条记录：

```sql
INSERT INTO record_format_demo(c1, c2, c3, c4) VALUES('eeee', 'fff', NULL, NULL);
```

这条记录的c1、c3、c4这3个列中c3和c4的值都为NULL，所以这3个列对应的二进制位的情况就是：

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/87549bf07d6c49966ba787a0.png)

所以这记录的NULL值列表用十六进制表示就是：0x06。

## 2. 键值为NULL的记录是怎么在B+树中存放的

对于InnoDB存储引擎来说，记录都是存储在页面中的（一个页面默认是16KB大小），这些页面可以作为B+树的节点而组成一个索引，类似这种样子（只是用下边的图举个B+树的例子而已，跟我们上边列举的表没关系）：

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/81cd3a32f4c84c0d28607a42.png)

聚簇索引和二级索引都对应着像上图一样的B+树（也就是说有多少个索引就有多少棵对应的B+树）。

**说明：**
- 对于<font color="red">聚簇索引，B+树每一层节点、页面中的记录是按照主键值进行排序</font>的；而对于<font color="red">二级索引，是按照给定的索引列的值进行排序</font>的。
- 对于<font color="red">聚簇索引，B+树叶子节点存储用户记录字段的完整数据（包括隐藏列）</font>；对于<font color="red">二级索引，B+树叶子节点对应的页面中存储的只是索引列的值 + 主键值</font>。

按规定，一条记录的主键值不允许存储NULL值，所以下边语句中的WHERE子句结果肯定为FALSE：

```
SELECT * FROM tbl_name WHERE primary_key IS NULL;
```

像这样的语句优化器自己就能判定出WHERE子句必定为NULL，所以压根儿不会去执行它，因为where子句根本不成立。

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/7470ed6564d37bddc3eb4e19f7.png)

对于二级索引来说，索引列的值可能为NULL。那对于索引列值为NULL的二级索引记录来说，它们被放在B+树的哪里呢？答案是：<font color="red">NULL值放在B+树的最左边</font>。比方说我们有如下查询语句：

```sql
SELECT * FROM s1 WHERE key1 IS NULL;
```

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/f422a754de74908a5ffee86866a5.png)

从图中可以看出，对于s1表的二级索引idx_key1来说，<font color="red">值为NULL的二级索引记录都被放在了B+树的最左边，SQL中的NULL值认为是列中最小的值</font>。这是因为设计InnoDB的大叔有这样的规定：

> We define the SQL null to be the smallest possible value of a field.

在通过二级索引idx_key1对应的B+树快速定位到叶子节点中符合条件的最左边的那条记录后，也就是本例中id值为521的那条记录之后，就可以顺着每条记录都有的next_record属性沿着由记录组成的单向链表去获取记录了，直到某条记录的key1列不为NULL。

## 3. 使不使用索引依据是什么

**优化器会分析使用索引与全盘扫表的成本对比。**
1. 读取二级索引记录的成本
2. 将二级索引记录执行回表操作，也就是到聚簇索引中找到完整的用户记录的操作所付出的成本

显然<font color="red">要扫描的二级索引记录条数越多，那么需要执行的回表操作的次数也就越多，达到了某个比例时，使用二级索引执行查询的成本也就超过了全表扫描的成本</font>（举一个极端的例子，比方说要扫描的全部的二级索引记录，那就要对每条记录执行一遍回表操作，自然不如直接扫描聚簇索引来的快）。

MySQL优化器在真正执行查询之前，对于每个可能使用到的索引来说，都会预先计算一下需要扫描的二级索引记录的数量。
```sql
SELECT * FROM s1 WHERE key1 IS NULL;
```
优化器会分析出此查询只需要查找key1值为NULL的记录，然后访问一下二级索引idx_key1，看一下值为NULL的记录有多少。这种在查询真正执行前优化器就率先访问索引来计算需要扫描的索引记录数量的方式称之为<font color="red">index dive</font>。

对于in查询，如下：
```sql
SELECT * FROM s1 WHERE key1 IN ('a', 'b', 'c', ... , 'zzzzzzz');
```

这样的话需要统计的key1值所在的区间就太多了，这样就不能采用index dive的方式去真正的访问二级索引idx_key1，而是需要采用之前在背地里产生的一些统计数据去估算匹配的二级索引记录有多少条（很显然根据统计数据去估算记录条数比index dive的方式精确性差了很多）。


<font color="red">反正不论采用index dive还是依据统计数据估算，最终要得到一个需要扫描的二级索引记录条数，如果这个条数占整个记录条数的比例特别大，那么就趋向于使用全表扫描执行查询，否则趋向于使用这个索引执行查询</font>。

所以WHERE子句中出现IS NULL、IS NOT NULL、!=这些条件仍然可以使用索引，本质上都是优化器去计算一下对应的二级索引数量占所有记录数量的比值而已。


参考文章：
- [MySQL中IS NULL、IS NOT NULL、!=不能用索引？胡扯！](https://juejin.im/post/6844903921450745863)
- [MySQL索引实现原理分析](https://blog.csdn.net/u013308490/article/details/83001060)