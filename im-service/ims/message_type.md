# 消息类型

### Message

```go
type Message struct {
	cmd  int
	seq  int
	version int
	flag int
	
	body interface{}
}
```

`Message`是最终存储到文件里面的消息格式，我们再来看看`storage_file.go`文件里最终写入文件的方法：

```go
func (storage *StorageFile) SaveMessage(msg *Message) int64
```

调用**SaveMessage**方法的上层方法为：

- SavePeerMessage
- SaveGroupMessage

**SavePeerMessage**

```go
func (storage *PeerStorage) SavePeerMessage(appid int64, uid int64, device_id int64, msg *Message) int64 {
	storage.mutex.Lock()
	defer storage.mutex.Unlock()
	msgid := storage.saveMessage(msg)

	last_id, last_peer_id := storage.getLastMessageID(appid, uid)

	off := &OfflineMessage2{appid:appid, receiver:uid, msgid:msgid, device_id:device_id, prev_msgid:last_id, prev_peer_msgid:last_peer_id}
	
	var flag int
	if storage.isGroupMessage(msg) {
		flag = MESSAGE_FLAG_GROUP
	}
	m := &Message{cmd:MSG_OFFLINE_V2, flag:flag, body:off}
	last_id = storage.saveMessage(m)

	if !storage.isGroupMessage(msg) {
		last_peer_id = last_id
	}
	storage.setLastMessageID(appid, uid, last_id, last_peer_id)
	return msgid
}
```

**SaveGroupMessage**

```go
func (storage *GroupStorage) SaveGroupMessage(appid int64, gid int64, device_id int64, msg *Message) int64 {
	storage.mutex.Lock()
	defer storage.mutex.Unlock()

	msgid := storage.saveMessage(msg)

	last_id, _ := storage.getLastGroupMessageID(appid, gid)
	lt := &GroupOfflineMessage{appid:appid, gid:gid, msgid:msgid, device_id:device_id, prev_msgid:last_id}
	m := &Message{cmd:MSG_GROUP_IM_LIST, body:lt}
	
	last_id = storage.saveMessage(m)
	storage.setLastGroupMessageID(appid, gid, last_id)
	return msgid
}
```

每条消息到IMS之后，都会存储两条Message，一条为消息本身，其后跟了一条元数据消息。

- 私聊消息：元数据消息类型为`MSG_OFFLINE_V2`，保存了应用ID、接收用户UID、设备ID、消息ID、上一条消息ID、上一条私聊消息ID这些元数据
- 群组消息：元数据消息类型为`MSG_GROUP_IM_LIST`，保存了应用ID、群组ID、设备ID、消息ID、上一条群组消息ID这些元数据

### EMessage

```go
type EMessage struct {
	msgid int64
	device_id int64
	msg   *Message
}
```

EMessage在Message的基础上多了两个字段：msgid、device_id。

存储在消息文件的消息类型是Message，除了消息本身外，还包括元数据。

读取消息时，不仅要读取消息本身数据，还需要读取元数据，所以使用了EMessage类型。

### HistoryMessage

```go
type HistoryMessage struct {
	MsgID     int64
	DeviceID  int64   //消息发送者所在的设备ID
	Cmd       int32
	Raw       []byte
}
```

**Raw**：为Message类型序列化之后的字节数据；

**Cmd**：是从Message中提取出来的命令；

可以看出，**HistoryMessage**和**EMessage**数据是等价的，不过EMessage是在IMS内部使用的类型，而HistoryMessage是RPC调用需要用到的类型。

### PeerHistoryMessage & GroupHistoryMessage

```go
type PeerHistoryMessage struct {
	Messages []*HistoryMessage
	LastMsgID int64
}
type GroupHistoryMessage PeerHistoryMessage
```

从代码可以看出，这两个类型互为别名，是同一个类型，都是用于客户端同步拉取历史消息RPC的返回值。

```go
func SyncMessage(addr string, sync_key *SyncHistory) *PeerHistoryMessage
func SyncGroupMessage(addr string , sync_key *SyncGroupHistory) *GroupHistoryMessage
```

### OfflineMessage2 vs OfflineMessage

对应的cmd值为：

```go
//个人消息队列 代替MSG_OFFLINE
const MSG_OFFLINE_V2 = 250  
//deprecated 兼容性
const MSG_OFFLINE = 254
```

V2版本的消息多了一个`prev_peer_msgid`字段，多了一个私聊消息指针，可以只遍历所有私聊消息！

### PeerMessage

```go
type PeerMessage struct {
	AppID     int64
	Uid       int64
	DeviceID  int64
	Cmd       int32
	Raw       []byte
}
```

私聊消息，用于RPC调用SavePeerMessage，Raw字段为Message类型序列化后的字节数据。

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

### GroupMessage

```go
type GroupMessage struct {
	AppID     int64
	GroupID   int64
	DeviceID  int64
	Cmd       int32
	Raw       []byte
}
```

群组消息，用于RPC调用SaveGroupMessage，Raw字段为Message类型序列化后的字节数据。

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

### HistoryMessage

```go
type HistoryMessage struct {
	MsgID     int64
	DeviceID  int64   //消息发送者所在的设备ID
	Cmd       int32
	Raw       []byte
}
type PeerHistoryMessage struct {
	Messages []*HistoryMessage
	LastMsgID int64
}
type GroupHistoryMessage PeerHistoryMessage
```

