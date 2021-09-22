## 1. 总结
- 如果分配内存的时候 eden 区内存几乎已经被分配完了，`当 Eden 区没有足够空间进行分配时，虚拟机将发起一次 Minor GC.GC `期间虚拟机又发现 allocation 无法存入 Survivor 空间(因为From、To空间太小)，所以只好通过 分配担保机制 把新生代的对象提前转移到老年代中去，老年代上的空间足够存放 allocation，所以不会出现 Full GC。执行 Minor GC 后，后面分配的对象如果能够存在 eden 区的话，还是会在 eden 区分配内存。
- 内存分配是在JVM在内存分配的时候，`新生代内存不足时，把新生代的存活的对象搬到老生代，然后新生代腾出来的空间用于为分配给最新的对象。这里老生代是担保人`。在不同的GC机制下，也就是不同垃圾回收器组合下，担保机制也略有不同。`在Serial+Serial Old( -XX:+UseSerialGC)的情况下，发现放不下就直接启动担保机制；在Parallel Scavenge+Serial Old( -XX:+UseParallelGC)的情况下，却是先要去判断一下要分配的内存是不是>=Eden区大小的一半，如果是那么直接把该对象放入老生代，否则才会启动担保机制`。
- `在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间`。
	- 大于。进行Minor GC。
	- 小于。`虚拟机查看HandlePromotionFailure是否允许担保失败`。
		- 不允许。不进行Minor GC，进行一次Full GC。
		- 允许。`虚拟机查看老年代最大可用连续空间是否大于历届晋升到老年代对象的平均大小`。
			- 大于。进行一次Minor GC。
			- 小于。不进行Minor GC，进行一次Full GC。





## 2. 结论验证

**堆内存模型简单回顾：**
- 新生代与老年代的内存默认比值`–XX:NewRatio`为2，即新生代占堆空间的1/3，老年代占堆空间的2/3。
- 新生代由Eden、From SurvivorRatio、To SurvivorRatio三部分组成，默认`-XX:SurvivorRatio`为8，即Eden:一个Survivor=8:1，所以Eden占新生代的8/10，From SurvivorRatio与To SurvivorRatio各占1/10。

**上面所说如下图所示：**

![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/jvm/images/20200820221225498.png)

### 2.1 UseSerialGC（Serial + Serial Old的收集器组合进行内存回收）

**设置虚拟机参数：**
```shell
# 堆内存最小20M 最大20M 新生代内存10M 打印GC信息 Eden与一个Survivor比8:1 指定垃圾收集器为客户端模式下的Serial+Serial Old的收集器组合进行内存回收
-Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+UseSerialGC
```

![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/java/images/20200824123950879.png)

**测试代码：**
```java
public class GcTest {
    private static final int MB = 1024 * 1024;
    public static void main(String[] args)  {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * MB];
        allocation2 = new byte[2 * MB];
        allocation3 = new byte[2 * MB];
        allocation4 = new byte[4 * MB];
    }
}
```
**运行结果：**
```txt
[GC (Allocation Failure) [DefNew: 7331K->403K(9216K), 0.0035550 secs] 7331K->6547K(19456K), 0.0035783 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 4582K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  eden space 8192K,  51% used [0x00000007bec00000, 0x00000007bf014930, 0x00000007bf400000)
  from space 1024K,  39% used [0x00000007bf500000, 0x00000007bf564f98, 0x00000007bf600000)
  to   space 1024K,   0% used [0x00000007bf400000, 0x00000007bf400000, 0x00000007bf500000)
 tenured generation   total 10240K, used 6144K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
   the space 10240K,  60% used [0x00000007bf600000, 0x00000007bfc00030, 0x00000007bfc00200, 0x00000007c0000000)
 Metaspace       used 2726K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 289K, capacity 386K, committed 512K, reserved 1048576K
```

**结果解析：**

发生了一次Minor GC，触发的条件是Allocation Failure，将新生代内存从7331k变成了403k，分配allocation1、allocation2、allocation3如下图所示：

![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/jvm/images/20200824123821074.png)

Eden区可用空间为8192K(新生代10M * 8/10)，分配allocation1、allocation2、allocation3已经用去了6M，Eden区还剩2048K，新生代from区还剩1024K，所以分配allcation4时已经不够通了，进行一次Minor GC，由于allocation1、allocation2、allocation3引用依然存在，无法回收，Eden区的内存也无法放入Survivor区（可用内存只有from区的1M），所以JVM就启动了内存分配的担保机制，把这6MB的三个对象allocation1、allocation2、allocation3直接转移到了老年代。GC输入结果为tenured generation   total 10240K, used 6144K。

而新生代的Eden由于allocation1、allocation2、allocation3已经转移到老年代，所以有空间存储allocation4，所以新生代GC结果为def new generation   total 9216K, used 4582K，而Eden区内存使用为eden space 8192K,  51% used。

