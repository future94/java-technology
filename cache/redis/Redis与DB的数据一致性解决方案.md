
## 1. 数据一致性
- 强一致性：写入什么就要就要读出什么，没有延迟。用户体验最好，但是性能消耗很高。
- 弱一致性：尽量在一段时间后，将数据尽可能的达到一致的状态。
- 最终一致性：算是弱一致性的特例，在一定时间后，将数据达到最终一致的状态。

## 2. Redis与DB的三种模式

- **旁路缓存模式** ：读数据（查看是否命中Cache，命中查询Cache返回，未命中查询DB，更新Cache后返回）。写数据（更新DB，删除Cache）。
- **读写穿透** ：读数据（查看是否命中Cache，命中查询Cache返回，未命中查询DB，更新Cache后返回）。写数据（如果没有Cache，直接写DB；如果有Cache，更新Cache，并由Cache同步写入DB）
- **异步缓存写入** ：对Cache进行读写操作，然后由Cache异步写入到DB。

### 2.1 旁路缓存模式

读数据：
![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/20210522212436746.png)

写数据：
![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/20210522212625984.png)

**为什么是先更新DB在删除Cache？**  

因为删除Cache比更新DB快的多，如果先删除Cache，这时候有人来请求，可能造成读取出来的Cache与DB数据不一致的情况。
如果我们先更新DB在删除缓存，就是可以降低这种情况的发生。

**先更新DB后删除缓存，Cache和DB数据一定一致嘛？**  

当时这条记录缓存已失效，线程t1是写操作（写db，删缓存），线程t2是读操作（读db，写缓存）。 时间线是：t2读db -> t1写db -> t1删缓存 -> t2写缓存。最后缓存里的是t1写之前的值，而我们期望的是t1写后的值。怎么解决后面会说。

**第一次永远不会走Cache的缺陷**  

可以缓存热点key来缓解

**频繁更新缓存对数据一致性的影响**  

数据库和缓存数据强一致场景，更新DB的时候同样更新cache，不过我们需要加一个锁/分布式锁来保证更新cache的时候不存在线程安全问题。也可以允许短暂的不一致情况。怎么后面会详细说。

### 2.2 读写穿透
与旁路缓存模式相比，由中间层Cache Provider进行操作。

读数据：
![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/20210522212527364.png)

- 查询的时候命中直接返回。
- 如果查询不到，由Cache Provider查询DB设置Cache后返回。

写数据：
![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/20210522212542188.png)

- 写入数据的时候，如果有缓存，由Cache Provider更新缓存并同步到DB中返回。
- 如果不存在，由Cache Provider直接操作DB后返回。

### 2.3 异步缓存写入

与读写穿透模式相似，异步缓存写入也是有Cache Provider对Cache和DB进行操作，但是他是异步操作了，并不能保证实时的一致性。

这种策略在我们平时开发过程中也非常非常少见，但是不代表它的应用场景少，比如消息队列中消息的异步写入磁盘、MySQL 的 InnoDB Buffer Pool 机制都用到了这种策略。

Write Behind Pattern 下 DB 的写性能非常高，非常适合一些数据经常变化又对数据一致性要求没那么高的场景，比如浏览量、点赞量。

## 3. Redis与MySQL双写一致性如何保证

上述问题产生原因：当时这条记录缓存已失效，线程t1是写操作（写db，删缓存），线程t2是读操作（读db，写缓存）。 时间线是：t2读db -> t1写db -> t1删缓存 -> t2写缓存。最后缓存里的是t1写之前的值，而我们期望的是t1写后的值。

### 3.1 缓存延时双删

先删除Cache，在更新DB，最后执行一个系统延时（一般是读DB加载Cache或者主从复制延时加几百毫秒），再次删除缓存防止可能存在的脏数据。

### 3.2 删除缓存重试机制

删除可能失败，会造成脏数据的残留，这时候我们可以加一个删除重试机制，删除失败的时候进行重试保证删除成功。

可以同步删除，也可以异步线程池删除，也可以放到队列中处理。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/131231231.png)

### 3.3 同步biglog异步删除缓存

业务代码有大量的入侵，我们可以用Canal订阅MySQL的binlog来做删除Cache的操作。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/1232131231.png)

## 4. 二级缓存的数据一致性

- 一级缓存：即本地缓存，存储在JVM中，如Guava、Map等等。
- 二级缓存：即集中缓存，存储在其他服务，如Redis、Memecache等。

### 4.1 一级缓存

优点：直接在JVM中，没有网络开销，速度非常快。
缺点：只能被自己访问，重启后丢失，占用JVM内存空间。

### 4.2 二级缓存

优点：在其他服务，多个服务可共用，不占用JVM内存。
缺点：有网络开销。

### 4.3 数据一致性

一级缓存与二级缓存也需要与DB做数据一致性。实现方式也差不多，利用Canal监听数据变化或者代码中维护，并利用MQ的订阅发布异步处理。更新一级缓存与二级缓存，并保证消息的最终一致性，如果需要强一致可以加分布式锁同步。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/20210605172721406.png)

## 5. 三级缓存的数据一致性

### 5.1 三级缓存

使用Nginx Lua共享字典作为L3本地缓存。每次请求过来，优先从nginx本地缓存中提取各种数据（这里nginx+lua脚本做页面动态生成的工作），结合页面模板，生成需要的页面。

### 5.2 数据一致性

三级缓存一般会用在超大流量的时候如秒杀场景缓存页面，一般设置热点数据永远不过期，通过 ngx.shared.DICT的缓存的LRU机制去淘汰。页面一般不会变化，如有变化也应该通知一级二级三级缓存进行更新变化。

**本人接触很少，欢迎大家提供完整的三级缓存方案及源码。**

参考文章：
- [3种常用的缓存读写策略](https://github.com/Snailclimb/JavaGuide/blob/master/docs/database/Redis/3%E7%A7%8D%E5%B8%B8%E7%94%A8%E7%9A%84%E7%BC%93%E5%AD%98%E8%AF%BB%E5%86%99%E7%AD%96%E7%95%A5.md)
- [Redis与DB的数据一致性解决方案（史上最全）](https://www.cnblogs.com/crazymakercircle/p/14853622.html)
- [分布式之数据库和缓存双写一致性方案解析](https://www.cnblogs.com/rjzheng/p/9041659.html)