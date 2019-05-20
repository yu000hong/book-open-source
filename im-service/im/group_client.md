# 群聊消息

处理群聊消息的是`GroupClient`，它继承自`Connection`。

```go
type GroupClient struct {
	*Connection
}
```

### 消息处理

**GroupClient**只处理三种类型的消息：

* MSG_GROUP_IM：
* MSG_SYNC_GROUP：
* MSG_GROUP_SYNC_KEY：

```go
func (client *GroupClient) HandleMessage(msg *Message) {
	switch msg.cmd {
	case MSG_GROUP_IM:
		client.HandleGroupIMMessage(msg)
	case MSG_SYNC_GROUP:
		client.HandleGroupSync(msg.body.(*GroupSyncKey))
	case MSG_GROUP_SYNC_KEY:
		client.HandleGroupSyncKey(msg.body.(*GroupSyncKey))
	}
}
```

### HandleGroupIMMessage

```go
func (client *GroupClient) HandleGroupIMMessage(message *Message) {
	msg := message.body.(*IMMessage)
	seq := message.seq		
	if client.uid == 0 {
		//校验是否登录认证
		return
	}
	if msg.sender != client.uid {
		//校验uid是否一致
		return
	}
	if message.flag & MESSAGE_FLAG_TEXT != 0 {
		//如果是文本消息，那么进行文本过滤
		FilterDirtyWord(msg)
	}
	msg.timestamp = int32(time.Now().Unix())

	group := group_manager.FindGroup(msg.receiver)
	if group == nil {
		//没有找到对应群组
		return
	}
	if !group.IsMember(msg.sender) {
		//没在群组里，不能发送群组消息
		return
	}
	if group.GetMemberMute(msg.sender) {
		//被禁言的群组成员不能发送群组消息
		return
	}
	if group.super {
		//超级群的处理逻辑
		client.HandleSuperGroupMessage(msg)
	} else {
		//普通群的处理逻辑
		client.HandleGroupMessage(msg, group)
	}
    //返回给发送终端一个ACK回复
	ack := &Message{cmd: MSG_ACK, body: &MessageACK{int32(seq)}}
	r := client.EnqueueMessage(ack)
	if !r {
		log.Warning("send group message ack error")
	}
	atomic.AddInt64(&server_summary.in_message_count, 1)
}
```

### HandleGroupMessage

```go
func (client *GroupClient) HandleGroupMessage(im *IMMessage, group *Group) {
	gm := &PendingGroupMessage{}
	gm.appid = client.appid
	gm.sender = im.sender	
	gm.device_ID = client.device_ID
	gm.gid = im.receiver
	gm.timestamp = im.timestamp
	
	members := group.Members()
	gm.members = make([]int64, len(members))
	i := 0
	for uid := range members {
		gm.members[i] = uid
		i += 1
	}

	gm.content = im.content
	deliver := GetGroupMessageDeliver(group.gid)
	m := &Message{cmd:MSG_PENDING_GROUP_MESSAGE, body: gm}
	deliver.SaveMessage(m)
}
```

### HandleSuperGroupMessage

```go
func (client *GroupClient) HandleSuperGroupMessage(msg *IMMessage) {
	m := &Message{cmd: MSG_GROUP_IM, version:DEFAULT_VERSION, body: msg}
	//将消息发送到IMS存储服务器进行存储
	msgid, err := SaveGroupMessage(client.appid, msg.receiver, client.device_ID, m)
	if err != nil {
		return
	}
	//推送外部通知
	PushGroupMessage(client.appid, msg.receiver, m)
	//发送同步的通知消息
	notify := &Message{cmd:MSG_SYNC_GROUP_NOTIFY, body:&GroupSyncKey{group_id:msg.receiver, sync_key:msgid}}
	client.SendGroupMessage(msg.receiver, notify)
}
```

### HandleGroupSync

根据终端提供的lastId（如果终端没有提供，那么取Redis中保存的值）去IMS存储服务器取最新未读取的消息列表，读取之后直接返回到此终端。

返回的数据流为：

- MSG_SYNC_GROUP_BEGIN
- ...(群聊消息)
- MSG_SYNC_GROUP_END

### HandleGroupSyncKey

终端会通过MSG_GROUP_SYNC_KEY消息来设置用户某个群组的最近消息ID，最终会保存到Redis Hash里（KEY：users_#{appid}_#{uid}，HASH-KEY：group_sync_key_#{gid}）。

```go
func (client *GroupClient) HandleGroupSyncKey(group_sync_key *GroupSyncKey) {
	if client.uid == 0 {
		//校验是否认证登录
		return
	}
	//设置用户的群组消息lastId
	group_id := group_sync_key.group_id
	last_id := group_sync_key.sync_key
	if last_id > 0 {
		s := &SyncGroupHistory{
			AppID:client.appid, 
			Uid:client.uid, 
			GroupID:group_id, 
			LastMsgID:last_id,
		}
		group_sync_c <- s
	}
}
```