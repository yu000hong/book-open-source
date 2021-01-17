# Kafka Broker 配置

| 配置参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `zookeeper.connect` | string | | ZK host string|
| `zookeeper.session.timeout.ms` | int | 6000 | zookeeper session timeout |
| `zookeeper.connection.timeout.ms` | int | 6000 | the max time that the client waits to establish a connection to zookeeper|
| `zookeeper.sync.time.ms` | int | 2000 | how far a ZK follower can be behind a ZK leader |
| `log.retention.ms` | int | 
| `log.retention.minutes` | int |
| `log.retention.hours` | int |
| `log.roll.ms` | int | 
| `log.roll.hours` | int | 
| `log.roll.jitter.ms` | int | 
| `log.roll.jitter.hours` | int |
| `broker.id` | int | | the broker id for this server |
| `message.max.bytes` | int | 1000000 | the maximum size of message that the server can receive |
| `num.network.threads` | int | 3 | the number of network threads that the server uses for handling network requests |
| `num.io.threads` | int | 8 | the number of io threads that the server uses for carrying out network requests |
| `background.threads` | int | 10 | the number of threads to use for various background processing tasks |
| `queued.max.requests` | int | 500 | the number of queued requests allowed before blocking the network threads |
| `port` | int | 9092 | the port to listen and accept connections on |
| `host.name` | string | null | hostname of broker. If this is set, it will only bind to this address. If this is not set, it will bind to all interfaces |
| `advertised.host.name` | string | same as `host.name` | hostname to publish to ZooKeeper for clients to use. In IaaS environments, this may need to be different from the interface to which the broker binds. If this is not set, it will use the value for "host.name" if configured. Otherwise it will use the value returned from java.net.InetAddress.getCanonicalHostName().
| `advertised.port` | int | same as `port` | the port to publish to ZooKeeper for clients to use. In IaaS environments, this may need to be different from the port to which the broker binds. If this is not set, it will publish the same port that the broker binds to. |
| `socket.send.buffer.bytes` | int | 102400 | the SO_SNDBUFF buffer of the socket sever sockets |
| `socket.receive.buffer.bytes` | int | 102400 | the SO_RCVBUFF buffer of the socket sever sockets |
| `socket.request.max.bytes` | int | 100\*1024\*1024 |