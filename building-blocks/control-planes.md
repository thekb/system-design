# Control Planes

Control planes have certain unique needs:

* Many independent resource lifecycles (clusters, workloads, policies, users, etc.)
* Idempotency and replayability are critical (crash recovery, retries, failover)
* Multi-region replication often needed
* Low-latency writes, but reads can be served from materialized views
* Strong auditability and compliance needs

WAL-based event sourcing fits this perfectly.

## Mental Model
In WAL-based event sourcing:

* The WAL (Write-Ahead Log) is your append-only stream of events.
* All state changes are written to WAL first.
* The "authoritative state" is not a database row — it’s the log.
* Materialized views (DB tables, caches, APIs) are built by replaying the WAL.

Think of it like this:
```
Client → Command → Validation → WAL Append → Materializer(s) → Read Models
```

## What exactly is in the WAL?
Each entry is typically:
```json
{
  "timestamp": "2025-06-15T19:30:00Z",
  "tenant_id": "tenant-123",
  "aggregate_id": "cluster-456",
  "version": 17,
  "event_type": "ClusterScalingRequested",
  "payload": {
    "new_node_count": 10
  },
  "metadata": {
    "schema_version": 2,
    "actor": "user-789"
  }
}
```
* aggregate_id: entity whose state this affects (e.g. cluster ID, workload ID)
* version: strictly increasing per-aggregate version (used for optimistic concurrency)
* event_type: what happened
* payload: event data

## How writes work
1. Client issues a command (desired intent)
2. Validation happens (precondition checks, auth, quotas, business logic)
3. If valid, control plane appends one or more events to WAL
4. WAL append is the only authoritative write
5. Downstream systems consume WAL for materializing current state

## Materializer Layer
This is where WAL gets translated into queryable state.
* Read Models: SQL tables, KV stores, document DBs
* Views: e.g. cluster health summaries, workload lists
* Derived State: e.g. quota usage, policy evaluations

Materializers subscribe to WAL and apply events to build state.


## Replay model
On startup or recovery, materializers can replay WAL from offset 0 (or from snapshot). This allows:
* Crash recovery
* Rebuilding corrupted read models
* Adding new read models retrospectively

## Why is WAL better for control planes vs direct DB writes?
| WAL model                | Traditional DB writes            |
| ------------------------ | -------------------------------- |
| Immutable log            | Mutable tables                   |
| Full audit history       | Often limited change history     |
| Easy to replicate        | Hard to replicate precisely      |
| Replayable               | Point-in-time recovery difficult |
| Idempotent application   | Race conditions possible         |
| Easier to evolve schemas | Schema migrations are riskier    |

## Replication & Multi-region
* WAL entries can be replicated across regions (e.g. Kafka, Kinesis, Pulsar, custom log replication).
* Each region runs its own materializer locally.
* If a region loses state → replay WAL → full recovery.
* Replication guarantees:
    * Ordered per aggregate ID (sharded WALs help)
    * Global ordering often unnecessary
    * Consistency models can be tuned: strong per-aggregate, eventual global.

## Versioning & Evolution
* Every event schema is versioned.
* Materializers must handle old and new event versions.
* Protobuf / Avro / JSON Schema commonly used.
* Allows safe rolling upgrades across control planes.

## Common optimizations
* Snapshots (periodic state capture to reduce replay time)
* WAL compaction (for long-lived aggregates)
* Partitioning WAL per resource type or aggregate ID
* WAL retention policies (e.g. keep full log 30d, compact after)
* WAL integrity checksums for audit

# One simple diagram
```
          +----------+
 Client → | Command  | 
          +----------+
                ↓
          +----------+
          |  WAL     |  (Append-only)
          +----------+
                ↓
      +--------+--------+
      | Materializer(s) |
      +--------+--------+
        ↓            ↓
    Read DBs    Derived State
```

## WAL Sharding Models

The goal of WAL sharding is simple:

* Maximize throughput
* Minimize contention
* Ensure per-aggregate consistency
* Enable horizontal scalability

### Baseline constraint

In event sourcing, order matters per aggregate (e.g. per cluster, per workload, per tenant).

But global order is rarely required.

Thus:
Shard the WAL such that:

* all events for one aggregate go to the same shard
* shards can be processed independently

### Sharding Keys

#### Aggregate ID based sharding (most common)

Example: shard by hash(aggregate_id) % N

Each cluster/workload always lands on same WAL shard.

Ensures strict order for that aggregate.

