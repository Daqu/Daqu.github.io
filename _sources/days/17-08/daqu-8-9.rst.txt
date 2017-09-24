daqu-8-9 日报
================

完成的工作~
-----------

-  阅读seaweedfs的storage部分代码，但是只完成了两个文件的阅读。进度缓慢
-  复习了\ `btree <http://blog.csdn.net/endlu/article/details/51720299>`__\ ，因为seaweedfs用这个作为文件的数据结构。具体可见\ ``storage/needle/btree_map.go``

谷歌就btree写了一个golang
包，能够基于内存实现btree。然后seaweedfs用btree作为数据结构来维护文件。

.. code:: go

    func NewBtreeMap() *BtreeMap {
        return &BtreeMap{
            tree: btree.New(32),
        }
    }

    func (cm *BtreeMap) Set(key Key, offset, size uint32) (oldOffset, oldSize uint32) {
        found := cm.tree.ReplaceOrInsert(NeedleValue{key, offset, size})
        if found != nil {
            old := found.(NeedleValue)
            return old.Offset, old.Size
        }
        return
    }

    ··· ···

上面是weed中关于btree的部分，可以看到这部分负责创建btree的实例。然后weedfs将它与文件对应起来。在\ ``needle_map.go``\ 中

.. code:: go

    type NeedleMapper interface {
        Put(key uint64, offset uint32, size uint32) error
        Get(key uint64) (element *needle.NeedleValue, ok bool)
        Delete(key uint64, offset uint32) error
        Close()
        Destroy() error
        ContentSize() uint64
        DeletedSize() uint64
        FileCount() int
        DeletedCount() int
        MaxFileKey() uint64
        IndexFileSize() uint64
        IndexFileContent() ([]byte, error)
        IndexFileName() string
    }

作者设置了NeedleMapper接口。因为前面btree模块中实现了这个接口中的Put、Get方法。这样Btree实例就可以通过这个接口和文件关联起来。

-  复习了golang的一些语法，在阅读过程中复习的
-  将weedfs在windows下的环境搭建作了总结

待完成的工作~
-------------

1. 继续阅读seaweedfs
