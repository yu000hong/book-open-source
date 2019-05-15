# 群组消息

GroupStorage负责群组消息的存储，对应源码文件为：`group_storage.go`。

```go
type GroupStorage struct {
	*StorageFile
	message_index  map[GroupID]int64 //记录每个群组最近的消息ID
}
type GroupID struct {
	appid  int64
	gid    int64
}
```

**GroupID**：通过appid和gid两个参数可以唯一的定位一个群组。

**GroupStorage**：GroupStorage在StorageFile的基础上维护了一个包含所有群组的索引信息。

> 消息索引全部放在内存中，在程序退出时，再全部保存到文件中。如果索引文件不存在或上次保存失败，则在程序启动的时候，从消息DB中重建索引，这需要遍历每一条消息！

### 公共方法

```go
func (storage *GroupStorage) SetLastGroupMessageID(appid int64, gid int64, msgid int64)
func (storage *GroupStorage) GetLastGroupMessageID(appid int64, gid int64) (int64, error)
func (storage *GroupStorage) SaveGroupMessage(appid int64, gid int64, device_id int64, msg *Message) int64
func (storage *GroupStorage) LoadGroupHistoryMessages(appid int64, uid int64, gid int64, msgid int64, ts int32, limit int) ([]*EMessage, int64)
```

**SetLastGroupMessageID**

设置群组消息索引数据（最近消息ID）。

**GetLastGroupMessageID**

获取群组消息索引数据（最近消息ID）。

**SaveGroupMessage**

写入`消息本身`到消息文件，同时写入`消息元数据`到消息文件，元数据包括：

- 应用ID
- 群组ID
- 设备ID
- 消息ID
- 上一条群组消息ID。

其中，通过`上一条群组消息ID`将消息连成了一个从后向前单向链表。我们在获取群组历史消息列表时，就是通过这个链表来读取消息列表数据的。

> `消息ID`：这个ID是指消息本身的位置。<br>
> `上一条群组消息ID`：这个ID是指向的消息元数据的位置，而非消息本身的位置！

**LoadGroupHistoryMessages**

获取**appid:gid**群组的历史消息列表，这里**uid**参数不起任何作用，只是进行简单的日志记录功能。

**ts**为用户uid入群的时间，获取群组历史消息时，会根据消息时间进行过滤，不会取到入群之前的历史消息。

**limit**限制了最多只取多少条历史消息。

**msgid**获取从msgid到目前最新的群组消息，不会取到msgid以前的历史消息。

这个方法有两个返回值，一是群组历史消息列表，二是该群组消息的最近消息ID。


### 私有方法

```go
func (storage *GroupStorage) readGroupIndex() bool
func (storage *GroupStorage) createGroupIndex()
func (storage *GroupStorage) repairGroupIndex()
func (storage *GroupStorage) removeGroupIndex()
func (storage *GroupStorage) cloneGroupIndex() map[GroupID]int64
func (storage *GroupStorage) saveGroupIndex(message_index map[GroupID]int64)
func (storage *GroupStorage) execMessage(msg *Message, msgid int64)
```

**readGroupIndex**

读取索引文件`group_index`，并将索引信息填充到**message_index**变量中。

如果索引文件不存在，返回false。

**createGroupIndex**

逐个遍历消息文件`message_N`，构建索引结构，将索引信息填充到**message_index**变量中。

**repairGroupIndex**

索引文件有可能会滞后于消息文件，比如程序意外退出没有来得及重新保存索引文件。针对这种情况，我们必须根据索引文件中的最近消息ID（last_id），从last_id位置开始读取消息文件，将其后的消息信息进行索引，构建一个完整的索引结构。

**removeGroupIndex**

删除索引文件`group_index`。

**cloneGroupIndex**

克隆一份message_index对应的完整索引数据。在刷新索引数据到索引文件时，为了及时释放锁，避免message_index长时间被锁住影响其他逻辑（比如SaveMessage），需要将索引数据克隆出来。

**saveGroupIndex**

将克隆出来的完整索引数据先写入临时文件`group_index_t`，然后将其重命名为`gourp_index`，保证原子性写入。

**execMessage**

```go
func (storage *GroupStorage) execMessage(msg *Message, msgid int64) {
	if msg.cmd == MSG_GROUP_IM_LIST {
		off := msg.body.(*GroupOfflineMessage)
		storage.setLastGroupMessageID(off.appid, off.gid, msgid)
	}
}
```

每次从主服务器同步消息数据的时候（调用`SaveSyncMessage`）都会调用**execMessage**方法，来设置群组消息索引。