Parallelizes well across shards.

Can dynamically scale shards if you support resharding.

Excellent for control planes with independent cluster lifecycles.

#### Tenant ID based sharding
Example: hash(tenant_id) % N

Ensures all aggregates for one tenant land on same shard.

Simplifies noisy neighbor isolation.

Simplifies cross-tenant query boundaries.

Can make materializers very tenant-centric.

Useful for SaaS-style multi-tenant control planes.

#### Hybrid sharding: (Tenant, Aggregate)
Example: hash(tenant_id, aggregate_id) % N

Balances fine-grained sharding while preserving tenant-level isolation.

Avoids “hot tenant” problem.

Useful when you have both very large tenants and many small tenants.

## WAL Shard Implementation Patterns

### Model 1: Kafka-style partitioned WAL
Use Kafka topics where partitions == WAL shards.

Shard count = initial capacity planning (resizable with care).

Producer routes events to partition via key.

Materializers consume from assigned partitions.

```
Kafka Topic: control-plane-wal
Partitions: 1024
Producer → partitioner → correct shard
Consumer Group → shard assignment
```
* Battle-tested at massive scale
* Horizontal scaling of consumers
* Natural checkpointing via Kafka offsets

### Model 2: Table-based WAL shards
Each shard is its own DB table:

```
wal_shard_0
wal_shard_1
...
wal_shard_N
```
Writer chooses table based on sharding key.

Readers independently tail assigned tables.

* Easier if you don’t want Kafka-level complexity
* Often used with relational DB-backed WALs
* Simpler for bootstrapped control planes

### Model 3: Logstream based WAL (Pulsar, custom)
WAL implemented as distributed logstream.

Shards implemented via topics or streams.

Built-in multi-region replication and retention.

* Pulsar or custom Raft-based WALs
* Fine-grained shard control
* Flexible replication models

## Monotonicity for aggregate version in WAL

### Why do we need monotonically increasing versions per aggregate?

* Ensure idempotent application of events
* Avoid lost updates
* Enable replay safety (only apply version N+1 after N)
* Allow optimistic concurrency control (OCC)
* Provide strong ordering per aggregate, while avoiding global coordination

### Core Problem
In a distributed system:

* We want to shard WAL writes across many writers/nodes.
* We still want per-aggregate version numbers to be strictly monotonic.
* But we don’t want to serialize all writes globally (it kills scalability).

### Observations to exploit
The strict monotonicity only needs to hold per aggregate.
* Cross-aggregate ordering isn’t usually required.
* This allows us to use per-aggregate sequencing models.


### Pattern 1: Single Writer Per Aggregate (natural leader)
* Assign exclusive ownership for each aggregate to a single WAL writer.
* That writer maintains in-memory version counter.
* Every command validated → increment version → append to WAL.
* No race condition exists.
* Failover requires leader transfer.
* Can be done via distributed locks (Zookeeper, etcd, Raft leases).
* Works very well when aggregates are naturally partitionable (e.g. per cluster).

```
Command → Leader → Read latest version → +1 → Write WAL event(version N+1)
```
Pros and Cons:
* `+` Super simple correctness
* `+` Low coordination
* `-` Leadership movement complexity

### Pattern 2: Optimistic Concurrency + CAS on WAL Append
* Client submits command with "expected version."
* WAL append happens via a CAS (compare-and-set) operation:
  "only append if latest version == expected version."
* If CAS fails → client retries or error out.
* WAL store needs to support conditional appends (some modern Kafka, EventStoreDB, SQL transactional inserts with WHERE clause).
```
Command(client expects version 7) → WAL → CAS: version==7? 
    - Yes → write version 8 
    - No → reject
```
Pros and Cons:
* `+` No leadership required
* `+` Full horizontal scaling
* `-` Write contention possible if multiple writers race on same aggregate

### Pattern 3: Centralized Version Allocator (per aggregate)

* Maintain a small service that issues version numbers per aggregate.
* Writers first request next version from allocator.
* Allocator maintains latest version state (may use Redis, etcd, etc).
* Then writer appends event with assigned version.
```
Command → Allocator: allocate version N+1 → WAL append with version N+1
```
Pros and Cons:
* `+` Centralizes version state
* `+` Guarantees monotonicity
* `-` Allocator can become bottleneck under heavy concurrency
* `+` Can be horizontally partitioned by aggregate ID (distributed allocators)

