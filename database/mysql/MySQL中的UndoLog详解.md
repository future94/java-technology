## undo log（回滚日志）

### 1. 概念
undo log和redo log记录物理日志不一样，它<font color="red">undo log是逻辑日志</font>。<font color="red">undo log有两个作用</font>：提供回滚和[多版本并发控制(MVCC)](MySQL中MVCC多版本并发控制详解.md)

由10改为20记录的undo log示意：

![image](https://raw.githubusercontent.com/future94/java-technology/master/database/mysql/images/a5bca37660f20b9ff6080ffcc.png)

### 2. undo log的存储方式

undo log是采用段(segment)的方式来记录的，每个undo操作在记录的时候占用一个undo log segment。`rollback segment`称为回滚段，每个回滚段中有1024个`undo log segment`。Mysql5.6之前只有一个回滚段，5.6以后默认128个，由参数`innodb_undo_logs`配置。

**undo log 默认存在共享表空间中，开启了innodb_file_per_table存放在每个表的.ibd文件中**。

### 3. undo log的delete / update机制
事务提交时，不会立刻删除undo log，后续mvcc机制还可能会用到，但是在事务提交的时候将该事务对应的undo log放入到删除列表中，后续通过purge线程删除。还会判断undo log分配的页是否可以重用，如果可以重用，则会分配给后面来的事务，避免为每个独立的事务分配独立的undo log页而浪费存储空间和性能。

- delete操作：不会直接删除，而是将delete对象打上delete flag，标记为删除，最终的删除操作是purge线程完成的。
- update操作，更新是否是主键列
	- 不是主键列：undo log记录更新之前记录。
	- 是主键列：先删除该行，然后在插入一行更新行。