### 2.2 UseParallelGC（Parallel Scavenge + Serial Old(PS MarkSweep)的收集器组合进行内存回收）

**设置虚拟机参数：**
```shell
-Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8 -XX:+UseParallelGC
```

![image](https://raw.githubusercontent.com/future94/java-technology/master/java-base/jvm/images/20200824123950879-1.png)

**测试代码：**
```java
public class GcTest {
    private static final int MB = 1024 * 1024;
    public static void main(String[] args)  {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * MB];
        allocation2 = new byte[2 * MB];
        allocation3 = new byte[2 * MB];
        allocation4 = new byte[4 * MB];
    }
}
```
**运行结果：**
```txt
Heap
 PSYoungGen      total 9216K, used 7495K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 8192K, 91% used [0x00000007bf600000,0x00000007bfd51f90,0x00000007bfe00000)
  from space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
  to   space 1024K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007bff00000)
 ParOldGen       total 10240K, used 4096K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  object space 10240K, 40% used [0x00000007bec00000,0x00000007bf000010,0x00000007bf600000)
 Metaspace       used 2727K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 289K, capacity 386K, committed 512K, reserved 1048576K
```

**结果解析：**
Eden区可用空间为8192K(新生代10M * 8/10)，分配allocation1、allocation2、allocation3已经用去了6M，Eden区还剩2048K，新生代from区还剩1024K，所以分配allcation4时已经不够通了，这时在GC前判断要分配的allcation4（4M） >= Eden内存容量的一半（8M / 2 = 4M）。所以直接将allocation4分配到老年代，老年代结果为ParOldGen       total 10240K, used 4096K。新生代结果为PSYoungGen      total 9216K, used 7495K。Eden区为eden space 8192K, 91% used。

**测试代码：**
```java
public class GcTest {
    private static final int MB = 1024 * 1024;
    public static void main(String[] args)  {
        byte[] allocation1, allocation2, allocation3, allocation4;
        allocation1 = new byte[2 * MB];
        allocation2 = new byte[2 * MB];
        allocation3 = new byte[2 * MB];
        allocation4 = new byte[3 * MB];
    }
}
```
**运行结果：**
```txt
[GC (Allocation Failure) [PSYoungGen: 7331K->640K(9216K)] 7331K->6792K(19456K), 0.0036830 secs] [Times: user=0.04 sys=0.00, real=0.01 secs] 
[Full GC (Ergonomics) [PSYoungGen: 640K->0K(9216K)] [ParOldGen: 6152K->6546K(10240K)] 6792K->6546K(19456K), [Metaspace: 2720K->2720K(1056768K)], 0.0042511 secs] [Times: user=0.03 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 9216K, used 3154K [0x00000007bf600000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 8192K, 38% used [0x00000007bf600000,0x00000007bf914930,0x00000007bfe00000)
  from space 1024K, 0% used [0x00000007bfe00000,0x00000007bfe00000,0x00000007bff00000)
  to   space 1024K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007c0000000)
 ParOldGen       total 10240K, used 6546K [0x00000007bec00000, 0x00000007bf600000, 0x00000007bf600000)
  object space 10240K, 63% used [0x00000007bec00000,0x00000007bf264bb8,0x00000007bf600000)
 Metaspace       used 2727K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 289K, capacity 386K, committed 512K, reserved 1048576K
```

**结果解析：**
Eden区可用空间为8192K(新生代10M * 8/10)，分配allocation1、allocation2、allocation3已经用去了6M，Eden区还剩2048K，新生代from区还剩1024K，所以分配allcation4时已经不够通了，这时在GC前判断要分配的allcation4（3M） < Eden内存容量的一半（8M / 2 = 4M）。所以进行一次Minor GC，由于allocation1、allocation2、allocation3引用依然存在，无法回收，Eden区的内存也无法放入Survivor区（可用内存只有from区的1M），所以JVM就启动了内存分配的担保机制，把这6MB的三个对象allocation1、allocation2、allocation3直接转移到了老年代，allocation4分配在Eden区。所以GC结果为[GC (Allocation Failure) [PSYoungGen: 7331K->640K(9216K)]。此时老年代的内存使用结果为ParOldGen       total 10240K, used 6546K 。新生代结果为PSYoungGen      total 9216K, used 3154K。Eden区结果为eden space 8192K, 38% used。

在发生Minor GC之前，虚拟机会先检查老年代最大可用的连续空间是否大于新生代所有对象总空间
- 大于。进行Minor GC。
- 小于。虚拟机查看HandlePromotionFailure是否允许担保失败。
	- 允许。虚拟机查看老年代最大可用连续空间是否大于历届晋升到老年代对象的平均大小。
		- 大于。进行一次Minor GC。
		- 小于。不进行Minor GC，进行一次Full GC。
	- 不允许。不进行Minor GC，进行一次Full GC。