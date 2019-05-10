# 索引文件

### 两种类型

- **peer**：点对点消息通讯
- **group**：群消息通讯

**索引文件位置：**

- peer：`%root%/peer_index`
- group：`%root%/group_index`


如果索引文件不存在，那么在ims启动的时候会根据消息内容文件进行重建；

如果索引文件读取出现问题，则会直接退出，说明文件已经破坏，需要人工干预。

### PeerStorage

```go
//在取离线消息时，可以对群组消息和点对点消息分别获取，
//这样可以做到分别控制点对点消息和群组消息读取量，避免单次读取超量的离线消息
type PeerStorage struct {
    *StorageFile
    
    //消息索引全部放在内存中,在程序退出时,再全部保存到文件中，
    //如果索引文件不存在或上次保存失败，则在程序启动的时候，从消息DB中重建索引，这需要遍历每一条消息
    message_index  map[UserID]*UserIndex //记录每个用户最近的消息ID
}
```

### GroupStorage

```go
type GroupStorage struct {
    *StorageFile

    message_index  map[GroupID]int64 //记录每个群组最近的消息ID
}
```