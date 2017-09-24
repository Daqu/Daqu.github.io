daqu-9-6 日报
================

过程
----

am
--

1.Storlet MultiInput例子看了。结论：确实是多个文件输入，多个文件输出

2.研究了正向代理、反向代理。因为实验室的电脑系统是Linux，没有好的远程登录软件

3.安装漂亮的git客户端，gitkraken。失败

4.看了Openstack Swift middleware文档。(没怎么看懂)

5.看了Middleware部分源代码 name\_check（没怎么看懂）

6.安装typora失败，原因是/etc/lsb\_realease中不是Ubuntu或者它支持的。我用的确实是Ubuntu，但是是kylin，然后它把系统标识改了。

7.安装vscode失败

8.尝试用gdebi解决问题6、 7。没有效果。

pm
--

1.重看MultiInput。结论：虽然是多文件输入和多文件输出，但是这都是一次请求内完成。重点是一次请求内完成。如果这个storlet能够同时处理多个请求，就很好，然而不是。

2.怀疑早上安装软件的失败原因都是系统标识不在deb包支持目录内。从kylin换成kubuntu。成功。

3.写了一篇关于middleware的输出（目的是为了主动学习）

4.看了Object storage API overview（一天一文档）

结论：介绍的用途很有意思（自动解压、浏览器上传文件）

未来的计划
----------

通过swift middleware实现合并
（一个或者多个middleware，实现一个或者多个功能？）（偶尔出现的想法）

完成的工作
----------

1.写了一篇关于swift middleware的输出（主动学习）

2.看了swift Object API文档

3.了解了python WSGI（架构、思想，没有写代码）
