
[1. 从网卡接收数据说起](#1-从网卡接收数据说起)
[2. 如何知道接收了数据](#2-如何知道接收了数据)
[3. 进程阻塞为什么不占用cpu资源](#3-进程阻塞为什么不占用cpu资源)
[4. 内核接收网络数据全过程](#4-内核接收网络数据全过程)
[5. 同时监视多个socket的简单方法](#5-同时监视多个socket的简单方法)
[6. epoll的设计思路](#6-epoll的设计思路)
[7. epoll的原理和流程](#7-epoll的原理和流程)
[8. epoll的实现细节](#8-epoll的实现细节)
[9. 结论](#9-结论)

## 1. 从网卡接收数据说起

从**硬件角度**看计算机是如何接受网络数据的。具体很复杂(其实我也不知道)，我们只需要知道<font color="red">计算机将网卡的数据写入到内存之中让操作系统可以读取到它</font>即可。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-e549406135abf440331de9dd8c3925e9_1440w.jpg)

## 2. 如何知道接收了数据

从**CPU角度**看计算机是如何接受到网络数据的。要理解这个过程，我们要先了解一个概念 `中断` 。

**什么是中断？**  
计算机执行程序是有优先级的，比如计算机收到断电信号、键盘数据、网卡收到数据等等，他们的优先级非常的高，需要CPU立刻对其作出相应，这时候CPU就需要中断正在执行的程序，这个过程就是**中断**。

所以接着上面的网卡接收数据说起，网卡收到了数据并写入内存中，这时候网卡会向CPU发送中断信号，CPU就知道网卡有数据来了，从而执行网卡的中断程序处理数据。

## 3. 进程阻塞为什么不占用cpu资源

从**操作系统角度**看计算机是如何接收到网络数据的。要理解这个过程，我们要先了解 `操作系统进程调度` ，阻塞是进程调度的关键一环。`阻塞` 是指进程在等待某事件发生之前的等待状态。下面是简单的socket的recv阻塞等待数据的代码：

```c
//创建socket
int s = socket(AF_INET, SOCK_STREAM, 0);
//绑定
bind(s, ...);
//监听
listen(s, ...);
//接受客户端连接
int c = accept(s, ...);
//接收客户端数据
recv(c, ...);
//将数据打印出来
printf(...);
```

### 阻塞的原理

**工作队列**

操作系统为了支持多任务，实现了进程调度的功能，会把进程分为 `运行` 、`就绪` 和 `等待` 等状态。运行状态是进程获得cpu使用权，正在执行代码的状态；就绪就是已经准备好了，等待分配CPU时间片；等待状态是阻塞状态，比如上述程序运行到recv时，程序会从运行状态变为等待状态，接收到数据后又变回运行状态。操作系统会分时执行各个运行状态的进程，由于速度很快，看上去就像是同时执行多个任务。

如下图，下面有3个进程，启动之后3个线程都会进入就绪状态放入工作队列中等待分配CPU时间片。进程A中运行的就是我们的代码（**整个文章我们都忽略线程的概念**）。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-2f3b71710f1805669a780a2d634f0626_1440w.jpg)

**等待队列**

当进程A运行到create_socket时候，操作系统会创建一个由文件系统管理（fd句柄）的Socket对象。该Socket对象由 `发送缓冲区`、`接收缓冲区`和`等待队列`等构成。如下图所示：

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-7ce207c92c9dd7085fb7b823e2aa5872_1440w.jpg)

当进程A运行到recv时候，操作系统将进程A加入到该Socket的等待队列中（实际上是将进程A的引用加入，不是将进程放进去了）。由于进程A放入了等待队列中，不需要在分配CPU时间片，所以阻塞不占用CPU资源。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-1c7a96c8da16f123388e46f88772e6d8_1440w.jpg)


**唤醒线程**

当Socket接收缓冲区收到数据时，就会唤醒等待队列中的进程，这时候进程A就可以继续执行了，由于缓冲区已经有数据了，所以recv可以接收到数据。

## 4. 内核接收网络数据全过程

1. 进程已经调用了recv阻塞等待数据，这时候网络传输数据到网卡。
2. 网卡接收到数据，将数据写入到内存之后等待CPU获取。
3. 网卡发送中断信号给CPU，CPU接收到信号之后执行中断程序（4和5都是中断程序的一部分）
4. 将网卡数据写入Socket缓冲区。
5. 唤醒等待队列中的进程A。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-696b131cae434f2a0b5ab4d6353864af_1440w.jpg)

6. 进程A结束阻塞被唤醒，在工作队列中恢复可以重新获得CPU时间片的资格后，接收到recv方法返回的数据。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-3e1d0a82cdc86f03343994f48d938922_1440w.jpg)

问题：
- 操作系统怎么知道哪个Socket有数据了？就是port。
- 一个端口操作系统怎么监听多个Socket发送的数据？就是多路复用。

## 5. 同时监视多个socket的简单方法


### 5.1 select
服务端需要监听多个客户端发来的数据，而recv只能接受到一个socket来的数据，那么我们怎么接收到多个Socket呢，其实就是epoll。而epoll是演化的结果，最开始采用的是 `select模式`。维护一个socket列表（fd_set数组），如果列表中没有数据，则挂起进程，如果列表中有一个socket准备好数据，那么就换行线程。

创建socket，并将所有的socket放到数组中维护，调用select监听这些额socket，当没有数据来时select会阻塞，当有socket数据准备好的时候，我们循环监听的socket列表得到对应socket的数据。select伪代码如下：
```c
//创建socket
int s = socket(AF_INET, SOCK_STREAM, 0);
//绑定
bind(s, ...);
//监听
listen(s, ...);
// 存放需要监听的socket
int fds[] = {s1, s2, ...};
while(true){
    int n = select(..., fds, ...)
    for(int i=0; i < fds.count; i++){
        if(FD_ISSET(fds[i], ...)){
            //fds[i]的数据处理
        }
    }
}
```

> 当调用select的时候，内核会先遍历一遍select看有无数据，有就不阻塞直接返回，所以select返回可能大于1，说明有多个Socket数据准备好了。

**流程如下图**

我们创建了3个socket，调用select之后，将进程A加入到3个socket等待队列中。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-0cccb4976f8f2c2f8107f2b3a5bc46b3_1440w.jpg)

这时候当Socket收到数据时，触发中断程序换行进程A。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-85dba5430f3c439e4647ea4d97ba54fc_1440w.jpg)

唤醒线程其实就是将进程A从等待队列中移除，并结束阻塞被唤醒，在工作队列中恢复可以重新获得CPU时间片的资格后。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-a86b203b8d955466fff34211d965d9eb_1440w.jpg)

