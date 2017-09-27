openstack swift middleware
==========================

在swift中提供了一种管道机制，它能按照一定的逻辑和顺序来处理文件请求。一个逻辑就是一个middleware，多个middleware按顺序连接起来就是一个管道。

架构
----

WSGI
全称Web服务器网关接口，它的作用是在协议之间进行转化。WSGI是一座桥梁，它可以将网络请求、响应等转化为python中的数据结构，反过来也可以。
在WSGI中把网络应用看成三部分，分别是WSGI server、WSGI middleware、WSGI
application。但一般只有WSGI server和WSGI application。

每一个swift middleware的本质其实是一个WSGI
middleware，它们需要实现一个call方法。swift在收到文件请求的时候将依次调用已经注册的middleware的call方法。
WSGI的架构很像洋葱，一个完整的swift管道可以包括一个或者多个swift
middleware，并且每个swift middleware都可以接受请求并返回响应。
举个例子，一个swift应用App有3个中间件实例M1、M2、M3，当一个网络请求进入到App时，它将经历以下过程：

M1 req-&gt;M2 req-&gt;M3 req-&gt;M3 res-&gt;M2 res-&gt;M1 res
（WSGI的顺序是M1-&gt;M2-&gt;M3）

这就像是剖洋葱，请求在进入WSGI的时候要像剖洋葱一样一层层地经过中间件。如果不进入下一层，当前层就会返回响应。每一层都能接受请求并返回响应，从架构上看它们是平级的，但从功能上看它们又是有顺序的。

本质
----

swift可以被看成是一个复杂的WSGI
应用。swift中的管道其实就是一个WSGI的中间件机制，只不过在swift中是处理IO请求。此外，由于每一个WSGI实例都能够接受请求并且返回响应，这意味着一个文件请求可以不经过所有的WSGI实例。
举个例子，一个文件请求经过上面提到的App，但是它在经过M2时被拦截了（因为M2不打算让它经过M3），那么它的下一站就是M2
res而不是M3 req。

用途
----

现在，swift一般利用middleware来完成一些辅助性工作，比如对上传的文件的名称合法性进行检查。但是middleware的能力不止于此，由于它可以拦截请求，并且支持管道化，因此它可以作为新功能模块的入口。
比如要拦截请求并将其转发到另外一个程序，这部分工作就可以单独用一个middleware来完成。
