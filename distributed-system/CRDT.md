# CRDT - Conflict-free Replicated Data Type

[https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)

In distributed computing, a conflict-free replicated data type (CRDT) is a data structure which can be replicated across multiple computers in a network, where the replicas can be updated independently and concurrently without coordination between the replicas, and where it is always mathematically possible to resolve inconsistencies that might come up.

The CRDT concept was formally defined in 2011 by Marc Shapiro, Nuno Preguiça, Carlos Baquero and Marek Zawirski. Development was initially motivated by collaborative text editing and mobile computing. CRDTs have also been used in online chat systems, online gambling, and in the SoundCloud audio distribution platform. The NoSQL distributed databases Redis, Riak and Cosmos DB have CRDT data types.

### Background

Concurrent updates to multiple replicas of the same data, without coordination between the computers hosting the replicas, can result in inconsistencies between the replicas, which in the general case may not be resolvable. Restoring consistency and data integrity when there are conflicts between updates may require some or all of the updates to be entirely or partially dropped.

**Accordingly, much of distributed computing focuses on the problem of how to prevent concurrent updates to replicated data. But another possible approach is optimistic replication, where all concurrent updates are allowed to go through, with inconsistencies possibly created, and the results are merged or "resolved" later**. In this approach, consistency between the replicas is eventually re-established via "merges" of differing replicas. While optimistic replication might not work in the general case, it turns out that there is a significant and practically useful class of data structures, CRDTs, where it does work — where it is mathematically always possible to merge or resolve concurrent updates on different replicas of the data structure without conflicts. This makes CRDTs ideal for `optimistic replication`.

