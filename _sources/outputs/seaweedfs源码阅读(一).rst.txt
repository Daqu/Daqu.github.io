seaweedfs源码阅读(一)
---------------------

1.随便看看
^^^^^^^^^^

作为一个新读者，一开始是很难把握项目的整个架构的。因此我先凭着感觉去看一下，然后便找到了\ ``storage``\ 目录。这个目录应该是讲怎么存储文件的，论文里面讲到needle代表着一个文件。那么先从\ ``storage/needle.go``\ 看起吧。

.. code:: go

    const (
        NeedleHeaderSize      = 16 //should never change this
        NeedlePaddingSize     = 8
        NeedleChecksumSize    = 4
        MaxPossibleVolumeSize = 4 * 1024 * 1024 * 1024 * 8
        TombstoneFileSize     = math.MaxUint32
        PairNamePrefix        = "Seaweed-"
    )

一开始就定义了一堆常量，不过从名字上比较好判断它们是干嘛的。但是目前只知道\ ``MaxPossibleVolumeSize``\ 应该意味着一个Volume最大是4G。然后接下来便是needle的定义。

.. code:: go

    type Needle struct {
        Cookie uint32 `comment:"random number to mitigate brute force lookups"`
        Id     uint64 `comment:"needle id"`
        Size   uint32 `comment:"sum of DataSize,Data,NameSize,Name,MimeSize,Mime"`

        DataSize     uint32 `comment:"Data size"` //version2
        Data         []byte `comment:"The actual file data"`
        Flags        byte   `comment:"boolean flags"` //version2
        NameSize     uint8  //version2
        Name         []byte `comment:"maximum 256 characters"` //version2
        MimeSize     uint8  //version2
        Mime         []byte `comment:"maximum 256 characters"` //version2
        PairsSize    uint16 //version2
        Pairs        []byte `comment:"additional name value pairs, json format, maximum 64kB"`
        LastModified uint64 //only store LastModifiedBytesLength bytes, which is 5 bytes to disk
        Ttl          *TTL

        Checksum CRC    `comment:"CRC32 to check integrity"`
        Padding  []byte `comment:"Aligned to 8 bytes"`
    }

可以看到，一个needle携带蛮多信息。除了本身的数据比如Data、Id、cookie之外，它还携带了额外的信息比如Mime、Pairs、ttl(time
to live?)等等。看来，一个needle是元数据和数据的结合。

看到这里，发现什么都不懂。那么，seaweedfs源码的第一步就从storage模块开始吧。

2.storage
^^^^^^^^^

needle.go
'''''''''

storage的目录结构如下：

.. code:: powershell

     D:\Temp\seaweedfs-master\weed\storage 的目录

    2017/08/04  12:37    <DIR>          .
    2017/08/04  12:37    <DIR>          ..
    2017/07/28  23:33             4,682 compact_map.go
    2017/07/28  23:33               533 crc.go
    2017/07/28  23:33             4,432 disk_location.go
    2017/07/28  23:33             1,218 file_id.go
    2017/08/04  12:37    <DIR>          needle
    2017/07/28  23:33             6,641 needle.go
    2017/07/28  23:33               232 needle_byte_cache.go
    2017/07/28  23:33             3,090 needle_map.go
    2017/07/28  23:33             4,247 needle_map_boltdb.go
    2017/07/28  23:33             3,807 needle_map_leveldb.go
    2017/07/28  23:33             3,423 needle_map_memory.go
    2017/07/28  23:33             8,831 needle_read_write.go
    2017/07/28  23:33             1,182 needle_test.go
    2017/07/28  23:33             1,179 replica_placement.go
    2017/07/28  23:33               313 replica_placement_test.go
    2017/07/28  23:33             9,740 store.go
    2017/07/28  23:33             1,474 store_vacuum.go
    2017/07/28  23:33             3,041 volume.go
    2017/07/28  23:33             1,996 volume_checking.go
    2017/07/28  23:33               367 volume_create.go
    2017/07/28  23:33               438 volume_create_linux.go
    2017/07/28  23:33               356 volume_id.go
    2017/07/28  23:33             1,581 volume_info.go
    2017/07/28  23:33               303 volume_info_test.go
    2017/07/28  23:33             3,471 volume_loading.go
    2017/07/28  23:33             6,815 volume_read_write.go
    2017/07/28  23:33             2,267 volume_super_block.go
    2017/07/28  23:33               403 volume_super_block_test.go
    2017/07/28  23:33             7,093 volume_sync.go
    2017/07/28  23:33             2,394 volume_ttl.go
    2017/07/28  23:33             1,060 volume_ttl_test.go
    2017/07/28  23:33             9,432 volume_vacuum.go
    2017/07/28  23:33             1,713 volume_vacuum_test.go
    2017/07/28  23:33               132 volume_version.go

接着上面对\ ``needle.go``\ 的阅读。在needle的结构声明后是一个简单的String函数，它的功能就是把needle的字段打印出来。接着是一个巨长的\ ``ParseUpload``\ 的函数，虽然这个函数有点长，大概八十行这样子。但是它的功能还是挺简单的，他负责分析http请求。它接受一个http请求，然后返回一个包含大部分needle字段信息的元组，它的签名如下：

.. code:: go

    func ParseUpload(r *http.Request) (
        fileName string, 
        data []byte, 
        mimeType string, 
        pairMap map[string]string, 
        isGzipped bool,
        modifiedTime uint64, 
        ttl *TTL, 
        isChunkedFile bool, 
        e error)

在接受到一个请求后，它首先分析请求头。把所有Seaweed-前缀的字段提取出来。再读取数据，值得注意的是数据的传送协议是\ ``multipart/form-data``\ 。

接下来是新建needle的函数，它的目的是将请求中的字段赋值给新的needle。这部分用到的结构体方法定义在\ ``needle_read_write.go``\ 。

needle\_read\_write.go
''''''''''''''''''''''

