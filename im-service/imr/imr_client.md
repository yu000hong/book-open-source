# IM客户端

`client.go`文件定义了**Client**类型，这个类型代表了一个IM服务器到当前IMR服务器的连接。

### 数据结构

```go
type Push struct {
	queue_name  string
	content     []byte
}
type Client struct {
	conn        *net.TCPConn
	app_route   *AppRoute
	wt          chan *Message
	pwt         chan *Push
}
```

**conn**：与IM服务器的底层连接

**app_route**：维护了与各个IM服务器连接的所有用户及聊天室信息

**wt**：分发消息的通道，IMR将消息分发到IM服务器对应的wt通道（后台协程会将消息发送到IM服务器）

**pwt**：消息推送的通道，参见[消息推送](push.html)

### 启动Client

```go
func NewClient(conn *net.TCPConn) *Client {
	client := new(Client)
	client.conn = conn
	client.pwt = make(chan *Push, 10000)
	client.wt = make(chan *Message, 10)
	client.app_route = NewAppRoute()
	return client
}
func (client *Client) Run() {
	go client.Write()
	go client.Read()
	go client.Push()
}
```

**Write()**：Write方法负责将wt通道收到的聊天消息写入到对应的底层连接；

**Push()**：Push方法负责将pwt通道收到的推送消息写入到Redis队列，参见[消息推送](push.html)；

**Read()**：Read方法负责接收IM服务器发送的消息，然后分派到不同处理器进行处理。

### HandleMessage

HandleMessage负责将收到的IM服务器消息进行分发，看代码：

```go
func (client *Client) HandleMessage(msg *Message) {
	switch msg.cmd {
	case MSG_SUBSCRIBE:
		client.HandleSubscribe(msg.body.(*SubscribeMessage))
	case MSG_UNSUBSCRIBE:
		client.HandleUnsubscribe(msg.body.(*AppUserID))
	case MSG_SUBSCRIBE_ROOM:
		client.HandleSubscribeRoom(msg.body.(*AppRoomID))
	case MSG_UNSUBSCRIBE_ROOM:
		client.HandleUnsubscribeRoom(msg.body.(*AppRoomID))
	case MSG_PUBLISH:
		client.HandlePublish(msg.body.(*AppMessage))
	case MSG_PUBLISH_GROUP:
		client.HandlePublishGroup(msg.body.(*AppMessage))
	case MSG_PUBLISH_ROOM:
		client.HandlePublishRoom(msg.body.(*AppMessage))
	default:
		log.Warning("unknown message cmd:", msg.cmd)
	}
}
```

**HandleSubscribe/Unsubscribe**：维护Route里面的uids映射

**HandleSubscribeRoom/UnsubscribeRoom**：维护Route里面的room_ids映射

**HandlePublish**：将单聊消息通过IMR消息路由器分发到对应的IM服务器

```go
func (client *Client) HandlePublish(amsg *AppMessage) {
	cmd := amsg.msg.cmd
	receiver := &AppUserID{appid:amsg.appid, uid:amsg.receiver}
	s := FindClientSet(receiver)
	offline := true
	for c := range(s) {
		if c.IsAppUserOnline(receiver) {
			offline = false
		}
	}
	//如果用户没有上线，那么将单聊写入消息推送队列
	if offline {
		if cmd == MSG_IM {
			client.PublishPeerMessage(amsg.appid, amsg.msg.body.(*IMMessage))
		} else if cmd == MSG_GROUP_IM {
			client.PublishGroupMessage(amsg.appid, []int64{amsg.receiver},
				amsg.msg.body.(*IMMessage))
		} else if cmd == MSG_CUSTOMER || 
			cmd == MSG_CUSTOMER_SUPPORT {
			client.PublishCustomerMessage(amsg.appid, amsg.receiver, 
				amsg.msg.body.(*CustomerMessage), amsg.msg.cmd)
		} else if cmd == MSG_SYSTEM {
			sys := amsg.msg.body.(*SystemMessage)
			if config.is_push_system {
				client.PublishSystemMessage(amsg.appid, amsg.receiver, sys.notification)
			}
		}
	}
	//持久化的消息不主动分发到IM服务器，因为我们采用的是客户端拉取的方式进行消息同步
	if cmd == MSG_IM || cmd == MSG_GROUP_IM || 
		cmd == MSG_CUSTOMER || cmd == MSG_CUSTOMER_SUPPORT || 
		cmd == MSG_SYSTEM {
		if amsg.msg.flag & MESSAGE_FLAG_UNPERSISTENT == 0 {
			return
		}
	}
    //其他类型消息直接分发到IM服务器（TODO 还有哪些非持久化消息？）
	msg := &Message{cmd:MSG_PUBLISH, body:amsg}
	for c := range(s) {
		//不发送给自身
		if client == c {
			continue
		}
		c.wt <- msg
	}
}
```

**HandlePublishGroup**：将群聊消息通过IMR消息路由器分发到对应的IM服务器

```go
func (client *Client) HandlePublishGroup(amsg *AppMessage) {
	gid := amsg.receiver
	group := group_manager.FindGroup(gid)
	//将群聊消息推送到没有上线的用户
	if group != nil && amsg.msg.cmd == MSG_GROUP_IM {
		msg := amsg.msg
		members := group.Members()
		im := msg.body.(*IMMessage);
		off_members := make([]int64, 0)
		for uid, _ := range members {
			if im.sender != uid && !IsUserOnline(amsg.appid, uid) {
				off_members = append(off_members, uid)
			}
		}
		if len(off_members) > 0 {
			client.PublishGroupMessage(amsg.appid, off_members, im)
		}
	}
	//当前只有MSG_SYNC_GROUP_NOTIFY可以发给终端
	if amsg.msg.cmd != MSG_SYNC_GROUP_NOTIFY {
		return
	}
	//群发给所有接入服务器
	s := GetClientSet()
	msg := &Message{cmd:MSG_PUBLISH_GROUP, body:amsg}
	for c := range(s) {
		//不发送给自身
		if client == c {
			continue
		}
		c.wt <- msg
	}
}
```

**HandlePublishRoom**：将聊天室消息通过IMR消息路由器分发到对应的IM服务器

```go
func (client *Client) HandlePublishRoom(amsg *AppMessage) {
	receiver := &AppRoomID{appid:amsg.appid, room_id:amsg.receiver}
	s := FindRoomClientSet(receiver)
	msg := &Message{cmd:MSG_PUBLISH_ROOM, body:amsg}
	for c := range(s) {
		//不发送给自身
		if client == c {
			continue
		}
		log.Info("publish room message")
		c.wt <- msg
	}
}
```
