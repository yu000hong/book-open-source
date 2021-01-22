# 消费者配置

| 配置参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `zookeeper.connect` | string | | ZK host string|
| `zookeeper.session.timeout.ms` | int | 6000 | zookeeper session timeout |
| `zookeeper.connection.timeout.ms` | int | 6000 | the max time that the client waits to establish a connection to zookeeper|
| `zookeeper.sync.time.ms` | int | 2000 | how far a ZK follower can be behind a ZK leader |
| `group.id` | string | | a string that uniquely identifies a set of consumers within the same consumer group |
| `consumer.id` | string | null | consumer id: generated automatically if not set |
| `socket.timeout.ms` | int | 30000 | the socket timeout for network requests. Its value should be at least `fetch.wait.max.ms`. |
| `socket.receive.buffer.bytes` | int | 64*1024 | the socket receive buffer for network requests |
| `fetch.message.max.bytes` | int | 1024*1024 | the number of byes of messages to attempt to fetch |
| `num.consumer.fetchers` | int | 1 | the number threads used to fetch data |
| `auto.commit.enable` | bool | true | if true, periodically commit to zookeeper the offset of messages already fetched by the consumer |
| `auto.commit.interval.ms` | int | 60*1000 | the frequency in ms that the consumer offsets are committed to zookeeper |
| `queued.max.message.chunks` | int | 2 | max number of message chunks buffered for consumption, each chunk can be up to `fetch.message.max.bytes`. |
| `rebalance.max.retries` | int | 4 | max number of retries during rebalance |
| `rebalance.backoff.ms` | int | 2000 | backoff time between retries during rebalance |
| `fetch.min.bytes` | int | 1 | the minimum amount of data the server should return for a fetch request. If insufficient data is available the request will block |
| `fetch.wait.max.ms` | int | 100 | the maximum amount of time the server will block before answering the fetch request if there isn't sufficient data to immediately satisfy `fetch.min.bytes`. |
| `refresh.leader.backoff.ms` | int | 200 | backoff time to refresh the leader of a partition after it loses the current leader |
| `offsets.channel.backoff.ms` | int | 1000 | backoff time to reconnect the offsets channel or to retry offset fetches/commits |
| `offsets.channel.socket.timeout.ms` | int | 10000 | socket timeout to use when reading responses for Offset Fetch/Commit requests. This timeout will also be used for the ConsumerMetdata requests that are used to query for the offset coordinator. |
| `offsets.commit.max.retries` | int | 5 | Retry the offset commit up to this many times on failure. This retry count only applies to offset commits during shut-down. It does not apply to commits from the auto-commit thread. It also does not apply to attempts to query for the offset coordinator before committing offsets. i.e., if a consumer metadata request fails for any reason, it is retried and that retry does not count toward this limit. |
| `offsets.storage` | string | zookeeper | Specify whether offsets should be committed to "zookeeper" (default) or "kafka" |
| `dual.commit.enabled` | bool |  | If you are using "kafka" as `offsets.storage`, you can dual commit offsets to ZooKeeper (in addition to Kafka). This is required during migration from zookeeper-based offset storage to kafka-based offset storage. With respect to any given consumer group, it is safe to turn this off after all instances within that group have been migrated to the new jar that commits offsets to the broker (instead of directly to ZooKeeper). |
| `auto.offset.reset` | string | largest | what to do if an offset is out of range. <br>**smallest**: automatically reset the offset to the smallest offset <br>**largest**: automatically reset the offset to the largest offset <br>**anything else**: throw exception to the consumer |
| `consumer.timeout.ms` | int | -1 | throw a timeout exception to the consumer if no message is available for consumption after the specified interval |
| `client.id` | striing | same as `group.id` | Client id is specified by the kafka consumer client, used to distinguish different clients |
| `exclude.internal.topics` | bool | true | Whether messages from internal topics (such as offsets) should be exposed to the consumer. |
| `partition.assignment.strategy` | string | range | Select a strategy for assigning partitions to consumer streams. Possible values: range, roundrobin |
