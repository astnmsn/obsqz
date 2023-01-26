
[Paper](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)

# Overview

Calvin is a transaction scheduling and data replication layer that uses a deterministic ordering guarantee which reduces the cost associated with distributed transactions to a level that makes them feasible to run at large scale.

Calvin:

- Supports disk-based storage
- Scales near linearly on a cluster (add note of what makes it "near")
- No single point of failure (as is the case with many leader based systems)

Calvin replicates transaction inputs rather than effects, and in so doing, is able to support multiple levels of consistency at no cost to transaction throughput

- Paxos-based strong consistency across distant replicas
- Others

Many other systems forgo transactions to enable high throughput, low latency, and availability

- Dynamo (Amazon)
- MongoDB
- CouchDB
- Cassandra

Others provide limited transaction support, generally within a single slice of data (local to one replica) to provide linear outward scalability

- Azure
- Bigtable/Megastore
- Oracle NoSQL database

Calvin runs alongside a non-transactional storage system, meaning it does not rely on any specific storage structure, nor does it manage any of the underlying data. It elevates existing database systems, using a "shared-nothing" architecture, that is "near-linearly" scalable and provides high availability and full ACID transactions, that can even span multiple partitions.

Calvin acts as a mediator between the client and the underlying database, utilizing distributed locks to remove the need for typical distributed commit protocols to perform replication.

Calvin can sit on top of existing storage systems because it operates outside of transaction boundaries, before database level locks come into play. To enable this, an agreement on how to perform a transaction must be reached by each replica, and execution must be guaranteed. Nodes that fail can recover by communicating with other replicas that were performing the same transaction in parallel, or replay the plan of record from that node. However, to avoid divergence in the case of failure, the activity plan must deterministic.  

Because Calvin guarantees the exact, ordered list of transactions to execute on all nodes, it does not require any distributed commit protocols.

### 4 Main Contributions:

1. Transaction scheduling and data replication layer that augments non-transactional storage, introducing strong consistency and ACID transactions, while maintaining near-linear scalability & high availability.
2. More scalable than previous distributed commit protocols, without a single point of failure
3. Data prefetching during the planning phase, that allows for operations on purely disk-resident data without extending the contention footprint of a transaction for the full duration of disk lookups
4. A fast checkpoint scheme that helps remove the need for physical REDO logging and its associated overhead

# System Architecture

Designed to work with any storage system that accepts a basic CRUD interface

Calvin organizes the system into 3 distinct layers of processing:

1. Sequencing
    1. Intercepts transaction inputs from the client
    2. Places them onto a global transaction queue that guarantees serial ordering across the replicas
    3. Handles replication and logging of the input sequence across nodes
    4. Distributed across all system replicas and also partitioned across every machine within each replica. This prevents the sequencer from being a single point of failure, and removes the throughput bottleneck that a single node sequencer, serving a many node subsystem, would create.
    5. This is the single point where replication occurs. Schedulers and storage do not need to communicate to other replicas as their actions/state are deterministic given the input from the sequencer.
    6. Replication can be performed using asynchronous leader/follower nodes, or via a synchronous Paxos-based implementation in which all sequencers in a replication group must agree on a combined batch of transaction requests per epoch.
2. Scheduling
    1. Utilizes the distributed locking scheme to guarantee equivalence in execution of the schedule provided by the previous layer, while allowing transactions to be performed in parallel across execution threads
    2. Each node's scheduler is only responsible for locking records that are stored within the partition that the node manages
    3. The locking protocol is very similar to traditional 2 phase locking but includes:
        1. Locks must be requested in the same order as the transaction requiring them appears within the sequence. This is achieved by serializing transactions within a single thread, and requesting all the locks a transaction will need throughout its life time, as it parses the global sequence of transactions
        2. The lock manager must grant locks in the order they are requested, and only release them when the corresponding transaction completes, and releases all of its locks.
3. Storage
    1. Handles all physical data layout
    2. Interacts with underlying database subsystem via the CRUD interface provided

All 3 layers can be partitioned across a cluster without any shared components between instances of each layer. They are typically run one per node, but can be adapted to fit the scaling characteristics of the underlying storage mechanism.

