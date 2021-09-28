
## 1. LRU算法

LRU算法是常用的内存淘汰算法，可以将不常用的key从内存中淘汰到，并且查找和添加的时间复杂度都是O(1)。通常采用 `LinkedList + HashMap` 来实现。

具体可以点击[LRU算法](../../algorithm/LRU算法.md)

## 2. redis内存淘汰机制

- **volatile-lru** ：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
- **volatile-ttl** ：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
- **volatile-random** ：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
- **allkeys-lru** ：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key（这个是最常用的）
- **allkeys-random** ：从数据集（server.db[i].dict）中任意选择数据淘汰
- **no-eviction** ：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！
4.0版本后增加以下两种：
- **volatile-lfu** ：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
- **allkeys-lfu** ：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的key

## 3. redis内存淘汰机制实现

### 3.1 redis中的LRU算法

redis中的LRU并不是真的LRU算法，并没有采用双向链表来维护key的使用情况。我们知道，redis的所有数据类型存储都依赖于key，如果每个key使用时都维护一个链表（头尾指针16字节），那么会消耗掉许多内存。作者认为既然LRU本来就是基于假设做出的算法，为什么不能模拟实现一个LRU呢。所以redis中的LRU算法并不能准确的删除掉最不常用的key。

我们在[Redis为什么这么快](Redis为什么这么快.md)中介绍了redis所有的数据类型都会有一个key，通过hashtable来存储，而value可能用各种不同的类型来合理存储，使用redisObject结构体来表示，而redisObject结构体中，有一项24bit长度的字段，字段里边保存的是一个时间戳（单位秒）用它来记录最后使用的时间。由于只有24bit，所以最多为194天。

```c
typedef struct redisObject {
    
    unsigned lru:LRU_BITS; //LRU_BITS为24bit
    
} robj;
```

每次获取key的时候，都会调用lookupKey来更新下（LRU或者LFU的方式），而`LRU_CLOCK`方法就是更新了redisObject中的时间戳。

```c
robj *lookupKey(redisDb *db, robj *key, int flags) {
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) { //如果配置的是lfu方式，则更新lfu
        updateLFU(val);
    } else {
        val->lru = LRU_CLOCK();//否则按lru方式更新
    }
}
```

redis最开始设计的非常简单，就是从hashtable中取出`maxmemory-samples`个key，淘汰掉其中最小的时间戳。

这种随机选key的情况效率可能会有些低，在3.0以后增加了LRU淘汰池pool，进一步提高了与LRU算法的近似效果。第一次随机一些key放入池子中，下次获取key的时候，将更小的lru值留在pool中，淘汰时淘汰掉pool中最小的key。

### 3.2 redis中的LFU算法

LRU算法可能会有一些问题，如下情况：

```txt
A~~A~~A~~A~~A~~A~~A~~A~~A~~A~~~|
B~~~~~B~~~~~B~~~~~B~~~~~~~~~~~B|
```

按照LRU算法，我们应该淘汰掉数据A，但是A的访问频率比B高很多，淘汰掉B可能更合理一点，所以Rdis4.0后提出了LFU算法，增加了使用频率。

<font color="red">LFU把原来的key对象的内部时钟的24位分成两部分，前16位还代表时钟（单位是分钟），后8位代表一个计数器</font>。16位的情况下如果还按照秒为单位就会导致不够用，所以一般这里以分钟为单位。前16位还用来计时，后8位用来表示访问频率，还是采用淘汰池pool的方式，留下时间更小的，淘汰时候淘汰掉计数器更小的。

Redis4.0之后为`maxmemory_policy`淘汰策略添加了两个LFU模式：

`volatile-lfu`：对有过期时间的key采用LFU淘汰算法
`allkeys-lfu`：对全部key采用LFU淘汰算法


计数器不是自然增加的，**还有2个配置可以调整LFU算法：**

- `lfu-log-factor`可以调整计数器counter的增长速度，lfu-log-factor越大，counter增长的越慢。
- `lfu-decay-time`是一个以分钟为单位的数值，可以调整counter的减少速度

```txt
# +--------+------------+------------+------------+------------+------------+
# | factor | 100 hits   | 1000 hits  | 100K hits  | 1M hits    | 10M hits   |
# +--------+------------+------------+------------+------------+------------+
# | 0      | 104        | 255        | 255        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 1      | 18         | 49         | 255        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 10     | 10         | 18         | 142        | 255        | 255        |
# +--------+------------+------------+------------+------------+------------+
# | 100    | 8          | 11         | 49         | 143        | 255        |
# +--------+------------+------------+------------+------------+------------+
```

增长计数器原码如下：

```c
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;
    double r = (double)rand()/RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    if (r < p) counter++;
    return counter;
}
```

参考文章：
- [Redis中的LRU算法实现](https://segmentfault.com/a/1190000017555834)
- [Redis中的LFU算法](https://www.cnblogs.com/linxiyue/p/10955533.html)

