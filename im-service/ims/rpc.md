# 存储服务RPC

im-service使用的RPC为：[valyala/gorpc](https://github.com/valyala/gorpc)

与存储服务进行RPC交互的是各个IM服务器：

- IM服务器在收到客户端的消息后，需要进行消息存储：**SavePeerMessage**，**SaveGroupMessage**
- 客户端需要知道有多少条新消息：**GetNewCount**
- 客户端需要获取最近最新消息：**GetLatestMessage**
- 客户端需要同步拉取历史消息：**SyncMessage**，**SyncGroupMessage**

```go
func ListenRPCClient() {
	dispatcher := gorpc.NewDispatcher()
	dispatcher.AddFunc("SavePeerMessage", SavePeerMessage)
	dispatcher.AddFunc("SaveGroupMessage", SaveGroupMessage)
	dispatcher.AddFunc("GetNewCount", GetNewCount)
	dispatcher.AddFunc("GetLatestMessage", GetLatestMessage)
	dispatcher.AddFunc("SyncMessage", SyncMessage)
	dispatcher.AddFunc("SyncGroupMessage", SyncGroupMessage)
	
	s := &gorpc.Server{
		Addr: config.rpc_listen,
		Handler: dispatcher.NewHandlerFunc(),
	}

	if err := s.Serve(); err != nil {
		log.Fatalf("Cannot start rpc server: %s", err)
	}
}
```

先来看几个参数类型：SyncHistory、SyncGroupHistory、HistoryRequest

```go
type SyncHistory struct {
	AppID     int64
	Uid       int64
	DeviceID  int64
	LastMsgID int64
}
type SyncGroupHistory struct {
	AppID     int64
	Uid       int64
	DeviceID  int64
	GroupID   int64
	LastMsgID int64
	Timestamp int32
}
type HistoryRequest struct {
	AppID     int64
	Uid       int64
	Limit     int32
}
```

**HistoryRequest**：没有DeviceID字段，无法记录用户最近消息ID，只能用于网页版拉取最新的消息。

### SavePeerMessage

存储私聊消息，返回此条消息对应的消息ID（这里是指消息本身位置，而非元数据位置）。

```go
func SavePeerMessage(addr string, m *PeerMessage) (int64, error) {
	atomic.AddInt64(&server_summary.nrequests, 1)
	atomic.AddInt64(&server_summary.peer_message_count, 1)
	msg := &Message{cmd:int(m.Cmd), version:DEFAULT_VERSION}
	msg.FromData(m.Raw)
	msgid := storage.SavePeerMessage(m.AppID, m.Uid, m.DeviceID, msg)
	return msgid, nil
}
```

PeerMessage结构参见：[消息类型](message_type.html)

### SaveGroupMessage

存储群组消息，返回此条消息对应的消息ID（这里是指消息本身位置，而非元数据位置）。

```go
func SaveGroupMessage(addr string, m *GroupMessage) (int64, error) {
	atomic.AddInt64(&server_summary.nrequests, 1)
	atomic.AddInt64(&server_summary.group_message_count, 1)
	msg := &Message{cmd:int(m.Cmd), version:DEFAULT_VERSION}
	msg.FromData(m.Raw)
	msgid := storage.SaveGroupMessage(m.AppID, m.GroupID, m.DeviceID, msg)
	return msgid, nil
}
```

GroupMessage结构参见：[消息类型](message_type.html)

### GetNewCount

```go
func GetNewCount(addr string, s *SyncHistory) (int64, error)
```

获取用户最新未读消息数量，从后往前遍历用户的消息链表，直到参数**SyncHistory.LastMsgID**。

### GetLatestMessage

```go
func GetLatestMessage(addr string, r *HistoryRequest) []*HistoryMessage
```

获取用户最近的**HistoryRequest.limit**条消息，返回数据类型为`[]*HistoryMessage`。

### SyncMessage

```go
func SyncMessage(addr string, sync_key *SyncHistory) *PeerHistoryMessage
```

客户端同步拖取消息，客户端会提供用户的**SyncHistory.lastMsgID**，存储服务器根据这个ID去同步最新达到的消息。

不仅返回一个列表**[]\*HistoryMessage**，同时返回用户消息链表中的**LastMsgID**。

PeerHistoryMessage结构参见：[消息类型](message_type.html)

### SyncGroupMessage

```go
func SyncGroupMessage(addr string , sync_key *SyncGroupHistory) *GroupHistoryMessage
```

客户端同步拖取群组消息，客户端会提供用户的**SyncGroupHistory.LastMsgID**，存储服务器根据这个ID去同步最新达到的消息。

同时，**SyncGroupHistory.Timestamp**表示用户的入群时间，不会拖取比入群时间更早的历史消息。

不仅返回一个列表**[]\*HistoryMessage**，同时返回用户消息链表中的**LastMsgID**。

GroupHistoryMessage结构参见：[消息类型](message_type.html)



