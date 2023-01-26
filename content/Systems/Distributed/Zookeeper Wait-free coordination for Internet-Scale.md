## Overview

Zookeeper is a foundational system that enables distributed systems and workloads by providing a kernel to develop complex coordination primitives. In this way, it serves no well-defined workload, but instead, positions itself as a building block for many of the highly specialized systems we see deployed in massive scale services today.

It builds upon several fundamentals, namely:

- The structure and interface found in generic file systems
- The wait-free characteristics of shared registers
- Event-driven mechanisms utilized by cache invalidation schemes

Along with these improvements, it guarantees:
 - per client FIFO execution
 - linearizable updates to the internal state
 - reads serviced purely from local servers.

Local reads allow Zookeeper to achieve hundreds of thousands of transactions per second with anywhere from 2-100:1 read to write ratio, which is incredibly important due to the read heavy nature of most Zookeeper deployments. These fast reads forgo precedence order for read operations, which allows Zookeeper to return stale znode values.

Like most systems, Zookeeper is replicated over an ensemble of servers to distribute load. In order to maintain consistency between these replicas, Zookeeper relies on Zab, a leader-based atomic broadcast protocol. Zab conveys state changing operations that pass through a leader to all follower nodes. 

Clients can issue multiple, outstanding asynchronous requests while still observing consistency, due to the guarantee of per client FIFO operations. This allows for efficient implementations for updates on the client side.

## At a High Level

Zookeepers basic design appears fairly common at a surface level. It is from this simplicity that it is able to provide so much power to systems that utilize it. Below is a brief description of the fundamental components. It will be followed by how these building blocks are able to ensure consistency and provide the wait free, event driven architecture that is able to achieve the guarantees and performance necessary to enable other coordination principles.

### Sessions

Clients are connected to a single Zookeeper server that services all of their reads throughout the life of the session. These sessions are assigned a timeout. If the server does not interact with the client within this timeout, they are deemed faulty and the sessions is closed. Sessions can also be explicitly closed by the client. 

### Znodes and the Data Tree

Znodes are the fundamental data objects within the zookeeper system. They store all the relevant information that clients wish to ensure consistency over. Znodes are arranged into a tree structure, where the path to a particular znode resembles the path to a file in a typical filesystem. Path components (directories) are referred to as namespaces, and are used as a way of providing multiple primitives to many applications, all interacting with the same Zookeeper instance. It is important to note, that each namespace is itself a znode and acts as the parent to the next level of hierarchy. 

There are two variants of znodes:

- Regular: manipulated via normal creates and deletes
- Ephemeral: created by clients, deleted explicitly or by the system when the corresponding client session is deleted

Znodes can also be created with the SEQUENTIAL flag, which allocates an ID to the node that is monotonically increasing within the context of the node's parent. 

### Client API

Znodes can be interacted with in typical CRUD fashion. Zookeeper allows for both synchronous and asynchronous versions. It further extends these operations with event-based watchers, and session level sync calls to ensure all pending writes have flushed from the leader to the connected replica.

- create(path, data, flags):
    - Creates a znode under the specified path, storing data[].
    - Returns the name of the new znode
    - Flags specifies the type of znode and and whether it is to be allocated sequentially
- delete(path, version):
    - Deletes the znode at path if it is at the expected version
- exists(path, watch):
    - Determines if a znode exists at path. Watch allows clients to receive an event if a node at path is ever created.
- getData(path, watch):
    - Returns data and znode metadata (version, etc.) at path. If the node exists, a watch is set on the znode if it exists.
- setData(path, data, version):
    - Writes data[] to path if the the version matches the current value stored in the znode metadata.
- getChildren(path, watch):
    - Returns the set of names (paths) for the children of the znode at path.
- sync(path):
    - Waits for all pending updates (at the time of invocation) to flush to the server that the client is connected to.
    - path is currently ignored?

Note: setting version = -1 ignores all version checks

## In the weeds

### Requests and Client Interaction

Reads:

As mentioned above, each client connects to a single Zookeeper server, which performs reads from a local replica. Each read is tagged with a zxid, that corresponds to the last transaction seen by the server. This zxid ensures FIFO ordering of reads by client, even in the event of a server failure. When a client reestablishes a connection, there are two possible states of the server it connects to.

1. Server is up to date with (or ahead of) zxid
    1. Server allows reconnection and begins servicing reads immediately
    2. Some state may need to be replayed to clients for the messages between the client zxid and the server zxid.
2. Server is behind zxid
    1. Server denies sessions creation, and client is rerouted to a server that is up date.
    2. Given that zxids are sent only when a quorum has reached concensus, it is guaranteed that the client can find another server that has seen zxid.

Watches:

These are also managed locally, when an update is received via Zab, the process looks at its local watcher table, keyed by the znode name, and pushes the new state to any client that is connected that exists in the watch table for that znode

Heartbeats:

Clients send occasional heartbeats when they perform no other interaction with the server in within the timeout configured. This avoids the overhead associated with unnecessary reconnections. Responses to heartbeats also include the zxid to reduce the amount of catchup data sent to a client in the case of a server failure.

Sync:

In some cases, stale reads are not acceptable from a system design perspective. For these cases, Zookeeper introduced the sync operation to ensure that all changes have been flushed to replicas for the znode specified in the request. Clients issue a sync, followed by a read request. The sync operates as a phantom write in which the leader places it onto the request queue for the follower executing the sync. Clients will observe the sync message only after all previous writes to the state have been performed on the local server. Therefore any subsequent reads are certain to be up to date to the point of the sync.

