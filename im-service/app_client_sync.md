# 终端消息同步

终端有三种类型：

- PLATFORM_IOS：iOS客户端
- PLATFORM_ANDROID：安卓客户端
- PLATFORM_WEB：网页终端

### 消息同步过程

1、当终端发送MSG_IM消息时，服务端会通知接收终端MSG_SYNC_NOTIFY，其中消息内容SyncKey为当前最新msgid（由IMS服务器返回）

2、终端收到这个SyncKey时，会向IM服务器发送MSG_SYNC消息，消息内容即为SyncKey

3、IM服务器调用IMS，获取从sync_key位置的最新消息

4、IM服务器发送MSG_SYNC_BEGIN＋历史消息列表＋MSG_SYNC_END

5、终端收到MSG_SYNC_END消息时，保存最新SyncKey

6、如果终端设置了SyncKeyHandler，那么将会向服务器发送MSG_SYNC_KEY消息，消息内容为最新SyncKey

7、服务器收到MSG_SYNC_KEY后，会比较当前Redis里面的SyncKey和终端发送的SyncKey

8、如果终端发送SyncKey较大，那么将其存入Redis；否则，do nothing



### 终端初始化消息同步

1、终端连接后，进行Token认证

2、发送MSG_SYNC，此时SyncKey为0

3、服务器收到SyncKey为0的消息同步请求，从Redis中获取对应的SyncKey

```
users_#{APPID}_#{UID}
用户状态数据，类型为Hash，hashKey如下：
* sync_key
* group_sync_key_#{GROUPID}
* forbidden
* unread
```

4、后面的流程保持一致

5、IM服务器调用IMS，获取从sync_key位置的最新消息

6、IM服务器发送MSG_SYNC_BEGIN＋历史消息列表＋MSG_SYNC_END

7、终端收到MSG_SYNC_END消息时，保存最新SyncKey




