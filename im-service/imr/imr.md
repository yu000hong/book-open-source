# IMR路由服务器

IM终端服务器会给IMS路由服务器发送两种大类型的消息：

- 聊天消息路由分发(Publish)
- 上线注册/下线注销(Subscribe/Unsubscribe)

### 聊天消息路由分发

IM终端服务器发送需要转发的消息到IMS时，IMR会根据内存中维护的路由信息进行分发，将消息原样转发到对应的IM服务器。

```
         MSG_PUBLISH(AppMessage)             |---> IM
IM------------------------------------->IMR------> IM
                                             |---> IM


      MSG_PUBLISH_GROUP(AppMessage)          |---> IM
IM------------------------------------->IMR------> IM
                                             |---> IM


      MSG_PUBLISH_ROOM(AppMessage)           |---> IM
IM------------------------------------->IMR------> IM
                                             |---> IM
```

### 上线注册/下线注销

IM终端发送的注册/注销消息，表明某个客户端的上线/下线状态，IMR路由服务器收到此类消息后会修改其维护的路由表信息。

```
         MSG_SUBSCRIBE(SubscribeMessage)
IM------------------------------------------->IMR

         MSG_UNSUBSCRIBE(AppUserID)
IM------------------------------------------->IMR

         MSG_SUBSCRIBE_ROOM(RoomID)
IM------------------------------------------->IMR

         MSG_UNSUBSCRIBE_ROOM(RoomID)
IM------------------------------------------->IMR
```

