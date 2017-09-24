daqu-9-11 日报
==================

完成的工作
----------

*am*

上课。矩阵论后听分布式

*pm*

1.openstack swift Ring（一天一文档）

-  ring的工作原理，挂载->存储
-  如何挂载？对一个设备或者文件计算一个hash值，这个hash值决定了其在ring上的位置
-  如何存储？已经挂载到ring上的文件按顺时针往前查找，找到第一个设备，即将其作为存储目的地
-  上述的问题？各存储设备所分布到的文件可能不均匀。解决方法：将ring先平均分配，再执行上述工作

2.Temp auth（没看完）

3.重新安装SAIO（失败。原因是centos的源更新太慢，以及找不到某个包liberasecode）

尝试\ ``yum -y install epel-realease``\ 安装第三方源，因为该源更新速度更快，但仍有一个包没有

4.rpm命令（了解）

5.\ `Linux命令大全 <http://www.man.linuxde.net>`__
