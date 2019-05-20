# 启动过程


### 可执行程序ims

通过make构建之后，便生成可执行程序ims。

```bash
$ ./ims --help
Usage of ./ims:
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

通过`ims --help`帮助信息可以看出，这些选项都是为了设置日志相关的配置的。

```bash
$ ./ims
Version:     2.0.0
Built:       2019年 2月27日 星期三 17时24分30秒 CST
Go version:  go version go1.10.3 darwin/amd64
Git branch:  master
Git commit:  93a580b
usage: ims config
```

执行ims可执行程序会打印一些构建信息，从命令行输出可以看出，还必须提供配置文件参数。

### mian()方法

```go
//删掉了部分无关紧要的代码
func main() {
	config = read_storage_cfg(flag.Args()[0])
	storage = NewStorage(config.storage_root)
	master = NewMaster()
	master.Start()
	if len(config.master_address) > 0 {
		slaver := NewSlaver(config.master_address)
		slaver.Start()
	}
	//刷新storage file
	go FlushLoop()
	go FlushIndexLoop()
	go waitSignal()
	if len(config.http_listen_address) > 0 {
		go StartHttpServer(config.http_listen_address)
	}
	go ListenSyncClient()
	ListenRPCClient()
}
```

**启动过程：**

1. 启动Master
2. 启动Slaver（如果设置了master_address）
3. 处理系统信号
4. 定时刷新消息文件
5. 定时刷新索引文件
6. 启动HttpServer（如果设置了http_listen_address）
7. 监听SyncClient
8. 监听RPCClient

#### 1. 启动Master

参见[]()

#### 2. 启动Slaver

参见[]()

#### 3. 处理系统信号

只会处理`SIGINT`和`SIGTERM`两种系统信号，当系统向ims进程发送这两种信号时，ims进程会刷新消息文件和索引文件，然后退出。

```go
switch sig {
case syscall.SIGTERM, syscall.SIGINT:
    storage.Flush()
    storage.SaveIndexFileAndExit()
}
```

#### 4. 定时刷新消息文件

每隔一秒刷新一次消息文件，看源码：

```go
func FlushLoop() {
	ticker := time.NewTicker(time.Millisecond * 1000)
	for range ticker.C {
		storage.Flush()
	}
}
```

#### 5. 定时刷新索引文件

每隔5分钟刷新一次索引文件，看源码：

```go
func FlushIndexLoop() {
	ticker := time.NewTicker(time.Second * 60 * 5)
	for range ticker.C {
		storage.FlushIndex()
	}
}
```

#### 6. 启动HttpServer

如果设置了`http_listen_address`配置选项，那么会启动HttpServer，用于获取服务器状态信息。

支持两个接口：

- `/summary`：包括GO协程数量、请求总数、单聊消息总数、群组消息总数
- `/stack`：打印当前堆栈信息

#### 7. 监听SyncClient

TODO

监听是否有来自从服务器的同步数据请求。

#### 8. 监听RPCClient

TODO

监听来自IM服务器的RPC请求，来自IM服务器的RPC请求有：

- SyncMessage
- SyncGroupMessage
- SavePeerMessage
- SaveGroupMessage
- GetNewCount
- GetLatestMessage

