# 客服消息

处理客服聊天功能的是`CustomerClient`，它继承自`Connection`。

```go
type CustomerClient struct {
	*Connection
}
```

### 消息处理

**CustomerClient**只处理两种类型的消息：

* MSG_CUSTOMER：顾客咨询客服（顾客->客服）
* MSG_CUSTOMER_SUPPORT：客服回复顾客（客服->顾客）

```go
func (client *CustomerClient) HandleMessage(msg *Message) {
	switch msg.cmd {
	case MSG_CUSTOMER:
		client.HandleCustomerMessage(msg)
	case MSG_CUSTOMER_SUPPORT:
		client.HandleCustomerSupportMessage(msg)
	}
}
```

### HandleCustomerMessage

顾客 --> 客服

```go
func (client *CustomerClient) HandleCustomerMessage(msg *Message) {
	cm := msg.body.(*CustomerMessage)
	cm.timestamp = int32(time.Now().Unix())
	if cm.customer_appid != client.appid {
		//校验appid
		return
	}
	if cm.customer_id != client.uid {
		//校验uid
		return
	}
	if cm.seller_id == 0 {
		//校验seller_id
		return
	}
	if (msg.flag & MESSAGE_FLAG_UNPERSISTENT) > 0 {
		//发送到IMR路由服务器，由IMR进行消息路由
		SendAppMessage(config.kefu_appid, cm.seller_id, msg)
		ack := &Message{cmd: MSG_ACK, body: &MessageACK{int32(msg.seq)}}
		client.EnqueueMessage(ack)		
		return
	}
	//将消息发送到IMS服务器，存到接收者的消息队列里
	msgid, err := SaveMessage(config.kefu_appid, cm.seller_id, client.device_ID, msg)
	if err != nil {
		return
	}
	//将消息发送到IMS服务器，存到发送者的消息队列里
	msgid2, err := SaveMessage(cm.customer_appid, cm.customer_id, client.device_ID, msg)
	if err != nil {
		return
	}
	//将消息发送到IMR路由服务器，由IMR决定是否进行推送
	PushMessage(config.kefu_appid, cm.seller_id, msg)
	//发送同步的通知消息
	notify := &Message{cmd:MSG_SYNC_NOTIFY, body:&SyncKey{msgid}}
	SendAppMessage(config.kefu_appid, cm.seller_id, notify)
	//发送给自己的其它登录点
	notify = &Message{cmd:MSG_SYNC_NOTIFY, body:&SyncKey{msgid2}}
	client.SendMessage(client.uid, notify)
    //返回给发送终端一个ACK回复
	ack := &Message{cmd: MSG_ACK, body: &MessageACK{int32(msg.seq)}}
	client.EnqueueMessage(ack)
}
```

### HandleCustomerSupportMessage

客服 --> 顾客

```go
func (client *CustomerClient) HandleCustomerSupportMessage(msg *Message) {
	cm := msg.body.(*CustomerMessage)
	if client.appid != config.kefu_appid {
		//校验appid
		return
	}
	if client.uid != cm.seller_id {
		//校验uid
		return
	}
	cm.timestamp = int32(time.Now().Unix())
	if (msg.flag & MESSAGE_FLAG_UNPERSISTENT) > 0 {	
		//发送到IMR路由服务器，由IMR进行消息路由
		SendAppMessage(cm.customer_appid, cm.customer_id, msg)
		ack := &Message{cmd: MSG_ACK, body: &MessageACK{int32(msg.seq)}}
		client.EnqueueMessage(ack)
		return
	}
	//将消息发送到IMS服务器，存到接收者的消息队列里
	msgid, err := SaveMessage(cm.customer_appid, cm.customer_id, client.device_ID, msg)
	if err != nil {
		return
	}
	//将消息发送到IMS服务器，存到发送者的消息队列里
	msgid2, err := SaveMessage(client.appid, cm.seller_id, client.device_ID, msg)
	if err != nil {
		return
	}
	//将消息发送到IMR路由服务器，由IMR决定是否进行推送
	PushMessage(cm.customer_appid, cm.customer_id, msg)
	//发送同步的通知消息
	notify := &Message{cmd:MSG_SYNC_NOTIFY, body:&SyncKey{msgid}}
	SendAppMessage(cm.customer_appid, cm.customer_id, notify)
	//发送给自己的其它登录点
	notify = &Message{cmd:MSG_SYNC_NOTIFY, body:&SyncKey{msgid2}}
	client.SendMessage(client.uid, notify)
    //返回给发送终端一个ACK回复
	ack := &Message{cmd: MSG_ACK, body: &MessageACK{int32(msg.seq)}}
	client.EnqueueMessage(ack)
}
```

