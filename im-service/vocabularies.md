# 一些词汇

### IM & IMS & IMR

IM：终端服务器（是否考虑IMC(client)或IMT(terminal)）

IMS：存储服务器

IMR：路由服务器

TODO：感觉叫终端服务器也不太合适，因为IM除了直接与终端连接外，还会通IMR/IMS保持通信连接！

### 终端

C/S模式里面，C端都可以称为客户端，比如IM调用IMR的RPC时，IM可以称为IMR的客户端；

为了避免语意混淆，这里的iOS客户端、安卓客户端以及WEB客户端都统一称为终端！

### RouteMessage

指的是IM终端服务器向IMR路由服务器发送消息，经由IMR再路由到对应的IM终端服务器。

IM --> IMS --> IM1,IM2···

### DispatchMessage

指的是IM终端服务器根据自己维护在内存中的路由信息（AppRoute）将消息分发到对应的终端，这些终端都是直接连接在该IM终端服务器上的。

IM --> Client1,Client2···

### SendMessage

SendMessage包含两个动作：DispatchMessage + RouteMessage

### SaveMessage

SaveMessage指IM终端服务器将消息发送到IMS存储服务器进行消息的持久化。




### UID、GID、RID

UID：用户ID

GID：群组ID

RID：聊天室ID
