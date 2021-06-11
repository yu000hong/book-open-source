# Eureka 2.0 Motivations

[Eureka 2.0 Motivations](https://github.com/Netflix/eureka/wiki/Eureka-2.0-Motivations)

### DISCONTINUED

The open source work on eureka 2.0 (as embodied by the 2.x branch) is discontinued. **The rationale for this is variously due to dependencies on some parallel open source projects that never came to full fruition, and also the big bang nature of the project**.

The majority of the motivations for eureka 2.0 is something that we (Netflix) still believe in, and have since realized internally. This is done in a much more evolutionary and incremental way, via variously:

- a simple, auto-scalable REST readonly cluster with the same APIs to help with scale.
- a simple, auto-scalable gRPC readonly cluster leveraging gRPC to provide streaming capabilities and better multi-language client support.

### Why Eureka 2.0?

Eureka in its current form is a very stable system, battle tested in large cloud deployments with tens of thousand nodes. However, it has the following major limitations:

- **Only support homogenous client views**: Eureka servers only expects the client to always get the entire registry and does not allow to get specific applications/VIP addresses. This imposes a memory limitation on all clients registering with Eureka, even if they need a very small subset of the Eureka’s registry.

- **Only supports scheduled updates**: Eureka client follows a poll model to fetch updates from the server routinely. This imposes an overhead on the client of doing a scheduled call to the server, even if there are no changes and also delays the updates by the poll interval.

- **Replication algorithm limits scalability**: Eureka follows a broadcast replication model i.e. all the servers replicate data and heartbeats to all the peers. This is simple and effective for the data set that eureka contains however replication is implemented by relaying all the HTTP calls that a server receives as is to all the peers. This limits scalability as every node has to withstand the entire write load on eureka.

Although Eureka 2.0 provides a more richer feature set, the above limitations are the major driving factors for the changes proposed in this version.

### Eureka 2.0 Improvements

Based on the above motivations, Eureka 2.0 achieves the following improvements:

- **Interest based subscription model for registry data**: A client of Eureka is able to select a part of the instance registry in which it is interested in and the eureka server only sends information about that part of the registry. Eg: A client can say I am only interested in application “WebFarm” and then the server will only send information about WebFarm instances. Eureka server provides various selection criterions and a way to update this interest set dynamically.

- **Push model from the server for any changes to the interest set**: Instead of the current pull model from the client, Eureka servers pushes updates for changes to the interest set, to the client.

- **Optimized replication**: As Eureka 1.0, Eureka 2.0 also follows a broadcast replication model i.e. every node replicates data to all other nodes. However, the replication algorithm is much more optimized eliminating the need of sending heartbeats for every instance in the registry. This drastically reduce the replication traffic and achieves much higher scalability.

- **Auto-scaled Eureka servers**: Eureka 2.0 divides the read (discovery of data) and write (registration) concerns into separate clusters. Since, the write load is predictable (proportional to the number of instances in a region), the write cluster is pre-scaled. On the other hand, read load is unpredictable (proportional to subscriptions from clients) and hence the read farm is auto-scaled.

- **Audit log**: Eureka 2.0 provides an elaborate audit log for any change done to the registry. This helps Eureka owners as well as users to get insight into debugging the state of individual application instances as exists in Eureka. The audit log by default is persisted to a log file, but different persistent storages can be plugged-in.

- **Dashboard**: Eureka 2.0 provides a rich dashboard (as opposed to very rudimentary dashboard of Eureka 1.0) with insights into Eureka internals with respect to registry views, server health, subscription state, audit log, etc.
