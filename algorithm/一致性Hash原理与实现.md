
## 1. 产生的背景

当我们分库分表、服务器负载的时候，如果单纯的采用hash取模的方式会有很多问题。比如采用hash取模的方式会产生数据分布不均匀的情况，扩容缩容也非常麻烦。这时候我们就可以用一致性hash方案解决。基于虚拟节点设计原理的一致性hash可以让数据分布更均匀。

## 2. 一致性Hash算法

### 2.1 概念
一致性Hash算法也是使用取模的方法，不过，上述的取模方法是对服务器的数量进行取模，而一致性的Hash算法是对 `2的32方取模`。即，一致性Hash算法将整个Hash空间组织成一个虚拟的圆环，Hash函数的值空间为 `0 ~ 2^32 - 1` (一个32位无符号整型)，整个哈希环如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/181738.png)

整个圆环以 `顺时针方向` 组织，圆环正上方的点代表0，0点右侧的第一个点代表1，以此类推。
第二步，我们将各个服务器使用Hash进行一个哈希，具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台服务器就确定在了哈希环的一个位置上，比如我们有三台机器，使用IP地址哈希后在环空间的位置如图:

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/232522.png)

将数据Key使用相同的函数Hash计算出哈希值，并确定此数据在环上的位置，从此位置沿环顺时针查找，遇到的服务器就是其应该定位到的服务器。

例如，现在有ObjectA，ObjectB，ObjectC三个数据对象，经过哈希计算后，在环空间上的位置如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/232655.png)

根据算法，ObjectA对应的是服务器A，ObjectB对应的是服务器B，ObjectC对应的是服务器C。

### 2.2 容错性

当服务器C挂了时候，影响的是服务器B与服务器C之间Hash值的数据，并不会影响所有的数据。

### 2.3 扩展性

如果我们在服务器C和服务器A之间增加一个服务器D，我们只需要对C到D之间的Hash值数据进行重新处理即可，其他的不需要变化。

### 2.4 数据倾斜的问题（虚拟节点）

当我们服务器比较少的时候，计算出来的hash可能不均匀，导致数据分配的不均匀，这是一致性hash算法还有一个 `虚拟节点` 的概念。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/1631374692593.jpg)

B'、C'、D' 是原始节点复制出来的虚拟节点，原本映射到节点D上的数据1和4，分别被映射到了B'和D'，最终会落到对应的真实节点B和D上。我们可以看到，虚拟节点让数据分布更均匀了。

## 3. 代码
```java
public interface IHashService {

    Long hash(String key);
}
```

```java
import java.nio.ByteBuffer;
import java.nio.ByteOrder;

public class HashService implements IHashService {

    /**
     * MurmurHash算法,性能高,碰撞率低
     * @param key String
     * @return Long
     */
    @Override
    public Long hash(String key) {
        ByteBuffer buf = ByteBuffer.wrap(key.getBytes());
        int seed = 0x1234ABCD;

        ByteOrder byteOrder = buf.order();
        buf.order(ByteOrder.LITTLE_ENDIAN);

        long m = 0xc6a4a7935bd1e995L;
        int r = 47;

        long h = seed ^ (buf.remaining() * m);

        long k;
        while (buf.remaining() >= 8) {
            k = buf.getLong();

            k *= m;
            k ^= k >>> r;
            k *= m;

            h ^= k;
            h *= m;
        }

        if (buf.remaining() > 0) {
            ByteBuffer finish = ByteBuffer.allocate(8).order(ByteOrder.LITTLE_ENDIAN);
            finish.put(buf).rewind();
            h ^= finish.getLong();
            h *= m;
        }

        h ^= h >>> r;
        h *= m;
        h ^= h >>> r;

        buf.order(byteOrder);
        return h;

    }
}
```

```java
public class Node {

    private String ip;
    private String name;

    public Node(String ip, String name) {
        this.ip = ip;
        this.name = name;
    }

    public String getIp() {
        return ip;
    }

    public void setIp(String ip) {
        this.ip = ip;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    /**
     * 使用IP当做hash的Key
     *
     * @return String
     */
    @Override
    public String toString() {
        return ip;
    }
}
```

```java
import java.util.Collection;
import java.util.SortedMap;
import java.util.TreeMap;

public class ConsistentHash<T> {

    // Hash函数接口
    private final IHashService iHashService;
    // 每个机器节点关联的虚拟节点数量
    private final int          numberOfReplicas;
    // 环形虚拟节点
    private final SortedMap<Long, T> circle = new TreeMap<>();

    public ConsistentHash(IHashService iHashService, int numberOfReplicas, Collection<T> nodes) {
        this.iHashService = iHashService;
        this.numberOfReplicas = numberOfReplicas;
        for (T node : nodes) {
            add(node);
        }
    }

    /**
     * 增加真实机器节点
     *
     * @param node T
     */
    public void add(T node) {
        for (int i = 0; i < this.numberOfReplicas; i++) {
            circle.put(this.iHashService.hash(node.toString() + i), node);
        }
    }

    /**
     * 删除真实机器节点
     *
     * @param node T
     */
    public void remove(T node) {
        for (int i = 0; i < this.numberOfReplicas; i++) {
            circle.remove(this.iHashService.hash(node.toString() + i));
        }
    }

    public T get(String key) {
        if (circle.isEmpty()) {
            return null;
        }

        long hash = iHashService.hash(key);

        // 沿环的顺时针找到一个虚拟节点
        if (!circle.containsKey(hash)) {
            SortedMap<Long, T> tailMap = circle.tailMap(hash);
            hash = tailMap.isEmpty() ? circle.firstKey() : tailMap.firstKey();
        }
        return circle.get(hash);
    }
}
```

5000条记录运行大致分布的很均匀。
```txt
192.168.0.1节点记录条数：491
192.168.0.2节点记录条数：477
192.168.0.3节点记录条数：556
192.168.0.4节点记录条数：418
192.168.0.5节点记录条数：538
192.168.0.6节点记录条数：510
192.168.0.7节点记录条数：426
192.168.0.8节点记录条数：585
192.168.0.9节点记录条数：461
192.168.0.10节点记录条数：538
```

参考文章：
- [一致性Hash原理与实现](https://www.jianshu.com/p/528ce5cd7e8f)
- [每日百万订单，这样的技术方案更靠谱](https://mp.weixin.qq.com/s/_7jJNNPI21uPhWM2F7vapA)