Requests that change the state of the database go through a leader and are atomically broadcast to replicas via Zab. 

### Zab

Zab guarantees that followers never diverge, but does not ensure that each follower maintains the same state at any single point in time. It is possible, and in fact fairly common, for servers to have applied more (or less) transactions that another.

Zab utilizes simple majority quorums by default.

It extends atomic broadcast by ensuring that changes are delivered in the order they are sent, and all changes from previous leaders are delivered to a newly established leader before it may communicate its own changes. This does provide stronger consistency guarantees, but comes at the cost of longer stalls when leaders rollover. 

TCP provides network level ordering, which removes the need for Zab to implement its own message ordering protocol.

Zookeeper repurposes the Zab leader as its own so that the process that creates transactions also proposes them.

Under healthy operation Zab observes exactly once delivery, but in the case of failure, duplicate messages may be sent because Zab does not persist a record of delivered message IDs. However, because state operations are idempotent, replaying messages will still result in the same end state.

### Storage and Recovery

Zookeeper maintains replica stores in memory. Each znode is limited to 1MB in size by default, but this can be increased by configuration. 

As part of a write request, the leader must calculate the resultant state of the database and transform it into a transaction to capture the new state. Concurrent requests could alter the state of the database and invalidate the resultant state of other transactions (as in the case of a version conditioned write). Without this precalculation of the end state, Zookeeper would be unable to maintain the idempotent operation that underpins its functionality. 

To ensure recoverability, updates are logged to disk before they are written to memory and served to clients. Snapshots of the database are maintained to avoid replaying the entire log when a faulty replica is replaced. Zookeeper avoids locking the entire database during the snapshotting process by performing a depth first scan of the tree, persisting to znodes to disk in the order of traversal. This process is known a  fuzzy snapshot. It is relevant to note that the snapshot itself may not be equivalent to the exact state of the database at any one point in time. However, state changes are idempotent in Zookeeper and a depth first traversal guarantees that we maintain the ordering of sequential node. Therefore, transactions can be reapplied via the log to reach the same end state.

## Problem

---

*Description of the current state of reality and why that is insufficient going forward.*

- Existing distributed systems attempt to address a narrow set of use cases and usually offer functionality focused on solving a specific problem that arises from coordination and consistency across multiple servers.
- Lack of flexibility requires developers to learn and deploy many different frameworks/services to solve the various issues they face.
    - Each of these comes with their own set of quarks and tradeoffs which developers must we aware of

*What need exists, and why is it important?*

- Systems require coordination and consistency for managing data and communication between distributed processes.

## Approach

---

*Describe the main design and method for which this attempts to solve an issue*

*How does this improve upon the current state or state of the art?*

- Zookeeper provides a "coordination kernel" that exposes an API to allow clients to build their own primitives
- This API exposes some of the functionality included in a basic file system, but does not manage file/directory handles or open/close patterns
- Allows for asynchronous and synchronous operations
- Allows for forced consistency through sync() method, which serves to flush all writes out to the client prior to the call
- FIFO operation order per client
- Linearizable Writes

*What tradeoffs does it make, and what features/outcomes does it prioritize?*

- Weak consistency for reads and watches (not globally linearizable, not precedence ordered)
    - Gives better performance per client, at the expense of consistent values across all users

*Does it take build upon any existing systems?*

- Built on top of Zab
    - Leader-based atomic broadcast protocol
    - Zab not used to totally order reads, but does broadcast writes to replicas
    - Uses simple majority quorums
    - Guarantees changes broadcast by a leader are delivered in the order they were sent and all changes from previous leaders are delivered to an established leader before it broadcasts its own changes
    - Leader chosen by Zab is also the leader chosen for Zookeeper
    - Delivers messages in order, but may redeliver during recovery, because messages are not persisted permanently

*Does it leverage any other foundational research in the design approach?*

- Many comparisons to Chubby Lock Service throughout
- Principles/structures of file systems

## Results

---

*Is it able to achieve the goal it initially set out?*

*How does it compare to other systems (additional guarantees, resource usage, speed, etc.)?*

## Deficiencies

---

*Do the authors acknowledge any shortcomings of the system?*

*What concerns do you have with the statements made?*

*What do you feel the authors may have left out, intentionally or unintentionally?*

*How has new research (if known) changed the framing of these results?*

## Use Cases

---

*How could this research be applied in other systems?*

- Often used for configuration management
- Can implement simple locks (without herd effect)
- Read/write locks
- Double Barrier
- Katta
    - Distributed indexer
    - divides work of indexing per shard
    - Tracks the status of slave servers and the master, handles failover, assignment of shards to slaves via Zookeeper
- Fetching Service (Yahoo)
    - Manages configuration data for master and page-fetching processes
    - Leader election
- Yahoo Message Broker
    - Manages distribution of topics, failover, and control system operation
    - Ephemeral znodes for the replica metadata (load & status)
    - Subscribers are managed as children of topics

*Does it have greater implications for the field as a whole?*

- Yes
- Because it sets out to provide a foundation for building primitives, many complex systems can utilize the underlying infrastructure to implement their own distributed systems

## Unanswered Questions

---

*Are there any statements that confuse you? (Can you do any additional research to clarify for yourself)*

*Are there any offshoots of this research that haven't been explored?*

## Takeaways

---

*Summarize the key points of the research*

*What information do you hope to retain from this?*

## Especially Unique, Interesting, or Clever

---

*Are there any points, techniques, insights, results, etc. that stand out?*