# 客户端连接

客户端连接逻辑涉及如下几个文件：

- `connection.go`
- `peer_client.go`
- `group_client.go`
- `room_client.go`
- `client.go`

### Connection

```go
type Connection struct {
	conn   interface{} //底层连接对象，两种可能：net.Conn 或 engineio.Conn
	closed int32 //客户端连接是否关闭
	
	forbidden int32 //是否被禁言
	notification_on bool //桌面在线时是否通知手机端
	online bool //是否在线

	sync_count int64 //私聊消息同步计数，用于判断是否是首次同步
	tc     int32 //write channel timeout/error count
	wt     chan *Message
	lwt    chan int
	pwt    chan []*Message //离线消息
	
	version int //客户端协议版本号

	tm     time.Time
	appid  int64
	uid    int64
	device_id string
	platform_id int8
	device_ID int64 //根据device_id和platform_id生成的唯一ID
	
	messages *list.List //待发送的消息队列 FIFO
	mutex  sync.Mutex
}
```

**字段说明：**

- conn：底层连接对象，有两种可能的类型**net.Conn**或**engineio.Conn**
- closed：客户端连接是否关闭
- forbidden：当前用户是否被禁言
- notification_on：桌面在线时是否通知手机端
- online：表示用户不再接受推送通知(apns, gcm)（online=!(notification_on&&!is_mobile)）
- sync_count：TODO
- tc：底层连接超时或失败的次数，只要出现一次超时或失败，不会再向该连接写入数据
- wt：
- lwt：
- pwt：
- version：客户端协议版本号
- tm：
- appid：应用ID
- uid：用户ID
- device_id：设备字符串
- platform_id：平台类型（1-iOS，2-Android，3-WEB）
- device_ID：根据device_id和platform_id生成的唯一ID
- messages：异步待写入的消息队列

**主要方法：**

- read：从客户端连接读取一条消息（Message）
- send：将消息（Message）写入到客户端
- close：关闭底层连接
- EnqueueMessage：将消息发送到wt通道
- EnqueueMessages：将消息列表发送到pwt通道
- EnqueueNonBlockMessage：将消息放入messages队列，如果队列长度超过1000，那么移除对头消息并插入该消息，同时通过lwt通道通知写入异步消息
- SendMessage：


