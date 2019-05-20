# 单聊消息

处理单聊消息的是`PeerClient`，它继承自`Connection`。

```go
type PeerClient struct {
	*Connection
}
```

### 消息处理

**PeerClient**只处理三种类型的消息：

* MSG_IM：
* MSG_RT：实时消息
* MSG_UNREAD_COUNT：
* MSG_SYNC：
* MSG_SYNC_KEY：

```go
func (client *PeerClient) HandleMessage(msg *Message) {
	switch msg.cmd {
	case MSG_IM:
		client.HandleIMMessage(msg)
	case MSG_RT:
		client.HandleRTMessage(msg)
	case MSG_UNREAD_COUNT:
		client.HandleUnreadCount(msg.body.(*MessageUnreadCount))
	case MSG_SYNC:
		client.HandleSync(msg.body.(*SyncKey))
	case MSG_SYNC_KEY:
		client.HandleSyncKey(msg.body.(*SyncKey))
	}
}
```

### HandleIMMessage



### HandleRTMessage

处理实时消息，直接将消息通过SendMessage发送到指定目的终端。

```go
func (client *PeerClient) HandleRTMessage(msg *Message) {
	rt := msg.body.(*RTMessage)
	if rt.sender != client.uid {
        //校验uid
		return
	}
	//使用SendMessage直接发送消息，不进行消息的持久化
	m := &Message{cmd:MSG_RT, body:rt}
	client.SendMessage(rt.receiver, m)
	atomic.AddInt64(&server_summary.in_message_count, 1)
}
```

### HandleUnreadCount

终端会通过MSG_UNREAD_COUNT消息来设置未读消息数量，最终会保存到Redis Hash里（KEY：users_appid_uid，HASH-KEY：unread）。

TODO 为何需要终端来设置这个值呢？

### HandleSync


根据终端提供的lastId（如果终端没有提供，那么取Redis中保存的值）去IMS存储服务器取最新未读取的消息列表，读取之后直接返回到此终端。

返回的数据流为：

- MSG_SYNC_BEGIN
- ...(群聊消息)
- MSG_SYNC_END

### HandleSyncKey

终端会通过MSG_SYNC_KEY消息来设置用户最近消息ID，最终会保存到Redis Hash里（KEY：users_#{appid}_#{uid}，HASH-KEY：sync_key）。





