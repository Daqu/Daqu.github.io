daqu-8-14 日报
==============

完成的工作\~
------------

-   继续阅读weedfs网络部分的代码
-   在weedfs上实现软件定义存储的可能性

一个weedfs的文件请求从客户端发出到文件读取可以粗略地概括为

文件请求 --发出--&gt; Master Server --转发--&gt; Volume Server
--查找--&gt; Volume --查找--&gt; 该Volume中的Super block(元数据)
--读取--&gt; needle

为了在weedfs实现类似storlet的功能，第一步可以先思考如何weedfs满足storlet的依赖。

1.storlet通过一个swift
mildleware实现请求拦截与转发，那么weedfs中存在类似的机制么？

> 答案是否定的。

2.storlet的特点就是充分利用了存储节点的计算能力，但是这种前提是存储系统必须是分布式的，因为只有分布式，存储节点才能具备有独立的计算能力。

> weedfs满足这个要求。

3.weedfs本身已经是针对小文件优化提出的解决方案，那么在weedfs上实现SDS的好处是什么？

> weedfs专注于小文件存储问题，小文件的优化策略与文件系统是紧耦合的。SDS的目的在于控制和数据分离，SDS
> on
> weedfs不仅能够降低小文件优化与weedfs的耦合度，提高weedfs的通用性，而且用户能够将部分业务逻辑放在存储上实现，降低了上层系统的复杂度。

4.如何实现SDS on weedfs?

> 参考swift，在Master Server、Volume
> Server中加入管道机制。每一个请求在进入server后将像水流过管道一样的遍历filter函数。在所有filter函数执行完之后继续其它工作。

5.假如已经有了SDS on weedfs，可以利用它做什么工作？

> 将一些weedfs的功能通过策略的方式实现，比如weedfs允许在请求.jpg格式的文件加入长、宽两个参数，这样返回的jpg文件的格式就是设定好的长宽。然而，目前这个功能是以模块函数的形式实现的，这是没有的耦合度提升。总而言之，通过SDS可以进行降低weedfs复杂度的工作，将一些模块解耦。从而在不减少功能的情况下降低系统复杂度。

下一步工作\~
------------

-   继续阅读代码
-   尝试边读边实现一个原型？

