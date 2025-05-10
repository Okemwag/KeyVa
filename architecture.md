# KeyVa Store Architecture

## High-Level Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                          Client Interface                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐    │
│  │    REST     │    │    gRPC     │    │  Command Line/SDK   │    │
│  └─────────────┘    └─────────────┘    └─────────────────────┘    │
└───────────────────────────────────────┬───────────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────────┐
│                          Request Handler                          │
│  ┌─────────────────────┐    ┌─────────────────────────────────┐   │
│  │ Authentication/Auth │    │        Rate Limiting            │   │
│  └─────────────────────┘    └─────────────────────────────────┘   │
└───────────────────────────────────────┬───────────────────────────┘
                                        │
┌───────────────────────────────────────▼───────────────────────────┐
│                          Core Engine                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐    │
│  │   Command   │    │ Transaction │    │ Concurrency Control │    │
│  │  Processor  │    │  Manager    │    │                     │    │
│  └─────────────┘    └─────────────┘    └─────────────────────┘    │
└───────────────────────────────────────┬───────────────────────────┘
                                        │
                 ┌─────────────────────┐▼┌─────────────────────┐
                 │                      │                      │
┌────────────────▼─────────────┐ ┌──────▼───────────────────┐ ┌▼────────────────────────┐
│        Storage Engine        │ │     Indexing Layer       │ │    Replication Layer    │
│  ┌─────────────────────────┐ │ │ ┌─────────────────────┐  │ │ ┌────────────────────┐  │
│  │     Memory Storage      │ │ │ │   B+Tree / LSM      │  │ │ │  Leader Election   │  │
│  ├─────────────────────────┤ │ │ ├─────────────────────┤  │ │ ├────────────────────┤  │
│  │   Persistent Storage    │ │ │ │     Hash Index      │  │ │ │   Log Shipping     │  │
│  ├─────────────────────────┤ │ │ └─────────────────────┘  │ │ ├────────────────────┤  │
│  │   Compaction Manager    │ │ └──────────────────────────┘ │ │  Conflict Resolution│ │
│  └─────────────────────────┘ │                             │ └────────────────────────┘
└────────────────────────────┬─┘                             │
                             │                               │
┌────────────────────────────▼───────────────────────────────▼─────┐
│                        Monitoring & Management                    │
│  ┌────────────┐  ┌────────────────┐  ┌────────────┐  ┌──────────┐ │
│  │  Metrics   │  │ Configuration  │  │   Backup   │  │  Logging │ │
│  └────────────┘  └────────────────┘  └────────────┘  └──────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. Storage Engine

The storage engine is responsible for actually storing and retrieving key-value pairs. Let's look at two popular approaches:

#### Hash Table Based (In-Memory)
```
┌─────────────────────────────────────────────────────────────┐
│                       Hash Table                            │
├─────────┬─────────┬─────────┬─────────┬─────────┬───────────┤
│ Bucket 0│ Bucket 1│ Bucket 2│   ...   │Bucket n-1│ Bucket n │
├─────────┼─────────┼─────────┼─────────┼─────────┼───────────┤
│┌───┬───┐│┌───┬───┐│         │         │┌───┬───┐│┌───┬───┐  │
││k1 │v1 │││k8 │v8 ││   NULL  │   ...   ││k45│v45│││k3 │v3 │  │
│└───┴───┘│└───┴───┘│         │         │└───┴───┘│└───┴───┘  │
│┌───┬───┐│         │         │         │┌───┬───┐│┌───┬───┐  │
││k5 │v5 ││   NULL  │   NULL  │   ...   ││k12│v12│││k77│v77│  │
│└───┴───┘│         │         │         │└───┴───┘│└───┴───┘  │
└─────────┴─────────┴─────────┴─────────┴─────────┴───────────┘
           hash(key) % n determines bucket placement
```

#### LSM Tree Based (Disk-Based)

```
┌─────────────────────────────────────────────────────────────┐
│                     Memory (MemTable)                       │
│  ┌─────────────────────────────────────────────────┐        │
│  │  Sorted Key-Value Pairs (e.g., Skip List/RBTree)│        │
│  └─────────────────────────────────────────────────┘        │
└──────────────────────────┬──────────────────────────────────┘
                           │ Flush when full
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     Disk (SSTable)                          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐     ┌─────────┐        │
│  │ Level 0 │ │ Level 0 │ │ Level 0 │ ... │ Level 0 │        │
│  └─────────┘ └─────────┘ └─────────┘     └─────────┘        │
│        ▼                                                    │
│  ┌──────────────────────────────────────────────┐           │
│  │               Level 1 (Larger)               │           │
│  └──────────────────────────────────────────────┘           │
│        ▼                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  Level 2 (Even Larger)              │    │
│  └─────────────────────────────────────────────────────┘    │
│        ▼                                                    │
│       ...                                                   │
└─────────────────────────────────────────────────────────────┘
             Periodically compacted to optimize space
```

