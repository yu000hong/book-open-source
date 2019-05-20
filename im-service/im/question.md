# 一些问题

### Redis挂了对IM服务器有何影响？

### MySQL挂了对IM服务器有何影响？

### `route.go`文件中GetUserIDs始终返回一个空集合，好像是有问题的？

```go
func (route *Route) GetUserIDs() IntSet {
	return NewIntSet()
}
```

不过，看代码这个方法只在AppRoute.GetUsers()方法中被调用，但AppRoute.GetUsers()是一个未使用方法！

### 为何HandleUnreadCount是直接设置Redis，而HandleSyncKey里是通过通道在另一个协程里面进行异步处理？

HandleUnreadCount：peer_client.go L204

HandleSyncKey：peer_client.go L133