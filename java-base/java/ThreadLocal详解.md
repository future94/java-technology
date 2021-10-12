## 1. ThreadLocal

### 1.1 ThreadLocal简介

通常情况下，我们创建的变量是可以被任何一个线程访问并修改的。**如果想实现每一个线程都有自己的专属本地变量该如何解决呢？** JDK中提供的`ThreadLocal`类正是为了解决这样的问题。 **`ThreadLocal`类主要解决的就是让每个线程绑定自己的值，可以将`ThreadLocal`类形象的比喻成存放数据的盒子，盒子中可以存储每个线程的私有数据。**

**如果你创建了一个`ThreadLocal`变量，那么访问这个变量的每个线程都会有这个变量的本地副本，这也是`ThreadLocal`变量名的由来。他们可以使用 `get（）` 和 `set（）` 方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题。**

再举个简单的例子： 

比如有两个人去宝屋收集宝物，这两个共用一个袋子的话肯定会产生争执，但是给他们两个人每个人分配一个袋子的话就不会出现这样的问题。如果把这两个人比作线程的话，那么ThreadLocal就是用来避免这两个线程竞争的。

<font color="red">注意：不应该在ThreadLocal中存放static变量</font>。

### 1.2 应用场景

- 每个场景都需要一个独享的对象（通常是工具类，比如SimpleDateFormat、Random等），一般使用ThreadLocal.withInitial初始化。
- 每个线程保存不存的全局变量信息（比如拦截器中的用户信息），可以减少参数传递的麻烦，一个用get、set对值操作。

### 1.3 ThreadLocal示例

相信看了上面的解释，大家已经搞懂 ThreadLocal 类是个什么东西了。

```java
import java.text.SimpleDateFormat;
import java.util.Random;

public class ThreadLocalExample implements Runnable{

     // SimpleDateFormat 不是线程安全的，所以每个线程都要有自己独立的副本
    private static final ThreadLocal<SimpleDateFormat> formatter = ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyyMMdd HHmm"));

    public static void main(String[] args) throws InterruptedException {
        ThreadLocalExample obj = new ThreadLocalExample();
        for(int i=0 ; i<10; i++){
            Thread t = new Thread(obj, ""+i);
            Thread.sleep(new Random().nextInt(1000));
            t.start();
        }
    }

    @Override
    public void run() {
        System.out.println("Thread Name= "+Thread.currentThread().getName()+" default Formatter = "+formatter.get().toPattern());
        try {
            Thread.sleep(new Random().nextInt(1000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //formatter pattern is changed here by thread, but it won't reflect to other threads
        formatter.set(new SimpleDateFormat());

        System.out.println("Thread Name= "+Thread.currentThread().getName()+" formatter = "+formatter.get().toPattern());
    }

}

```

Output:

```
Thread Name= 0 default Formatter = yyyyMMdd HHmm
Thread Name= 0 formatter = yy-M-d ah:mm
Thread Name= 1 default Formatter = yyyyMMdd HHmm
Thread Name= 2 default Formatter = yyyyMMdd HHmm
Thread Name= 1 formatter = yy-M-d ah:mm
Thread Name= 3 default Formatter = yyyyMMdd HHmm
Thread Name= 2 formatter = yy-M-d ah:mm
Thread Name= 4 default Formatter = yyyyMMdd HHmm
Thread Name= 3 formatter = yy-M-d ah:mm
Thread Name= 4 formatter = yy-M-d ah:mm
Thread Name= 5 default Formatter = yyyyMMdd HHmm
Thread Name= 5 formatter = yy-M-d ah:mm
Thread Name= 6 default Formatter = yyyyMMdd HHmm
Thread Name= 6 formatter = yy-M-d ah:mm
Thread Name= 7 default Formatter = yyyyMMdd HHmm
Thread Name= 7 formatter = yy-M-d ah:mm
Thread Name= 8 default Formatter = yyyyMMdd HHmm
Thread Name= 9 default Formatter = yyyyMMdd HHmm
Thread Name= 8 formatter = yy-M-d ah:mm
Thread Name= 9 formatter = yy-M-d ah:mm
```

从输出中可以看出，Thread-0已经改变了formatter的值，但仍然是thread-2默认格式化程序与初始化值相同，其他线程也一样。

上面有一段代码用到了创建 `ThreadLocal` 变量的那段代码用到了 Java8 的知识，它等于下面这段代码，如果你写了下面这段代码的话，IDEA会提示你转换为Java8的格式(IDEA真的不错！)。因为ThreadLocal类在Java 8中扩展，使用一个新的方法`withInitial()`，将Supplier功能接口作为参数。