### 2. Indexing Layer 

```
┌───────────────────────────────────────────────────────────┐
│                      Index Structure                      │
│                                                           │
│                          ┌─┐                              │
│                          │R│                              │
│                          └┬┘                              │
│                   ┌───────┴───────┐                       │
│                   │               │                       │
│                  ┌┴┐             ┌┴┐                      │
│                  │A│             │N│                      │
│                  └┬┘             └┬┘                      │
│           ┌───────┴───┐       ┌───┴────┐                  │
│           │           │       │        │                  │
│          ┌┴┐         ┌┴┐     ┌┴┐      ┌┴┐                 │
│          │B│         │F│     │P│      │Z│                 │
│          └┬┘         └┬┘     └┬┘      └─┘                 │
│     ┌─────┴─────┐     │       │                           │
│     │           │     │       │                           │
│    ┌┴┐         ┌┴┐   ┌┴┐     ┌┴┐                          │
│    │C│         │E│   │G│     │Q│                          │
│    └─┘         └─┘   └─┘     └─┘                          │
│                                                           │
└───────────────────────────────────────────────────────────┘
                B+ Tree or similar structure
```

### 3. Transaction Manager

```
┌───────────────────────────────────────────────────────────┐
│                   Transaction Manager                     │
│                                                           │
│  ┌─────────────────┐     ┌──────────────────────────┐     │
│  │ Transaction Log │<───>│ Transaction Queue/Pool   │     │
│  └─────────────────┘     └──────────────────────────┘     │
│          ▲                          │                     │
│          │                          ▼                     │
│  ┌─────────────────┐     ┌──────────────────────────┐     │
│  │   Write-Ahead   │     │ Transaction Coordinator  │     │
│  │   Log (WAL)     │<───>│   (Commit/Rollback)      │     │
│  └─────────────────┘     └──────────────────────────┘     │
│                                     │                     │
│                                     ▼                     │
│                         ┌──────────────────────────┐      │
│                         │ Concurrency Control      │      │
│                         │ (MVCC/Locks/Timestamps)  │      │
│                         └──────────────────────────┘      │
└───────────────────────────────────────────────────────────┘
```

### 4. Replication Layer

