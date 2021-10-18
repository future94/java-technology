
## 1. 什么是LRU算法

一般用在内存淘汰机制中，我们都知道内存是十分宝贵的，因为他的运行速度比磁盘快很多，但是内存又是有限的，没有磁盘那么大，所以我们就要淘汰掉内存中不常用的数据。

下图演示 LRU 原理，假设内存只能容纳3个页大小，按照 7 0 1 2 0 3 0 4 的次序访问页。假设内存按照栈的方式来描述访问时间，在上面的，是最近访问的，在下面的是，最远时间访问的，LRU就是这样工作的。

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/v2-584ed398c35ba76250cfb2f01b20ec0c_1440w.jpg)

核心思想一共就固定大小的内存，如果不够，就淘汰掉最后操作的数据。

## 2. 实现思路

[leetcode第146题LRU缓存机制](https://leetcode-cn.com/problems/lru-cache/)

hash表 + 双向链表。这样可以满足获取和插入都在 O(1) 时间复杂度内。hash表操作为 O(1)，链表可以记录插入的顺序，让最先插入的数据被淘汰掉。如下图所示：

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/v2-09f037608b1b2de70b52d1312ef3b307_1440w.png)

![image](https://raw.githubusercontent.com/future94/java-technology/master/algorithm/images/1566782-20190713105655398-1688289084.jpg)

## 3. 实现代码

```java
import java.util.HashMap;
import java.util.Map;

/**
 * @author weilai
 */
public class LRUCache {

    public static void main(String[] args) {
        LRUCache lruCache = new LRUCache(2);
        lruCache.put(1, 1); // 缓存是 {1=1}
        lruCache.put(2, 2); // 缓存是 {1=1, 2=2}
        System.out.println(lruCache.get(1));
        lruCache.put(3, 3); // 该操作会使得关键字 2 作废，缓存是 {1=1, 3=3}
        System.out.println(lruCache.get(2));
        lruCache.put(4, 4); // 该操作会使得关键字 1 作废，缓存是 {4=4, 3=3}
        System.out.println(lruCache.get(1));
        System.out.println(lruCache.get(3));
        System.out.println(lruCache.get(4));
    }

    private Map<Integer, Node> map;

    private DoubleLinkedList cache;

    private int capacity;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        map = new HashMap<>();
        cache = new DoubleLinkedList();
    }

    public int get(int key) {
        if (!map.containsKey(key)) {
            return -1;
        }
        int value = map.get(key).value;
        put(key, value);
        return value;
    }

    public void put(int key, int value) {
        Node node = new Node(key, value);
        if (map.containsKey(key)) {
            cache.delete(map.get(key));
            cache.addHead(node);
            map.put(key, node);
        } else {
            if (map.size() >= capacity) {
                int k = cache.deleteLast();
                map.remove(k);
            }
            map.put(key, node);
            cache.addHead(node);
        }
    }
}

class DoubleLinkedList {

    public Node head;

    public Node tail;

    public DoubleLinkedList() {
        Node node = new Node(0, 0);
        head = node;
        tail = node;
        head.next = tail;
        tail.prev = head;
    }

    public void addHead(Node node) {
        node.next = head.next;
        node.prev = head;

        head.next.prev = node;
        head.next = node;
    }

    public int delete(Node node) {
        node.next.prev = node.prev;
        node.prev.next = node.next;
        return node.key;
    }

    public int deleteLast() {
        if (head.next == tail) {
            return -1;
        }
        return delete(tail.prev);
    }
}


class Node {

    public Integer key;

    public Integer value;

    public Node next;

    public Node prev;

    public Node(Integer key, Integer value) {
        this.key = key;
        this.value = value;
    }
}
```

## 4. redis中的LRU算法

![Redis内存淘汰机制](../cache/redis/Redis内存淘汰机制.md)




推荐视频教程：
- [LRU Cache 实现剖析](https://www.bilibili.com/video/BV1hp4y1x7MH?from=search&seid=393840591896601714&spm_id_from=333.337.0.0)