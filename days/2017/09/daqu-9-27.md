daqu-9-27 日报
==============

过程
----

1.  重构一下小文件存储的中间件(遇到并修复了一些bug)

2.  上传小文件应该返回seaweed的响应(实现)

3.  完成了小文件上传部分的调试(即小文件上传功能完成)

4.  在尝试调试小文件下载部分时发现没有处理fid

    **因为GET请求或者是DELETE请求时需要把fid作为uri附加到请求地址中去**

    --&gt; 解决方法:引入路由模块

下一步计划
----------

1.  开始调试下载部分代码
2.  引入路由模块，因为它是调试下载部分代码的前提

