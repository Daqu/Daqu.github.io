# 高级消息队列协议在sds中的应用的思路

sds的特点在于控制与数据分离，然而控制层的形态是怎么样的仍是一个待探究的问题。本文将结合海量小文件存储的问题探讨高级消息队列协议在sds中应用的思路。

## 用sds处理小文件存储的思路

海量小文件的存储问题主要是由于小文件数量太多而引起元数据过大的问题，这样会导致在查询时会先要花费大量资源去遍历元数据的情况。为了解决这个问题，目前的思路一般是先将小文件合并再存储。现有的很多方案是在文件系统级别进行合并，但是这种方案会导致小文件问题和文件系统紧紧耦合在一起，从而降低了文件系统的通用性。那么，如何在文件系统外实现合并，并且对于用户来说他是看不到这种合并的过程的。这便是用sds的方式解决小文件存储问题，即在用户和文件系统之间额外增加一层控制层，业务逻辑都在控制层中完成，从而数据层（文件系统）不需要太多改动，用户也感受不到这种变化。  
如何进行合并？一般的思路是：  
```
接收小文件
if 大文件达到大小上限:
	存储当前大文件
	新建一个大文件
	将小文件放入大文件
else:
	将小文件放入大文件
```
## 现有方案的缺陷

Crystal实现了一个控制层，它在文件请求和文件系统之间架设了一条管道，这条管道由多个管子组成，每根管子就是一个filter。文件请求在到达文件系统前需要先经过各个filter并被处理。形象一点就是:  

```
request --> filter1 --> filter2 --> filter3 --> filesystem
```
在这里，我们并不关注这种机制的结果，而是关注它本身。那么，可以先分析一下上述机制的缺陷。在Crystal中，管道是单通道的，并且在管道中不允许停顿。在某些情况下，crystal会很容易产生堵塞。举个例子，还是上面那个图。假如某个时间段有10个请求，filter1处理单个请求的时间是1s，filter2处理单个请求的时间是1s，filter3处理单个请求的时间是10s。这个处理过程如下:  
```
t=0
	filter(10)
t=1
	filter(9) filter2(1)
t=3
	filter(8) filter2(1) filter3(1)
t=4
	filter(7) filter2(1) filter3(2)
t=5
	filter(6) filter2(1) filter3(3)
t=6
	filter(5) filter2(1) filter3(4)
t=7
	filter(4) filter2(1) filter3(5)
... ...
t=12
	filter(0) filter2(0) filter3(9)⁉️
t=13
	filter(0) filter2(0) filter3(8)
```
可以看到，在这种机制下，如果filter间的处理时间差距很大，某个时间花费较大的filter会出现堵塞的情况，往严重一定说就有可能出现内存不足，因为要缓存请求，在海量小文件存储的场景下，这个问题又被放大了。

## 改进方法

为了解决上面提到的问题。可以在filter之间增加一个用于缓存请求的队列。改进之后的模型如下:  

```
request⬇️
queue0 🚋🚋🚋🚋🚋⬇️
filter1⬇️
queue1 🚋🚋🚋🚋🚋⬇️
filter2⬇️
queue2 🚋🚋🚋🚋🚋⬇️
filter3⬇️
queue3 🚋🚋🚋🚋🚋⬇️
filesystem
```

在改进之后，filter不用再花时间维护堵塞的请求。filter只需在处理完当前请求后向关联的队列发起一个GET请求即可拿到下一个文件请求。

## 分析

就目前来说，用管道来描述crystal的控制层这种抽象是没有问题的，即便是前面提到的改进过后的crystal也可以用管道来描述。因为它的控制层的逻辑并不是很复杂，因此管道这种抽象对于它来说已经足够了。但是是不是可以大胆地说，crystal做得还不够好，它包括的场景还不够多？😊假如这种观点成立的话，又有一个新的问题。有没有一种通用性比crystal的管道抽象更强的抽象呢？答案是肯定的，这种抽象就是amqp协议。

## amqp协议简介

AMQP是一个提供统一消息服务的应用层标准协议，基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，不同开发语言等条件的限制。

### rabbitmq的一些概念（amqp协议的一个实现）

- **Broker**:    接收和分发消息的应用，RabbitMQ Server就是Message Broker。
- **Virtual host**:    出于多租户和安全因素设计的，把AMQP的基本组件划分到一个虚拟的分组中，类似于网络中的namespace概念。当多个不同的用户使用同一个RabbitMQ server提供的服务时，可以划分出多个vhost，每个用户在自己的vhost创建exchange／queue等。
- **Connection**: publisher／consumer和broker之间的TCP连接。断开连接的操作只会在client端进行，Broker不会断开连接，除非出现网络故障或broker服务出现问题。
- **Channel**: 如果每一次访问RabbitMQ都建立一个Connection，在消息量大的时候建立TCP Connection的开销将是巨大的，效率也较低。Channel是在connection内部建立的逻辑连接，如果应用程序支持多线程，通常每个thread创建单独的channel进行通讯，AMQP method包含了channel id帮助客户端和message broker识别channel，所以channel之间是完全隔离的。Channel作为轻量级的Connection极大减少了操作系统建立TCP connection的开销。
- **Exchange**: message到达broker的第一站，根据分发规则，匹配查询表中的routing key，分发消息到queue中去。常用的类型有：direct (point-to-point), topic (publish-subscribe) and fanout (multicast)。
- **Queue**:    消息最终被送到这里等待consumer取走。一个message可以被同时拷贝到多个queue中。
- **Binding**: exchange和queue之间的虚拟连接，binding中可以包含routing key。Binding信息被保存到exchange中的查询表中，用于message的分发依据。

[rabbitmq简介](http://www.cnblogs.com/diegodu/p/4971586.html)

## amqp与sds

为什么amqp与sds能扯上关系？因为在笔者看来，目前sds的代表crystal，它所提出的控制层模型其实是amqp的生产者-消费者模型的一个子集。前面提到的改进的crystal控制层模型其实就是amqp中最简单的点对点通信模型。稍有不同的是crystal中的filter需要先跟前一个filter通信完才能和下一个filter通信。

既然amqp能够描述crystal的控制层模型，并且支持更复杂的模型，为什么不可以考虑使用amqp协议来描述sds的控制层呢？目前，sds的应用场景还不多，因此crystal这种方案的优点会很突出。但是随着sds的发展，crystal的控制层过于简单的问题会越来越突出。