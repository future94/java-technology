
## 1. bitmap

Bitmap 的底层数据结构用的是 String 类型的 SDS 数据结构来保存位数组，Redis 把每个字节数组的 8 个 bit 位利用起来，每个 bit 位 表示一个元素的二值状态（不是 0 就是 1）。

### 1.1 相关命令

- `setbit key offset value`: 设置key在offset上value。offset单位为bit，value只能为0或1。
- `getbit key offset`: 获取key在offset上value。offset单位为bit。不存在也会返回0。
- `bitcount key [start end]`: 获取key在offset为start到end上值为1的数量。offset单位为byte，支持负数，比如 -1 表示最后一个字节，而 -2 表示倒数第二个字节。

offset从0开始的，0-7为第1个字节，8-15位第2个字节，所以如下情况：

```redis
127.0.0.1:6379> setbit weilai 1 1
(integer) 0
127.0.0.1:6379> setbit weilai 4 1
(integer) 0
127.0.0.1:6379> setbit weilai 8 1
(integer) 0
127.0.0.1:6379> getbit weilai 1
(integer) 1
127.0.0.1:6379> getbit weilai 2
(integer) 0
127.0.0.1:6379> getbit weilai 3
(integer) 0
127.0.0.1:6379> getbit weilai 4
(integer) 1
127.0.0.1:6379> getbit weilai 5
(integer) 0
127.0.0.1:6379> getbit weilai 8
(integer) 1
127.0.0.1:6379> bitcount weilai
(integer) 3
127.0.0.1:6379> bitcount weilai 0 0
(integer) 2
127.0.0.1:6379> bitcount weilai 0 -1
(integer) 3
127.0.0.1:6379> bitcount weilai 0 1
(integer) 3
127.0.0.1:6379> bitcount weilai 1 1
(integer) 1
127.0.0.1:6379> bitcount weilai 2 1
(integer) 0
```

- `bitop operation destkey key [key ...]`: 对不同key的二进制存储数据进行位运算，opration取值为(AND、OR、NOT、XOR)。
- `bitpos key bit [start] [end]`: 返回用户在offset为start到end其中第一个值为bit(0或1)的offset。

### 1.2 实战

#### 1.2.1 亿级别用户的在线状态

如用户ID为1000上线，我们设置offset为1000的值为1。
```redis
setbit loginStatus 1000 1
```

获取用户是否在线，0不在线，1在线。
```redis
getbit loginStatus 1000
```

获取用户在线的数量
```redis
bitcount loginStatus
```

#### 1.2.2 用户每月的签到情况

在签到统计中，每个用户每天的签到用 1 个 bit 位表示，一年的签到只需要 365 个 bit 位。一个月最多只有 31 天，只需要 31 个 bit 位即可。

key 可以设计成 `uid:sign:{userId}:{yyyyMM}`，月份的每一天的值 - 1 可以作为 offset（因为 offset 从 0 开始，所以 offset = 日期 - 1）。

如用户ID为1000的在2021年9月24号打卡
```redis
setbit uid:sign:1000:202109 23 1
```

查询用户ID为1000的在2021年9月24号是否打卡
```redis
getbit uid:sign:1000:202109 23
```

获取用户ID为1000在9月份的签到天数

```redis
bitcount uid:sign:1000:202109
```

获取用户ID为1000在9月份第一次打卡的时间
```redis
bitpos uid:sign:1000:202109 1
```

#### 1.2.3 连续签到用户总数

我们把每天的日期当作key，userid当作offset，打卡了，将value设置为1。

```redis
// 用户userId为1000在23号打卡
setbit day:23 1000 1
// 用户userId为1001在23号打卡
setbit day:23 1001 1
// 用户userId为1000在24号打卡
setbit day:24 1000 1
// 在连续两天内都打卡的用户数量(将key为day:23 day:24做and运算生成一个新的key为continuous)
bitop and continuous day:23 day:24
bitcount continuous
```

## 2. HyperLogLog

这是一种用于基数统计的数据集合类型，即使数据量很大，计算基数需要的空间也是固定的。

每个 HyperLogLog 最多只需要花费 12KB 内存就可以计算 2 的 64 次方个元素的基数。

Redis 对 HyperLogLog 的存储进行了优化，在计数比较小的时候，存储空间采用系数矩阵，占用空间很小。

只有在计数很大，稀疏矩阵占用的空间超过了阈值才会转变成稠密矩阵，占用 12KB 空间。