```java
 private static final ThreadLocal<SimpleDateFormat> formatter = new ThreadLocal<SimpleDateFormat>(){
        @Override
        protected SimpleDateFormat initialValue()
        {
            return new SimpleDateFormat("yyyyMMdd HHmm");
        }
    };
```

### 1.4 Thread、ThreadLocal、ThreadLocalMap之间的关系

Thread类中有属性ThreadLocalMap，用来存储线程私有的数据。ThreadLocal相当于对ThreadLocalMap的操作封装。ThreadLocalMap中的key为每个ThreadLocal对象的hash值，value为存储的具体数据。`ThreadLocalMap与HashMap相似，但解决冲突情况不一样，HashMap采用的是拉链法，ThreadLocalMap采用线性探测法（找下一个空位置）`。

- 因为存储线程私有数据的ThreadLocalMap时Thread的私有属性，所以可以做到数据变量线程私有。
- ThreadLocalMap的key对不同的ThreadLocal对象的hash值，所以每个线程可以存放很多的私有的ThreadLocal数据。

![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/java/images/0664df7c14549edc2da804dd78cde6d7.png)

## 2. ThreadLocal原理

从 `Thread`类源代码入手。

```java
public class Thread implements Runnable {
 ......
//与此线程有关的ThreadLocal值。由ThreadLocal类维护，createMap时创建。
ThreadLocal.ThreadLocalMap threadLocals = null;

//与此线程有关的InheritableThreadLocal值。由InheritableThreadLocal类维护，createMap时创建。
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
 ......
}
```

从上面`Thread`类 源代码可以看出`Thread` 类中有一个 `threadLocals` 和 一个  `inheritableThreadLocals` 变量，它们都是 `ThreadLocalMap`  类型的变量,我们可以把 `ThreadLocalMap`  理解为`ThreadLocal` 类实现的定制化的 `HashMap`。默认情况下这两个变量都是null，只有当前线程调用 `ThreadLocal` 类的 `set`或`get`方法时才创建它们，实际上调用这两个方法的时候，我们调用的是`ThreadLocalMap`类对应的 `get()`、`set() `方法。

### 2.1 hash值的计算

通过AtomicInteger自增来获得ThreadLocalMap的key，即ThreadLocal的hash值。
```java
private final int threadLocalHashCode = nextHashCode();

private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
	return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

### 2.2 `set()`方法

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

```java
private void set(ThreadLocal<?> key, Object value) {
	Entry[] tab = table;
	int len = tab.length;
	int i = key.threadLocalHashCode & (len-1);

	for (Entry e = tab[i];
		 e != null;
		 e = tab[i = nextIndex(i, len)]) {
		ThreadLocal<?> k = e.get();

		if (k == key) {
			e.value = value;
			return;
		}

		// 清理key为null时候的情况，避免内存泄漏
		if (k == null) {
			replaceStaleEntry(key, value, i);
			return;
		}
	}

	tab[i] = new Entry(key, value);
	int sz = ++size;
	if (!cleanSomeSlots(i, sz) && sz >= threshold)
		rehash();
}
```

### 2.3 `get()`方法

如果没有调用过set方法，走下面的setInitialValue进行初始化并返回初始化的值。
```java
public T get() {
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null) {
		ThreadLocalMap.Entry e = map.getEntry(this);
		// 如果没有调用过set方法，e为null，走下面的setInitialValue进行初始化。
		if (e != null) {
			@SuppressWarnings("unchecked")
			T result = (T)e.value;
			return result;
		}
	}
	return setInitialValue();
}

private T setInitialValue() {
	// 获取初始化设置的值，初始化时候必须重写initialValue方法，所以已经提供了初始化的值，直接调用get返回null。
	T value = initialValue();
	Thread t = Thread.currentThread();
	ThreadLocalMap map = getMap(t);
	if (map != null)
		map.set(this, value);
	else
		createMap(t, value);
	return value;
}
```

### 2.4 `withInitial()`方法
初始化采用懒加载的方式，并不会立刻初始化，当第一次调用get方法时才会初始化。返回一个SuppliedThreadLocal对象储存初始化的值，当get的时候初花初始化。初始化时候重写initialValue方法，但ThreadLocal并没有实现直接返回null。
```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
	return new SuppliedThreadLocal<>(supplier);
}
```

### 2.5 `remove()`方法

找到key删除当前ThreadLocal的值，将引用设置为null防止内存泄漏，排查现有ThreadLocalMap清楚垃圾数据
```java
public void remove() {
	 ThreadLocalMap m = getMap(Thread.currentThread());
	 if (m != null)
		 // 调用ThreadLocalMap的remove方法
		 m.remove(this);
 }
 
 private void remove(ThreadLocal<?> key) {
	Entry[] tab = table;
	int len = tab.length;
	int i = key.threadLocalHashCode & (len-1);
	for (Entry e = tab[i];
		 e != null;
		 e = tab[i = nextIndex(i, len)]) {
		if (e.get() == key) {
			// 引用设置为null，方便gc识别
			e.clear();
			// 去除陈旧的对象键值对(相当于帮派清理门户，就是将没用的东西清理出去)
			expungeStaleEntry(i);
			return;
		}
	}
}

