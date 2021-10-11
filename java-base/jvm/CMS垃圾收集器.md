
## 1. 简介

CMS是老年代垃圾收集器，在收集过程中可以与用户线程并发操作。它可以与Serial收集器和Parallel New收集器搭配使用。CMS牺牲了系统的吞吐量来追求收集速度，适合追求垃圾收集速度的服务器上。可以通过JVM启动参数：`-XX:+UseConcMarkSweepGC`来开启CMS。

## 2. 收集7个阶段

1. 初始标记(CMS-initial-mark) ,会导致stw;
2. 并发标记(CMS-concurrent-mark)，与用户线程同时运行；
3. 预清理（CMS-concurrent-preclean），与用户线程同时运行；
4. 可被终止的预清理（CMS-concurrent-abortable-preclean） 与用户线程同时运行；
5. 重新标记(CMS-remark) ，会导致swt；
6. 并发清除(CMS-concurrent-sweep)，与用户线程同时运行；
7. 并发重置状态等待下次CMS的触发(CMS-concurrent-reset)，与用户线程同时运行；

**图解说明：**
- 白色对象，表示自身未被标记；
- 灰色对象，表示自身被标记，但内部引用未被处理；
- 黑色对象，表示自身被标记，内部引用都被处理；

**发生GC之前java堆的对象如下：**
![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/jvm/images/dhiuhi12321312sdad.png)

### 2.1 初始标记

初始标记仅仅只是标记一下GC Roots能直接关联到的对象，速度很快。

**初始标记做两件事：**
1. 标记GC Roots能关联到的老年代对象。
2. 遍历新生代对象，标记关联到的老年代对象。

**初始标记之后java堆的对象如下：**
![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/jvm/images/jiu2389u912jdsad.png)

**在Java语言里，可作为GC Roots对象的包括如下几种：**
- 虚拟机栈(栈桢中的本地变量表)中的引用的对象
- 方法区中的类静态属性引用的对象
- 方法区中的常量引用的对象
- 本地方法栈中JNI的引用的对象

为了加快此阶段处理速度，减少停顿时间，可以开启初始标记并行化 `-XX:+CMSParallelInitialMarkEnabled`，同时调大并行标记的线程数，线程数不要超过cpu的核数。

### 2.2 并发标记

遍历初始标记中存活的对象，继续递归标记这些对象的可达对象。

因为该阶段并发执行的，在运行期间可能发生新生代的对象晋升到老年代、或者是直接在老年代分配对象、或者更新老年代对象的引用关系等等，对于这些对象，都是需要进行重新标记的，否则有些对象就会被遗漏，发生漏标的情况。

为了提高重新标记的效率，该阶段会把上述对象所在的Card标识为Dirty，后续只需扫描这些Dirty Card的对象，避免扫描整个老年代。

下图中绿色就是变化的对象，Card标识为Dirty。  
![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/jvm/images/ji8789123128uw9dauisoj.png)

### 2.3 预清理

通过参数`CMSPrecleaningEnabled`选择关闭该阶段，默认启用，主要做两件事情：

1. 处理新生代已经发现的引用，比如在并发阶段，在Eden区中分配了一个A对象，A对象引用了一个老年代对象B（这个B之前没有被标记），在这个阶段就会标记对象B为活跃对象。如2.2图中的右侧绿线。
2. 在并发标记阶段，如果老年代中有对象内部引用发生变化，会把所在的Card标记为Dirty（其实这里并非使用CardTable，而是一个类似的数据结构，叫ModUnionTalble），通过扫描这些Table，重新标记那些在并发标记阶段引用被更新的对象（晋升到老年代的对象、原本就在老年代的对象）。如2.2图中的引用变化。

### 2.4 可终止的预清理

该阶段发生的前提是，新生代Eden区的内存使用量大于参数`CMSScheduleRemarkEdenSizeThreshold` 默认是2M，如果新生代的对象太少，就没有必要执行该阶段，直接执行重新标记阶段。

**为什么需要这个阶段，存在的价值是什么？**

因为CMS GC的终极目标是降低垃圾回收时的暂停时间，所以在该阶段要尽最大的努力去处理那些在并发阶段被应用线程更新的老年代对象，这样在暂停的重新标记阶段就可以少处理一些，暂停时间也会相应的降低。<font color="red">核心就是提前处理一些对象，让重新标记阶段可以少处理一些；如果这时候发生了Young GC，也可以减少重新标记的时间</font>。可以加入参数`-XX:+CMSScavengeBeforeRemark`，在重新标记之前，先执行一次ygc，回收掉年轻带的对象无用的对象，并将对象放入幸存带或晋升到老年代，这样再进行年轻带扫描时，只需要扫描幸存区的对象即可，一般幸存带非常小，这大大减少了扫描时间。

**在该阶段，主要循环的做两件事：**

1. 处理 From 和 To 区的对象，标记可达的老年代对象
2. 和上一个阶段一样，扫描处理Dirty Card中的对象

**当然了，这个逻辑不会一直循环下去，打断这个循环的条件有三个：**

1. 可以设置最多循环的次数 `CMSMaxAbortablePrecleanLoops`，默认是0，意思没有循环次数的限制。
2. 如果执行这个逻辑的时间达到了阈值`CMSMaxAbortablePrecleanTime`，默认是5s，会退出循环。
3. 如果新生代Eden区的内存使用率达到了阈值`CMSScheduleRemarkEdenPenetration`，默认50%，会退出循环。（这个条件能够成立的前提是，在进行Precleaning时，Eden区的使用率小于十分之一）

