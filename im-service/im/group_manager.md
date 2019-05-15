# 群组管理

### 数据结构


```go
type GroupManager struct {
	mutex  sync.Mutex
	groups map[int64]*Group
	ping     string
	action_id int64
	dirty     bool
}
```

### Group

```go
type Group struct {
	appid   int64
	gid     int64
	super   bool //超大群
	mutex   sync.Mutex
	//key:成员id value:入群时间|(mute<<31)
	members map[int64]int64
}
```

群组数据结构包含了应用ID、群组ID、是否超级群、群组成员，其中群组成员包括了用户ID、入群时间及是否禁言。

**Group主要方法**：

- NewGroup：创建一个普通群组对象
- NewSuperGroup：创建一个超级群组对象
- AddMember：向群组中添加一个用户
- RemoveMember：从群组中移除一个用户
- Members：获取群组所有成员
- IsMember：判断用户是否在群里
- IsEmpty：判断群组是否为空
- GetMemberTimestamp：获取用户入群时间
- SetMemberMute：设置用户是否在群内禁言
- GetMemberMute：获取用户是否在群内禁言
- CreateGroup：创建群组，插入数据库表`group`，然后返回群组ID
- DeleteGroup：删除群组，从表`group`中删除，并且删除`group_member`表中的所有该群组用户
- LoadAllGroup：从数据库表`group`中读取所有群组，并且从`group_member`表中加载群组的所有用户
- LoadGroupMember：从数据库表`group_member`中读取该群组的所有用户
