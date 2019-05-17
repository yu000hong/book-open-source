# 配置文件

### 配置参数

* listen
* mysqldb_source
* redis_address
* redis_password(可选)
* redis_db(可选)
* is_push_system(可选)
* http_listen_address(可选)

### listen

IMR服务器监听地址

### mysqldb_source

数据库连接地址，IMR服务器需要从数据库加载群组信息

### redis_address & redis_password & redis_db

Redis连接地址，IMR服务器需要订阅Redis频道来保持群组信息一致性；同时，消息推送也需要使用Redis队列。

**redis_password**默认值为空字符串，**redis_db**默认值为0。

### is_push_system

是否推送系统消息，默认值为false，看代码：

```go
if config.is_push_system {
    client.PublishSystemMessage(amsg.appid, amsg.receiver, sys.notification)
}
```

### http_listen_address

HTTP服务器地址，默认值为空字符串，表示不开启。

HTTP服务器提供了两个服务：

- /online：获取单个用户在线状态
- /all_online：获取当前所有在线的用户