As an example, a one-way Boolean event flag is a trivial CRDT: one bit, with a value of true or false. True means some particular event has occurred at least once. False means the event has not occurred. Once set to true, the flag cannot be set back to false. (An event, having occurred, cannot un-occur.) The resolution method is "true wins": when merging a replica where the flag is true (that replica has observed the event), and another one where the flag is false (that replica hasn't observed the event), the resolved result is true — the event has been observed.

### Types of CRDTs

There are two approaches to CRDTs, both of which can provide strong eventual consistency: 

- **operation-based CRDTs**
- **state-based CRDTs**

The two alternatives are theoretically equivalent, as one can emulate the other. However, there are practical differences. `State-based CRDTs` are often simpler to design and to implement; their only requirement from the communication substrate is some kind of gossip protocol. Their drawback is that the entire state of every CRDT must be transmitted eventually to every other replica, which may be costly. In contrast, `operation-based CRDTs` transmit only the update operations, which are typically small. However, operation-based CRDTs require guarantees from the communication middleware; that the operations are not dropped or duplicated when transmitted to the other replicas, and that they are delivered in causal order.

**Operation-based CRDTs**

Operation-based CRDTs are also called `commutative replicated data types`, or `CmRDT`s. CmRDT replicas propagate state by transmitting only the update operation. For example, a CmRDT of a single integer might broadcast the operations (+10) or (−20). Replicas receive the updates and apply them locally. The operations are commutative. However, they are not necessarily idempotent. The communications infrastructure must therefore ensure that all operations on a replica are delivered to the other replicas, without duplication, but in any order.

Pure operation-based CRDTs are a variant of operation-based CRDTs that reduces the metadata size.

**State-based CRDTs**

State-based CRDTs are called `convergent replicated data types`, or `CvRDTs`. In contrast to CmRDTs, CvRDTs send their full local state to other replicas, where the states are merged by a function which must be commutative, associative, and idempotent. The merge function provides a join for any pair of replica states, so the set of all states forms a semilattice. The update function must monotonically increase the internal state, according to the same partial order rules as the semilattice.

Delta state CRDTs (or simply Delta CRDTs) are optimized state-based CRDTs where only recently applied changes to a state are disseminated instead of the entire state.

**Comparison**

While CmRDTs place more requirements on the protocol for transmitting operations between replicas, they use less bandwidth than CvRDTs when the number of transactions is small in comparison to the size of internal state. However, since the CvRDT merge function is associative, merging with the state of some replica yields all previous updates to that replica. Gossip protocols work well for propagating CvRDT state to other replicas while reducing network use and handling topology changes.

Some lower bounds on the storage complexity of state-based CRDTs are known.

### Known CRDTs

- G-Counter (Grow-only Counter)
- PN-Counter (Positive-Negative Counter)
- G-Set (Grow-only Set)
- 2P-Set (Two-Phase Set)
- LWW-Element-Set (Last-Write-Wins-Element-Set)
- OR-Set (Observed-Remove Set)
- Sequence CRDTs

**G-Counter (Grow-only Counter)**

```
payload integer[n] P
    initial [0,0,...,0]
update increment()
    let g = myId()
    P[g] := P[g] + 1
query value() : integer v
    let v = Σi P[i]
compare (X, Y) : boolean b
    let b = (∀i ∈ [0, n - 1] : X.P[i] ≤ Y.P[i])
merge (X, Y) : payload Z
    let ∀i ∈ [0, n - 1] : Z.P[i] = max(X.P[i], Y.P[i])
```

This CvRDT implements a counter for a cluster of n nodes. Each node in the cluster is assigned an ID from 0 to n - 1, which is retrieved with a call to myId(). Thus each node is assigned its own slot in the array P, which it increments locally. Updates are propagated in the background, and merged by taking the max() of every element in P. The compare function is included to illustrate a partial order on the states. The merge function is commutative, associative, and idempotent. The update function monotonically increases the internal state according to the compare function. This is thus a correctly-defined CvRDT and will provide strong eventual consistency. The CmRDT equivalent broadcasts increment operations as they are received.

**PN-Counter (Positive-Negative Counter)**

```
payload integer[n] P, integer[n] N
    initial [0,0,...,0], [0,0,...,0]
update increment()
    let g = myId()
    P[g] := P[g] + 1
update decrement()
    let g = myId()
    N[g] := N[g] + 1
query value() : integer v
    let v = Σi P[i] - Σi N[i]
compare (X, Y) : boolean b
    let b = (∀i ∈ [0, n - 1] : X.P[i] ≤ Y.P[i] ∧ ∀i ∈ [0, n - 1] : X.N[i] ≤ Y.N[i])
merge (X, Y) : payload Z
    let ∀i ∈ [0, n - 1] : Z.P[i] = max(X.P[i], Y.P[i])
    let ∀i ∈ [0, n - 1] : Z.N[i] = max(X.N[i], Y.N[i])
```

A common strategy in CRDT development is to combine multiple CRDTs to make a more complex CRDT. In this case, two G-Counters are combined to create a data type supporting both increment and decrement operations. The "P" G-Counter counts increments; and the "N" G-Counter counts decrements. The value of the PN-Counter is the value of the P counter minus the value of the N counter. Merge is handled by letting the merged P counter be the merge of the two P G-Counters, and similarly for N counters. Note that the CRDT's internal state must increase monotonically, even though its external state as exposed through query can return to previous values.

**G-Set (Grow-only Set)**

```
payload set A
    initial ∅
update add(element e)
    A := A ∪ {e}
query lookup(element e) : boolean b
    let b = (e ∈ A)
compare (S, T) : boolean b
    let b = (S.A ⊆ T.A)
merge (S, T) : payload U
    let U.A = S.A ∪ T.A
```

The G-Set (grow-only set) is a set which only allows adds. An element, once added, cannot be removed. The merger of two G-Sets is their union.

**2P-Set (Two-Phase Set)**

```
payload set A, set R
    initial ∅, ∅
query lookup(element e) : boolean b
    let b = (e ∈ A ∧ e ∉ R)
update add(element e)
    A := A ∪ {e}
update remove(element e)
    pre lookup(e)
    R := R ∪ {e}
compare (S, T) : boolean b
    let b = (S.A ⊆ T.A ∧ S.R ⊆ T.R)
merge (S, T) : payload U
    let U.A = S.A ∪ T.A
    let U.R = S.R ∪ T.R
```

Two G-Sets (grow-only sets) are combined to create the 2P-set. With the addition of a remove set (called the "tombstone" set), elements can be added and also removed. Once removed, an element cannot be re-added; that is, once an element e is in the tombstone set, query will never again return True for that element. The 2P-set uses "remove-wins" semantics, so remove(e) takes precedence over add(e).

**LWW-Element-Set (Last-Write-Wins-Element-Set)**

LWW-Element-Set is similar to 2P-Set in that it consists of an "add set" and a "remove set", with a timestamp for each element. Elements are added to an LWW-Element-Set by inserting the element into the add set, with a timestamp. Elements are removed from the LWW-Element-Set by being added to the remove set, again with a timestamp. An element is a member of the LWW-Element-Set if it is in the add set, and either not in the remove set, or in the remove set but with an earlier timestamp than the latest timestamp in the add set. Merging two replicas of the LWW-Element-Set consists of taking the union of the add sets and the union of the remove sets. When timestamps are equal, the "bias" of the LWW-Element-Set comes into play. A LWW-Element-Set can be biased towards adds or removals. The advantage of LWW-Element-Set over 2P-Set is that, unlike 2P-Set, LWW-Element-Set allows an element to be reinserted after having been removed.

**OR-Set (Observed-Remove Set)**

OR-Set resembles LWW-Element-Set, but using unique tags instead of timestamps. For each element in the set, a list of add-tags and a list of remove-tags are maintained. An element is inserted into the OR-Set by having a new unique tag generated and added to the add-tag list for the element. Elements are removed from the OR-Set by having all the tags in the element's add-tag list added to the element's remove-tag (tombstone) list. To merge two OR-Sets, for each element, let its add-tag list be the union of the two add-tag lists, and likewise for the two remove-tag lists. An element is a member of the set if and only if the add-tag list less the remove-tag list is nonempty.[2] An optimization that eliminates the need for maintaining a tombstone set is possible; this avoids the potentially unbounded growth of the tombstone set. The optimization is achieved by maintaining a vector of timestamps for each replica.[15]

**Sequence CRDTs**

A sequence, list, or ordered set CRDT can be used to build a Collaborative real-time editor, as an alternative to Operational transformation (OT).

Some known Sequence CRDTs are Treedoc, RGA, Woot, Logoot, and LSEQ. CRATE is a decentralized real-time editor built on top of LSEQSplit (an extension of LSEQ) and runnable on a network of browsers using WebRTC. LogootSplit was proposed as an extension of Logoot in order to reduce the metadata for sequence CRDTs. MUTE is an online web-based peer-to-peer real-time collaborative editor relying on LogootSplit algorithm.

### Industry use

`Nimbus Note` is a collaborative note-taking application that uses the Yjs CRDT for collaborative editing. 

`Redis` is a distributed, highly available and scalable in-memory database that uses CRDTs for implementing globally distributed databases based on and fully compatible with Redis open source.

SoundCloud open-sourced `Roshi`, a LWW-element-set CRDT for the SoundCloud stream implemented on top of Redis.

`Riak` is a distributed NoSQL key-value data store based on CRDTs.[25] League of Legends uses the Riak CRDT implementation for its in-game chat system, which handles 7.5 million concurrent users and 11,000 messages per second.[26] Bet365, stores hundreds of megabytes of data in the Riak implementation of OR-Set.[27]

`TomTom` employs CRDTs to synchronize navigation data between the devices of a user.

Phoenix, a web framework written in Elixir, uses CRDTs to support real time multi-node information sharing in version 1.2.[29]

Facebook implements CRDTs in their Apollo low-latency "consistency at scale" database.[30]

Teletype for Atom employs CRDTs to enable developers share their workspace with team members and collaborate on code in real time.[31]

Haja Networks' OrbitDB uses operation-based CRDTs in its core data structure, IPFS-Log.[32]

Apple implements CRDTs in the Notes app for syncing offline edits between multiple devices.[33]