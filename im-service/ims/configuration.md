# 配置文件

### 配置参数

* rpc_listen
* storage_root
* kefu_appid
* http_listen_address（可选项）
* sync_listen
* master_address（可选项）
* is_push_system（可选项）
* group_limit
* limit

### rpc_listen

RPC监听地址，形式为：ip:port，如：

```
rpc_listen=:13333
```

### storage_root

消息存储文件的根目录，如：

```
storage_root=/tmp/ims
```

### kefu_appid

客服的appid，TODO，默认值：0。

```
kefu_appid=0
```

### http_listen_address

**可选项**，服务器运行状态信息的监听地址。

```
http_listen_address=:8080
```

### sync_listen



### master_address

**可选项**，主服务器的IP地址，形式为：ip:port。

如果设定了master_address，那么表明该ims为从服务器。

```
master_address=172.16.133.222:8888
```

### is_push_system

**可选项**，TODO，默认值：0.

```
is_push_system=0
```

### group_limit

普通群离线消息的数量限制，默认值：0。

```
group_limit=0
```

### limit

离线消息的数量限制，默认值：3000。

```
limit=3000
```