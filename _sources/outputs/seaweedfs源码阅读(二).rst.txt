seaweedfs源码阅读(二)
=====================

1.简要介绍
----------

如果是阅读一个成熟的项目，最好是先阅读早期的版本，因为阅读一个项目的重点在于理解其思想。比如Linux，Linux目前的版本(4.x)的代码规模已经达到几百万行，阅读这样一个庞然大物的难度是无法想象的。但是Linux
0.11版本的规模只有2万行，可谓是小巧玲珑，从这个版本出发可谓是省时省力。seaweedfs也是一样，因此这次来简单介绍一下seaweedfs早期版本\ ``2011-12-13 08:17 53814``\ 。

2.要解决的任务
--------------

可以简单的分为两类，一个是如何处理http请求，另外一个是如何将数据存储。具体点说，前者要是实现基于http的文件接口，包括查找、下载、上传。这里的难点在于如何处理文件的上传。后者的目的是为了将文件比如文本文件与逻辑结构Needle对应起来，并将其写入到Volume中。

3.解决的思路
------------

网络请求通过golang的标准库中的路由组件进行处理，一个请求在被处理后调用存储模块相应的函数并进行相关的任务。

4.怎么存储
----------

先从存储的最小单位Needle说起，在weedfs中needle是存储的“原子”。举个例子，要将一张已经上传的图片存储到weedfs中该怎么做呢？

1. 拿到一张图片，这个条件已经满足了。
2. 新建一个Needle。注意，这里的Needle还不是图片，它只是一群字段的集合，只是一段内存而已

.. code:: go

      Cookie uint8 "用于唯一标识"
      Key uint64 "文件ID"
      AlternateKey uint32 "额外文件id"
      Size uint32 "数据大小"
      Data []byte "存储数据"
      Checksum int32 "校验码"
      Padding []byte "Aligned to 8 bytes"

一个Needle是由Cookie+Key+AltKey三者组合并唯一标识的，Size和Data用于保存数据，在这里目前这两个字段为空，最后的Checksum和Padding用于校验

3. 将Needle和图片对应起来。怎么对应，将图片写入到Needle.Data和补充完Needle.Size就可以了。注意这里写入的是图片对应的字节流。
4. 现在已经有一个Needle，并且这个Needle也存储有对应的图片了。下一步该做什么呢？\ ***注册***\ ！每一个Needle都要向Needle\_Map注册，只有这样才能找得到它们。因为每一个Needle都要被存储到一个大文件比如某个Volume服务器上的某个Volume中。在这里，Volume就代表一个大文件。当Needle存储在大文件中时，从物理实质看，每一个needle都是被顺序追加写入到大文件的末尾。那么，Needle\_Map怎么找到它们呢？每一个Needle\_Map维护着一个哈希表，它的键是Needle的key，它的值是一个叫做NeedleValue的数据结构，它的定义如下

.. code:: go

    type NeedleValue struct{
      Offset uint32 "Volume offset" //since aligned to 8 bytes, range is 4G*8=32G
      Size uint32 "Size of the data portion"
    }

该怎么理解这个结构呢？它其实是Needle在Volume上的存储标识。为什么这么说?因为要在Volume上找到一个Needle事实上只需要知道你在哪里开始存储的，你的大小是多少。知道这两点我就可以找到你。

5. 现在，我们已经有了一个Needle和一个Needle\_Map。那么下一步就是找一个Volume。如果没有Volume咋办，创建一个!那么一个Volume长啥样啊?

.. code:: go

    type Volume struct {
        Id                  uint64  //标识
        dir                 string  //所在目录
        dataFile, indexFile *os.File    //对应的大文件和索引文件
        nm                  *NeedleMap  //needleMap

        accessChannel chan int  //用于各Volume通信和master通信
    }

一个Volume起到的作用主要是存储和维护。存储needle和这个Volume的元数据信息。维护主要是和主节点保持通信

6. 现在已经有Volume了，但是还是不能存储，为啥？因为找不到！因此我们还需要一个类似Needle\_Map的东西也就是Volume\_Map。对于每一个Volume来说，它只会记录它在哪个目录和有哪些needle。但是这对于找到它来说远远不够，是的我知道你这个Volume在哪个目录。但是是哪台电脑的目录啊？这个目录下可能有多个Volume，哪一个是你哇？这个问题就是由Volume\_Map来解决的？它的结构如下

.. code:: go

    type Mapper struct {
        dir              string //map文件在哪个目录
        FileName         string //哪个是map文件
        Id2Machine map[uint32][]*Machine
        LastId uint32
    }

在这个版本中，Volume\_Map维护一个directorymap文件。这个文件告诉程序该如何找到它要找的Volume。

7.现在Needle，Needle\_Map，Volume，Volume\_Map都有了，程序终于知道怎么找到一张图片

::

    网络操作··· ···
    ->Volume_Map（找Volume）
    ->Volume（找Needle_Map）
    ->Needle_Map（根据key找到value，也就是needle在volume中的位置）
    ->根据Needle_Map的信息从Volume中读取字节流
    ->字节流转换
    ->拿到Needle
    ->读取Needle代表的文件

因此，程序可以放心地将图片存储到needle中。也就是执行Needle.write函数。

至此，一个文件的存储工作就完成了。

5.怎么发送文件请求
------------------

前面介绍了文件是怎么存储的，但是就使用来说还有一个问题。就是我的使用请求怎么发出去啊？在这个版本中，weedfs的思路很简单，每一个文件请求都是HTTP请求。weedfs会将HTTP请求转换为相应模块的函数调用。

6.总结
------

这次解读了weedfs的早期版本，功能很简陋，但是核心功能已经实现了，网络到存储这部分的功能已经基本实现，但是用户到网络的部分只能说有个原型。
