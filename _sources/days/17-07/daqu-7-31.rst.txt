daqu-7-31 日报
==================

完成的工作~
-----------

研究storlet代码：

1. Scommon包是storlet的主要部分，阅读这个即可。推荐阅读的类。

-  StorletOutStream.java 此类描述了storlet处理的文件流是什么样的
-  StorletLogger.java
   介绍了storlet的日志记录方式，通过这个类可以了解为什么println函数没有用的原因
-  RangFileInputStream.java 介绍了storlet怎么处理流的，重要！！

| 以上代码都没有精读 ​
| 2.
  Storlet的输入和输出都是通过stream来完成的，这不仅包括swift文件的传输，还包括log函数。

搭建storlet环境：

3. 网络是用的铁通，出口带宽太少。正在换成联通中。

下一步的计划~
-------------

1. 速速搭好storlet all in
   one的环境并成功运行测试，并将其输出成虚拟机。然后传到小分队的群里

2. 在SAIO上检查storlet demo的代码。熟悉工作流程。
