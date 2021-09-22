**synchronized发生异常会自动释放锁**

在 Java 早期版本中，synchronized属于重量级锁，效率低下，因为监视器锁（monitor）是依赖于底层的操作系统的 Mutex Lock 来实现的，Java 的线程是映射到操作系统的原生线程之上的互斥锁来实现的。如果要挂起或者唤醒一个线程，都需要操作系统帮忙完成，而操作系统实现线程之间的切换时需要从用户态转换到内核态，这个状态之间的转换需要相对比较长的时间，时间成本相对较高，这也是为什么早期的 synchronized 效率低的原因。庆幸的是在 Java 6 之后 Java 官方对从 JVM 层面对synchronized 较大优化，所以现在的 synchronized 锁效率也优化得很不错了。JDK1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。

## 1. 使用方式

- **修饰实例方法方法** ：作用于当前对象实例加锁，进入同步代码前要获得当前对象实例的锁。synchronized(this)。
- **修饰静态方法** ：也就是给当前类加锁，会作用于类的所有对象实例。
	因为静态成员不属于任何一个实例对象，是类成员（ static 表明这是该类的一个静态资源，不管new了多少个对象，只有一份）。所以如果一个线程A调用一个实例对象的非静态 synchronized 方法，而线程B需要调用这个实例对象所属类的静态 synchronized 方法，是允许的，不会发生互斥现象，因为访问静态 synchronized 方法占用的锁是当前类的锁，而访问非静态 synchronized 方法占用的锁是当前实例对象锁。
- **修饰代码块** ：指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。最好别用字符串，因为JVM中，字符串常量池具有缓存功能。


## 2. 双重校验锁实现对象单例（线程安全）：

通过volatile保证线程的可见性并防止指令的重新排序，synchronized加锁防止重复实例化。

```java
public class Singleton {

    private volatile static Singleton uniqueInstance;

    private Singleton() {
    }

    public static Singleton getUniqueInstance() {
       //先判断对象是否已经实例过，没有实例化过才进入加锁代码
        if (uniqueInstance == null) {
            //类对象加锁
            synchronized (Singleton.class) {
                if (uniqueInstance == null) {
                    uniqueInstance = new Singleton();
                }
            }
        }
        return uniqueInstance;
    }
}
```

## 3. synchronized 关键字的底层原理

```java
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("synchronized 代码块");
        }
    }

    public synchronized void synchronizedMethod() {
        System.out.println("synchronized 方法");
    }
}
```

通过 JDK 自带的 javap 命令查看 SynchronizedDemo 类的相关字节码信息：首先切换到类的对应目录执行 `javac SynchronizedDemo.java` 命令生成编译后的 .class 文件，然后执行`javap -c -s -v -l SynchronizedDemo.class。`

synchronized代码块：  

