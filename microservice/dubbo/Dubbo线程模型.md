
## Dubbo中线程模型

如果事件处理的逻辑能迅速完成，并且不会发起新的 IO 请求，比如只是在内存中记个标识，则直接在 IO 线程上处理更快，因为减少了线程池调度。

但如果事件处理逻辑较慢，或者需要发起新的 IO 请求，比如需要查询数据库，则必须派发到线程池，否则 IO 线程阻塞，将导致不能接收其它请求。

如果用 IO 线程处理事件，又在事件处理过程中发起新的 IO 请求，比如在连接事件中发起登录请求，会报“可能引发死锁”异常，但不会真死锁。


Dubbo请求流程如下：

![image](https://raw.githubusercontent.com/future94/java-technology/master/microservice/dubbo/images/dubbo-protocol.jpg)

由上图，我们需要配置不同的Dispatcher和ThreadPool

```xml
<dubbo:protocol name="dubbo" dispatcher="all" threadpool="fixed" threads="100" />
```

Dubbo基于Netty实现，netty中有两种线程池，其中 EventLoopGroup(boss) 主要用来接受客户端的链接请求，并把接受的请求分发给 EventLoopGroup(worker) 来处理，boss和worker线程组我们称为 `IO线程` 。同时 Dubbo 还提供了业务线程池，对于一个请求，根据请求消息是由 IO线程处理 还是业务线程池处理，Dubbo提供了5种 Dispatcher。
 
## Dispatcher

- **all** <font color="red">所有消息都派发到线程池</font>，包括请求，响应，连接事件，断开事件，心跳等。
- **direct** 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
- **message** 只有<font color="red">请求和响应消息派发到线程池</font>，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
- **execution** 只有<font color="red">请求消息派发到线程池</font>，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
- **connection** 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。

## ThreadPool

- **fixed** 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
- **cached** 缓存线程池，空闲一分钟自动删除，需要时重建。任务数量超过maximumPoolSize时直接抛出异常
- **limited** 可伸缩线程池，但池中的线程数只会增长不会收缩。只增长不收缩的目的是为了避免收缩时突然来了大流量引起的性能问题。
- **eager** 优先创建Worker线程池。在任务数量大于corePoolSize但是小于maximumPoolSize时，优先创建Worker来处理任务。当任务数量大于maximumPoolSize时，将任务放入阻塞队列中。阻塞队列充满时抛出RejectedExecutionException。(相比于cached:cached在任务数量超过maximumPoolSize时直接抛出异常而不是将任务放入阻塞队列)


## Dubbo线程模型与Tomcat线程模型有什么区别

### Tomcat支持的请求模式

- BIO ：同步阻塞模式。每个请求来时，都需要创建一个线程来处理请求，线程开销较大。
- NIO ：同步非阻塞模式。NIO中Connector中比BIO多了一个Poller：主要用来轮询事件列表中的事件，判断连接是否可读可写。然后生成任务定义器，放入Executor线程。如：去银行填表，有一个人专门负责轮询看是否有人填好，填好了就让他去窗口办理业务。
- APR ：利用JNI（Java Native Interface）调用本地API，大幅度提高了tomcat的IO性能利用操作系统来解决。
- AIO ：异步非阻塞莫模式。如：去银行填表，没人看是否填好，填好了自己去窗口。适用于链接很多，处理时间比较长的场景。

### TomcatNio线程模型

![image](https://raw.githubusercontent.com/future94/java-technology/master/microservice/dubbo/images/20170712085200173.png)

以NioEndpoint为例，分为LimitLatch、Acceptor、Poller、SocketProcessor、Excutor5个部分。

- **LimitLatch** ：连接控制器。负责控制最大可以接受多少个连接。默认为1000。
- **Acceptor** ：接收连接，转发给Queue。默认有一个线程，接收到连接请求，将请求事件注册到事件队列中，由Poller取出。
- **Poller** ：将已经就绪的事件取出，封装为SocketProcessor后交给Worker执行。默认 `Math.min(2,Runtime.getRuntime().availableProcessors())`
- **Worker** ：请求处理，从socket获取数据，交给对应的mapping执行后返回给socket。


参考文章：
- [线程模型](https://dubbo.apache.org/zh/docs/advanced/thread-model/)
- [Tomcat多线程模型浅析](https://blog.csdn.net/u011552404/article/details/80301692)