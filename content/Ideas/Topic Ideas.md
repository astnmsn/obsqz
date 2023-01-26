1. Bloom Filters
2. Types of log compaction [Universal](https://zhangyuchi.gitbooks.io/rocksdbbook/content/Universal-Compaction.html) [Comparison of many](https://www.alibabacloud.com/blog/an-in-depth-discussion-on-the-lsm-compaction-mechanism_596780)
3. Schedulers (Cooperative v Preemptive) -> [Tokio](https://tokio.rs/blog/2019-10-scheduler) [Go](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
4. Concurrency Primitives
5. How atomics work in a CPU
	1. Rust -> std::sync::atomic::[Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html#)
	2. C++ -> [reference](https://en.cppreference.com/w/cpp/atomic/memory_order#Sequentially-consistent_ordering)
6. MVCC
7. DNS
8. Load balancing algorithms
9. Cacheing -> [Good place to start](https://blog.frankel.ch/choose-cache/1/)
	1. Patterns
	2. Eviction Strategies
	3. Invalidation
10. Cache Coherence
11. Concurrent Data structures
12. Distributed hash tables [Overview](https://codethechange.stanford.edu/guides/guide_kademlia.html) []
13. Database consistency/isolation levels
14. Content Addressing (IPFS)
15. Merkle Dags -> [Lessons](https://proto.school/merkle-dags)
16. Merkle trees
17. Public Key Cryptography
18. Semantic Search
19. SIMD
20. [RocksDB Book](https://zhangyuchi.gitbooks.io/rocksdbbook/content/)
21. [Threading](https://en.wikipedia.org/wiki/Thread_(computing))
22. Postgres terminology -> [Post](https://www.crunchydata.com/blog/challenging-postgres-terminology)
23. Package deep dives
	1. tokio
	2. rayon
	3. crossbeam
	4. mio
	5. flashmap