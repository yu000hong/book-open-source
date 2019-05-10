# 主从数据同步


### Master

**数据结构**

```go
type Master struct {
    ewt       chan *EMessage

    mutex     sync.Mutex
    clients   map[*SyncClient]struct{}
}
```

Master表示当前服务的角色为主服务器，Master维护了其对应的从服务器与当前Master的连接列表clients。

mutex互斥锁的目的就是同步访问这些clients。

**主要工作**

1. 维护一个大小为1000的消息队列
2. 每当ewt通道接收到消息时，将消息放入消息队列
3. 如果消息队列已满，那么调用SendBatch()方法将消息同步到从服务器
4. 如果达到一个周期（60秒），那么也会调用SendBatch()方法进行数据同步

**ewt通道**

Master会监听这个通道，每当有消息存入文件后，同时会将该消息发往Mater的ewt通道。

### Slaver

**数据结构**

```go
type Slaver struct {
    addr string
}
```

Slaver表示当前服务的角色为从服务器，里面addr字段表示其对应的主服务器的网络地址。

只有配置了`master_address`参数，才会启动Slaver从服务器，去与主服务器Master进行数据同步操作。

**主要工作**

1. 给Master发送MSG_STORAGE_SYNC_BEGIN命令（带上本地nextMessageId）
2. 不断接收Master发送来的同步命令：MSG_STORAGE_SYNC_MESSAGE 或 MSG_STORAGE_SYNC_MESSAGE_BATCH
3. 根据Master的同步命令将数据同步到本地存储

注：从代码看，Master只会批量的与Slaver进行数据同步，也即只会发送MSG_STORAGE_SYNC_MESSAGE_BATCH命令。

代码：

```go
msgid := storage.NextMessageID()
cursor := &SyncCursor{msgid}
log.Info("cursor msgid:", msgid)

msg := &Message{cmd:MSG_STORAGE_SYNC_BEGIN, body:cursor}
seq += 1
msg.seq = seq
SendMessage(conn, msg)

for {
	msg := ReceiveStorageSyncMessage(conn)
	if msg == nil {
		return
	}

	if msg.cmd == MSG_STORAGE_SYNC_MESSAGE {
		emsg := msg.body.(*EMessage)
		storage.SaveSyncMessage(emsg)
	} else if msg.cmd == MSG_STORAGE_SYNC_MESSAGE_BATCH {
		mb := msg.body.(*MessageBatch)
		storage.SaveSyncMessageBatch(mb)
	} else {
		log.Error("unknown message cmd:", Command(msg.cmd))
	}
}
```

### SyncClient

**数据结构**

```go
type SyncClient struct {
    conn      *net.TCPConn
    ewt       chan *Message
}
```

conn为对应从服务器的连接

ewt通道用于主服务器与SyncClient的通信，SyncClient从ewt通道接收到消息时，会直接发送到对应的从服务器达到数据同步的目的。

Master主服务器上，与每个从服务器Slaver建立连接之后，都会生成一个与之对应的SyncClient实例，这个实例其实就是维护了主服务器与从服务器的连接关系。Master主服务器上维护了一个从服务器连接集合clients。

**主要工作**

1. 验证从服务器（如果conn连接没有发送数据过来，或者发送的命令不是MSG_STORAGE_SYNC_BEGIN，表明不是合法Slaver，丢弃该连接）
2. 根据从服务器传上来的nextMessageId，将主服务器上基于此id的最新消息同步回从服务器（需要从存储文件中获取对应消息）
3. 监听ewt通道，如果收到从Master传来的消息，将其同步到从服务器

问题：在从服务器连接上主服务器进行数据同步的过程中，监听ewt通道之前，如果有消息进来，这些消息感觉会丢失哦？！


我们可以配置多层级主从复制架构！

Slaver与Master间的通讯协议：

1. Slaver发送MSG_STORAGE_SYNC_BEGIN（根据当前的nextMessageID）
2. Master根据Slaver发送的nextMessageID去本地加载最新的消息列表
3. 根据nextMessageID去文件中读取消息，每5000条数据形成一个batch，发送到Slaver［MSG_STORAGE_SYNC_MESSAGE_BATCH］
4. 将Slaver对应的连接SyncClient加入到Master的clients集合里
5. 通过SyncClient.ewt通道来实时同步新消息
6. storage_file.go文件里的saveMessage()方法每写入一条消息就会发送一条EMessage到Master.ewt通道，Master会将消息放入消息队列，当消息队列已满（1000条）或者超过一定时限（60秒）时，就会遍历Master.clients，并通过client.ewt通道发送消息。
7. 当Slaver连接断开时，会从Master.clients集合中移除。
