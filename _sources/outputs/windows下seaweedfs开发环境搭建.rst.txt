windows下seaweedfs开发环境搭建
==============================================

1.必备条件
^^^^^^^^^^

1. `curl <https://curl.haxx.se/download.html>`__ Win64 -
   Generic或者Win32 - Generic的7zip
   `安装过程 <http://www.cnblogs.com/xing901022/p/4652624.html>`__
2. `weedfs编译版本 <https://github.com/chrislusf/seaweedfs/releases>`__
   weedfs的安装非常简单，windows下只有一个执行文件\ ``weed.exe``\ 。因此只需将它放进环境变量路径就可以了，为了方便可以将weed.exe放入curl解压目录的bin文件夹下。

2.阅读源码配置
^^^^^^^^^^^^^^

1. `go语言安装 <https://www.golangtc.com/download>`__
   值得注意的是在安装后需要新建GOPATH目录并将其加入环境变量
2. `gogland <https://www.jetbrains.com/go/>`__
