# 消息路由

消息路由主要涉及三个文件：`route.go`、`app_route.go`、`channel.go`。


### Route

```go
type Route struct {
	appid        int64
	mutex        sync.Mutex
	clients      map[int64]ClientSet
	room_clients map[int64]ClientSet
}
```

**主要字段：**

- appid：应用ID
- clients：应用内所有上线的客户端
- room_clients：应用内的所有进入聊天室的客户端

**主要方法：**

- AddClient：当客户端上线连接到IM服务器时，将其添加到clients集合
- RemoveClient：当客户端下线关闭连接后，将其从clients集合删除
- FindClientSet：找出某用户所有上线的客户端连接
- AddRoomClient：当客户端进入某聊天室后，将其连接添加进room_clients集合
- RemoveRoomClient：当客户端退出聊天室后，将其从room_clients集合删除
- FindRoomClientSet：找出某聊天室的所有客户端连接
- IsOnline：判断用户是否不再接受推送通知，有一个或多个客户端online的用户返回true
- GetUserIDs：返回该应用当前上线的所有用户集合

### AppRoute

```go
type AppRoute struct {
	mutex sync.Mutex
	apps  map[int64]*Route
}
```

Route维护了某个应用内所有上线客户端的一个连接情况；

AppRoute则是维护了所有应用的所有上线客户端的连接。

**主要方法**

- NewAppRoute：创建一个AppRoute实例对象。
- AddRoute：添加一个应用路由。
- FindRoute：查找appid对应的应用路由。
- FindOrAddRoute：如果存在appid对应的路由对象，那么找到并返回，否则创建一个新的应用路由。

### Channel

```go
type Subscriber struct {
	uids map[int64]int
	room_ids map[int64]int
}
type Channel struct {
	addr            string
	wt              chan *Message

	mutex           sync.Mutex
	subscribers     map[int64]*Subscriber

	dispatch        func(*AppMessage)
	dispatch_group  func(*AppMessage)
	dispatch_room   func(*AppMessage)
}
```

**Subscriber**

当前IM服务器上已经连接的用户和聊天室的数量情况。

- uids：uid对应的value值高16位表示online数量，低16位表示连接总数量
- room_ids：room_id对应的value值高16位表示online数量，低16位表示连接总数量

**Channel**

Channel表示当前IM服务器与IMR路由服务器的连接，每个Channel里面都维护了一个subscribers映射，其键位appid。

- addr：IMR路由服务器的地址
- wt：IM把要发送到IMR的消息写入wt通道，另外有一个协程会处理发送到IMR服务器的任务
- subscribers：通过subscribers映射就能查找某个用户是否上线，该用户有多少个客户端正在连接IM服务器
- dispatch：分发单聊消息方法
- dispatch_group：分发群组消息方法
- dispatch_room：分发聊天室消息方法

### 主要方法

**Publish**

```go
func (channel *Channel) Publish(amsg *AppMessage) {
	msg := &Message{cmd: MSG_PUBLISH, body: amsg}
	channel.wt <- msg
}
```

向IMR发送一个MSG_PUBLISH类型消息，通过IMR路由发送AppMessage单聊消息。

**PublishGroup**

```go
func (channel *Channel) PublishGroup(amsg *AppMessage) {
	msg := &Message{cmd: MSG_PUBLISH_GROUP, body: amsg}
	channel.wt <- msg
}
```

向IMR发送一个MSG_PUBLISH_GROUP类型消息，通过IMR路由发送AppMessage群组消息。

**PublishRoom**

```go
func (channel *Channel) PublishRoom(amsg *AppMessage) {
	msg := &Message{cmd: MSG_PUBLISH_ROOM, body: amsg}
	channel.wt <- msg
}
```

向IMR发送一个MSG_PUBLISH_ROOM类型消息，通过IMR路由发送AppMessage聊天室消息。

**路由订阅与取消**

以下几个方法只是修改或读取subscribers映射数据，不会与IMR进行通信：

- AddSubscribe
- RemoveSubscribe
- GetAllSubscriber
- AddSubscribeRoom
- RemoveSubscribeRoom
- GetAllRoomSubscriber

```go
func (channel *Channel) Subscribe(appid int64, uid int64, online bool) {
	count, online_count := channel.AddSubscribe(appid, uid, online)
	if count == 0 {
		//新用户上线
		on := 0
		if online {
			on = 1
		}
		id := &SubscribeMessage{appid: appid, uid: uid, online:int8(on)}
		msg := &Message{cmd: MSG_SUBSCRIBE, body: id}
		channel.wt <- msg
	} else if online_count == 0 && online {
		//手机端上线
		id := &SubscribeMessage{appid: appid, uid: uid, online:1}
		msg := &Message{cmd: MSG_SUBSCRIBE, body: id}
		channel.wt <- msg
	}
}
```

```go
func (channel *Channel) Unsubscribe(appid int64, uid int64, online bool) {
	count, online_count := channel.RemoveSubscribe(appid, uid, online)
	if count == 1 {
		//用户断开全部连接
		id := &AppUserID{appid: appid, uid: uid}
		msg := &Message{cmd: MSG_UNSUBSCRIBE, body: id}
		channel.wt <- msg
	} else if count > 1 && online_count == 1 && online {
		//手机端断开连接,pc/web端还未断开连接
		id := &SubscribeMessage{appid: appid, uid: uid, online:0}
		msg := &Message{cmd: MSG_SUBSCRIBE, body: id}
		channel.wt <- msg		
	}
}
```

```go
func (channel *Channel) ReSubscribe(conn *net.TCPConn, seq int) int {
	subs := channel.GetAllSubscriber()
	for appid, sub := range(subs) {
		for uid, count := range(sub.uids) {
			//低16位表示总数量 高16位表示online的数量
			c2 := count>>16&0xffff
			on := 0
			if c2 > 0 {
				on = 1
			}
			id := &SubscribeMessage{appid: appid, uid: uid, online:int8(on)}
			msg := &Message{cmd: MSG_SUBSCRIBE, body: id}
			seq = seq + 1
			msg.seq = seq
			SendMessage(conn, msg)
		}
	}
	return seq
}
```

```go
func (channel *Channel) SubscribeRoom(appid int64, room_id int64) {
	count := channel.AddSubscribeRoom(appid, room_id)
	if count == 0 {
		id := &AppRoomID{appid: appid, room_id: room_id}
		msg := &Message{cmd: MSG_SUBSCRIBE_ROOM, body: id}
		channel.wt <- msg
	}
}
```

```go
func (channel *Channel) UnsubscribeRoom(appid int64, room_id int64) {
	count := channel.RemoveSubscribeRoom(appid, room_id)
	if count == 1 {
		id := &AppRoomID{appid: appid, room_id: room_id}
		msg := &Message{cmd: MSG_UNSUBSCRIBE_ROOM, body: id}
		channel.wt <- msg
	}
}
```

```go
func (channel *Channel) ReSubscribeRoom(conn *net.TCPConn, seq int) int {
	subs := channel.GetAllRoomSubscriber()
	for _, id := range(subs) {
		msg := &Message{cmd: MSG_SUBSCRIBE_ROOM, body: id}
		seq = seq + 1
		msg.seq = seq
		SendMessage(conn, msg)
	}
	return seq
}
```