关于详细原理，可以查看[HyperLogLog 算法的原理讲解](https://www.cnblogs.com/linguanh/p/10460421.html)

去重复会有0.81%的误差，典型应用场景可以用HyperLogLog来计算UV。

**相关命令：**
- `pfadd key element [element ...]`：将指定的element放入key中。
- `pfcount key [key ...]`：返回指定key中数量（多个key则返回去重复后相加后的结果）。
- `pfmerge destkey sourcekey [sourcekey ...]`：将指定的sourcekey合并之后生成新的destkey。

**相关例子：**
```redis
//用户ID1访问文章ID1
pfadd aid:1 1
//用户ID2访问文章ID1
pfadd aid:1 1
//用户ID1访问文章ID2
pfadd aid:2 1
//计算文章ID1的UV(2)
pfcount aid:1
//计算文章ID1的UV(1)
pfcount aid:2
//计算整个站(就两个文章)文章UV（2）
pfcount aid:1 aid:2
//生成整个站的UV
pfmerge all aid:1 aid:2
//计算整个站(就两个文章)文章UV（2）
pfcount all
```

## 3. String常见用法

### 3.1 热点数据缓存

通过key-value结构，存储一些热点数据，已减轻数据库的压力。

### 3.2 分布式session

由于redis是独立部署的服务，所以可以将session放入redis中，以达到多个项目共享的目的

### 3.3 分布式锁

通过 `set key value EX seconds NX` 或者 `setnx key value` 来实现分布式锁。

### 3.4 计数器

可以通过 `incr` 、`incrby` 、`decr`、`decrby`来实现计数器的统计。

- 浏览量、点赞数量等。
- 分布式ID生成。
- 限流数量。

## 4. Hash常见用法

理论上String能做的，hash也都能做。相比与string的优点是 hash可以对一系列的数据进行归类，hash可以对一系列的某一个字段进行修改。如：我们使用String + Json存储，当我们修改了其中一个时，需要将整个json字符串重新生成并写入，而hash只需要改对应字段数据即可。

### 4.1 购物车

如：以用户id为key，商品id为field，商品数量为value，恰好构成了购物车的3个要素。如下图：

![image](967517-20190405010903324-1410775270.png)

### 4.2 缓存对象

由于对象也是对象名称，字段，值。与hash非常吻合，所以我们可以用hash来存储对象，而且修改指定字段时候也不需要像String + json那样全部修改。

### 4.3 相同系列归类

如我们发放优惠券，10块钱100张，20块钱100张，50块钱100张，如果使用string：

```redis
// 设置
set ticket:10 100
set ticket:20 100
set ticket:50 100
// 查看所有优惠卷
keys ticket:*
// 删除
del ticket:10 ticket:20 ticket:50
```

我们使用hash就可以统一归类
```redis
// 设置
hset ticket 10 100 20 100 50 100
// 删除
del ticket
```

## 5. List常见用法

### 5.1 消息队列

- 实现类似MQ中间件一样的消息队列。
- 而且双向链表即支持先出先出，也只是先入后出，

### 5.2 存储数据

- 数据放入其中是有顺序的，如时间线。
- 存储一系列消息集合，如最新评论、访问记录。

## 6. Set常见用法

### 6.1 随机取值

利用 `spop key [count]` 命令可以随机取出几个值。如：随机抽取幸运用户。

### 6.2 不重复的数据集合

数据量不大的不重复数据集合，如点赞、商品标签等。

```redis
// 用户3001点赞了ID为1001的商品
sadd like:1001 3001
// 用户3002点赞了ID为1001的商品
sadd like:1001 3002
// 用户3001取消点赞ID为1001的商品
srem like:1001 3001
// 用户3001是否点赞ID为1001的商品
sismember like:1001 3001
// 点赞ID为10001的商品所有用户：
smembers like:1001
// ID为10001的商品点赞数
scard like:1001

// 添加商品标签（去重复）
sadd tag:1001 tag1 tag2 tag3 tag1
```

### 6.3 交集、差集、并集等操作

#### 交集

**如寻找共同好友：**

```redis
// user1的好友
sadd user1 u2 u3 u4
// user1的好友
sadd user2 u1 u4
// user1和user2的共同好友
sinter user1 user2
```

#### 差集

**如可能认识的人**

```redis
// user1的好友
sadd user1 u2 u3 u4
// user1的好友
sadd user2 u1 u4
// user2可能认识的人（u2、u3）
sdiff user1 user2
```

#### 并集

**求互动（互动=点赞+转发）人数**
```redis
// 添加点赞人
sadd link1 u2 u3 u4
// 添加转发人
sadd interactive1 u1 u4
// 对ID1互动的人(u1、u2、u3、u4)
sunion link1 interactive1
```

## 7. Zset常见用法

### 7.1 排行榜

我们可以设置score来决定数据在集合中的位置，从而来生成排行榜数据

```redis
// 设置排行
zadd lookTop 100 张三 98 李四 97 王五
// 增加分数
zincrby lookTop 1 张三
// 查询[98,200] 分数的项，offset从0开始，最多获取10个
zrangebyscore lookTop 98 200 limit 0 10
// offset从0开始，最多获取5个
zrevrange ZREVRANGE lookTop 0 5
```

### 7.2 延时队列

利用score对数据进行排序，将过期时间设置为score，我们可以按顺序依次处理就可以了。第一个数据没过期其他肯定也没过期。

### 7.3 滑动窗口限流

滑动窗口是限流常见的一种策略。如果我们把一个用户的 ID 作为 key 来定义一个 zset ，member 或者 score 都为访问时的时间戳。我们只需统计某个 key 下在指定时间戳区间内的个数，就能得到这个用户滑动窗口内访问频次，与最大通过次数比较，来决定是否允许通过。

## 8. GEO 

GEO底层才用的ZSET存储结构，通过[GeoHash算法](../../algorithm/GeoHash算法.md)计算出经纬度的score值。

通过命令 `geoadd key longitude latitude member` 来添加对应元素的坐标。

```redis
geoadd localtion 13.361389 38.115556 "mem1" 15.087269 37.502669 "mem2"
```

通过命令 `georedius key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]` 来获取指定经纬度周围的数据。

```redis
georedius localtion 15.087269 37.502669 km ASC COUNT 10
```

通过`zrem key member` 来删除其中的数据
```redis
georedius localtion mem1
```