![[calvin-1.png]]
Process ordering:
1. Requests are compiled into 10 millisecond batches by Sequencers and replicated to the  sequencers of the corresponding partition on replica nodes.
2. Sequencers send these batches to every partition within their replica with the following info:
    1. Sequencer unique node ID
    2. The epoch number (which is synchronously incremented across the entire system)
    3. All transaction inputs collected that the recipient will need to participate in
3. Scheduler interleaves the transactions sent by the sequencer within their replica, thus constructing a global ordering
4. For each transaction in the ordering the scheduler grants the transaction all of its required locks, and hands it off to a worker thread to be executed.
5. Each worker thread proceeds in five phases:
    1. Read/write set analysis
        1. The elements of the r/w set that exist in the partition for the node
        2. The set of nodes at which elements of the write set are stored
        3. Active participants: those where writes occur
        4. Passive participants: those where only reads occur
    2. Perform local reads
    3. Serve remote reads
        1. Forward to parallel worker threads on other actively participating nodes
    4. Collect remote reads
    5. Execute (Transaction logic and applying writes)
        1. Only local writes (to the nodes partition)

Calvin does not natively support transactions which must perform reads in order to determine the read/write set because the deterministic locking protocol it is based upon requires that the set of locks needed by a transaction be known before the transaction is executed.

To get around this Calvin does allow for Optimistic Lock Location Prediction; an inexpensive, low-isolation, unreplicated, read-only query that can be run prior to the transaction to uncover the full read/write. Because the results of these queries are not guarded by the transaction's boundary, they must be rechecked on execution, and the process restarted. This is commonly the case for secondary indexes (but because secondary indexes are applied to values that should be rarely updated, the process should not often need be restarted).

## Calvin with Disk-Based Storage

Because Calvin must follow the transaction order determined by the scheduler, it cannot reorder transactions on the fly (as in many traditional databases). This means that it cannot move non-overlapping transactions up in the order if others are stalled (for disk access).

To remedy this Calvin incurs the cost prior to acquiring locks. It introduces a delay (fixed or deterministic) into the scheduling process to account for any cold-fetch of data from disk. It issues a request to preload this data into memory so that it will be resident when the transaction actually occurs.

Tuning for the length of this delay is highly important to performance. Too low and the system throughput will suffer the same contention issues that exist without the prefetch. Too high and the system could overload memory while waiting for longer than is necessary and adds un-utilized latency.

The other component that proves non-trivial is determining which documents are already resident in memory. A lookup table must be stored at all sequencers which becomes intractable at large scale. The only alternative to not tracking, is to introduce the delay for all transactions. 

## Checkpointing

As with many systems that maintain a log of transactions, it is possible to reconstruct a database by replaying the full list of transactions in order. However, this is generally very inefficient. Instead Calvin, and most other systems, take periodic snapshots of the database state, and replay only the transaction after the snapshot.

Remote reads to failed machines within a replica are likely to slow or halt until the afflicted machine is able to replay the transaction log from the point of the last snapshot.

Naive Synchronous Checkpointing:

- Snapshot is generated at a point in time from a replica. Client interaction is unaffected during this period, assuming there are no other failures in the system.
- The snapshot replica may fall significantly behind the leader, and catching up may prove difficult in high load systems. This becomes especially concerning if the leader fails during the process, and the snapshot is called into action.

Asynchronous Zig-Zag

- Two copies of each record are stored, plus two additional bits that signify which copy to read and which to overwrite.
- During normal operation, the read bit is set equivalent to the write bit to ensure the newest value is read.
- At the time of a checkpoint, the write bit is set to the opposite of the read bit for all values, so that new writes will not blast the values present at the time of the checkpoint. This requires a period of quiescence (only DBA operations can be performed) to ensure consistency.
- The frozen copy (that is not being written to, or read from) is used to generate the snapshot.
- This method retains the last snapshot value at all points in time in the ¬write copy.
- Calvin's improvement:
    - Because of its global serial order, it does not require quiescence.
    - Calvin can specify a point the serial order, around which it can start storing multiple versions of records (before and after).
    - Once all before transactions have completed, those versions become immutable and a thread can begin checkpointing them to disk.
    - To cleanup the, all records that have both a before and after version stored can dispose of the before version.

Full Multiversioning Storage

- Allows read only queries to be executed without acquiring locks
- Calvin can simply scan (SELECT *) over the database and log the result to a file