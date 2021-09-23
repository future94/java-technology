- [1. 概览](#1-概览)
- [2. 基于内存实现](#2-基于内存实现)
- [3. I/O 多路复用模型](#3-IO-多路复用模型)
- [4. 单线程模型](#4-单线程模型)
- [5. 高效的数据结构](#5-高效的数据结构)
	- [5.1 Redis5种数据结构与底层的6种数据结构](#51-Redis5种数据结构与底层的6种数据结构)
	- [5.2 SDS 简单动态字符](#52-SDS-简单动态字符)
	- [5.3 LinkedList 双端列表](#53-LinkedList-双端列表)
	- [5.4 ZipList 压缩列表](#54-ZipList-压缩列表)
	- [5.5 quicklist 快速列表](#55-quicklist-快速列表)
	- [5.6 HashTable 哈希表](#56-HashTable-哈希表)
	- [5.7 IntSet 整数集合](#57-IntSet-整数集合)
	- [5.8 SkipList 跳表](#58-SkipList-跳表)
- [6. 合理的数据编码](#6-合理的数据编码)

## 1. 概览

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/640.png)

## 2. 基于内存实现

Redis数据存储在内存中，读邪恶操作不需要磁盘的IO操作，CPU直接操作内存即可，所以速度非常快。

## 3. I/O 多路复用模型

Redis 采用 I/O 多路复用技术，并发处理连接。采用了 epoll + 自己实现的简单的事件框架。epoll 中的读、写、关闭、连接都转化成了事件，然后利用 epoll 的多路复用特性，绝不在 IO 上浪费一点时间。

什么是epoll呢？详细的可以点击[IO多路复用及Epoll原理](../../os/IO多路复用及Epoll原理.md)了解，这里我们只简单的说下。

**普通的IO读写步骤**（后面的步骤单词自己起的）
- 服务端阻塞等待客户端建立连接, `accept`。
- 从socket中阻塞读取请求，`read`。
- 解析客户端发送的命令， `parse`。
- 执行命令，`exec`。
- 向socket中写数据，`write`。

由于accept、read操作会阻塞线程，而parse、write又和核心命令执行没有什么关系，所以普通的IO操作性能非常低。

epoll 提供了基于事件的回调机制，即针对不同事件的发生，调用相应的事件处理器。

epoll会创建一个对象维护等待队列，当socket中数据准备好了之后，就会直接唤醒程序告诉那个Socket数据准备就绪，将数据直接转发到工作线程即可。

## 4. 单线程模型

Redis6.0执行命令的核心模块还是单线程的，但是IO相关操作已经为多线程了，如网络数据的读写、协议的解析等等。

redis的瓶颈不在CPU而是在网络IO上，所以redis的核心模块还是单线程的。

**单线程优点：**
- 可以减少CPU上下文和多线程切换的开销。
- 不需要引入锁开销来保证多线程操作共同数据时的线程安全问题。
- 代码简介，逻辑更清晰。

## 5. 高效的数据结构

### 5.1 Redis5种数据结构与底层的6种数据结构

**五种数据结构：**
- **String** ：缓存、计数器、分布式锁等。
- **Hash** ：用户信息、Hash 表等。
- **List** ：链表、队列、微博关注人时间轴列表等。
- **Set** ：去重、赞、踩、共同好友等。
- **SortedSet** ：访问量排行榜、点击量排行榜等。

**六种底层数据结构：**
- **SDS** 简单动态字符
- **LinkedList** 双端列表
- **ZipList** 压缩列表
- **HashTable** 哈希表
- **IntSet** 整数集合
- **SkipList** 跳跃表

#### 对应关系如下表

| 数据结构 | 对应底层数据结构 |
|---|---|
| String | SDS |
| Hash | ZipList / HashTable |
| List | ZipList / LinkedList |
| Set | IntSet / HashTable |
| SortedSet | ZipList / SkipList |

对应关系如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/640-1.png)

### 5.2 SDS 简单动态字符

#### 字符串结构

redis使用C语言实现，数据结构如下：
```c
struct sdshdr{
     //记录buf数组中已使用字节的数量
     //等于 SDS 保存字符串的长度
     int len;
     //记录 buf 数组中未使用字节的数量
     int free;
     //字节数组，用于保存字符串
     char buf[];
}
```

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/1120165-20180528075607627-218845583.png)

#### 查询字符串长度优化

C语言字符串有弊端，比如是以`\0`字符结束，每次获取字符串长度都需要遍历到`\0`处才知道长度。Redis 自定义了结构体`sdshdr`来存储字符串，加上了len。不需要通过`\0`来获取字符串长度。

#### 支持二进制数据存储

String还能支持二进制数据存储，因为二进制数据中存在`\0`不会被认为是结束，而是通过sdshdr中的len判断。


#### 空间预分配

当SDS进行修改时，会进行空间的预分配，避免修改时候字符串的内存重新分配次数。策略如下：
- 数据小于1M时候，多分配先有空间+1（之前len为10，修改之后len为21 = 10 + 10 + 1）。
- 数据大于等于1M时候，修改后多分配1M空间（之前len为1.5M，修改之后为2.5M = 1.5M + 1M）。

#### 惰性空间释放

当对 SDS 进行字符串缩短操作时，程序并不会回收多余的内存空间，而是使用 free 字段将这些字节数量记录下来不释放，后面如果需要 append 操作，则直接使用 free 中未使用的空间，减少了内存的分配。

### 5.3 LinkedList 双端列表

双端列表很好的支持`先进先出`或者`先进后出`。

```c
// 链表
typedef struct list{
     //表头节点
     listNode *head;
     //表尾节点
     listNode *tail;
     //链表所包含的节点数量
     unsigned long len;
     //节点值复制函数
     void (*dup) (void *ptr);
     //节点值释放函数
     void (*free) (void *ptr);
     //节点值对比函数
     int (*match) (void *ptr, void *key);
}list;

// 每个节点
typedef struct listNode{
      struct listNode *prev;
      struct listNode * next;
      void * value;  
}listNode;
```

**Redis 的链表实现的特性可以总结如下：**

1. **双端** ：链表节点带有 prev 和 next 指针，获取某个节点的前置节点和后置节点的复杂度都是 O（1）。
2. **无环** ：表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL，对链表的访问以 NULL 为终点。
3. **带表头指针和表尾指针** ：通过 list 结构的 head 指针和 tail 指针，程序获取链表的表头节点和表尾节点的复杂度为 O（1）。
4. **带链表长度计数器** ：程序使用 list 结构的 len 属性来对 list 持有的链表节点进行计数，程序获取链表中节点数量的复杂度为 O（1）。
5. **多态** ：链表节点使用 void* 指针来保存节点值，并且可以通过 list 结构的 dup、free、match 三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值。

> Redis3.2+版本对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist.

### 5.4 ZipList 压缩列表

压缩列表的原理：<font color="red">压缩列表并不是对数据利用某种算法进行压缩，而是将数据按照一定规则编码在一块连续的内存区域，目的是节省内存</font>。

```c
struct ziplist<T> {
    int32 zlbytes; // 整个压缩列表占用字节数
    int32 zltail_offset; // 最后一个元素距离压缩列表起始位置的偏移量，用于快速定位到最后一个节点
    int16 zllength; // 元素个数
    T[] entries; // 元素内容列表，挨个挨个紧凑存储
    int8 zlend; // 标志压缩列表的结束，值恒为 0xFF
}
```

形式如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/640-2.png)

压缩列表当我们查询第一个和最后一个时候非长快，O(1)。当寻找其他元素的时候就需要遍历了，O(n)。

> Redis3.2+版本对列表数据结构进行了改造，使用 quicklist 代替了 ziplist 和 linkedlist.

### 5.5 quicklist 快速列表

**quicklist 是 ziplist 和 linkedlist 的混合体，它将 linkedlist 按段切分，每一段使用 ziplist 来紧凑存储，多个 ziplist 之间使用双向指针串接起来**。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/640-3.png)


### 5.6 HashTable 哈希表

在redis中，无论哪一种数据结构，都需要有一个Key，对应一个Value，只是Value有5种数据结构。Redis整体就是用一张hashtable来保存所有的键值对，与java中的hashmap类似，采用`数据+链表`的方式存储数据。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/640-4.png)

当键过长时，hash冲突就越多，导致链表过长从而影响查询性能，这时候会触发rehash操作。

**渐进式 rehash:**
rehash的时候，如果Key十分大，整个rehash事件就会很长，所以redis每次根据已使用的空间扩大一倍创建另一张hash表。而且并不是一下子全都迁移过去，而是将索引一个一个的迁移过去，这就是`渐进式 rehash`。

### 5.7 IntSet 整数集合

当一个Set集合中只包含整数值元素并且数量不多时，Redis使用整数集合来存储。

```c
typedef struct intset{
     //编码方式
     uint32_t encoding;
     //集合包含的元素数量
     uint32_t length;
     //保存元素的数组
     int8_t contents[];
}intset;
```

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/2021081714382565.png)

contents 数组是整数集合的底层实现：整数集合的每个元素都是 contents 数组的一个数组项（item），<font color="red">各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项</font>。length 属性记录了整数集合包含的元素数量，也即是 contents 数组的长度。

### 5.8 SkipList 跳表

跳表（skiplist）是一种有序数据结构，它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

跳跃表支持平均 O（logN）、最坏 O（N）复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

跳表在链表的基础上，增加了多层级索引，通过索引位置的几个跳转，实现数据的快速定位，如下图所示：

- 查找：如果我们想查找40，那么起始位置在最高层1，下一个是28，28比40小，所以我们跳到28，下一个是100，100大于40，所以我们向下走到28，下一个是66，66比40大，所以我们向下走到28，下一个是40，正好找到。
- 增加：首先确定插入的层数，先查找到插入的位置。要不要向上插入。有一种方法是假设抛一枚硬币，如果是正面就累加，直到遇见反面为止，最后记录正面的次数作为插入的层数。当确定插入的层数k后，则需要将新元素插入到从底层到k层。
- 删除：在各个层中找到包含指定值的节点，然后将节点从链表中删除即可，如果删除以后只剩下头尾两个节点，则删除这一层。

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/1632398900966.jpg)

```c
typedef struct zskiplist {
    //表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    //表中节点的数量
    unsigned long length;
    //表中层数最大的节点的层数
    int level;
} zskiplist;

typedef struct zskiplistNode {
    /* 存储字符串类型数据 redis3.0版本中使用robj类型表示，但是在redis4.0.1中直接使用sds类型表示 */
    sds ele;
    // 存储排序的分值
    double score;
    // 后退指针，指向当前节点最底层的前一个节点
    struct zskiplistNode *backward;
    // 层，柔性数组，随机生成1-64的值
    struct zskidictEntryplistLevel {
    	//指向本层下一个节点
        struct zskiplistNode *forward;
        //本层下个节点到本节点的元素个数
        unsigned long span;
    } level[];
} zskiplistNode;
```

![image](https://raw.githubusercontent.com/future94/java-technology/master/cache/redis/images/2021081714382357.jpg)

## 6. 合理的数据编码

例如当我们创建一个数据`set key value`的时候，我们会创建一个redis对象，用redisObject存储。

```c
typedef struct redisObject {
	//类型 对象类型(对应String、Hash、List、Set、SortedSet)
    unsigned type:4;
    //编码
    unsigned encoding:4;
    // LRU_BITS为24bit 记录最后一次被命令程序访问的时间
    unsigned lru:LRU_BITS; 
    //引用计数
    int refcount;
    //指向底层实现数据结构的指针
    void *ptr;
} robj;
```

**type**  
对应String、Hash、List、Set、SortedSet。可以使用`type key`命令查看类型。

```txt
127.0.0.1:6379> set wei lai
OK
127.0.0.1:6379> type wei
string
```

**encoding**  
对于每一种数据类型来说，底层的支持可能是多种数据结构，什么时候使用哪种数据结构，这就涉及到了编码转化的问题。

```txt
127.0.0.1:6379> OBJECT encoding wei
"embstr"
127.0.0.1:6379> set lai 1
OK
127.0.0.1:6379> OBJECT encoding lai
"int"
127.0.0.1:6379>
```

**encoding对数据的优化：**
- string：
	- 数字时候为int。
	- 非数字小于等于44为`embstr`编码，大于44为`raw`编码（低版本为39，为什么是39和44，自己看下吧）。
- hash：
	- ziplist：Hash 对象保存的所有键值对的键和值的字符串长度均小于 64 字节 **并且** Hash 对象保存的键值对数量小于 512 个。
	- hashtable：其他情况就是hashtable。
- list：
	- ziplist：字符串长度小于64 **并且** 元素的个数小于512个。
	- linkedlist：其他情况就是linkedlist。
- set：
	- intset：存储的值为整数 **并且** 个数小于一定个数。
	- hashtable：其他情况就是hashtable。
- zset：
	- ziplist：Zset 元素的成员长度都小于 64 字节 **并且** Zset 元素的成员个数小于128个。
	- skiplist：其他情况就是skiplist。
