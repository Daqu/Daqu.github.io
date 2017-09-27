openstack storlet简单介绍
=========================

storlet的特点
-------------

storlet无法操作硬盘，网络甚至是swift的请求环境(可以理解为不能修改请求么？)

storlet在执行时不能被理解为一般意义上的工作进程，特别是那种需要大量临时内存的进程。因为它工作在一个沙盒环境中，不能使用太多内存。作者不建议用storlet创建临时文件。storlet的运行过程一般是从输入流读取数据，执行任务，然后将结果写入到输出流。

值得注意的是，每一个storlet读取和存储的数据会根据调用方式的不同而发生改变：

-   **下载**--如果是在一个对象下载过程中调用storlet。那么用户接受到的输入流(inputstream)将是经过storlet转化(transform)过的。也就是说这时storlet接收到的数据不是存储在swift中的数据。
-   **上传**
    --如果是在一个对象上传过程中调用storlet。那么存储在swift中的数据将不会是它原来的数据，而是经过storlet转化的数据。
-   **操作(object
    copy)**--如果storlet要对一个对象进行持续操作，那么它的输出将会工作(keep
    in)在一个该对象的副本上。

storlet是怎么被调用的
---------------------

storlet的本质是swift的一个对象，存储在名叫storlet的容器中。但是storlet实现了一种底层机制(engine)使得这些storlet对象能够被动态的调用，storlet的生命流程可以简单地概括为拦截io请求-&gt;将请求重定向到storlet-&gt;返回storlet定制过的输出流。storlet的这种机制(engine)本质是swift的中间件。

storlet是怎么部署的
-------------------

storlet部署的过程可以分为三步，分别是编写storlet、编译打包storlet、将storlet作为一个对象上传到swift。再简单一点，可以说storlet的部署就是将打包好的storlet(包括依赖)上传到swift的特定的container中。

storlet engine的本质
--------------------

前面提到，storlet
engine其实是swift的中间件。每一个swift请求都会经过部署在swift上的中间件。那么一个请求在经过storlet
engine时发生了什么事情呢？storlet engine在往下说，它其实是一个python
WSGI，起到路由的作用。关于WSGI可以看这个[这个](http://www.nowamagic.net/academy/detail/1330310)