```
┌───────────────────────────────────────────────────────────┐
│                     Replication                           │
│                                                           │
│                   ┌─────────────┐                         │
│              ┌───>│   Leader    │<────┐                   │
│              │    └─────────────┘     │                   │
│              │          │             │                   │
│              │          │             │                   │
│      ┌───────┼──────────┼─────────────┼────────────┐      │
│      │       │          │             │            │      │
│      ▼       ▼          ▼             ▼            │      │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐    │      │
│ │Follower │ │Follower │ │Follower │ │Follower │    │      │
│ │   1     │ │   2     │ │   3     │ │   4     │────┘      │
│ └─────────┘ └─────────┘ └─────────┘ └─────────┘           │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

## Detailed Component Breakdown

### 1. Client Interface Layer

* **REST API**
  * Handles HTTP requests (GET, PUT, DELETE)
  * JSON/Protocol Buffers data format
  * Well-defined endpoints (/kv/{key})

* **gRPC Interface**
  * Efficient binary communication
  * Strong type checking with .proto files
  * Streaming capabilities for bulk operations

* **Client SDK/Libraries**
  * Language-specific wrappers (Go, Java, Python)
  * Connection pooling
  * Automatic retries and backoff strategies

### 2. Storage Engine Components

* **Memory Table (MemTable)**
  * In-memory buffer for recent writes
  * Typically implemented as a skip list or red-black tree
  * Provides fast access to hot data

* **Persistent Storage Structures**
  * **SSTables (Sorted String Tables)**
    * Immutable files containing sorted key-value pairs
    * Block-based layout with index for efficient lookups
    * Tiered or leveled organization
  
  * **Segment Files**
    * Fixed-size data chunks
    * Sequential writes for performance
    * Possible append-only format

* **Write-Ahead Log (WAL)**
  * Sequential log of all write operations
  * Enables crash recovery
  * Periodically checkpointed

* **Data Compression**
  * Block-level compression (LZ4, Snappy, zstd)
  * Dictionary-based compression for repeated patterns
  * Prefix compression for similar keys

* **Compaction Manager**
  * Merges multiple SSTables into fewer, larger ones
  * Removes deleted/stale entries
  * Reclaims disk space
  * Optimizes read performance

### 3. Indexing Layer Components

* **Primary Index**
  * Maps keys directly to data location
  * Can be hash-based (for random access) or B+Tree (for range queries)

* **Secondary Indexes**
  * Optional indexes on value fields
  * Enables more complex queries

* **Bloom Filters**
  * Probabilistic data structure
  * Quickly checks if a key might exist
  * Reduces unnecessary disk I/O

* **Block Cache**
  * Caches frequently accessed data blocks
  * LRU/LFU eviction policies
  * Configurable size

### 4. Concurrency Control

* **Locking Mechanisms**
  * Fine-grained locks (key-level)
  * Read-write locks for better parallelism
  * Deadlock detection/prevention

* **Multi-Version Concurrency Control (MVCC)**
  * Maintains multiple versions of data
  * Readers don't block writers
  * Timestamp-based versioning

* **Isolation Levels**
  * Read Uncommitted
  * Read Committed
  * Repeatable Read
  * Serializable

### 5. Replication Components

* **Consensus Protocol**
  * Raft/Paxos for leader election
  * Log replication for data consistency
  * Fault tolerance through quorum

* **Replication Modes**
  * Synchronous (strong consistency)
  * Asynchronous (eventual consistency)
  * Semi-synchronous (configurable durability)

* **Conflict Resolution**
  * Vector clocks
  * Last-write-wins
  * Custom merge functions

### 6. Sharding & Partitioning

```
┌───────────────────────────────────────────────────────────────┐
│                      Key Space                                │
│                                                               │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐     │
│  │Partition │Partition │Partition │Partition │Partition │     │
│  │    1     │    2     │    3     │    4     │    5     │     │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘     │
│                                                               │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐   │
│  │                │  │                │  │                │   │
│  │    Node 1      │  │     Node 2     │  │     Node 3     │   │
│  │ ┌────┐ ┌────┐  │  │  ┌────┐ ┌────┐ │  │  ┌────┐ ┌────┐ │   │
│  │ │Par1│ │Par5│  │  │  │Par2│ │Par3│ │  │  │Par4│ │Rep1│ │   │
│  │ └────┘ └────┘  │  │  └────┘ └────┘ │  │  └────┘ └────┘ │   │
│  │ ┌────┐         │  │  ┌────┐        │  │  ┌────┐        │   │
│  │ │Rep2│         │  │  │Rep4│        │  │  │Rep3│        │   │
│  │ └────┘         │  │  └────┘        │  │  └────┘        │   │
│  └────────────────┘  └────────────────┘  └────────────────┘   │
└───────────────────────────────────────────────────────────────┘
      (ParX = Partition X, RepY = Replica of Partition Y)
```

* **Partitioning Strategies**
  * Range partitioning (by key ranges)
  * Hash partitioning (by key hash)
  * Consistent hashing (for dynamic scaling)

* **Partition Management**
  * Rebalancing logic
  * Partition splitting/merging
  * Hot spot detection

### 7. Monitoring & Management

* **Metrics Collection**
  * Throughput (ops/sec)
  * Latency percentiles
  * Storage utilization
  * Cache hit rates

* **Administrative Tasks**
  * Backup/restore functionality
  * Node addition/removal
  * Configuration management

* **Logging**
  * Operation logs
  * Error logs
  * Slow query logs

## Key Concepts to Implement in Go

1. **Data Structures**
   * Memory-efficient storage representations
   * Go's maps and sync.Map for in-memory storage
   * Custom B+Tree or LSM implementations

2. **Concurrency Primitives**
   * Mutexes, RWMutexes
   * goroutines and channels for parallel processing
   * sync.WaitGroup for coordinated operations

3. **File I/O**
   * os package for file operations
   * mmap for memory-mapped files
   * buffered I/O for performance

4. **Networking**
   * net/http for REST API
   * gRPC for efficient client-server communication
   * Connection pooling and management

5. **Serialization**
   * encoding/json for human-readable format
   * encoding/gob or Protocol Buffers for binary format

6. **Testing & Benchmarking**
   * Unit tests with edge cases
   * Benchmarks for performance metrics
   * Fault injection testing
