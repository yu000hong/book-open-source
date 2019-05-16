# 群组管理

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

### 群组方法

```go
func CreateGroup(db *sql.DB, appid int64, master int64, name string, super int8) int64
func DeleteGroup(db *sql.DB, group_id int64) bool
func LoadAllGroup(db *sql.DB) (map[int64]*Group, error)
func LoadGroupMember(db *sql.DB, group_id int64) (map[int64]int64, error)
```

**CreateGroup**：创建群组，插入数据库表`group`，然后返回群组ID

**DeleteGroup**：删除群组，从表`group`中删除，并且删除`group_member`表中的所有该群组用户

**LoadAllGroup**：从数据库表`group`中读取所有群组，并且从`group_member`表中加载群组的所有用户

**LoadGroupMember**：从数据库表`group_member`中读取该群组的所有用户

> ⚠️ **CreateGroup**和**DeleteGroup**两个方法虽然定义了，但是处于未使用状态，也就是说，在IM服务器这边是不需要处理群组的创建删除以及群成员的添加移除工作的。
> 正确的操作流程是：在应用服务器中进行群组的创建删除以及群成员的添加移除，然后自增**action_id**，并同时将操作事件发布对应的到Redis频道；IM服务器订阅Redis频道
> 收到操作事件后，会进行相应的操作来维护内存中群组信息与数据库的一致性；除了通过接收操作事件，IM服务器会定期去比较**action_id**，当发现自己的**action_id**
> 值落后时，会触发`LoadAllGroup()`操作，重新加载数据库中的所有群组及成员信息。

### GroupManager

```go
type GroupManager struct {
	mutex       sync.Mutex
	groups      map[int64]*Group
	ping        string
	action_id   int64
	dirty       bool
}
```

**groups**：群组管理器维护在内存中的所有群组及其组员的信息

**ping**：IM服务器启动时随机生成的一个字符串

**action_id**：通过这个action_id值前后是否一致来判断群组信息是否有改变

**dirty**：表明群组管理器维护在内存中的群组数据已经不一致，需要重新加载

群组管理器会在内存中维护所有群组及其群成员的信息，它的主要工作就是维护内存中的数据与MySQL数据库中数据的一致性。
群组管理器通过**action_id**、**dirty**、以及**Redis频道**来维护内存数据与数据库数据的一致性。

**GroupManager主要方法**：

- getActionID：获取Redis缓存中**groups\_actions**的值，解析出当前的action_id值。
- checkActionID：通过getActionID获取当前action_id的值，与群组管理器中维护的值进行比较；如果两个值相等，那么do nothing；如果两个值不等，那么调用ReloadGroup重新加载数据库中的群组信息。
- parseAction：解析从Redis频道中获取的内容数据（action），action由三部分组成："prev_id:action_id:content"。
- handleAction：分派ACTION事件到对应的事件处理程序。
- ReloadGroup：重新加载所有群组及其群信息。
- GetGroups：获取所有的群组列表。
- FindGroup：根据gid获取对应的Group群组对象。
- FindUserGroups：获取用户所在的所有群组列表。
- Ping：发布`ping`频道，触发一致性检测。
- HandleCreate：处理群组创建事件，事件内容为："gid,appid,super"。
- HandleDisband：处理群组删除事件，事件内容为："gid"。
- HandleMemberAdd：处理群成员添加事件，事件内容为："gid,uid"。
- HandleMemberRemove：处理群成员删除事件，事件内容为："gid,uid"。
- HandleMute：处理群成员禁言事件，事件内容为："gid,uid,mute"。
- HandleUpgrade：处理群组升级为超级群事件，事件内容为："gid,appid,super"。

**handleAction**

```go
func (group_manager *GroupManager) handleAction(data string, channel string) {
	r, prev_id, action_id, content := group_manager.parseAction(data)
	if r {
		if group_manager.action_id != prev_id {
			//reload later
			group_manager.dirty = true
		}
		if channel == "group_create" {
			group_manager.HandleCreate(content)
		} else if channel == "group_disband" {
			group_manager.HandleDisband(content)
		} else if channel == "group_member_add" {
			group_manager.HandleMemberAdd(content)
		} else if channel == "group_member_remove" {
			group_manager.HandleMemberRemove(content)
		} else if channel == "group_upgrade" {
			group_manager.HandleUpgrade(content)
		} else if channel == "group_member_mute" {
			group_manager.HandleMute(content)
		}
		group_manager.action_id = action_id
	}	
}
```

先通过解析action获得prev_id、action_id、content，然后比较群组管理器的action_id是否与prev_id相等。

如果不相等，那么表明Redis频道推送的ACTION事件没有按序到达或者漏掉一些事件，因此需要设置dirty标志以待稍后重新加载。

根据频道channel值，将ACTION事件分派到特定的事件处理程序：HandleCreate、HandleDisband。。。




### 启动流程

群组管理器启动代码：

```go
group_manager = NewGroupManager()
group_manager.Start()
```

**NewGroupManager()**

```go
func NewGroupManager() *GroupManager {
    //生成随机字符串
	now := time.Now().Unix()
	r := fmt.Sprintf("ping_%d", now)
	for i := 0; i < 4; i++ {
		n := rand.Int31n(26)
		r = r + string('a' + n)
	}
	//创建实例对象
	m := new(GroupManager)
	m.groups = make(map[int64]*Group)
	m.ping = r
	m.action_id = 0
	m.dirty = true
	return m
}
```

**Start()**

```go
func (group_manager *GroupManager) Start() {
    //加载数据库中群组及其组员信息
    group_manager.load()
    //订阅Redis相关频道，根据收到的发布信息来维护内存中群组及组员信息
    //订阅超时时间设置为300秒，比发布ping信息多10秒，避免超时
    go group_manager.Run()
    //每隔290秒调用一次Ping()方法，触发群组管理器检测数据一致性
	go group_manager.PingLoop()
}
```