# 私聊消息

PeerStorage负责私聊消息存储，对应源码文件为：`peer_storage.go`。

```go
type PeerStorage struct {
	*StorageFile
	message_index  map[UserID]*UserIndex //记录每个用户最近的消息ID
}
type UserID struct {
	appid  int64
	uid    int64
}
type UserIndex struct {
	last_id int64	
	last_peer_id int64
}
```

**UserID**：通过appid和uid两个参数可以唯一的定位一位用户。

**UserIndex**：UserIndex维护了该用户上一条消息ID和上一条私聊消息ID两个索引位置。

**PeerStorage**：PeerStorage在StorageFile的基础上维护了一个包含所有用户的索引信息。

> 消息索引全部放在内存中，在程序退出时，再全部保存到文件中。如果索引文件不存在或上次保存失败，则在程序启动的时候，从消息DB中重建索引，这需要遍历每一条消息！

### 公共方法

```go
func (storage *PeerStorage) SetLastMessageID(appid int64, receiver int64, last_id int64, last_peer_id int64)
func (storage *PeerStorage) GetLastMessageID(appid int64, receiver int64) (int64, int64)
func (storage *PeerStorage) SavePeerMessage(appid int64, uid int64, device_id int64, msg *Message) int64
func (storage *PeerStorage) LoadHistoryMessages(appid int64, receiver int64, sync_msgid int64,  group_limit int, limit int) ([]*EMessage, int64)
func (storage *PeerStorage) LoadLatestMessages(appid int64, receiver int64, limit int) []*EMessage
func (storage *PeerStorage) GetNewCount(appid int64, uid int64, last_received_id int64) int
```

**SetLastMessageID**

设置私聊消息索引数据（最近消息ID）。

**GetLastMessageID**

获取私聊消息索引数据（最近消息ID）。

**SavePeerMessage**

写入`消息本身`到消息文件，同时写入`消息元数据`到消息文件，元数据包括：

- 应用ID
- 用户ID
- 设备ID
- 消息ID
- 上一条消息ID
- 上一条私聊消息ID

其中，通过`上一条消息ID`和`上一条私聊消息ID`将消息分别连成了两个从后向前单向链表，这两个链表是有一定重合部分的。我们在获取历史消息列表时，就是通过这两个链表来读取消息列表数据的。

> `消息ID`：这个ID是指消息本身的位置。<br>
> `上一条私聊消息ID`：这个ID是指向的消息元数据的位置，而非消息本身的位置！
> `上一条消息ID`：这个ID也是指向的消息元数据的位置，而非消息本身的位置！

**LoadHistoryMessages**

从后往前获取**appid:receiver**用户的历史消息列表，直到**sync_msgid**指定的位置。

**group_limit**获取的消息超过group_limit数量后，只获取私聊消息，0表示不限制。

**limit**限制了最多只取多少条历史消息，0表示不限制。

**msgid**获取从msgid到目前最新的群组消息，不会取到msgid以前的历史消息。

这个方法有两个返回值，一是历史消息列表，二是该群组消息的最近消息ID。

**LoadLatestMessages**

从消息文件中读取用户最近的**limit**条消息，包括私聊消息和群组消息。

**GetNewCount**

从当前用户的last_message_id(最近消息ID，保存在message_index索引结构中)开始往前遍历该用户的历史消息，直到参数**last_received_id**指定的位置为止。
因为用户自己发送的数据也会写入用户的消息列表，所以在遍历计数时，一定要过滤掉用户自己发送的消息。


### 私有方法

```go
func (client *PeerStorage) isGroupMessage(msg *Message) bool
func (client *PeerStorage) isSender(msg *Message, appid int64, uid int64) bool
func (storage *PeerStorage) readPeerIndex() bool
func (storage *PeerStorage) createPeerIndex()
func (storage *PeerStorage) repairPeerIndex()
func (storage *PeerStorage) removePeerIndex()
func (storage *PeerStorage) clonePeerIndex()
func (storage *PeerStorage) savePeerIndex(message_index  map[UserID]*UserIndex)
func (storage *PeerStorage) execMessage(msg *Message, msgid int64)
```

**isGroupMessage**

判断是否为群组消息，代码：

```go
func (client *PeerStorage) isGroupMessage(msg *Message) bool {
	return msg.cmd == MSG_GROUP_IM || msg.flag & MESSAGE_FLAG_GROUP != 0
}
```

**isSender**

判断用户是否为消息发送者，代码：

```go
func (client *PeerStorage) isSender(msg *Message, appid int64, uid int64) bool {
	if msg.cmd == MSG_IM || msg.cmd == MSG_GROUP_IM {
		m := msg.body.(*IMMessage)
		if m.sender == uid {
			return true
		}
	}
	if msg.cmd == MSG_CUSTOMER {
		m := msg.body.(*CustomerMessage)
		if m.customer_appid == appid && 
			m.customer_id == uid {
			return true
		}
	}
	if msg.cmd == MSG_CUSTOMER_SUPPORT {
		m := msg.body.(*CustomerMessage)
		if config.kefu_appid == appid && 
			m.seller_id == uid {
			return true
		}
	}
	return false
}
```

TODO: MSG_CUSTOMER & MSG_CUSTOMER_SUPPORT

**readPeerIndex**

读取索引文件`peer_index`，并将索引信息填充到**message_index**变量中。

如果索引文件不存在，返回false。

**createPeerIndex**

逐个遍历消息文件`message_N`，构建索引结构，将索引信息填充到**message_index**变量中。

**repairPeerIndex**

索引文件有可能会滞后于消息文件，比如程序意外退出没有来得及重新保存索引文件。针对这种情况，我们必须根据索引文件中的最近消息ID（last_id），从last_id位置开始读取消息文件，将其后的消息信息进行索引，构建一个完整的索引结构。

**removePeerIndex**

删除索引文件`group_index`。

**clonePeerIndex**

克隆一份message_index对应的完整索引数据。在刷新索引数据到索引文件时，为了及时释放锁，避免message_index长时间被锁住影响其他逻辑（比如SaveMessage），需要将索引数据克隆出来。

**savePeerIndex**

将克隆出来的完整索引数据先写入临时文件`peer_index_t`，然后将其重命名为`peer_index`，保证原子性写入。

**execMessage**

```go
func (storage *PeerStorage) execMessage(msg *Message, msgid int64) {
	if msg.cmd == MSG_OFFLINE {
		off := msg.body.(*OfflineMessage)
		storage.setLastMessageID(off.appid, off.receiver, msgid, msgid)
	} else if msg.cmd == MSG_OFFLINE_V2 {
		off := msg.body.(*OfflineMessage2)
		last_peer_id := msgid		
		if ((msg.flag & MESSAGE_FLAG_GROUP) != 0) {
			_, last_peer_id = storage.getLastMessageID(off.appid, off.receiver)			
		}
		storage.setLastMessageID(off.appid, off.receiver, msgid, last_peer_id)
		
	}
}
```

每次从主服务器同步消息数据的时候（调用`SaveSyncMessage`）都会调用**execMessage**方法，来设私聊消息索引。