### Pattern 4: Embedded WAL Versioning (for pure leaderless designs)
* WAL itself generates version based on log offset.
* Version = "WAL shard sequence number."
* If WAL is partitioned per aggregate → WAL offset naturally becomes version.
* Used in some log-structured control planes (Pulsar-style).
Pros and Cons:
* `+` Zero coordination
* `+` Scales trivially
* `-` Requires WAL partition-per-aggregate
* `-` Cannot support aggregates spread across shards

### Summary 
| Model             | Coordination         | Scalability | Complexity | Where Used                  |
| ----------------- | -------------------- | ----------- | ---------- | --------------------------- |
| Single Writer     | per-aggregate leader | high        | simple     | Raft, Etcd                  |
| Optimistic CAS    | optimistic on WAL    | high        | medium     | EventStoreDB, Kafka         |
| Version Allocator | central allocator    | medium-high | medium     | DynamoDB conditional writes |
| WAL Offset        | embedded in log      | very high   | hard       | Pulsar, per-aggregate WAL   |

## WAL Write Protocol

### Design goals
* Multi-tenant
* High write throughput
* Low latency
* Strong per-aggregate ordering
* WAL sharded for scale
* Safe failure recovery
* Easy replay + idempotency

### High-Level Flow
```
Client Request  →  Command Processor  →  WAL Write Protocol  →  WAL Append  →  Materializer
```
### Core Concepts
| Term         | Meaning                                                     |
| ------------ | ----------------------------------------------------------- |
| Aggregate    | Entity whose state evolves (e.g. cluster, workload, policy) |
| Aggregate ID | Unique identifier for the aggregate                         |
| Tenant ID    | Customer/account identity                                   |
| WAL Shard    | A partition of the WAL                                      |
| Version      | Monotonically increasing sequence for aggregate             |

### Protocol
1. Command comes in
    * Client submits Command(aggregate_id, command_type, payload)
    * If available: client may include expected_version (OCC)

2. Read latest version (optional, depends on model)
    * If you're using OCC (CAS model): read current version.
    * If you're using single-writer or allocator model: may skip.

3. Version Assignment
    * Depending on your versioning model (recall our previous discussion):
    * Model	How version assigned
      * Single Writer	Increment local in-memory version
      * CAS	Use client-provided expected_version
      * Allocator	Request version from allocator service
      * WAL Offset	Use WAL offset as version

4. Build WAL Event
    * Construct full event object:
      ```json
      {
        "aggregate_id": "cluster-123",
        "tenant_id": "tenant-abc",
        "event_type": "ClusterScaleUp",
        "payload": { "replicas": 5 },
        "version": 42,
        "timestamp": "2025-06-15T20:00:00Z",
        "schema_version": 1
      }
      ```
5. Determine WAL shard
    * Compute shard by:
    * shard_id = hash(aggregate_id) % WAL_SHARD_COUNT
    * This guarantees all events for aggregate go to same shard.
    * WAL writers can route writes accordingly.

6.  WAL Append
    * Append event to the WAL shard.
    * WAL guarantees:
      * Strict ordering per shard.
      * Idempotent appends.
      * CAS check if applicable.

7. Success Response
    * Acknowledge to client once WAL append succeeds.
    * Downstream materialization is async.

### Failure Scenarios
| Failure                 | Handling                       |
| ----------------------- | ------------------------------ |
| WAL append fails        | Retry with idempotency         |
| CAS conflict            | Client retries after refresh   |
| Version allocator fails | Retry / fallback               |
| Shard unavailable       | Retry with backoff or failover |

### WAL Store Properties
| WAL Feature           | Why needed                 |
| --------------------- | -------------------------- |
| Append-only           | Event sourcing correctness |
| Per-shard ordering    | Per-aggregate consistency  |
| CAS append (optional) | OCC semantics              |
| Retention policies    | Cost control, compaction   |
| Replayable            | Recovery & materialization |
| Partition awareness   | Efficient routing          |

### Multi-Tenant Isolation Considerations
| Layer          | Strategy                                               |
| -------------- | ------------------------------------------------------ |
| WAL Shards     | Tenant-aware shard keys (TenantID, AggregateID hybrid) |
| WAL Throughput | Per-tenant quotas                                      |
| WAL Storage    | Tenant-scoped retention policies                       |
| Materializers  | Tenant-scoped replay processes                         |
| Backpressure   | Tenant-isolated throttling                             |

## WAL Dispatcher

### Why ?

