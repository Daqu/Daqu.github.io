将seaweedfs变成SDS的思路
========================

seaweedfs变成SDS的思路总结一下就是

1.  先调整而不是重构，不修改功能
2.  先调整架构，再调整模块

1.分层
------

将各功能模块按照[MVC](https://baike.baidu.com/item/MVC%E6%A1%86%E6%9E%B6/9241230?fr=aladdin&fromid=85990&fromtitle=MVC)的设计理念分为控制、数据层

**问题**：seaweedfs中的网络部分倾向于控制，文件部分倾向于数据。但是按照这种思路处理无法解决在SDS原型(二)中提到的问题，就是网络请求转为文件请求后，从文件请求开始到文件响应发出这部分无法做控制。原因是按照MVC的思想，文件部分已经是数据层了，从网络应用的角度来说这是对的，但是从存储系统来说这是不对的，因为在底层的文件读取函数被调用前发生的过程都可以算入控制。

总结一下，MVC思想分层的问题在于控制层太小，数据层太大。

2.管道化
--------

将seaweedfs的一次请求-响应的过程看成是请求流入管道，并最终流出管道成为响应的过程。SDS的控制体现在数据在经过管道时会发生变化，管子组合在一起形成管道，也就是控制层。

**问题**：管道是个很抽象的概念，如何将seaweedfs中一次请求-响应的过程抽象为一个操作序列是个问题。举个例子，比如seaweedfs的网络部分已经被看成是一个管子，这根管子负责网络请求的收发和转化。为了更细腻的控制，可以将这根管子拆成更多的管子。为了更清晰的架构，可以将这根管子和其它管子组合成更大的管子。问题就是，如何描述这种机制，用管子组成管道的机制。

总结一下，管道化难以抽象。

3.插件化
--------

将seaweedfs看成是一块插满插头的插座。将各功能模块看成是插头，将要实现的控制层看成是插座。插座加插头形成seaweedfs。他的特点在于各功能模块之间的耦合度很低。

**问题**：根据插件化的介绍，数据层变得模糊，或者是数据层重要性不如控制层重要，因为它只是个插头，虽然没了这个插头不行。但从架构上说好像不大好。

总结一下，先不提实现难度，优点和缺点都不突出。