![file](https://raw.githubusercontent.com/future94/java-technology/master/java-base/java/images/20200229174919747.png)

synchronized方法：  

![file](https://raw.githubusercontent.com/future94/java-technology/master/java-base/java/images/20200229174941356.png)

### 3.1 synchronized代码块

**synchronized 同步语句块的实现使用的是 monitorenter 和 monitorexit 指令，其中 monitorenter 指令指向同步代码块的开始位置，monitorexit 指令则指明同步代码块的结束位置** 。 

当执行 monitorenter 指令时，线程试图获取锁也就是获取 monitor(**monitor对象存在于每个Java对象的对象头中**，synchronized 锁便是通过这种方式获取锁的，也是为什么Java中任意对象可以作为锁的原因) 的持有权。当计数器为0则可以成功获取，获取后将锁计数器设为1也就是加1。相应的在执行 monitorexit 指令后，将锁计数器设为0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

#### monitor原理及执行步骤

<font color="red">因为每个对象头中都存在monitor锁对象，所以java中可以用对象加锁。</font>

对象结构大概分为对象头、实例变量和填充字节。
- 对象头
	- Mark Word（标记字段）：存储对象的hashCode、锁信息或分代年龄或GC标志等信息
	- Klass Pointer（类型指针）：对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例
	- 数组长度：只有数组对象保存了这部分数据。该数据在32位和64位JVM中长度都是32bit。
- 实例变量：实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的，还是在子类中定义的，都需要记录起来。
- 填充字节：第三部分对齐填充并不是必然存在的，也没有特别的含义，它仅仅起着占位符的作用。由于HotSpot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说，就是对象的大小必须是8字节的整数倍。而对象头部分正好是8字节的倍数（1倍或者2倍），因此，当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。

每一个锁都对应一个monitor对象，在HotSpot虚拟机中它是由ObjectMonitor实现的（C++实现）。每个对象都存在着一个monitor与之关联，对象与其monitor之间的关系有存在多种实现方式，查看部分ObjectMonitor.cpp源码：

```c++
ObjectMonitor () {
	// 指向持有ObjectMonitor对象的线程地址。
	_owner = NULL;

	//  存放调用wait方法，而进入等待状态的线程的队列。
	_WaitSet = NULL

	// 这里是等待锁block状态的线程的队列。
	_EntryList = NULL;

	// 锁的重入次数。
	_recursions = 0;

	// 线程获取锁的次数。
	_count = 0;
}
```

执行步骤：

1. 当多个线程同时访问时，这些线程先被放进 **_EntryList队列** 中，线程处于blocked状态。
2. 当其中的一个线程获取到对象的monitor后，该线程进入running状态， **_owner指向当前线程，_count加1**表示锁被一个线程获取到。
3. 当running状态线程调用wait()方法，释放当前线程的monitor对，并进入waiting状态。**_owner变为NULL，_count减1，将线程加入都_WaitSet队列中，当有线程notify()该线程时，将该线程加入到_EntryList队列中参与锁的竞争** 。
4. 当线程执行完成时，**释放monitor对象，\_owner为NULL，_count减1** 。

更多详细请看[JVM底层又是如何实现synchronized的](https://www.open-open.com/lib/view/open1352431526366.html)

### 3.2 synchronized方法

方法头部使用的是 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

## 4. JDK1.6之后synchronized优化

### 4.1 锁的四种状态（无锁状态、偏向锁、轻量级锁、重量级锁）
JDK6之前只有两个状态：无锁、有锁（重量级锁），而在JDK6之后对synchronized进行了优化，新增了两种状态，总共就是四个状态：无锁状态、偏向锁、轻量级锁、重量级锁。

### 4.2 锁膨胀
上面讲到锁有四种状态，并且会因实际情况进行膨胀升级，其膨胀方向是：无锁——>偏向锁——>轻量级锁——>重量级锁，并且膨胀方向不可逆。

#### 4.2.1 偏向锁
情况：<font color="red">在大多数情况下，锁不存在多线程竞争，总是由同一线程多次获得，那么此时引入偏向锁。</font>一句话总结它的作用：**减少同一线程获取锁的代价**。

**偏向锁原理与升级过程：**
- 当线程1访问代码块并获取锁对象时，会在java对象头和栈帧中记录偏向锁的ThreadID。
- 因为**偏向锁不会主动释放锁**，因此以后线程1再次获取锁时会比较当前线程的ThreadID与jva对象头中的ThreadID是否相等
    - 相等就直接执行，无需CAS来加锁解锁
    - 如果不相等（其他线程需要获取锁对象，如线程2），查看线程1是否存活。
        - 不存活则锁对象被重置为无锁状态，其他线程（线程2）可以竞争将其设置为偏向锁。
        - 存活状态下则立刻查找该线程（线程1）的栈帧信息
            - 如果需要继续持有这个锁对象，那么会**暂停该线程（线程1），撤销偏向锁，升级为轻量级锁进行后续操作**。
            - 如果不再需要这个锁对象，那么将锁对象设置为无锁状态，重新进行偏向锁竞争。

#### 4.2.2 轻量级锁
情况：<font color="red">当竞争锁对象的线程不多，并且线程持有锁的时间也不长时，那么此时引入轻量级锁。</font>轻量级锁是由偏向锁升级而来，当存在第二个线程申请同一个锁对象时，偏向锁就会立即升级为轻量级锁。注意这里的第二个线程只是申请锁，不存在两个线程同时竞争锁，可以是一前一后地交替执行同步块。

轻量级锁：<font color="red">由于阻塞线程需要CPU从用户状态转到内核状态代价比较大，如果刚线程阻塞这个锁就被释放了时候代价太大，所以这个时候不会阻塞线程，而是通过CAS操作让它自旋等待锁对象的释放。</font>

**轻量级锁原理与升级过程：**
线程1获取轻量级锁时会把锁对象的对象头MarkWord复制一份到线程1的栈帧中创建的用于存储锁记录的空间（DisplacedMarkWord），然后使用CAS把对象中的内存替换为线程1存储的锁记录（DisplacedMarkWord）的地址。如果在线程1复制对象头的同时（在线程1CAS之前），线程2也准备获取锁，复制了对象头到线程2的锁记录空间中，但是在线程2CAS的时候发现线程1已经把对象头换了，线程2的CAS失败，那么线程2就尝试使用自旋锁来等在线程1释放锁。

升级为重量级锁：<font color="red">长时间自旋会导致CPU消耗，达到一定自旋次数还没有释放锁或者线程1还在执行线程2还在自旋，这时候又有线程3来竞争这个对象争锁，此时轻量级锁会升级为重量级锁。</font>

#### 4.2.3 重量级锁
情况：<font color="red">当多个线程同时在竞争锁对象时，那么此时引入重量级锁</font>。

重量级锁：<font color="red">阻塞所有等待竞争的线程，防止CPU空转，阻塞等待线程1释放锁后进入无锁状态重新竞争。</font>

### 4.3 锁消除
虚拟机的运行时编译器在运行时如果检测到一些要求同步的代码上不可能发生共享数据竞争，则会去掉这些锁。

如下面情况就不需要加锁：
```java
public void method1() {
    Object o = new Object();
    synchronized (o) {
        System.out.println("锁消除之前的代码");
    }
}

public void method2() {
    Object o = new Object();
    System.out.println("锁消除之前的代码");
}
```

### 4.4 锁粗化
将临近的可公用的代码块用同一个锁合并起来。

```java
public void method1() {
    synchronized (SynchronizedDemo.class) {
        System.out.println("锁粗化之前的代码1");
    }

    synchronized (SynchronizedDemo.class) {
        System.out.println("锁粗化之前的代码2");
    }

    System.out.println("锁");

}

public void method2() {
    synchronized (SynchronizedDemo.class) {
        System.out.println("锁粗化之后的代码1");
        System.out.println("锁粗化之后的代码2");
    }

    System.out.println("锁");
}
```

### 4.5 自旋锁与自适应自旋锁

轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。

**自旋锁：** 许多情况下，共享数据的锁定状态持续时间较短，切换线程不值得，通过让线程执行循环等待锁的释放，不让出CPU。如果得到锁，就顺利进入临界区。如果还不能获得锁，那就会将线程在操作系统层面挂起，这就是自旋锁的优化方式。但是它也存在缺点：如果锁被其他线程长时间占用，一直不释放CPU，会带来许多的性能开销。

**自适应自旋锁：** 这种相当于是对上面自旋锁优化方式的进一步优化，它的自旋的次数不再固定，其自旋的次数由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定，这就解决了自旋锁带来的缺点。


## 5. synchronized与lock的区别

| 对比项 | synchronized | lock |
---|---|---
实现层面 | JVM | JUC
锁释放 | jvm执行完或者异常会自动释放 | 手动调用unlock方法释放
锁获取 | 无限等待 | 提供多种方式获取锁
锁状态 | 获取不到 | 可获取
锁类型 | 可重入锁、非公平锁、不可中断 | 可以根据实现接口订制
线程调度 | 基于Object类中的wait()、notify() | 基于condition
