# Roshi

[soundcloud/roshi](https://github.com/soundcloud/roshi)

Roshi is a large-scale CRDT set implementation for timestamped events. Roshi implements a time-series event storage via a LWW-element-set CRDT with limited inline garbage collection. Roshi is a stateless, distributed layer on top of Redis and is implemented in Go. It is partition tolerant, highly available and eventually consistent.

At a high level, Roshi maintains sets of values, with each set ordered according to (external) timestamp, newest-first. Roshi provides the following API:

- Insert(key, timestamp, value)
- Delete(key, timestamp, value)
- Select(key, offset, limit) []TimestampValue

Roshi stores a sharded copy of your dataset in multiple independent Redis instances, called a cluster. Roshi provides fault tolerance by duplicating clusters; multiple identical clusters, normally at least 3, form a farm. Roshi leverages CRDT semantics to ensure consistency without explicit consensus.

