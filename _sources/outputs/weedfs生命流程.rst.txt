weedfs生命流程
==============

(入口)./weed.go
---------------

总入口，执行过程如下

.. code:: go

    func main() {
        glog.MaxSize = 1024 * 1024 * 32 
        rand.Seed(time.Now().UnixNano())
        flag.Usage = usage
        flag.Parse()    //设置一下背景

        args := flag.Args()
        if len(args) < 1 {
            usage()
        }
    //如果第一个参数是help，则跳转到help
        if args[0] == "help" {
            help(args[1:])
            for _, cmd := range commands {
                if len(args) >= 2 && cmd.Name() == args[1] && cmd.Run != nil {
                    fmt.Fprintf(os.Stderr, "Default Parameters:\n")
                    cmd.Flag.PrintDefaults()
                }
            }
            return
        }   
    //主入口，根据commands构造参数，并执行run方法
        for _, cmd := range commands {
            if cmd.Name() == args[0] && cmd.Run != nil {
                cmd.Flag.Usage = func() { cmd.Usage() }
                cmd.Flag.Parse(args[1:])
                args = cmd.Flag.Args()
                IsDebug = cmd.IsDebug
                if !cmd.Run(cmd, args) {
                    fmt.Fprintf(os.Stderr, "\n")
                    cmd.Flag.Usage()
                    fmt.Fprintf(os.Stderr, "Default Parameters:\n")
                    cmd.Flag.PrintDefaults()
                }
                exit()
                return
            }
        }
    //有错误就输出
        fmt.Fprintf(os.Stderr, "weed: unknown subcommand %q\nRun 'weed help' for usage.\n", args[0])
        setExitStatus(2)
        exit()
    }

其中cmd.run函数的签名如下，接受一个命令和参数

.. code:: go

    Run func(cmd *Command, args []string) bool

Command的结构如下

.. code:: go

    var Commands = []*Command{
        cmdBenchmark,   //测试
        cmdBackup,      //备份
        cmdCompact,     //合并？备份
        cmdCopy,        //文件复制
        cmdFix,         //重建索引
        cmdServer,      //同时启动Master服务和Volume服务
        cmdMaster,      //Master服务
        cmdFiler,       //启动文件服务器，处理rest请求
        cmdUpload,      //上传文件
        cmdDownload,    //下载文件
        cmdShell,       //开启交互命令行
        cmdVersion,     //打印weedfs版本
        cmdVolume,      //Volume服务
        cmdExport,      //列出所有文件或者将数据导出
        cmdMount,       //挂载filer到usespace，FUSE
    }

入口函数会使用range语法遍历Command里面的成员，不空则执行对应的命令

(请求)./weed/command/master.go
------------------------------

如果执行的是master命令，那么这次命令将启动master服务。当执行master命令时，程序将从入口函数切换到master模块，执行该模块下的runMaster函数。这个函数最主要的功能是新建了一个mux实例。

.. code:: go

    // 节选func run   r := mux.NewRouter()
        ms := weed_server.NewMasterServer(r,
                                          *mport,
                                          *metaFolder,
                                          *volumeSizeLimitMB, 
                                          *volumePreallocate,
                                          *mpulse,               
                                       *defaultReplicaPlacement, 
                                          *garbageThreshold,
                                          masterWhiteList, 
                                          *masterSecureKey,
        )Master(cmd *Command, args []string) bool
        r := mux.NewRouter()
        ms := weed_server.NewMasterServer(r, *mport, 
                                          *metaFolder,
                                          *volumeSizeLimitMB, 
                                          *volumePreallocate,
                                          *mpulse, 
                                       *defaultReplicaPlacement, 
                                          *garbageThreshold,
                                          masterWhiteList, 
                                          *masterSecureKey,
        )

    golang自带的\ `http.SeverMux路由实现 <http://studygolang.com/articles/4890>`__\ 简单,本质是一个map[string]Handler,是请求路径与该路径对应的处理函数的映射关系。实现简单功能也比较单一
    1. 不支持正则路由， 这个是比较致命的 2.
    只支持路径匹配，不支持按照Method，header，host等信息匹配，所以也就没法实现RESTful架构

    而gorilla/mux是一个强大的路由，小巧但是稳定高效，不仅可以支持正则路由还可以按照Method，header，host等信息匹配，可以从我们设定的路由表达式中提取出参数方便上层应用，而且完全兼容http.ServerMux

具体的路由处理函数被放到了weed/server/master\_server.go里面

.. code:: go

        r.HandleFunc("/", ms.uiStatusHandler)
        r.HandleFunc("/ui/index.html", ms.uiStatusHandler)
        r.HandleFunc("/dir/assign", ms.proxyToLeader(ms.guard.WhiteList(ms.dirAssignHandler)))
        r.HandleFunc("/dir/lookup", ms.proxyToLeader(ms.guard.WhiteList(ms.dirLookupHandler)))
        r.HandleFunc("/dir/status", ms.proxyToLeader(ms.guard.WhiteList(ms.dirStatusHandler)))
        r.HandleFunc("/col/delete", ms.proxyToLeader(ms.guard.WhiteList(ms.collectionDeleteHandler)))
        r.HandleFunc("/vol/lookup", ms.proxyToLeader(ms.guard.WhiteList(ms.volumeLookupHandler)))
        r.HandleFunc("/vol/grow", ms.proxyToLeader(ms.guard.WhiteList(ms.volumeGrowHandler)))
        r.HandleFunc("/vol/status", ms.proxyToLeader(ms.guard.WhiteList(ms.volumeStatusHandler)))
        r.HandleFunc("/vol/vacuum", ms.proxyToLeader(ms.guard.WhiteList(ms.volumeVacuumHandler)))
        r.HandleFunc("/submit", ms.guard.WhiteList(ms.submitFromMasterServerHandler))
        r.HandleFunc("/delete", ms.guard.WhiteList(ms.deleteFromMasterServerHandler))
        r.HandleFunc("/{fileId}", ms.proxyToLeader(ms.redirectHandler))
        r.HandleFunc("/stats/counter", ms.guard.WhiteList(statsCounterHandler))
        r.HandleFunc("/stats/memory", ms.guard.WhiteList(statsMemoryHandler))

可以看到master部分负责的路由，对应的方法也是被二次封装了。被封装成

.. code:: go

    func (ms *MasterServer) proxyToLeader(f func(w http.ResponseWriter, r *http.Request)) func(w http.ResponseWriter, r *http.Request)

不但如此，请求和响应也被封装到了一个函数里面。它的签名是

.. code:: go

    func (g *Guard) WhiteList(f func(w http.ResponseWriter, r *http.Request)) func(w http.ResponseWriter, r *http.Request)

在这里weeedfs通过封装的方式将请求交给了安全模块/weed/guard。但目前这里似乎是一片空白。。。因为guard也是继续转发请求，但是这次是直接交到后台程序。

至此，整个请求部分就完成了。

(响应)