* WAL is sharded for scalability.
* We have multiple WAL shards (Kafka partitions, Pulsar topics, DB tables, etc).
* We have multiple WAL writers / command processors.
* We need to ensure exclusive shard ownership per writer to avoid conflicts.
* We need fast failover on writer node failure.

This is where Global Dispatcher + WAL Writer Election architecture comes in.

| Component          | Responsibility                          |
| ------------------ | --------------------------------------- |
| Global Dispatcher  | Assign shard ownership to WAL writers   |
| WAL Writers        | Handle incoming commands, write to WAL  |
| Coordination Store | Manage shard leases and ownership state |

```
+-----------------+        +-------------------+
| Client Commands |  -->   |  Global Dispatcher|
+-----------------+        +-------------------+
                                   |
                          Assigns Shards via Leases
                                   |
               +---------+--------+--------+---------+
               | WAL Writer 1 | WAL Writer 2 | WAL Writer N |
               +---------+--------+--------+---------+
                                   |
                          Write WAL Shard (exclusive)

```

### The Global Dispatcher

The dispatcher handles:
* Shard assignment (initial)
* Lease acquisition (heartbeat-based)
* Failure detection (lease expiry)
* Fair distribution across healthy writers
* Rebalancing on writer join/leave

### The Lease Protocol (core of HA election)
At its heart, this is basically:
1.	Dispatcher assigns WAL shard S to writer W
2.	Writer W holds a lease for S (e.g. 30 seconds)
3.	Writer sends heartbeats to renew lease
4.	If lease expires → dispatcher reassigns S

This prevents double-writers on a shard and failures are automatically handled
via lease expiry

## The Coordination Store
This is usually built on top of a strongly consistent store:
| Option         | Example                          |
| -------------- | -------------------------------- |
| Distributed KV | etcd, Consul, Zookeeper          |
| Consensus DB   | FoundationDB                     |
| Cloud-native   | DynamoDB with conditional writes |

Dispatcher writes shard ownership state here.

### Writer Election per WAL shard
Sometimes you don’t need a global dispatcher at all if your WAL backend 
supports per-shard leader election.

| Approach               | Example                               |
| ---------------------- | ------------------------------------- |
| Kafka partition leader | Kafka brokers handle partition leader |
| Pulsar topic ownership | Pulsar broker manages topic ownership |
| Raft log               | Leader per Raft shard                 |

If your WAL storage can natively manage shard ownership, you don’t need an external dispatcher.

### Two Main Models
#### Model A — Centralized Dispatcher (external coordination)
* Global dispatcher assigns shards.
* Writers only write WAL if they hold lease.
* Good for heterogeneous backends (DB tables, generic queues).
Example:
* Multi-tenant SaaS using PostgreSQL or DynamoDB WAL shards.

#### Model B — Embedded Shard Ownership (backend-managed)
* WAL storage itself handles shard ownership.
* Writers consume from assigned shards automatically.

Example:
* Kafka partition leaders
* Pulsar topic assignments

### Failure Handling Example
| Failure                   | Resolution                                 |
| ------------------------- | ------------------------------------------ |
| WAL Writer crashes        | Lease expires → dispatcher reassigns shard |
| Dispatcher crashes        | Use HA dispatcher with leader election     |
| Coordination store outage | Strongly consistent store needed           |
| Split-brain               | Lease expiry prevents multiple writers     |

###  Backpressure & Load Balancing

The dispatcher can also consider:

* Current load per writer
* Tenant affinity
* Hot-shard detection
* Priority workloads

Dynamic rebalancing is possible (though harder if you need strict replay ordering).

### Example Flow
#### First assignment:
* WAL shard count = 1024
* WAL writers = 8 nodes
* Dispatcher assigns ~128 shards per writer
* Each writer holds leases for its shards
* Writers renew leases every few seconds

#### Writer crash:
* Writer 3 crashes
* Dispatcher sees missed heartbeats
* Dispatcher reassigns its 128 shards across remaining writers
* Replay resumes from last checkpoint per shard

### Real-World Examples
| System           | Dispatcher Model                                      |
| ---------------- | ----------------------------------------------------- |
| Kafka            | Partition leader election handled by Kafka controller |
| Pulsar           | Topic ownership assigned to brokers                   |
| DynamoDB Streams | Client-side shard lease management                    |
| Kafka Connect    | External coordinator assigns task partitions          |
| Zookeeper        | Curator recipes for distributed shard ownership       |

