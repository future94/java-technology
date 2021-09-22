

## 1. 执行流程

先创建核心线程，当核心线程不够时放入阻塞队列中等待核心线程执行，当阻塞队列满时创建非核心线程，当非核心线程达到最大时候，触发阻塞策略。

## 2. 参数

```java
ExecutorService executorService = new ThreadPoolExecutor(
        int corePoolSize,
        int maximumPoolSize,
        long keepAliveTime,
        TimeUnit unit,
        BlockingQueue<Runnable> workQueue,
        ThreadFactory threadFactory,
        RejectedExecutionHandler handler);
```

参数说明：
- **corePoolSize** ：核心线程数，一旦创建将不会再释放。如果创建的线程数还没有达到指定的核心线程数量，将会继续创建新的核心线程，直到达到最大核心线程数后，核心线程数将不在增加；如果没有空闲的核心线程，同时又未达到最大线程数，则将继续创建非核心线程；如果核心线程数等于最大线程数，则当核心线程都处于激活状态时，任务将被挂起，等待空闲线程来执行。
- **maximumPoolSize** ：最大线程数，允许创建的最大线程数量。如果最大线程数等于核心线程数，则无法创建非核心线程；如果非核心线程处于空闲时，超过设置的空闲时间，则将被回收，释放占用的资源。
- **keepAliveTime** ：也就是当线程空闲时，所允许保存的最大时间，超过这个时间，线程将被释放销毁，但只针对于非核心线程。
- **unit** ：时间单位。
- **wordQueue** ：任务队列，存储暂时无法执行的任务，等待空闲的核心线程来执行，如果超过队列数量，创建空闲线程来执行任务。
- **threadFactory** ：线程工厂，用于创建线程。
- **handler** ：拒绝策略。当线程边界和队列容量已经达到最大时，用于处理阻塞时的程序。

## 3. BlockingQueue阻塞队列（7种）

- **SynchronousQueue** ：同步阻塞队列。该线程池中没有容量。如Cache线程池(核心为0，最大线程为int最大)用的就是这个，由上面流程我们知道，来任务时，没有核心线程，队列存不下，直接创建非核心线程执行。
- **ArrayBlockingQueue** ：数组实现的有界队列。初始化时就要设置好容量，默认<font color="red">生产和消费使用的是同一把锁</font>。使用一把 `ReentrantLock` + 两个`Condition` 实现。
- **LinkedBlockingQueue** ：链表实现的队列。默认最大值为 `Integer.MAX_VALUE`，可以指定容量。默认<font color="red">生产和消费使用的是两把把锁</font>。使用一两把 `ReentrantLock` + `Condition` 实现。
- **LinkedBlockingDeque** ：链表实现的双向队列。默认<font color="red">生产和消费使用的是同一把锁</font>。使用一把 `ReentrantLock` + 两个`Condition` 实现。
- **PriorityBlockingQueue** ：可以指定优先级数组实现的有界队列。默认大小为11，最大值为 `Integer.MAX_VALUE - 8`。<font color="red">此队列指保证优先级最高的在队列头部，并不保证相同元素的位置，也不保证除了最高优先级之外元素的排序</font>。
- **LinkedTransferQueue** ：链表实现的生产／消费模式队列。实现了 `TransferQueue` 接口，内部无锁。源码分析[并发编程—LinkedTransferQueue](https://www.jianshu.com/p/ae6977886cec)
- **DelayQueue** ：支持延时获取元素的无界阻塞队列。队列使用PriorityQueue来实现。队列中的元素必须实现Delayed接口，在创建元素时可以指定多久才能从队列中获取当前元素。只有在延迟期满时才能从队列中提取元素。我们可以将DelayQueue运用在以下应用场景：
	- 缓存系统的设计：可以用DelayQueue保存缓存元素的有效期，使用一个线程循环查询DelayQueue，一旦能从DelayQueue中获取元素时，表示缓存有效期到了。
	- 定时任务调度。使用DelayQueue保存当天将会执行的任务和执行时间，一旦从DelayQueue中获取到任务就开始执行，从比如TimerQueue就是使用DelayQueue实现的。

## 4. 拒绝策略

- ThreadPoolExecutor.AbortPolicy ：直接抛出异常。
- ThreadPoolExecutor.CallerRunsPolicy ：如果线程池未关闭，由调用者执行。
- ThreadPoolExecutor.DiscardPolicy ：直接忽略掉
- ThreadPoolExecutor.DiscardOldestPolicy ：丢弃最老的任务，然后加入到线程池中。