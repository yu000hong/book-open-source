# 存储服务RPC

im-service使用的RPC为：[valyala/gorpc](https://github.com/valyala/gorpc)

与存储服务进行RPC交互的是各个IM服务器：

- 各个IM服务器在接收到客户端的消息后，需要进行消息存储
- 客户端需要知道有多少条新消息
- 客户端需要获取最近最新消息
- 客户端需要同步消息

```go
func ListenRPCClient() {
	dispatcher := gorpc.NewDispatcher()
	dispatcher.AddFunc("SyncMessage", SyncMessage)
	dispatcher.AddFunc("SyncGroupMessage", SyncGroupMessage)
	dispatcher.AddFunc("SavePeerMessage", SavePeerMessage)
	dispatcher.AddFunc("SaveGroupMessage", SaveGroupMessage)
	dispatcher.AddFunc("GetNewCount", GetNewCount)
	dispatcher.AddFunc("GetLatestMessage", GetLatestMessage)
	
	s := &gorpc.Server{
		Addr: config.rpc_listen,
		Handler: dispatcher.NewHandlerFunc(),
	}

	if err := s.Serve(); err != nil {
		log.Fatalf("Cannot start rpc server: %s", err)
	}
}
```

### SyncMessage

客户端同步拖取消息，客户端会提供用户的lastMessageId，存储服务器根据这个ID去同步最新达到的消息。

```go
func SyncMessage(addr string, sync_key *SyncHistory) *PeerHistoryMessage {
	return nil
}
type SyncHistory struct {
	AppID     int64
	Uid       int64
	DeviceID  int64
	LastMsgID int64
}
```

