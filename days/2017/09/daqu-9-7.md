daqu-9-7 日报
=============

过程
----

1.Swift Architectural Overview （一天一文档）

-   Ring-&gt;mapping、name-&gt;physical store
-   storage policy-&gt;分层。不仅可以分三层
-   {Object server -&gt;simple blob storage server

存储的是二进制文件（内容+元数据，但是元数据存储形式不一样，是以xattr的形式存储）

}

-   Container server **listing 存储在数据库里**
-   Account server **存储在数据库里**

2.测试xattr（环境manjaro 17.03）

使用attr命令往一个空文件附加一个键值对 key：name、value：daqu
