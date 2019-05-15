# 启动过程


### 可执行程序im

通过make构建之后，便生成可执行程序im。

```bash
$ ./im --help
Usage of ./im:
  -alsologtostderr
    	log to standard error as well as files
  -log_backtrace_at value
    	when logging hits line file:N, emit a stack trace
  -log_dir string
    	If non-empty, write log files in this directory
  -logtostderr
    	log to standard error instead of files
  -stderrthreshold value
    	logs at or above this threshold go to stderr
  -v value
    	log level for V logs
  -vmodule value
    	comma-separated list of pattern=N settings for file-filtered logging
```

通过`im --help`帮助信息可以看出，这些选项都是为了设置日志相关的配置的。

```bash
$ ./im
Version:     2.0.0
Built:       2019年 2月27日 星期三 17时24分30秒 CST
Go version:  go version go1.10.3 darwin/amd64
Git branch:  master
Git commit:  93a580b
usage: im config
```

执行im可执行程序会打印一些构建信息，从命令行输出可以看出，还必须提供配置文件参数。

### mian()方法

```go
//删掉了部分无关紧要的代码
func main() {
	//读取配置文件
	config = read_storage_cfg(flag.Args()[0])
	//创建Redis连接池
	redis_pool = NewRedisPool(config.redis_address, config.redis_password, config.redis_db)
	//构建IMS存储服务RPC的连接池
	rpc_clients = make([]*gorpc.DispatcherClient, 0)
	for _, addr := range(config.storage_rpc_addrs) {
		c := &gorpc.Client{Conns: 4, Addr: addr}
		c.Start()
		dispatcher := gorpc.NewDispatcher()
		dispatcher.AddFunc("SyncMessage", SyncMessageInterface)
		dispatcher.AddFunc("SyncGroupMessage", SyncGroupMessageInterface)
		dispatcher.AddFunc("SavePeerMessage", SavePeerMessageInterface)
		dispatcher.AddFunc("SaveGroupMessage", SaveGroupMessageInterface)
		dispatcher.AddFunc("GetLatestMessage", GetLatestMessageInterface)
		dc := dispatcher.NewFuncClient(c)
		rpc_clients = append(rpc_clients, dc)
	}
	//构建超级群存储服务RPC的连接池
	if len(config.group_storage_rpc_addrs) > 0 {
		//......
	} else {
		group_rpc_clients = rpc_clients
	}
	//构建IMR路由服务RPC的连接池
	route_channels = make([]*Channel, 0)
	for _, addr := range(config.route_addrs) {
		channel := NewChannel(addr, DispatchAppMessage, DispatchGroupMessage, DispatchRoomMessage)
		channel.Start()
		route_channels = append(route_channels, channel)
	}
	//构建超级群路由服务RPC的连接池
	if len(config.group_route_addrs) > 0 {
		//......
	} else {
		group_route_channels = route_channels
	}
	//创建敏感词过滤器
	if len(config.word_file) > 0 {
		filter = sensitive.New()
		filter.LoadWordDict(config.word_file)
	}
	//启动群组管理器
	group_manager = NewGroupManager()
	group_manager.Start()
	//TODO
	group_message_delivers = make([]*GroupMessageDeliver, config.group_deliver_count)
	for i := 0; i < config.group_deliver_count; i++ {
		q := fmt.Sprintf("q%d", i)
		r := path.Join(config.pending_root, q)
		deliver := NewGroupMessageDeliver(r)
		deliver.Start()
		group_message_delivers[i] = deliver
	}
	//订阅Redis的禁言频道：speak_forbidden
	go ListenRedis()
	//TODO
	go SyncKeyService()
	//启动HTTP服务器，用于获取IM服务器状态信息以及WEB客户端的即时通信
	go StartHttpServer(config.http_listen_address)
	//TODO do nothing server？
	StartRPCServer(config.rpc_listen_address)
	//启动WebSocket
	go StartSocketIO(config.socket_io_address, config.tls_address, config.cert_file, config.key_file)
	//启动SSL服务器，监听客户端连接
  if config.ssl_port > 0 && len(config.cert_file) > 0 && len(config.key_file) > 0 {
		go ListenSSL(config.ssl_port, config.cert_file, config.key_file)
	}
	//启动TCP服务器，监听客户端连接
	ListenClient()
}
```