如果执行这个逻辑的时候发生了Young GC，那么对于后面的重新标记阶段来说，大大减轻了扫描年轻代的负担，但是发生Young GC并非人为控制，所以只能祈祷这5s内可以来一次Young GC。可以加入参数`-XX:+CMSScavengeBeforeRemark`，在重新标记之前，先执行一次ygc，回收掉年轻带的对象无用的对象，并将对象放入幸存带或晋升到老年代，这样再进行年轻带扫描时，只需要扫描幸存区的对象即可，一般幸存带非常小，这大大减少了扫描时间。不过，这种参数有利有弊，利是降低了Remark阶段的停顿时间，弊的是在新生代对象很少的情况下也多了一次YGC，最可怜的是在AbortablePreclean阶段已经发生了一次YGC，然后在该阶段又傻傻的触发一次。

```txt
...
1678.150: [CMS-concurrent-preclean-start]
1678.186: [CMS-concurrent-preclean: 0.044/0.055 secs]
1678.186: [CMS-concurrent-abortable-preclean-start]
1678.365: [GC 1678.465: [ParNew: 2080530K->1464K(2044544K), 0.0127340 secs] 
1389293K->306572K(2093120K), 
0.0167509 secs]
1680.093: [CMS-concurrent-abortable-preclean: 1.052/1.907 secs]  
....
```

在上面GC日志中，1678.186启动了可终止的预清理阶段，在随后不到2s就发生了一次YGC。

### 2.5 重新标记

由于上述阶段是并发执行的，所以在运行期间可能产生了新的引用关系。

1. 老年代的新对象被GC Roots引用
2. 老年代的未标记对象被新生代对象引用
3. 老年代已标记的对象增加新引用指向老年代其它对象
4. 新生代对象指向老年代引用被删除
5. 。。。

上述对象中可能有一些已经在预清理阶段和可中断的预清理阶段被处理过，但总存在没来得及处理的，所以还有进行如下的处理：

1. 遍历新生代对象，重新标记
2. 根据GC Roots，重新标记
3. 遍历老年代的Dirty Card，重新标记，这里的Dirty Card大部分已经在预清理阶段处理过。

### 2.6 并发清理

通过以上5个阶段的标记，老年代所有存活的对象已经被标记并且现在要通过Garbage Collector采用清扫的方式回收那些不能用的对象了。**这个阶段主要是清除那些没有标记的对象并且回收空间**。
remark过程标记活着的对象，从GCRoot的可达性判断对象活着，但无法标记“死亡”的对象。如果在初始标记阶段被标记为活着，并发运行过程中“死亡”，remark过程无法纠正，因此变为浮动垃圾，需等待下次gc的到来。这一部分垃圾就称为`浮动垃圾`。


### 2.7 并发重置状态

并发重置对象状态，等待下次CMS的触发。

## 3. 相关问题

### 3.1 如何减少重新标记停顿时间

一般CMS的GC耗时80%都在remark阶段，如果发现remark阶段停顿时间很长，可以尝试添加该参数：`-XX:+CMSScavengeBeforeRemark`。在执行remark操作之前先做一次Young GC，目的在于减少年轻代对老年代的无效引用，降低remark时的开销。

### 3.2 内存碎片化

CMS是基于标记-清除算法的，CMS只会删除无用对象，不会对内存做压缩，会造成内存碎片，这时候我们需要用到这个参数：`-XX:CMSFullGCsBeforeCompaction=n`。意思是说在上一次CMS并发GC执行过后，到底还要再执行多少次full GC才会做压缩。默认是0，也就是在默认配置下每次CMS GC进行full GC的时候都会做压缩。 如果把CMSFullGCsBeforeCompaction配置为10，就会让上面说的第一个条件变成每隔10次真正的full GC才做一次压缩。

### 3.3 concurrent mode failure

在CMS GC进行时，用户线程也在并发执行。当年轻代满了进行Young GC时，需要老年代进行内存担保，这时候老年代也放不下时；或者做Minor GC的时候，新生代救助空间放不下，需要放入老年代，而老年代也放不下而产生的。

**设置cms触发时机有两个参数：**

1. `-XX:+UseCMSInitiatingOccupancyOnly`:不需要垃圾收集器自动调节。如果不指定, 只是用设定的回收阈值CMSInitiatingOccupancyFraction,则JVM仅在第一次使用设定值,后续则自动调整会导致上面的那个参数不起作用。
2. `-XX:CMSInitiatingOccupancyFraction=70`:是指设定CMS在对内存占用率达到70%的时候开始GC。

**解决办法：**
- 降低CMSInitiatingOccupancyFraction值，让其尽早进行GC保证有足够的空间。也不能太小，这样就会频繁触发gc。

### 3.4 promotion failed

在进行Minor GC时，Survivor Space放不下，对象只能放入老年代，而此时老年代也放不下造成的，多数是由于老年带有足够的空闲空间，但是由于碎片较多，新生代要转移到老年带的对象比较大,找不到一段连续区域存放这个对象导致的。

**解决办法：**
- 因为内存碎片导致的大对象提升失败，cms需要进行空间整理压缩，设置CMSFullGCsBeforeCompaction参数，对碎片空间进行压缩。
- 因为提升过快导致的，说明Survivor 空闲空间不足，那么可以尝试调大 Survivor。
- 因为老年代空间不够导致的，尝试将CMS触发的阈值调低。

### 3.5 优缺点

**优点：**
- 并发高，停顿时间少。

**缺点：**
- CPU比较敏感。
- 内存碎片化。
- 只能收集老年代。
- 无法清理浮动垃圾



参考文章：
- [CMS垃圾收集器](https://www.jianshu.com/p/86e358afdf17)
- [图解CMS垃圾回收机制，你值得拥有](https://www.jianshu.com/p/2a1b2f17d3e4)