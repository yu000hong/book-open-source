# IM终端服务器

### DispatchAppMessage & DispatchGroupMessage & DispatchRoomMessage

这三个方法是IM终端服务器根据自己的路由信息，将消息发送到对应的终端

### FilterDirtyWord

敏感词过滤，配置参数为：**word_file**。如果配置了过滤词文件，那么将开启敏感词过滤功能。

敏感词过滤功能的实现使用了函数库：[importcjj/sensitive](https://github.com/importcjj/sensitive)

### SendAppMessage & SendAppGroupMessage

这两个方法就是根据当前IM的路由信息以及IMR的路由信息将消息发送到对应的终端。

### SaveMessage & SaveGroupMessage

这两个方法是将消息发送到IMS存储服务器进行消息的持久化，并返回消息持久化之后的msgid

### PublishMessage & PublishGroupMessage

这两个方法将消息发送到IMR路由服务器。

### PushMessage & PushGroupMessage

```go
//离线消息推送
func PushMessage(appid int64, uid int64, m *Message) {
	PublishMessage(appid, uid, m)
}
```

```go
//超级群，离线消息推送
func PushGroupMessage(appid int64, group_id int64, m *Message) {
	now := time.Now().UnixNano()
	amsg := &AppMessage{appid:appid, receiver:group_id, msgid:0, timestamp:now, msg:m}
	group := group_manager.FindGroup(amsg.receiver)
	if group == nil {
		return
	}
	channels := make(map[*Channel]struct{})
	members := group.Members()
	for member := range members {
		channel := GetChannel(member)
		if _, ok := channels[channel]; !ok {
			channels[channel] = struct{}{}
		}
	}
	for channel, _ := range channels {
		channel.Publish(amsg)
	}
}
```

这两个方法都是用于离线消息推送。

PushMessage和PublishMessage作用一致，将消息发送到IMR路由服务器。

PushGroupMessage并没有直接调用PublishGroupMessage，而是遍历群组里的所有成员，再逐一调用PublishMessage进行离线消息的推送！！！

### GetChannel & GetGroupChannel & GetRoomChannel

选择需要交互的IMR路由服务器。

GetChannel和GetRoomChannel由参数`route_addrs`配置，根据uid或room_id取模选择对应的IMR；

GetGroupChannel由参数`group_route_addrs`配置，根据gid取模选择对应的IMR；

如果不配置`group_route_addrs`参数，那么直接使用`route_addrs`参数的值。

### GetStorageRPCClient & GetGroupStorageRPCClient

选择需要交互的IMS存储服务器。

GetStorageRPCClient：获取"单聊消息/普通群聊消息/客服消息"的IMS存储服务器，由参数`storage_rpc_addrs`进行配置

GetGroupStorageRPCClient：获取超级群消息的IMS存储服务器，由参数`group_storage_rpc_addrs`进行配置

### SyncKeyService

一个单独的协程来处理终端上报的SyncKey（包括用户的和群组的），将其保存在Redis中。

SyncKey的值为：终端收到的最新消息的ID（lastMsgId）,这个值只能往上增大，不能减小！

```go
func SyncKeyService() {
	for {
		select {
		case s := <- sync_c:
			origin := GetSyncKey(s.AppID, s.Uid)
			if s.LastMsgID > origin {
				SaveSyncKey(s.AppID, s.Uid, s.LastMsgID)
			}
			break
		case s := <- group_sync_c:
			origin := GetGroupSyncKey(s.AppID, s.Uid, s.GroupID)
			if s.LastMsgID > origin {
				SaveGroupSyncKey(s.AppID, s.Uid, s.GroupID, s.LastMsgID)
			}
			break
		}
	}
}
```