**select的进化史**
1. 用户代码中自旋实现所有阻塞socket的监听。但是每次判断socket是否产生数据，都涉及到用户态到内核态的切换。
2. 于是select改进 ：将fd_set传入内核态，由内核判断是否有数据返回，使用自旋来时刻的去判断socket列表中是否有数据达到。
3. 于是select改进：使用等待队列，让线程在没有资源时park（阻塞），当有数据到达时唤醒select线程，去处理socket。

**缺点**
1. 每次select都需要将进程加入到所有监听的socket的等待队列中，每次获取到数据，都需要将进程从所有监听的socket的等待队列中移除。而这两次遍历，都需奥将监听的socket列表传入到内核，从用户态进入内核态，开销非常大，所以限制最大监听的socket数量为1024。
2. 进程A被唤醒之后，还需要在遍历一次监听的socket列表，才能知道是哪个socket有数据准备好了。

### 5.2 poll

对于select监听socket最大为1024的限制，产生了poll。poll基本与select一致，只是存储监听Socket队列的数据结构使用链表（pollfd结构）。

6. epoll的设计思路

epoll是对select和poll的改进。

#### 等待队列和阻塞功能分离（缺点1）

调用select每次都需要将进程加入到等待队列，有socket就绪之后还需要将进程从多有socket的等待队列中移除。下次select还要重复上面的操作。而epoll把进程添加到等待队列和阻塞线程分开。大多数情况下进程监听的socket都是固定的，即使接收到数据之后下次阻塞进程监听的socket不会变化，所以先调用epoll_ctl创建等待队列，在调用epoll_wait阻塞线程，下次直接阻塞即可，不需要进行等待队列的相关操作。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-5ce040484bbe61df5b484730c4cf56cd_1440w.jpg)