// 去除陈旧的对象键值对，如key为null的value等，防止内存泄漏
private int expungeStaleEntry(int staleSlot) {
	Entry[] tab = table;
	int len = tab.length;

	// expunge entry at staleSlot
	tab[staleSlot].value = null; // 当前的值置为null，key的话，在刚刚之前clear已经清除了
	tab[staleSlot] = null; // 将整个键值对清除
	size--; // 数量减一

	// Rehash until we encounter null 直到遇到null，然后rehash操作
	Entry e;
	int i;
	// 从当前的staleSlot后面的位置开始，直到遇到null为止
	for (i = nextIndex(staleSlot, len);
		 (e = tab[i]) != null;
		 i = nextIndex(i, len)) {
		 // 获取键对象，也就是map中的key对象
		ThreadLocal<?> k = e.get();
		// 如果为null,跟上面情况一样，直接清除值和整个entry，数量减一
		if (k == null) {
			e.value = null;
			tab[i] = null;
			size--;
		} else { // 如果不为null，表名当前key存在
			// 此时就是再哈希操作了
			int h = k.threadLocalHashCode & (len - 1);
			if (h != i) { // 如果不等的话，表明与之前的hash值不同这个元素需要重新存储
				tab[i] = null; // 将这个地方设置为null

				// Unlike Knuth 6.4 Algorithm R, we must scan until
				// null because multiple entries could have been stale.
				while (tab[h] != null) // 从当前h位置找一个为null的地方将当前元素放下
					h = nextIndex(h, len);
				tab[h] = e;
			}
		}
	}
	return i; // 返回的是第一个entry为null的下标
}
 ```

### 2.6 总结
通过上面这些内容，我们足以通过猜测得出结论：**最终的变量是放在了当前线程的 `ThreadLocalMap` 中，并不是存在 `ThreadLocal` 上，`ThreadLocal` 可以理解为只是`ThreadLocalMap`的封装，传递了变量值。** `ThrealLocal` 类中可以通过`Thread.currentThread()`获取到当前线程对象后，直接通过`getMap(Thread t)`可以访问到该线程的`ThreadLocalMap`对象。

**每个`Thread`中都具备一个`ThreadLocalMap`，而`ThreadLocalMap`可以存储以`每个ThreadLocal对象`为key（经过hashCode算出数组下标）的键值对。** 比如我们在同一个线程中声明了两个 `ThreadLocal` 对象的话，会使用 `Thread`内部都是使用仅有那个`ThreadLocalMap` 存放数据的，`ThreadLocalMap`的 key 就是 `ThreadLocal`对象，value 就是 `ThreadLocal` 对象调用`set`方法设置的值。`ThreadLocal` 是 map结构是为了让每个线程可以关联多个 `ThreadLocal`变量。这也就解释了 ThreadLocal 声明的变量为什么在每一个线程都有自己的专属本地变量。

`ThreadLocalMap`是`ThreadLocal`的静态内部类。

## 3. ThreadLocal 内存泄露问题

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用,而 value 是强引用。当线程终止时，Thread会被GC回收，所以ThreadLocal的value也会被回收，但是当我们使用线程池时，线程不会被GC回收，所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。这样一来，`ThreadLocalMap` 中就会出现key为null的Entry。假如我们不做任何措施的话，value 永远无法被GC 回收，这个时候就可能会产生内存泄露。ThreadLocalMap实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 等，方法的时候，会调用expungeStaleEntry方法清理掉 key 为 null 的记录。使用完 `ThreadLocal`方法后 最好手动调用`remove()`方法等避免内存泄漏。

```java
// extends WeakReference
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    // 强引用
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        // key为弱引用
        super(k);
        value = v;
    }
}
```

**弱引用介绍：**

> 如果一个对象只具有弱引用，那就类似于**可有可无的生活用品**。弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它 所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。
>
> 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。
