daqu-8-7 日报
================

完成的工作~
-----------

1. 简单复习了haystack的论文

2. 

   -  haystack的特点是足够简单（相比与其它方案）

-  合并是基于物理卷(逻辑概念是volume，是的物理概念和逻辑概念都是volume)合并，volume维护当前卷下的元数据
-  needle可以被理解为精简过的inode?
-  haystack是对象存储，不是文件系统。这点跟swift很像。文中似乎提到在不同场景下可以切换底层文件系统

3. 阅读了seaweedfs的代码导读，理解了seaweedfs的代码结构

4. 

   -  seaweedfs是haystack的开源实现

-  seaweedfs可以简单分为集群模块和文件模块，集群模块的目的是为了维护分布式系统，文件模块的目的则是实现haystack。

下一步工作~
-----------

1. 深入阅读seaweedfs文件模块的代码
2. 思考如何将haystack中的小文件优化封装成一个模块
3. 思考如何在haystack中实现存储控制和数据的分离