### Why not always use a centralized dispatcher?
* Centralized dispatcher is generic: works with any WAL backend.
* Backend-managed ownership (Kafka/Pulsar) is much simpler — but you're locked into the backend’s capabilities.
* For control planes where you own both the WAL and the coordination layer — external dispatcher gives you more flexibility (multi-region active-active, tenant-based affinity, noisy neighbor protection, etc.)

### Hybrid Pattern: Shard Balancer + Lease Coordinator
* **Shard Balancer** (compute optimal shard assignment based on load)
* **Lease Coordinator** (enforces lease locks per shard)
* **WAL Writers** (only write if holding valid lease)

### Summary
| Design                    | Benefits                       | Tradeoff                  |
| ------------------------- | ------------------------------ | ------------------------- |
| Global Dispatcher + Lease | Full control, backend agnostic | Slightly more complex     |
| Backend-managed ownership | Simpler, offloaded             | Less flexible             |
| Hybrid Balancer + Lease   | Tenant-aware balancing         | Complex but most powerful |

## Why do we need exactly-once replay?
* Materializer may crash & restart.
* WAL may have duplicates (e.g. writer retries).
* WAL may contain out-of-order replays on reshard/rebalance.
* Network failures can lead to partial commits.

The key contract we want:

> Every event is applied once and only once to the aggregate state — even under retries, duplicates, failovers, and crashes.

### Simplify the problem: scope per aggregate
* We don't need global exactly-once.
* We only need per-aggregate exactly-once. Aggregate = cluster, workload, namespace, etc.\
* WAL versioning gives us strong ordering per aggregate.
* We can use this version to drive idempotency.

### The Core Replay Rule
The Version-Based Idempotency Contract:
```python
if event.version == current_version + 1:
    apply(event)
    current_version += 1
elif event.version <= current_version:
    skip(event)  # duplicate, safe to ignore
else:
    fail("version gap detected")
```
* Simple
* Fully idempotent
* Deterministic replay
* Crash safe

### Why this works
* WAL guarantees append-only, ordered events per aggregate.
* WAL version numbers are monotonic per aggregate.
* Materializer keeps durable checkpoint: {aggregate_id, current_version}

As long as you checkpoint version updates transactionally with applying the event, you're safe.

### Exactly-once = Atomic Apply + Checkpoint
At materializer:
* Start transaction
* Apply event → update state
* Update checkpoint (current_version = event.version)
* Commit transaction

If crash occurs:
* On replay: WAL replays from last checkpointed version.
* Duplicates are naturally skipped.

### How to make apply + checkpoint atomic
| Option            | Technique                                              |
| ----------------- | ------------------------------------------------------ |
| SQL DB            | Single transaction (UPDATE state, UPDATE version)      |
| Key-Value store   | CAS operations (conditional update if version matches) |
| Embedded log      | Log compaction with snapshots                          |
| Distributed store | Distributed transaction manager                        |


### Materializer Replay Flow
1. Read WAL event (shard offset N)
2. Lookup aggregate current_version
3. Compare versions
    - If event.version == current_version + 1 → Apply
    - If event.version <= current_version → Skip
    - If event.version > current_version + 1 → Fail-fast
4. Commit state + version
5. Advance WAL offset checkpoint

### WAL Shard Offset vs Aggregate Version
Important:
Materializer needs two independent checkpoints:
| Checkpoint                   | Purpose                            |
| ---------------------------- | ---------------------------------- |
| WAL offset checkpoint        | Progress on WAL shard stream       |
| Aggregate version checkpoint | Last applied version per aggregate |

* WAL offset allows fast resume after restart.
* Aggregate version ensures exactly-once apply.

### Exactly-once with at-least-once WAL

You do not need exactly-once delivery from WAL itself.
You only need:

* At-least-once delivery (can deliver duplicates)
* Strict ordering per aggregate

Then materializer enforces exactly-once via version checking.

### DO NOT FORGET
* Don't forget to atomically checkpoint version and state update.
* Don't allow aggregate state to apply out-of-order events.
* Don't replay WAL across multiple materializer instances without shard isolation.
* Don't reblance shards without proper checkpoint handoff.

### High-throughput Materializer
| Optimization             | How                                          |
| ------------------------ | -------------------------------------------- |
| Shard-level parallelism  | Each WAL shard processed independently       |
| Aggregate-level batching | Apply events per aggregate in batches        |
| Write-ahead checkpoints  | Pre-allocate WAL windows with prefetch       |
| Async commit buffers     | Pipeline apply + checkpoint stages           |
| Snapshotting             | Periodically checkpoint full aggregate state |






