这部分定义了needle的一些关于读和写的方法。除了给needle结构体中赋值的方法外，剩下的方法都是关于needle如何操作io的。下面描述一下其中一个方法\ ``Append``\ 。

.. code:: go

    func (n *Needle) Append(w io.Writer, version Version) (size uint32, actualSize int64, err error) {
        if s, ok := w.(io.Seeker); ok {
            if end, e := s.Seek(0, 1); e == nil {
                defer func(s io.Seeker, off int64) {
                    if err != nil {
                        if _, e = s.Seek(off, 0); e != nil {
                            glog.V(0).Infof("Failed to seek %s back to %d with error: %v", w, off, e)
                        }
                    }
                }(s, end)
            } else {
                err = fmt.Errorf("Cannot Read Current Volume Position: %v", e)
                return
            }
        }
        
    ··· ···

首先，将Writer
w通过类型断言转化Seeker。接着通过Seek方法从第0个字节开始读取数据，seek方法的两个参数分别代表偏移量和偏移起点，这里指的是从当前指针所在位置偏移0个字节。这里使用了设置匿名函数立即执行的方法，然后因为有defer修饰，所以是逆序立即执行。Seeker第二个参数有相应的常量

    SEEK\_SET int = 0 //从文件的起始处开始设置 offset SEEK\_CUR int = 1
    //从文件的指针的当前位置处开始设置 offset SEEK\_END int = 2
    //从文件的末尾处开始设置 offset

接下来就是数据写入的过程了。不过奇怪的是weedfs居然有两个版本。

.. code:: go

        switch version {
        case Version1:
            header := make([]byte, NeedleHeaderSize)
            util.Uint32toBytes(header[0:4], n.Cookie)
            util.Uint64toBytes(header[4:12], n.Id)
            n.Size = uint32(len(n.Data))
            size = n.Size
            util.Uint32toBytes(header[12:16], n.Size)
            if _, err = w.Write(header); err != nil {
                return
            }
            if _, err = w.Write(n.Data); err != nil {
                return
            }
            actualSize = NeedleHeaderSize + int64(n.Size)
            padding := NeedlePaddingSize - ((NeedleHeaderSize + n.Size + NeedleChecksumSize) % NeedlePaddingSize)
            util.Uint32toBytes(header[0:NeedleChecksumSize], n.Checksum.Value())
            _, err = w.Write(header[0 : NeedleChecksumSize+padding])
            return
        case Version2:
            header := make([]byte, NeedleHeaderSize)
            util.Uint32toBytes(header[0:4], n.Cookie)
            util.Uint64toBytes(header[4:12], n.Id)
            n.DataSize, n.NameSize, n.MimeSize = uint32(len(n.Data)), uint8(len(n.Name)), uint8(len(n.Mime))
            if n.DataSize > 0 {
    ··· ···

就功能上来说，两个版本的作用都是一样的。因此在这里只分析第一个版本。它首先根据header大小(这里是16字节)分配了一块内存，然后逐个将needle的字段转为字节码。最后写进去，末尾会有校验验证，如果前面写失败的话会重试。

剩下的就是跟读写操作相关的函数了。

needle\_map
'''''''''''

这部分由好几个文件构成，分别是基本数据结构\ ``needle_map``\ 和几个实例比如\ ``needle_map_leveldb``\ 等等。这部分主要是定义了NeedleMapper这个接口和baseNeedleMapper、mapMetric这两个结构。关于map的作用，目前我的理解还只是停留在论文的解释中。

    每个Store机器管理多个物理卷。每个物理卷存有百万张图片。读者可以将一个物理卷想象为一个非常大的文件（100GB），保存为\ ``/hay/haystack<logical volume id>``\ 。Store机器仅需要逻辑卷ID和文件offset就能非常快的访问一个图片。这是Haystack设计的主旨：不需要磁盘操作就可以检索文件名、偏移量、文件大小等元数据。Store机器会将其下所有物理卷的文件描述符（open的文件“句柄”，卷的数量不多，数据量不大）缓存在内存中。同时，图片ID到文件系统元数据（文件、偏移量、大小等）的映射（后文简称为“内存中映射”）是检索图片的重要条件，也会全部缓存在内存中。为了快速的检索needle，Store机器需要为每个卷维护一个内存中的key-value映射。映射的Key就是（needle.key+needle.alternate\_key）的组合，映射的Value就是needle的flag、size、卷offset（都以byte为单位）。如果Store机器崩溃、重启，它可以直接分析卷文件来重新构建这个映射（构建完成之前不处理请求）。

由上可知，map的存在是为了让涉及到元数据的操作不经过磁盘操作，因此map的作用是为了把元数据存储到本地的数据库上。

Volume
''''''

Volume就是物理卷了，它的结构如下：

.. code:: go

    type Volume struct {
        Id            VolumeId
        dir           string
        Collection    string
        dataFile      *os.File
        nm            NeedleMapper
        needleMapKind NeedleMapType
        readOnly      bool

        SuperBlock

        dataFileAccessLock sync.Mutex
        lastModifiedTime   uint64 //unix time in seconds

        lastCompactIndexOffset uint64
        lastCompactRevision    uint16
    }

除了基本的信息外，每一个Volume都有一个NeedleMapper。这也证实了每一个卷负责维护该卷上文件的元数据。