```c
int s = socket(AF_INET, SOCK_STREAM, 0);   
bind(s, ...)
listen(s, ...)

int epfd = epoll_create(...);
//将所有需要监听的socket添加到epfd中
epoll_ctl(epfd, s1, s2, ...);

while(true){
    int n = epoll_wait(...)
    for(接收到数据的socket){
        //处理
    }
}
```

#### 增加就绪列表（缺点2）

调用select之后，我们只知道有socket数据就绪了，但是不知道是哪个socket，所以我们需要遍历一遍才知道哪些socket就绪了。如果内核维护一个已经就绪了的socket列表，我们就不需要遍历直接知道了。

如socket2、socket3准备好了，将其加入到rdlist中，这样直接获取到准备好的socket了。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-5c552b74772d8dbc7287864999e32c4f_1440w.jpg)

## 7. epoll的原理和流程

### 7.1 创建eventpoll对象
调用epoll_create时，会创建一个eventpoll对象，就是上面伪代码的epfd对象。eventpoll也是fd中的，也有等待队列。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-e3467895734a9d97f0af3c7bf875aaeb_1440w.jpg)

### 7.2 添加监听队列

调用epoll_ctl时，将Socket添加到监听队列中。中断程序不在直接操作线程，而是操作eventpoll对象。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-b49bb08a6a1b7159073b71c4d6591185_1440w.jpg)

### 7.3 接收数据

接收到数据时，中断程序将socket引用放入到就绪列表中。epoll_wait唤醒并返回就绪列表中的socket。

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-18b89b221d5db3b5456ab6a0f6dc5784_1440w.jpg)

### 7.4 阻塞和唤醒进程

假设计算机中正在运行进程A和进程B，在某时刻进程A运行到了epoll_wait语句。如下图所示，内核会将进程A放入eventpoll的等待队列中，阻塞进程。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-90632d0dc3ded7f91379b848ab53974c_1440w.jpg)

当socket接收到数据，中断程序一方面修改rdlist，另一方面唤醒eventpoll等待队列中的进程，进程A再次进入运行状态。也因为rdlist的存在，进程A可以知道哪些socket发生了变化。如下图：

![image](https://raw.githubusercontent.com/future94/java-technology/master/os/images/v2-40bd5825e27cf49b7fd9a59dfcbe4d6f_1440w.jpg)

## 8. epoll的实现细节

### 8.1 epoll等待队列使用红黑树的数据结构

当有数据来时，内核要快速的找到是哪一个socket有数据，而且要支持监听多个socket。对于链表来说，查找是O(n)，不如红黑树的O(logn)。

**为啥不用哈希表而用红黑树呢？**
主要感觉还是内存方面，如果内存要求苛刻的项目，就用红黑树；如果内存足够大，牺牲内存换取更快的速度，哈希完全适合。可能不对，欢迎指点。

### 8.2 epoll就绪队列使用双向链表的数据结构

就绪列表需要支持快速插入和删除，当有socket准备好要放入，处理完成时候要删除掉，所以选择双向链表。

## 9. 结论

对比项 | select | poll | epoll
---|---|---|---
等待队列 | 每次select都会重置等待队列 | 每次poll都会重置等待队列 | 事先维护好等待队列，下次循环不在维护
等待监听上限 | 1024 | 链表 | 红黑树
消息传递 | 需要将监听列表从用户态拷到内核态 | 需要将监听列表从用户态拷到内核态 | 内核与用户共享同一块内存（eventpoll对象）
获取就绪的socket | 轮询 | 轮询 | 回掉函数，直接返回就绪的socket


参考文章：
- [epoll 或者 kqueue 的原理是什么](https://www.zhihu.com/question/20122137)
- [epoll的本质](https://zhuanlan.zhihu.com/p/64746509)