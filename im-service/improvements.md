# 改进意见

### 很多命名含糊不清

SendMessage、DispatchMessage、PushMessage、PublishMessage

无法根据命名来确定其用途？！

### PeerXXX, GroupXXX 统一一下

有的地方表示单聊，但是直接用的**XXX**，有的地方是群聊+单聊也是用的**XXX**。

这里最好区分一下，将单聊命名为**PeerXXX**，群聊为**GroupXXX**，群聊+单聊用**XXX**。

主要有哪些呢（TODO）？

### 修改命名

**消息分发**：PUBLISH -> ROUTE

- MSG_PUBLISH       -> MSG_ROUTE_PEER
- MSG_PUBLISH_GROUP -> MSG_ROUTE_GROUP
- MSG_PUBLISH_ROOM  -> MSG_ROUTE_ROOM

DispatchMessage：消息分发，指的是IM根据自己维护的路由信息将消息直接分发的终端；

RouteMessage：消息路由，指的是IM将消息路由到IMR服务器，IMR再根据其路由信息将消息路由到对应IM服务器；

**上线注册/下线注销**

- MSG_SUBSCRIBE         -> MSG_REGISTER
- MSG_UNSUBSCRIBE       -> MSG_UNREGISTER
- MSG_SUBSCRIBE_ROOM    -> MSG_REGISTER_ROOM
- MSG_UNSUBSCRIBE_ROOM  -> MSG_UNREGISTER_ROOM

### AppUserID -> UserID

### AppRoomID -> RoomID

### group_id -> GroupID

目前，group_id要求全局唯一，也即是在所有应用里，group_id不能重复。

后面期望改成和UserID/RoomID一样，都把app_id带上。

### AppMessage -> DispatchMessage

### 聊天室对应消息类型命名统一

MSG_ENTER_ROOM -> MSG_ROOM_ENTER

MSG_LEAVE_ROOM -> MSG_ROOM_LEAVE

MSG_ROOM_IM <=> MSG_ROOM_IM

HandleEnterRoom -> HandleRoomEnter

HandleLeaveRoom -> HandleRoomLeave

HandleRoomIM <=> HandleRoomIM

### execMessage 需要重新命名

### MSG_SYNC_GROUP -> MSG_GROUP_SYNC

MSG_GROUP_IM <=> MSG_GROUP_IM

MSG_GROUP_SYNC_KEY <=> MSG_GROUP_SYNC_KEY

MSG_SYNC_GROUP -> MSG_GROUP_SYNC
