# 聊天室

处理聊天室相应功能的是`RoomClient`，它继承自`Connection`。

一个终端一次最多只能进入一个聊天室，RoomClient会保存字段：**room_id**。

```go
type RoomClient struct {
	*Connection
	room_id int64
}
```

### 终端消息处理

```go
func (client *Client) HandleMessage(msg *Message) {
	switch msg.cmd {
	case MSG_AUTH_TOKEN:
		client.HandleAuthToken(msg.body.(*AuthenticationToken), msg.version)
	case MSG_ACK:
		client.HandleACK(msg.body.(*MessageACK))
	case MSG_PING:
		client.HandlePing()
	}
	client.PeerClient.HandleMessage(msg)
	client.GroupClient.HandleMessage(msg)
	client.RoomClient.HandleMessage(msg)
	client.CustomerClient.HandleMessage(msg)
}
```

从代码可以看出，IM服务器接收到终端发来的消息后，会分发到四种类型的Client：

- PeerClient
- GroupClient
- RoomClient
- CustomerClient

具体的不同类型Client是否需要处理某种类型的消息，由其自己决定。

**RoomClient**只处理三种消息类型：

* MSG_ENTER_ROOM：进入聊天室
* MSG_LEAVE_ROOM：离开聊天室
* MSG_ROOM_IM：聊天室消息

```go
func (client *RoomClient) HandleMessage(msg *Message) {
	switch msg.cmd {
	case MSG_ENTER_ROOM:
		client.HandleEnterRoom(msg.body.(*Room))
	case MSG_LEAVE_ROOM:
		client.HandleLeaveRoom(msg.body.(*Room))
	case MSG_ROOM_IM:
		client.HandleRoomIM(msg.body.(*RoomMessage), msg.seq)
	}
}
```

### HandleEnterRoom

```go
func (client *RoomClient) HandleEnterRoom(room *Room){
	if client.uid == 0 {
        //如果终端没有进行登录认证，那么do nothing
		log.Warning("client has't been authenticated")
		return
	}
	room_id := room.RoomID()
	if room_id == 0 || client.room_id == room_id {
        //如果room_id不合法或者终端已经在这个聊天室了，那么do nothing
		return
	}
	route := app_route.FindOrAddRoute(client.appid)
	if client.room_id > 0 {
        //如果终端已经在另外一个聊天室了，那么必须先退出那个聊天室
        channel := GetRoomChannel(client.room_id)
        //告知IMR路由服务器退出聊天室，维护IMR的聊天室路由信息
        channel.UnsubscribeRoom(client.appid, client.room_id)
        //修改当前IM服务器上的聊天室路由信息
		route.RemoveRoomClient(client.room_id, client.Client())
	}
    //进入新的聊天室
    client.room_id = room_id
    //修改当前IM服务器上的聊天室路由信息
	route.AddRoomClient(client.room_id, client.Client())
    channel := GetRoomChannel(client.room_id)
    //修改IMR服务器上的聊天室路由信息
	channel.SubscribeRoom(client.appid, client.room_id)
}
```

### HandleLeaveRoom

```go
func (client *RoomClient) HandleLeaveRoom(room *Room) {
	if client.uid == 0 {
        //如果终端没有进行登录认证，那么do nothing
		log.Warning("client has't been authenticated")
		return
	}

	room_id := room.RoomID()
	if room_id == 0 {
        //如果room_id不合法，那么do nothing
		return
	}
	if client.room_id != room_id {
        //如果当前所在聊天室不是准备要退出的聊天室，do nothing
		return
	}
    route := app_route.FindOrAddRoute(client.appid)
    //修改当前IM服务器上的聊天室路由信息
	route.RemoveRoomClient(client.room_id, client.Client())
	channel := GetRoomChannel(client.room_id)
    //修改IMR服务器上的聊天室路由信息
    channel.UnsubscribeRoom(client.appid, client.room_id)
    //重置room_id为0，表明已退出聊天室
	client.room_id = 0
}
```

### HandleRoomIM

```go
func (client *RoomClient) HandleRoomIM(room_im *RoomMessage, seq int) {
	if client.uid == 0 {
        //校验是否登录验证
		return
	}
	room_id := room_im.receiver
	if room_id != client.room_id {
        //校验是否非法room_id
        client.room_id)
		return
	}

	fb := atomic.LoadInt32(&client.forbidden) 
	if (fb == 1) {
        //校验用户是否被禁言
		return
	}
    //将聊天室消息直接转发到连接当前IM服务器的所有进入聊天室的终端
	m := &Message{cmd:MSG_ROOM_IM, body:room_im}
	route := app_route.FindOrAddRoute(client.appid)
	clients := route.FindRoomClientSet(room_id)
	for c, _ := range(clients) {
		if c == client.Client() {
			continue
		}
		c.EnqueueNonBlockMessage(m)
	}
    //将聊天室消息通过IMR路由服务器分发到其他进入聊天室的终端
	amsg := &AppMessage{appid:client.appid, receiver:room_id, msg:m}
	channel := GetRoomChannel(client.room_id)
	channel.PublishRoom(amsg)
    //返回给发送终端一个ACK回复
	client.wt <- &Message{cmd: MSG_ACK, body: &MessageACK{int32(seq)}}
}
```

### Logout

当终端连接断开的时候，同时会调用`RoomClient.Logout()`，以退出聊天室。

Logout方法的作用就是退出聊天室，与HandleLeaveRoom比起来只是少做了一些不必要的判断。

```go
func (client *RoomClient) Logout() {
	if client.room_id > 0 {
		channel := GetRoomChannel(client.room_id)
		channel.UnsubscribeRoom(client.appid, client.room_id)
		route := app_route.FindOrAddRoute(client.appid)
		route.RemoveRoomClient(client.room_id, client.Client())		
	}
}
```

