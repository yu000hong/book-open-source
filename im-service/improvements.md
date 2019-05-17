# 改进意见

### PeerXXX, GroupXXX 统一一下

有的地方表示私聊，但是直接用的**XXX**，有的地方是群聊+私聊也是用的**XXX**。

这里最好区分一下，将私聊命名为**PeerXXX**，群聊为**GroupXXX**，群聊+私聊用**XXX**。

主要有哪些呢（TODO）？

### 修改命名

**消息分发**：PUBLISH -> DISPATCH

- MSG_PUBLISH       -> MSG_DISPATCH_PEER
- MSG_PUBLISH_GROUP -> MSG_DISPATCH_GROUP
- MSG_PUBLISH_ROOM  -> MSG_DISPATCH_ROOM

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