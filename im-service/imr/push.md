# 消息推送

### IsROMApp

```go
func (client *Client) IsROMApp(appid int64) bool {
	return false
}
```

始终返回false，具体意义未知（TODO）

### PushQueue

PushQueue方法是最终将推送消息写入Redis队列的唯一方法，所有其他上层方法最终都要通过它来完成。

```go
func (client *Client) PushQueue(ps []*Push) {
	conn := redis_pool.Get()
	defer conn.Close()

	begin := time.Now()	
	conn.Send("MULTI")
	for _, p := range(ps) {
		conn.Send("RPUSH", p.queue_name, p.content)
	}
	_, err := conn.Do("EXEC")
	end := time.Now()
	duration := end.Sub(begin)
	if err != nil {
		log.Info("multi rpush error:", err)
	} else {
		log.Infof("mmulti rpush:%d time:%s success", len(ps), duration)
	}
	if  duration > time.Millisecond*PUSH_QUEUE_TIMEOUT {
		log.Warning("multi rpush slow:", duration)
	}
}
```

### PushChan

将推送消息发送到pwt通道，直接上代码：

```go
func (client *Client) PushChan(queue_name string, b []byte) {
	select {
	case client.pwt <- &Push{queue_name, b}:
	default:
		log.Warning("rpush message timeout")		
	}	
}
```

### Push

这是一个后台协程，负责把pwt通道收到的推送消息写入到Redis队列（调用PushQueue）。

WAIT_TIMEOUT：500毫秒

PUSH_LIMIT：1000条

如果推送消息累积了1000条，或者等待超过500毫秒，那么调用PushQueue完成Redis队列的写入工作。

### PublishPeerMessage

推送消息写入**push_queue**或**push_queue_\#\{appid\}**队列。

```go
func (client *Client) PublishPeerMessage(appid int64, im *IMMessage) {
	conn := redis_pool.Get()
	defer conn.Close()
	v := make(map[string]interface{})
	v["appid"] = appid
	v["sender"] = im.sender
	v["receiver"] = im.receiver
	v["content"] = im.content
	b, _ := json.Marshal(v)
	var queue_name string
	if client.IsROMApp(appid) {
		queue_name = fmt.Sprintf("push_queue_%d", appid)
	} else {
		queue_name = "push_queue"
	}
	client.PushChan(queue_name, b)		
}
```
### PublishGroupMessage

推送消息写入**group_push_queue**或**group_push_queue_\#\{appid\}**队列。

```go
func (client *Client) PublishGroupMessage(appid int64, receivers []int64, im *IMMessage) {
	conn := redis_pool.Get()
	defer conn.Close()
	v := make(map[string]interface{})
	v["appid"] = appid
	v["sender"] = im.sender
	v["receivers"] = receivers
	v["content"] = im.content
	v["group_id"] = im.receiver
	b, _ := json.Marshal(v)
	var queue_name string
	if client.IsROMApp(appid) {
		queue_name = fmt.Sprintf("group_push_queue_%d", appid)
	} else {
		queue_name = "group_push_queue"
	}
	client.PushChan(queue_name, b)	
}
```

### PublishCustomerMessage

推送消息写入**customer_push_queue**队列。

```go
func (client *Client) PublishCustomerMessage(appid, receiver int64, cs *CustomerMessage, cmd int) {
	conn := redis_pool.Get()
	defer conn.Close()
	v := make(map[string]interface{})
	v["appid"] = appid
	v["receiver"] = receiver
	v["command"] = cmd
	v["customer_appid"] = cs.customer_appid
	v["customer"] = cs.customer_id
	v["seller"] = cs.seller_id
	v["store"] = cs.store_id
	v["content"] = cs.content
	b, _ := json.Marshal(v)
	client.PushChan("customer_push_queue", b)	
}
```

### PublishSystemMessage

推送消息写入**system_push_queue**队列。

```go
func (client *Client) PublishSystemMessage(appid, receiver int64, content string) {
	conn := redis_pool.Get()
	defer conn.Close()
	v := make(map[string]interface{})
	v["appid"] = appid
	v["receiver"] = receiver
	v["content"] = content
	b, _ := json.Marshal(v)
	client.PushChan("system_push_queue", b)
}
```
