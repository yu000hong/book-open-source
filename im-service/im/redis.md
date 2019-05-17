# Redis使用情况

### users\_\#\{app\_id\}\_\#\{uid\}

类型：hash

描述：用户的一些状态信息

- sync\_key：用户最近消息ID（lastMsgId）
- group\_sync\_key\_\#\{gid\}：用户最近的群聊消息ID
- forbidden：用户是否禁言
- unread：未读消息数量

### access\_token\_\#\{token\}

类型：hash

描述：进行IM客户端连接认证，获取认证后的用户信息

- app\_id：应用ID
- user\_id：用户UID
- notification\_on：是否开启通知
- forbidden：是否禁言

### statistics\_users\_\#\{app\_id\}

类型：hyperloglog

元素：用户UID

描述：统计某个APP大概有多少上线过的用户

### statistics\_dau\_\#\{date\}\_\#\{app\_id\}

类型：hyperloglog

元素：用户UID

描述：统计某个APP在某天的活跃用户数

### devices\_\#\{device\_id\}\_\#\{platform\_id\}

类型：string

描述：根据客户端上传的device\_id字符串和platform\_id平台，获取其在IM服务器对应的设备ID（整型）

### devices\_id

类型：string

描述：这是一个计数器，用于生成唯一的设备ID（整型），当通过`devices_#{device_id}_#{platform_id}`无法获取到对应的设备ID时，就会从这个计数器中生成一个ID

### groups\_actions

类型：string

描述：获取群组管理器对应的**action\_id**，其值为："%d:%d"，第一个整数应该是上一个action\_id，第二个整数为当前action\_id。

### ACTION频道

ACTION频道包括：

- group\_create
- group\_disband
- group\_member\_add
- group\_member\_remove
- group\_upgrade
- group\_member\_mute

频道内容为：prev\_id:action\_id:content

### RELOAD频道

RELOAD频道为：**group_manager.ping**，group\_manager.ping是在IM服务器启动时随机生成的一个字符串。

当IM服务器收到RELOAD频道的订阅消息时，会根据action\_id和dirty去判断是否需要重新加载群组及其群成员信息。

### 禁言频道

禁言频道为：**speak_forbidden**

频道内容为：**appid,uid,forbidden**

### 推送队列

类型：LIST

推送队列包括：

- system_push_queue
- customer_push_queue
- group_push_queue
- group_push_queue_\#\{appid\}
- push_queue
- push_queue_\#\{appid\}