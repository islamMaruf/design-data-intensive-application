# Chapter 11: Stream Processing

## Introduction

In Chapter 10, we explored batch processing—analyzing large, fixed datasets. But what if data arrives continuously? Stream processing bridges batch and real-time systems, processing unbounded data as it arrives.

**Stream processing** handles data in motion:
- **E-commerce**: Update inventory in real-time
- **Finance**: Detect fraudulent transactions instantly
- **IoT**: Process sensor data from millions of devices
- **Social Media**: Generate trending topics on the fly

This chapter covers:
- **Event streams**: Modeling continuous data flow
- **Message brokers**: Kafka, RabbitMQ, and pub/sub systems
- **Stream processing frameworks**: Flink, Kafka Streams, Spark Streaming
- **Windowing**: Aggregating unbounded data
- **Event time vs processing time**: Handling late and out-of-order events
- **Stateful stream processing**: Joins, aggregations, and pattern detection

Understanding stream processing is essential for building modern, responsive applications that react to events as they happen.

## The Foundation: Understanding Logs

Before diving into stream processing, **we need to understand logs**—one of the most fundamental data structures in computer science. Logs are at the heart of:
- **PostgreSQL**: Write-ahead log (WAL)
- **MySQL**: Binary log (binlog)
- **Kafka**: Partition logs
- **Event sourcing**: Append-only event logs
- **Distributed systems**: Replicated logs for consensus

### What is a Log?

**Log**: An append-only, ordered sequence of records.

```
Simple log structure:
┌─────┬─────┬─────┬─────┬─────┬─────┐
│ R1  │ R2  │ R3  │ R4  │ R5  │ ... │  ← Append new records here
└─────┴─────┴─────┴─────┴─────┴─────┘
  ↑                                  ↑
  Oldest                          Newest
```

**Properties:**
- **Append-only**: New records added to the end
- **Immutable**: Once written, records never change
- **Ordered**: Each record has a unique position (offset)
- **Sequential**: Written sequentially on disk

**Why logs matter**: They're **incredibly fast** for writes, even on modern SSDs. Sequential writes are 10-100x faster than random writes.

### Logs in PostgreSQL: The Write-Ahead Log (WAL)

**The Write-Ahead Log (WAL)** is how PostgreSQL achieves durability and performance simultaneously.

#### How PostgreSQL Writes Work

When you INSERT a row into PostgreSQL, here's what happens:

```
┌─────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Architecture                       │
└─────────────────────────────────────────────────────────────────┘

Step 1: Insert request arrives
  INSERT INTO users VALUES (123, 'Alice', 'alice@example.com')

Step 2: Update in-memory cache (Buffer Pool)
┌──────────────────────────────────────────┐
│  Buffer Pool (RAM)                       │
│  ┌────────────┐  ┌────────────┐         │
│  │  Page 1    │  │  Page 2    │  ← Modified (dirty)
│  │  users(1)  │  │  users(123)│         │
│  └────────────┘  └────────────┘         │
└──────────────────────────────────────────┘
                  │
                  │ Mark as "dirty" (needs flush to disk)
                  ▼

Step 3: Write to Write-Ahead Log (WAL)
┌──────────────────────────────────────────┐
│  Write-Ahead Log (on disk)               │
│  ┌─────┬─────┬─────┬─────┬─────┐        │
│  │ ... │ INS │ UPD │ DEL │ INS │ ← Append new entry
│  └─────┴─────┴─────┴─────┴─────┘        │
│        "INSERT users(123, 'Alice', ...)" │
└──────────────────────────────────────────┘
                  │
                  │ fsync() - guarantee durability
                  ▼
Step 4: Return success to client ✓

Step 5: Eventually flush dirty pages to disk (async)
┌──────────────────────────────────────────┐
│  Data Files (on disk)                    │
│  users.dat, indexes.dat, ...             │
│  Updated later (not immediately)         │
└──────────────────────────────────────────┘
```

#### Why This Design is Fast

**Key insight**: Writing sequentially to the WAL is **much faster** than random writes to data files.

**The numbers:**
- **Sequential write**: ~200-500 MB/s on SSD, ~100 MB/s on spinning disk
- **Random write**: ~10-50 MB/s on SSD, ~1 MB/s on spinning disk
- **Speedup**: 10-100x faster
**This was even more crucial with spinning disks** (pre-2010s):
- Random seeks required physically moving the disk head (5-10ms per seek)
- Sequential writes: No seeks needed
- **Result**: 100x performance difference
**Even on modern SSDs**, sequential writes are still faster:
- Better wear leveling (SSDs degrade with writes)
- More efficient caching
- Fewer internal garbage collection cycles

#### The Write-Ahead Log Workflow

**Normal operation:**

```
1. Application INSERT → PostgreSQL
2. PostgreSQL updates buffer pool (RAM)
3. PostgreSQL appends to WAL (disk) ← Fast sequential write
4. PostgreSQL returns "committed" to application ✓
5. Later: Background process flushes dirty pages to data files
```

**On crash recovery:**

```
1. PostgreSQL crashes before flushing dirty pages
2. On restart: Read WAL from last checkpoint
3. Replay all WAL entries to reconstruct state
4. Database fully recovered! ✓
```

**Example scenario:**

```sql
-- Time 10:00:00: Insert users
INSERT INTO users VALUES (1, 'Alice');  -- Written to WAL immediately
INSERT INTO users VALUES (2, 'Bob');    -- Written to WAL immediately

-- Time 10:00:01: CRASH! Dirty pages not yet flushed to disk

-- Time 10:00:02: PostgreSQL restarts
-- Reads WAL: "INSERT users(1, 'Alice')"
--            "INSERT users(2, 'Bob')"
-- Replays operations → State fully recovered
-- Users Alice and Bob are in the database ✓
```

### Logs in MySQL: The Binary Log (binlog)

**MySQL's Binary Log** serves similar purposes to PostgreSQL's WAL:
- **Durability**: Records all writes
- **Replication**: Replicas read binlog to stay in sync
- **Point-in-time recovery**: Replay binlog to any timestamp

**Binary log entry example:**

```
# at 154
#210115 10:30:45 server id 1  end_log_pos 234
Query   thread_id=10    exec_time=0     error_code=0
SET TIMESTAMP=1610707845;
INSERT INTO users VALUES (123, 'Alice', 'alice@example.com');
```

**Key difference from PostgreSQL WAL:**
- **WAL**: Physical changes (page-level modifications)
- **Binlog**: Logical changes (SQL statements or row changes)

### Why Logs are Fundamental to Stream Processing

**The connection to Kafka and stream processing:**

1. **Kafka partitions ARE logs**: Each Kafka partition is an append-only log file
2. **Event streams ARE logs**: Sequence of events is a log
3. **Log-based replication**: Databases stream WAL/binlog to replicas
4. **Change Data Capture (CDC)**: Tools like Debezium stream database logs to Kafka

**The log is a unifying abstraction:**

```
Database:     Changes → WAL/binlog → Replicas
Kafka:        Events → Partition log → Consumers
Event Store:  Events → Event log → Subscribers
Blockchain:   Transactions → Blockchain → Nodes
```

**All use the same pattern**: Append-only, ordered, immutable log.

### Sequential Writes: The Performance Secret

**Why databases, Kafka, and stream processors all use logs:**

**Spinning Disk Example (historical, but illustrates the principle):**

```
Random writes:
┌─────────────────────────────────────┐
│  Disk platter (spinning)             │
│    ┌──┐     ┌──┐     ┌──┐           │
│    │ 1│     │ 3│     │ 2│  ← Seek to each location (5-10ms each!)
│    └──┘     └──┘     └──┘           │
│         Seek ← Slow!                 │
└─────────────────────────────────────┘
Total time: 3 writes × 10ms = 30ms

Sequential writes:
┌─────────────────────────────────────┐
│  Disk platter (spinning)             │
│  ┌──┬──┬──┬──┬──┬──┬──┐             │
│  │ 1│ 2│ 3│ 4│ 5│ 6│ 7│  ← No seeks! Just rotate and write
│  └──┴──┴──┴──┴──┴──┴──┘             │
└─────────────────────────────────────┘
Total time: 3 writes × 0.1ms = 0.3ms  (100x faster!)
```

**Modern SSD**: Still benefits from sequential writes:
- **Internal parallelism**: SSDs can write multiple blocks simultaneously
- **Wear leveling**: Sequential writes spread wear evenly
- **Write amplification**: Random writes cause more internal writes
- **Result**: 2-5x speedup with sequential writes

### Logs in Stream Processing: Kafka's Design

**Kafka is essentially a distributed log system**:
- Each **partition** is an append-only log file
- **Producers** append messages to the log
- **Consumers** read from the log at their own pace
- **Retention**: Logs kept for days/weeks (configurable)

```
Kafka Partition (log file):
/var/kafka/logs/user_clicks-0/
┌─────────────────────────────────────────────────────────┐
│ Segment 0: 00000000000000000000.log                     │
│ ┌──────┬──────┬──────┬──────┬──────┬──────┐            │
│ │ Msg0 │ Msg1 │ Msg2 │ Msg3 │ ...  │ MsgN │            │
│ └──────┴──────┴──────┴──────┴──────┴──────┘            │
│ Offset:  0      1      2      3            N            │
└─────────────────────────────────────────────────────────┘

Append operation (producer write):
  - Sequential write to end of log
  - O(1) time complexity
  - ~200,000-500,000 messages/second per partition

Read operation (consumer read):
  - Sequential read from offset
  - O(1) time complexity
  - Multiple consumers read independently
```

**Why this works for stream processing:**
- **Fast writes**: Producers don't block on slow consumers
- **Independent consumers**: Each maintains its own offset
- **Replay capability**: Can re-read old messages
- **Durability**: Data persisted to disk

### The Universal Pattern: Log-Structured Systems

**Many modern systems are log-structured:**

**1. Log-Structured Merge Trees (LSM-trees):**
- Used by: Cassandra, HBase, RocksDB, LevelDB
- Writes go to in-memory log, then flushed to disk logs

**2. Append-only databases:**
- CouchDB: All document versions stored in append-only file
- EventStoreDB: Pure event sourcing database (log of events)

**3. File systems:**
- Log-structured file systems (LFS): All writes append to log
- Modern flash file systems use log-structured design

**4. Distributed consensus:**
- Raft/Paxos: Replicated state machines with replicated logs
- Zookeeper: Transaction log for metadata

**The common thread**: Append-only logs provide:
- **Performance**: Fast sequential writes
- **Durability**: Easy to fsync and replicate
- **Simplicity**: Simple data structure, easy to reason about
- **Debuggability**: Complete history of what happened

### Summary: Why Logs Matter for Stream Processing

**Key insights:**

1. **Logs are fundamental**: Used in databases, stream processors, distributed systems
2. **Performance**: Sequential writes are 10-100x faster than random writes
3. **Durability**: Easy to make durable with fsync
4. **Stream processing IS log processing**: Kafka partitions, event streams are all logs
5. **The log is the unifying abstraction**: One pattern applies across many systems

**When you use Kafka, you're using a distributed log**. When you process streams, you're reading from logs. Understanding logs is essential for understanding stream processing.

## Part 1: Transmitting Event Streams

### What is an Event?

**Event**: A small, immutable record of something that happened.

**Examples:**
```json
{
  "event_type": "page_view",
  "user_id": "user_123",
  "url": "/products/laptop",
  "timestamp": "2024-01-15T10:30:45.123Z",
  "ip_address": "192.168.1.100"
}

{
  "event_type": "purchase",
  "user_id": "user_456",
  "product_id": "prod_789",
  "amount": 1299.99,
  "timestamp": "2024-01-15T10:31:20.456Z"
}

{
  "event_type": "sensor_reading",
  "device_id": "temp_sensor_42",
  "temperature": 22.5,
  "timestamp": "2024-01-15T10:32:01.789Z"
}
```

**Properties:**
- **Immutable**: Once created, never modified
- **Timestamped**: When the event occurred
- **Self-contained**: All relevant information included

### Event Streams vs Batch Data

```
Batch (bounded):
┌─────────────────────────────────────┐
│ [event1, event2, event3, ..., eventN] │
└─────────────────────────────────────┘
    (process all at once)

Stream (unbounded):
──→ event1 ──→ event2 ──→ event3 ──→ ...
    (process continuously)
```

### Message Brokers

**Message broker**: Intermediary that receives messages from producers and delivers to consumers.

**Architecture:**

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  Producer 1  │─────→│              │─────→│  Consumer 1  │
└──────────────┘      │    Message   │      └──────────────┘
┌──────────────┐─────→│    Broker    │─────→┌──────────────┐
│  Producer 2  │      │   (Kafka)    │      │  Consumer 2  │
└──────────────┘      │              │      └──────────────┘
┌──────────────┐─────→│              │─────→┌──────────────┐
│  Producer 3  │      └──────────────┘      │  Consumer 3  │
└──────────────┘                            └──────────────┘
```

**Message Broker vs Database:**

| Aspect | Message Broker | Database |
|--------|----------------|----------|
| **Delete messages** | Yes (after consumption) | No (keeps data) |
| **Queries** | No complex queries | SQL queries |
| **Indexing** | No secondary indexes | Multiple indexes |
| **Use case** | Real-time messaging | Long-term storage |
| **Performance** | High throughput | Optimized for reads |

## Part 2: Apache Kafka

**Kafka** is a distributed streaming platform—the de facto standard for event streaming.

### The Origins: LinkedIn's Need for Scale

**Kafka was invented by engineers at LinkedIn** in the early 2010s to solve massive data pipeline challenges. Before Kafka, LinkedIn struggled with:
- **Tens of thousands of events per second**: Page views, profile updates, connection requests
- **Multiple destination systems**: Hadoop for analytics, search indexes, monitoring systems
- **Point-to-point connections**: Each producer connected directly to each consumer (N×M connections)
- **No buffering**: If a consumer was slow, producers would block or drop data

#### The Kafka Paper and Its Impact

The original Kafka paper (published around 2011-2012, with updates in 2017) is **highly accessible** and worth reading:
- **Short and high-level**: No heavy mathematics or complex proofs
- **Clear architecture**: Explains brokers, topics, partitions, producers, consumers
- **Performance benchmarks**: Shows comparisons with ActiveMQ and RabbitMQ
- **Trade-off discussion**: Honest about design decisions

**Why Kafka succeeded:**
- **Purpose-built for high throughput**: Designed to ingest millions of events per second
- **Durability without complexity**: Simple log-based storage on disk
- **Horizontal scalability**: Add more brokers and partitions as needed
- **Tens of thousands of companies** now use Kafka: Uber, Netflix, Airbnb, LinkedIn, Twitter, and countless others

#### The Trade-offs: Performance vs Durability

**Key insight from the paper**: Kafka traded off some traditional database guarantees for performance.

**What Kafka prioritized:**
1. **High throughput**: Handle millions of messages per second
2. **Low latency**: Milliseconds from producer to consumer
3. **Horizontal scalability**: Add brokers linearly
4. **Simple design**: Sequential disk writes, simple replication

**What Kafka initially relaxed** (improved in later versions):
1. **Durability**: Early versions allowed occasional message loss
2. **Exactly-once semantics**: Initially only at-least-once or at-most-once
3. **Complex transactions**: Cross-partition transactions came later
4. **Strong ordering**: Only guaranteed within a partition

**The rationale**: For event logs, metrics, and analytics, **occasional data loss is acceptable**. Losing a few page view events out of billions won't break your analytics. This trade-off allowed Kafka to achieve:
- **800,000+ messages/sec** throughput (early benchmarks)
- **2-3ms latency** end-to-end
- **Petabytes of data** retention on commodity hardware

**Modern Kafka (2015+)**: Added exactly-once semantics, transactions, stronger durability—but still maintains the high-performance architecture.

#### Why Not Just Use a Traditional Message Queue?

**Traditional message queues (RabbitMQ, ActiveMQ)** focus on:
- Delivering messages reliably to a single consumer
- Complex routing rules
- Acknowledgments and retries
- Small message volumes (thousands/sec)

**Kafka focuses on:**
- **Log-based architecture**: All messages in append-only log
- **Multiple independent consumers**: Each reads at their own pace
- **Massive throughput**: Millions of messages per second
- **Long retention**: Keep messages for days/weeks (not just until consumed)

**Example**: If your app server sends 100,000 page view events per second, RabbitMQ would struggle. Kafka handles it easily, stores it on disk, and lets multiple consumers (analytics, monitoring, ML) all read the same stream independently.

### Kafka Architecture

**Key Concepts:**
- **Topic**: Category/feed name (e.g., "user_clicks", "purchases")
- **Partition**: Ordered, immutable sequence of messages
- **Producer**: Publishes messages to topics
- **Consumer**: Subscribes to topics and reads messages
- **Consumer Group**: Multiple consumers working together
- **Broker**: Kafka server that stores data

```
Topic: "user_clicks" (3 partitions)

┌────────────────────────────────────────────────────────────┐
│  Partition 0                                               │
│  ┌────┬────┬────┬────┬────┬────┐                          │
│  │msg0│msg3│msg6│msg9│    │    │ ← Offset 0, 1, 2, 3, ... │
│  └────┴────┴────┴────┴────┴────┘                          │
└────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────┐
│  Partition 1                                               │
│  ┌────┬────┬────┬────┬────┬────┐                          │
│  │msg1│msg4│msg7│    │    │    │                          │
│  └────┴────┴────┴────┴────┴────┘                          │
└────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────┐
│  Partition 2                                               │
│  ┌────┬────┬────┬────┬────┬────┐                          │
│  │msg2│msg5│msg8│    │    │    │                          │
│  └────┴────┴────┴────┴────┴────┘                          │
└────────────────────────────────────────────────────────────┘

Producers write to partitions (round-robin or by key)
Consumers read from assigned partitions
```

### Producing Messages

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092']
});

const producer = kafka.producer();

async function sendMessage() {
  await producer.connect();
  
  // Send message
  const event = {
    user_id: 'user_123',
    action: 'click',
    url: '/products/laptop',
    timestamp: '2024-01-15T10:30:45Z'
  };
  
  const result = await producer.send({
    topic: 'user_clicks',
    messages: [
      { value: JSON.stringify(event) }
    ]
  });
  
  console.log(`Sent to partition ${result[0].partition}, offset ${result[0].offset}`);
  
  // Disconnect
  await producer.disconnect();
}

sendMessage().catch(console.error);
```

**Partitioning Strategies:**

```javascript
// 1. Round-robin (default if no key)
await producer.send({
  topic: 'topic',
  messages: [{ value: JSON.stringify(message) }]
});

// 2. By key (messages with same key go to same partition)
await producer.send({
  topic: 'topic',
  messages: [
    { 
      key: user_id,  // Ensures ordering per user
      value: JSON.stringify(message) 
    }
  ]
});

// 3. Custom partitioner
const producer = kafka.producer({
  createPartitioner: () => {
    return ({ topic, partitionMetadata, message }) => {
      const key = message.key?.toString();
      
      // Route VIP users to dedicated partition
      if (key && key.startsWith('vip_')) {
        return 0;  // Partition 0 for VIPs
      } else {
        const numPartitions = partitionMetadata.length;
        const hash = key ? hashCode(key) : Math.floor(Math.random() * numPartitions);
        return Math.abs(hash) % (numPartitions - 1) + 1;
      }
    };
  }
});

function hashCode(str) {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash) + str.charCodeAt(i);
    hash = hash & hash; // Convert to 32-bit integer
  }
  return hash;
}
```

### Consuming Messages

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092']
});

const consumer = kafka.consumer({ 
  groupId: 'analytics_group' 
});

async function consumeMessages() {
  await consumer.connect();
  
  // Subscribe to topic from beginning
  await consumer.subscribe({ 
    topic: 'user_clicks', 
    fromBeginning: true  // Start from beginning
  });
  
  // Consume messages
  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());
      console.log(`User ${event.user_id} clicked ${event.url}`);
      
      // Process event
      await processClick(event);
      
      // Auto-commit enabled by default
    }
  });
}

consumeMessages().catch(console.error);

// Graceful shutdown
process.on('SIGINT', async () => {
  await consumer.disconnect();
  process.exit(0);
});
```

**Consumer Groups:**

```
Topic: "user_clicks" (3 partitions)

Consumer Group "analytics":
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Consumer 1   │     │ Consumer 2   │     │ Consumer 3   │
│              │     │              │     │              │
│ Reads P0     │     │ Reads P1     │     │ Reads P2     │
└──────────────┘     └──────────────┘     └──────────────┘

Each partition assigned to ONE consumer in the group
→ Parallel processing
→ At-least-once delivery guarantee per group

Multiple consumer groups can read same topic independently:
Group "analytics" → Reads all messages
Group "monitoring" → Also reads all messages (independent offset)
```

### Kafka Guarantees

**Ordering:**
- **Within partition**: Messages are ordered
- **Across partitions**: No ordering guarantee

```javascript
// Guarantee: All messages for user_123 are ordered
await producer.send({
  topic: 'topic',
  messages: [
    { key: 'user_123', value: JSON.stringify(message1) },
    { key: 'user_123', value: JSON.stringify(message2) }
  ]
});
// message1 arrives before message2 (same partition)

// No guarantee: messages for different users
await producer.send({
  topic: 'topic',
  messages: [
    { key: 'user_123', value: JSON.stringify(message1) },
    { key: 'user_456', value: JSON.stringify(message2) }
  ]
});
// Could arrive in any order (different partitions)
```

**Delivery Semantics:**

**1. At-most-once**: Messages may be lost but never redelivered
```javascript
// Consumer auto-commits offset BEFORE processing
const consumer = kafka.consumer({ groupId: 'mygroup' });

await consumer.run({
  autoCommit: true,
  autoCommitInterval: 5000,  // 5 seconds
  eachMessage: async ({ message }) => {
    try {
      await process(message);
    } catch (error) {
      // Message lost if processing fails
      console.error('Processing failed, message lost');
    }
  }
});
```

**2. At-least-once**: Messages never lost but may be redelivered
```javascript
// Consumer commits offset AFTER processing
const consumer = kafka.consumer({ groupId: 'mygroup' });

await consumer.run({
  autoCommit: false,  // Manual commit
  eachMessage: async ({ topic, partition, message }) => {
    await process(message);
    
    // Commit only after success
    await consumer.commitOffsets([{
      topic,
      partition,
      offset: (parseInt(message.offset) + 1).toString()
    }]);
    // If crash before commit → redelivery → duplicate processing
  }
});
```

**3. Exactly-once**: Messages processed exactly once (Kafka 0.11+)
```javascript
// Transactional producer + idempotent processing
const producer = kafka.producer({
  transactionalId: 'my_transactional_id',
  idempotent: true  // Enable idempotence
});

await producer.connect();

// Begin transaction
const transaction = await producer.transaction();

try {
  await transaction.send({
    topic: 'topic',
    messages: [{ value: JSON.stringify(message) }]
  });
  
  // Commit transaction
  await transaction.commit();
} catch (error) {
  // Abort on error
  await transaction.abort();
}

// Combined with idempotent consumer logic
// (check message ID, use database constraints, etc.)
```

### Kafka Persistence and Replication

**Storage:**
```
Each partition is an append-only log file:

/var/kafka/logs/user_clicks-0/
  00000000000000000000.log  ← Segment 0 (1GB)
  00000000000000123456.log  ← Segment 1 (1GB)
  00000000000000246912.log  ← Segment 2 (active)

Messages are kept for configurable retention:
  - Time-based: 7 days (default)
  - Size-based: 100GB per partition
```

**Replication:**
```
Topic: "user_clicks" (replication factor = 3)

┌─────────────────────────────────────────────────────┐
│ Broker 1 (Leader for P0)                           │
│ ┌──────────────┐                                   │
│ │ Partition 0  │ ← Leader (reads and writes)      │
│ └──────────────┘                                   │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Broker 2 (Follower for P0, Leader for P1)         │
│ ┌──────────────┐ ┌──────────────┐                 │
│ │ Partition 0  │ │ Partition 1  │                 │
│ │ (replica)    │ │ (leader)     │                 │
│ └──────────────┘ └──────────────┘                 │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Broker 3 (Follower for P0 & P1)                   │
│ ┌──────────────┐ ┌──────────────┐                 │
│ │ Partition 0  │ │ Partition 1  │                 │
│ │ (replica)    │ │ (replica)    │                 │
│ └──────────────┘ └──────────────┘                 │
└─────────────────────────────────────────────────────┘

If Broker 1 fails:
  → Broker 2 or 3 elected as new leader for P0
  → No data loss (replicas are in-sync)
```

### The Architecture Problem: Why You Need Kafka

Let's understand **why you need a message broker** like Kafka with a real-world scenario.

#### Scenario: E-commerce Analytics Pipeline

**Your setup:**
- **App servers**: Handle user requests (page views, clicks, purchases)
- **PostgreSQL**: Stores user data, orders, products
- **ClickHouse**: Time-series analytics database for page views, metrics
- **Dashboard**: Real-time analytics for business intelligence

**Initial architecture (naive approach):**

```
┌──────────────┐
│ App Server 1 │──────┐
└──────────────┘      │
┌──────────────┐      │    ┌─────────────────┐
│ App Server 2 │──────┼───→│   ClickHouse    │
└──────────────┘      │    │ (Analytics DB)  │
┌──────────────┐      │    └─────────────────┘
│ App Server 3 │──────┘              ↑
└──────────────┘                     │
                                     │
                            ┌────────────────┐
                            │   Dashboard    │
                            │     App        │
                            └────────────────┘

App servers send events directly to ClickHouse:
  - Page views
  - Button clicks
  - Search queries
  - User behavior tracking
```

**This looks simple, but it has serious problems!**

#### Problem 1: Throughput Mismatch

**Black Friday scenario:**

```
Normal day:
  - 10,000 events/second
  - ClickHouse capacity: 10,000 inserts/second ✓

Black Friday (5x traffic):
  - 50,000 events/second
  - ClickHouse capacity: Still 10,000 inserts/second ✗

Result:
  ┌──────────────┐
  │ App Server 1 │─────→ 15,000 events/sec ╲
  ├──────────────┤                           ╲
  │ App Server 2 │─────→ 15,000 events/sec  ├──→ ClickHouse OVERLOADED
  ├──────────────┤                           ╱     Rejects connections
  │ App Server 3 │─────→ 20,000 events/sec ╱
  └──────────────┘
         ↓
  App servers must buffer events
         ↓
  RAM fills up → Out of memory → CRASH
```

**The problem**: ClickHouse becomes a **bottleneck**. App servers can't scale independently.

#### Problem 2: Backpressure and Cascading Failures

**What happens when ClickHouse can't keep up:**

```
Step 1: ClickHouse slows down
  - High CPU (100%)
  - Disk I/O maxed out
  - Insert latency: 50ms → 500ms → 5 seconds

Step 2: App servers must wait
  - Each request blocks for 5 seconds waiting for ClickHouse
  - User requests pile up
  - App server threads exhausted

Step 3: App servers buffer events
  - "I'll hold events in RAM and retry later"
  - RAM usage: 1GB → 5GB → 10GB → 16GB (max) → OUT OF MEMORY
  - App server 1 crashes

Step 4: Traffic redirected to remaining servers
  - App servers 2 and 3 now handle 1.5x traffic
  - They also crash

Step 5: Complete outage
  - All app servers down
  - Users see 503 Service Unavailable
  - Revenue loss: $$$
```

**This is called "cascading failure"**: One slow component brings down the entire system.

#### Problem 3: Tight Coupling

**Every app server connects directly to ClickHouse:**

```
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ App 1  │ │ App 2  │ │ App 3  │ │ App N  │
└───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘
    │          │          │          │
    └──────────┴──────────┴──────────┼──────→ ClickHouse
                                     │
                                     └──────→ Elasticsearch (search)
                                     │
                                     └──────→ Monitoring System
                                     │
                                     └──────→ ML Pipeline

Problems:
  - N app servers × M consumers = N×M connections
  - If ClickHouse goes down, app servers crash
  - Can't add new consumers without modifying app servers
  - Can't replay old events
```

#### Problem 4: No Buffering

**Without an intermediate buffer:**
- Events processed immediately or lost
- Can't handle traffic spikes
- Can't replay for debugging
- Can't recover from consumer failures

#### Problem 5: Can't Scale Consumers Independently

**Example:** You want to add ML training pipeline that processes events:

```
Without Kafka:
  - Must modify app servers to send events to ML system
  - ML system must keep up with app server speed
  - If ML system slow → app servers slow
  - Tight coupling again
With Kafka:
  - ML system just subscribes to Kafka topic
  - Processes at its own pace
  - No impact on app servers
  - Can replay old events for model training
```

### The Solution: Kafka as Intermediary

**Revised architecture with Kafka:**

```
┌──────────────┐
│ App Server 1 │──────┐
└──────────────┘      │
┌──────────────┐      │    ┌─────────────────┐
│ App Server 2 │──────┼───→│     Kafka       │
└──────────────┘      │    │  (Message       │
┌──────────────┐      │    │   Broker)       │
│ App Server 3 │──────┘    └────────┬────────┘
└──────────────┘                    │
                        ┌───────────┼───────────┬────────────┐
                        │           │           │            │
                        ▼           ▼           ▼            ▼
              ┌──────────────┐ ┌────────┐ ┌─────────┐ ┌─────────┐
              │  ClickHouse  │ │  ES    │ │ Monitor │ │   ML    │
              │ (Analytics)  │ │(Search)│ │ System  │ │Pipeline │
              └──────────────┘ └────────┘ └─────────┘ └─────────┘

Producers (App Servers):
  - Send events to Kafka
  - Return immediately (async)
  - Don't care about consumers

Kafka:
  - Buffers events on disk
  - High throughput (millions/second)
  - Durable, replicated storage
  - Retains events for days/weeks

Consumers:
  - Read from Kafka at their own pace
  - Independent of producers
  - Can replay old events
  - Failure doesn't affect producers
```

#### How Kafka Solves Each Problem

**Solution 1: Handles throughput mismatch**

```
Black Friday traffic spike:
  - App servers send 50,000 events/sec to Kafka ✓
  - Kafka buffers them on disk (TB of space)
  - ClickHouse reads at 10,000 events/sec ✓
  - Kafka has 5 seconds of buffered events
  - Eventually ClickHouse catches up ✓

No crashes, no data loss! ✓
```

**Solution 2: Decouples producers and consumers**

```
App servers:
  - Send to Kafka
  - Return immediately
  - Don't wait for consumers
  - Can't be blocked by slow consumers

Consumers:
  - Read at their own pace
  - If ClickHouse slow → Events buffer in Kafka
  - If ClickHouse crashes → Kafka keeps events until it recovers
```

**Solution 3: Independent scaling**

```
Scale producers:
  - Add more app servers → Just point them to Kafka
  - Kafka scales horizontally (add more brokers)

Scale consumers:
  - Add more ClickHouse nodes → Subscribe to different partitions
  - Add new consumers → Just subscribe to Kafka topic
  - No changes to producers needed
```

**Solution 4: Replay and debugging**

```
Scenario: Bug in analytics code, wrong results for last 7 days

Without Kafka:
  - Events already consumed and gone
  - Can't reprocess
  - Must wait 7 days for new data

With Kafka:
  - Events retained for 7+ days
  - Fix bug in analytics code
  - Replay events from 7 days ago
  - Recompute correct results ✓
```

**Solution 5: Multiple consumers**

```
Same events, multiple consumers:
  - ClickHouse (analytics)
  - Elasticsearch (full-text search)
  - Monitoring system (alerts)
  - ML pipeline (training data)
  - Data warehouse (ETL)

Each consumer:
  - Independent offset
  - Independent pace
  - Can start/stop independently
  - All read the same events from Kafka
```

### Real-World Use Case: Amazon.com Scale

**The numbers:**
- **Normal day**: ~10 million events/second (page views, clicks, searches)
- **Prime Day / Black Friday**: ~50-100 million events/second
- **Millions of users**: Simultaneous activity
- **Hundreds of consumers**: Analytics, recommendations, fraud detection, inventory management

**With Kafka:**
- **Producers** (app servers): Send events at peak rate
- **Kafka**: Buffers petabytes of events
- **Consumers**: Process at their own pace
- **Result**: System stays up during traffic spikes ✓

**Without Kafka:**
- Direct connection overload
- Cascading failures
- System outages
- Lost revenue $$$ 

### Why Large Companies Track Everything

**From the SRT discussion**: Large companies (Amazon, Facebook, Google, LinkedIn) track **every user action**:
- Page views (which pages visited, how long)
- Button clicks (which buttons, in what order)
- Mouse movements and scrolling (heatmaps)
- Search queries (what users search for)
- Shopping cart behavior (what added, what removed)
- A/B test assignments (which variant user saw)

**Why?**
1. **Optimize conversions**: Placing items in the right spot increases sales
2. **Personalization**: Recommend products based on behavior
3. **A/B testing**: Which button color/text performs better?
4. **Fraud detection**: Unusual patterns indicate fraud
5. **Business intelligence**: What's trending? What's not used?

**The scale**: For a site like Amazon:
- **Billions of events per day**
- **Hundreds of millions per second** during peaks
- **Terabytes of log data per day**

**Kafka makes this possible**: Without Kafka or similar systems, handling this scale would be nearly impossible.

### Summary: When Do You Need Kafka?

**Use Kafka when:**
- ✓ High-throughput event streams (thousands+ events/second)
- ✓ Multiple consumers need same events
- ✓ Need to replay events
- ✓ Need to decouple producers and consumers
- ✓ Need buffering for traffic spikes
- ✓ Building data pipelines

**Don't need Kafka when:**
- ✗ Low throughput (< 100 events/second)
- ✗ Single consumer
- ✗ Don't need durability
- ✗ Simple request/response (use HTTP/REST instead)

**The core insight**: Kafka is a **distributed, durable, scalable log** that decouples producers and consumers, enabling massive-scale data pipelines.

## Part 3: Stream Processing Frameworks

### Stream Processing Patterns

**1. Stateless Transformations**

```javascript
// Filter events
const isPurchase = (event) => event.event_type === 'purchase';

const purchases = stream.filter(isPurchase);

// Map/Transform events
const enrichEvent = (event) => ({
  ...event,
  processed_at: new Date().toISOString(),
  category: getProductCategory(event.product_id)
});

const enriched = stream.map(enrichEvent);

// FlatMap (one-to-many)
const extractWords = (sentence) => sentence.split(' ');

const words = sentences.flatMap(extractWords);
```

**2. Stateful Transformations**

```javascript
// Aggregation (requires state)
function* countByUser(stream) {
  const counts = new Map();
  
  for (const event of stream) {
    const userId = event.user_id;
    const currentCount = counts.get(userId) || 0;
    counts.set(userId, currentCount + 1);
    
    yield { userId, count: counts.get(userId) };
  }
}
```

### Apache Flink

**Flink** is a true stream processor with powerful windowing and state management.

**Basic Stream Processing:**

```javascript
const { Kafka } = require('kafkajs');
const { Transform } = require('stream');

// Set up Kafka
const kafka = new Kafka({
  clientId: 'flink-app',
  brokers: ['localhost:9092']
});

const consumer = kafka.consumer({ groupId: 'flink_group' });

async function processStream() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'user_clicks', fromBeginning: true });
  
  // Create stream processing pipeline
  await consumer.run({
    eachMessage: async ({ message }) => {
      // Parse JSON
      const event = JSON.parse(message.value.toString());
      
      // Filter purchases
      if (event.event_type === 'purchase') {
        // Count purchases per user (stateful)
        await incrementUserPurchaseCount(event.user_id);
        
        const count = await getUserPurchaseCount(event.user_id);
        console.log(`User ${event.user_id}: ${count} purchases`);
      }
    }
  });
}

// State management (in-memory for this example)
const purchaseCounts = new Map();

async function incrementUserPurchaseCount(userId) {
  const current = purchaseCounts.get(userId) || 0;
  purchaseCounts.set(userId, current + 1);
}

async function getUserPurchaseCount(userId) {
  return purchaseCounts.get(userId) || 0;
}

processStream().catch(console.error);
```

**Windowing:**

```javascript
// Tumbling window: non-overlapping, fixed-size
class TumblingWindow {
  constructor(sizeMs) {
    this.sizeMs = sizeMs;
    this.windows = new Map();
  }
  
  add(userId, event) {
    const windowStart = Math.floor(event.timestamp / this.sizeMs) * this.sizeMs;
    const key = `${userId}_${windowStart}`;
    
    if (!this.windows.has(key)) {
      this.windows.set(key, []);
    }
    this.windows.get(key).push(event);
    
    return this.windows.get(key).reduce((sum, e) => sum + e.amount, 0);
  }
}

const window = new TumblingWindow(5 * 60 * 1000);  // 5 minutes

// Sliding window: overlapping windows  
class SlidingWindow {
  constructor(sizeMs, slideMs) {
    this.sizeMs = sizeMs;
    this.slideMs = slideMs;
    this.events = [];
  }
  
  add(userId, event) {
    this.events.push({ userId, event, timestamp: Date.now() });
    
    // Remove old events
    const cutoff = Date.now() - this.sizeMs;
    this.events = this.events.filter(e => e.timestamp >= cutoff);
    
    // Aggregate
    return this.events
      .filter(e => e.userId === userId)
      .reduce((sum, e) => sum + e.event.amount, 0);
  }
}

const slidingWindow = new SlidingWindow(10 * 60 * 1000, 5 * 60 * 1000);
```

**Window Types Visualization:**

```
Event Timeline:
─────────1─────2───3────────4──5───────6────7──────8────→ time

Tumbling Window (size=5):
[─────Window 1─────][─────Window 2─────][─────Window 3─────]
 1,2,3              4,5                  6,7,8

Sliding Window (size=10, slide=5):
[──────Window 1──────]
          [──────Window 2──────]
                    [──────Window 3──────]
1,2,3,4,5           4,5,6,7             6,7,8

Session Window (gap=3):
[──Window 1──]      [──Window 2──][Window 3]
 1,2,3               4,5            6,7,8
```

### Kafka Streams

**Kafka Streams** is a lightweight library for stream processing (no separate cluster needed).

```java
// Java example
Properties props = new Properties();
props.put(StreamsConfig.APPLICATION_ID_CONFIG, "word-count");
props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

StreamsBuilder builder = new StreamsBuilder();

// Read from topic
KStream<String, String> textLines = builder.stream("text-input");

// Word count
KTable<String, Long> wordCounts = textLines
    .flatMapValues(line -> Arrays.asList(line.toLowerCase().split("\\W+")))
    .groupBy((key, word) -> word)
    .count();

// Write to topic
wordCounts.toStream().to("word-count-output");

// Start
KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

**Stateful Processing:**

```java
// Aggregation with state
KGroupedStream<String, Purchase> grouped = purchases.groupByKey();

KTable<String, Double> totalSpent = grouped.aggregate(
    () -> 0.0,  // Initializer
    (key, purchase, aggregate) -> aggregate + purchase.getAmount(),  // Adder
    Materialized.with(Serdes.String(), Serdes.Double())  // State store
);
```

### Spark Structured Streaming

**Spark Structured Streaming** treats streams as unbounded tables.

```javascript
// JavaScript equivalent using Kafka + SQL-like processing
const { Kafka } = require('kafkajs');
const { DateTime } = require('luxon');

const kafka = new Kafka({
  clientId: 'spark-streaming-app',
  brokers: ['localhost:9092']
});

const consumer = kafka.consumer({ groupId: 'stream-example' });

async function streamProcessing() {
  await consumer.connect();
  await consumer.subscribe({ topic: 'user_clicks', fromBeginning: false });
  
  // Windowed aggregation state
  const windowCounts = new Map();
  
  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      const timestamp = DateTime.fromISO(event.timestamp);
      
      // 5-minute tumbling windows
      const windowStart = timestamp.startOf('minute')
        .minus({ minutes: timestamp.minute % 5 })
        .toISO();
      
      const key = `${windowStart}_${event.url}`;
      
      // Increment count
      windowCounts.set(key, (windowCounts.get(key) || 0) + 1);
      
      // Output to console
      console.log({
        window: windowStart,
        url: event.url,
        count: windowCounts.get(key)
      });
    }
  });
}

streamProcessing().catch(console.error);
```

**Micro-Batch Model:**

```
Spark Streaming processes data in micro-batches:

Stream:
───→ event ───→ event ───→ event ───→ event ───→ ...

Batched (every 1 second):
[batch 1: events 1-100] → [batch 2: events 101-200] → ...

Processing:
  Batch 1 → RDD → Transformations → Output
  Batch 2 → RDD → Transformations → Output
  ...

Latency: Bounded by batch interval (1-10 seconds typical)
```

## Part 4: Event Time vs Processing Time

### The Challenge: Late and Out-of-Order Events

**Scenario:** Events generated on devices, sent over network.

```
Device Time (Event Time):
Device 1: event_a @ 10:00:00
Device 2: event_b @ 10:00:01
Device 1: event_c @ 10:00:02

Network delays → Arrive at server:
Processing Time: event_b @ 10:00:05
                 event_c @ 10:00:08
                 event_a @ 10:00:10  ← Late
```

**Problem:** Which timestamp to use for windowing?

### The Message Ordering Problem: A Deep Dive

From the SRT discussion, **one of the biggest challenges in distributed stream processing is maintaining message order**. Let's explore this with a concrete example.

#### The Scenario: Multiple Producers and Consumers

**Setup:**
```
┌───────────────┐                ┌───────────────┐
│  Producer 1   │───────────────→│               │
│ (App Server)  │                │     Kafka     │
└───────────────┘                │    Broker     │
                                 │               │
┌───────────────┐                │               │
│  Producer 2   │───────────────→│               │
│ (App Server)  │                │               │
└───────────────┘                └───────┬───────┘
                                         │
                          ┌──────────────┼──────────────┐
                          │              │              │
                          ▼              ▼              ▼
                   ┌──────────┐   ┌──────────┐  ┌──────────┐
                   │Consumer 1│   │Consumer 2│  │Consumer 3│
                   └──────────┘   └──────────┘  └──────────┘
```

**What we want:**
- Process messages in the order they were sent
- M1 (first) → M2 (second) → M3 (third) → M4 (fourth)

**What actually happens:**

#### Example: The Out-of-Order Scenario

```
Timeline:
─────────────────────────────────────────────────────────────────→ time

Producer 1 sends M1 at t=0:
  M1 → Kafka → Assigned to Consumer 1 → Processed ✓

Producer 2 sends M2 at t=1:
  M2 → Kafka → Assigned to Consumer 2 → Processed ✓

Producer 1 sends M3 at t=2:
  M3 → Kafka → Assigned to Consumer 2 → ...

Producer 2 sends M4 at t=3:
  M4 → Kafka → Assigned to Consumer 1 → Processed ✓

PROBLEM: Consumer 2 crashes before acknowledging M3
  - M3 never confirmed
  - Kafka doesn't know if M3 was processed
  - Must resend M3

Kafka resends M3 to Consumer 1 at t=5:
  M3 → Consumer 1 → Processed (finally) ✓

Final processing order: M1 → M2 → M4 → M3 
Expected order:        M1 → M2 → M3 → M4 ✓
```

**Visual representation:**

```
┌──────────────────────────────────────────────────────┐
│ Expected Timeline (Event Time Order)                 │
├──────────────────────────────────────────────────────┤
│  M1 ──→ M2 ──→ M3 ──→ M4                             │
│  t=0    t=1    t=2    t=3                            │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│ Actual Processing (Processing Time Order)            │
├──────────────────────────────────────────────────────┤
│  M1 ──→ M2 ──→ M4 ──→ M3  ← M3 arrives late!        │
│  t=0    t=1    t=4    t=5                            │
│                  ↑                                    │
│                  └─ Consumer crash caused redelivery │
└──────────────────────────────────────────────────────┘
```

#### Why This Happens: Common Causes

**1. Consumer failures:**
```
Consumer receives M3 → Starts processing → CRASH
  → Never sends acknowledgment
  → Kafka resends to different consumer
  → Arrives out of order
```

**2. Network delays:**
```
M3 sent at t=2 → Network congestion → Arrives at t=10
M4 sent at t=3 → Fast path → Arrives at t=4
Result: M4 processed before M3
```

**3. Partitioning:**
```
Messages with different keys go to different partitions:
  M1 (key=A) → Partition 0 → Consumer 1
  M2 (key=B) → Partition 1 → Consumer 2
  M3 (key=A) → Partition 0 → Consumer 1
  M4 (key=B) → Partition 1 → Consumer 2

If Consumer 2 processes faster than Consumer 1:
  M2 and M4 processed before M1 and M3
```

**4. Parallel processing:**
```
Consumer 1 gets M1, M3, M5 (same partition)
  - Processes them in parallel (3 threads)
  - M5 finishes first
  - M1 finishes second
  - M3 finishes third
  
Result: M5 → M1 → M3 (out of order within partition!)
```

#### The Trade-offs: Ordering vs Scalability

**Option 1: Strict ordering (single partition, single consumer)**

```
┌─────────────────────────────────────────────────┐
│ Kafka Topic: "user_events" (1 partition)       │
│ ┌──────┬──────┬──────┬──────┬──────┐           │
│ │  M1  │  M2  │  M3  │  M4  │  M5  │           │
│ └──────┴──────┴──────┴──────┴──────┘           │
└──────────────────────┬──────────────────────────┘
                       │
                       ▼
                ┌──────────────┐
                │  Consumer 1  │ (single consumer)
                └──────────────┘

Pros:
  ✓ Perfect ordering
  ✓ Simple to reason about

Cons:
  ✗ No parallelism
  ✗ Limited throughput (one consumer)
  ✗ Single point of failure
  ✗ Can't scale horizontally
```

**Option 2: Partitioned for scale (multiple partitions, multiple consumers)**

```
┌──────────────────────────────────────────────────┐
│ Kafka Topic: "user_events" (3 partitions)        │
│                                                   │
│ Partition 0: M1 ─── M4 ─── M7                   │
│ Partition 1: M2 ─── M5 ─── M8                   │
│ Partition 2: M3 ─── M6 ─── M9                   │
└───────┬──────────┬──────────┬───────────────────┘
        │          │          │
        ▼          ▼          ▼
  ┌─────────┐ ┌─────────┐ ┌─────────┐
  │Consumer1│ │Consumer2│ │Consumer3│
  └─────────┘ └─────────┘ └─────────┘

Pros:
  ✓ High throughput (parallel processing)
  ✓ Horizontal scalability
  ✓ Fault tolerance (if one consumer fails, others continue)

Cons:
  ✗ Ordering only within partition (not across partitions)
  ✗ M2 might be processed before M1
  ✗ Complex failure handling
```

**Option 3: Partition by key (ordering per key)**

```
Messages partitioned by user_id:
  M1 (user=Alice) → Partition 0
  M2 (user=Bob)   → Partition 1
  M3 (user=Alice) → Partition 0  ← Same partition as M1
  M4 (user=Bob)   → Partition 1  ← Same partition as M2

Result:
  - All Alice's messages in order ✓
  - All Bob's messages in order ✓
  - But Alice's and Bob's messages NOT ordered relative to each other
    (which is usually fine!)

This is the most common pattern! ✓
```

#### How Kafka Guarantees Ordering

**Kafka's guarantee**:
- Messages with the **same key** go to the **same partition**
- Messages within a **partition** are **totally ordered**
- Messages **across partitions** are **NOT ordered**

```
Example: E-commerce order events

producer.send('orders', key='user_123', value={'action': 'add_to_cart'})
producer.send('orders', key='user_123', value={'action': 'checkout'})
producer.send('orders', key='user_123', value={'action': 'payment'})

All three go to the SAME partition (because same key='user_123')
→ Processed in order: add_to_cart → checkout → payment ✓

But orders for user_456 might be interleaved:
  user_123: add_to_cart
  user_456: add_to_cart  ← Different partition
  user_123: checkout
  user_456: checkout
  user_123: payment
  user_456: payment

This is fine! We only need ordering per user, not across users.
```

#### Handling the Ordering Problem: Practical Solutions

**Solution 1: Accept eventual consistency**
```javascript
// For analytics, slight ordering issues are acceptable
// Example: Page view counts might be off by 1-2 temporarily
// Eventually correct as late messages arrive

if (lateMessage) {
  // Just count it anyway
  // Final daily count will be correct
}
```

**Solution 2: Use sequence numbers**
```javascript
// Producer adds sequence number
const message = {
  user_id: 'user_123',
  sequence: 42,  // ← Monotonically increasing per user
  action: 'purchase',
  timestamp: Date.now()
};

// Consumer checks sequence
let lastSequence = {};
function process(message) {
  const expected = (lastSequence[message.user_id] || 0) + 1;
  
  if (message.sequence < expected) {
    // Duplicate - already processed
    return; // Skip
  }
  
  if (message.sequence > expected) {
    // Out of order - buffer for later
    buffer.add(message);
    return;
  }
  
  // In order - process now
  processMessage(message);
  lastSequence[message.user_id] = message.sequence;
  
  // Check if buffered messages can now be processed
  processBuffered(message.user_id);
}
```

**Solution 3: Use watermarks (advanced)**
```
Watermark = "No messages with timestamp < T will arrive"

Example:
  - Current time: 10:00:10
  - Watermark lag: 5 seconds
  - Watermark: 10:00:05
  
Meaning:
  "Any messages timestamped before 10:00:05 have already arrived"
  "Safe to close windows ending at 10:00:05"

If message arrives at 10:00:10 with timestamp 10:00:03:
  → LATE (before watermark)
  → Handle specially (discard, side output, or recompute)
```

**Solution 4: Partition by key (recommended)**
```javascript
// Partition by user_id ensures per-user ordering
producer.send({
  topic: 'user_events',
  key: user_id,  // ← All events for same user go to same partition
  value: event
});

// Consumer processes partition sequentially
// → All events for user_123 are in order ✓
```

#### The Bottom Line: Ordering Trade-offs

**Key insights:**

1. **Perfect global ordering is expensive**: Single partition, no parallelism
2. **Per-key ordering is practical**: Partition by key, scale horizontally
3. **Eventual consistency is often good enough**: Analytics, monitoring
4. **Failure handling is complex**: Consumer crashes, network issues, redeliveries

**Most applications use per-key ordering:**
- E-commerce: Order events per user
- IoT: Sensor readings per device
- Logs: Log entries per service
- Social media: Posts/likes per user

**Quote from the SRT:**
> "These are like many different trade-offs between like well what if I run this all on a single server and only have one consumer that kind of simplifies the time sequencing aspect but then that might not scale well."

The ordering problem is **fundamental to distributed systems**. There's no perfect solution—only trade-offs between consistency, scalability, and complexity.

### Event Time Processing

**Event Time**: When event actually occurred (source timestamp)
**Processing Time**: When event is processed (server timestamp)

```javascript
// Extract event timestamp
const extractTimestamp = (event) => {
  return new Date(event.timestamp).getTime();
};

// Watermark strategy (5 second out-of-orderness)
class WatermarkStrategy {
  constructor(maxOutOfOrderMs) {
    this.maxOutOfOrderMs = maxOutOfOrderMs;
    this.maxTimestamp = 0;
  }
  
  updateWatermark(eventTimestamp) {
    this.maxTimestamp = Math.max(this.maxTimestamp, eventTimestamp);
    return this.maxTimestamp - this.maxOutOfOrderMs;
  }
  
  isLate(eventTimestamp) {
    const watermark = this.maxTimestamp - this.maxOutOfOrderMs;
    return eventTimestamp < watermark;
  }
}

const watermarkStrategy = new WatermarkStrategy(5000); // 5 seconds

// Process events
function processEvent(event) {
  const eventTime = extractTimestamp(event);
  const watermark = watermarkStrategy.updateWatermark(eventTime);
  
  if (watermarkStrategy.isLate(eventTime)) {
    console.log('Late event:', event);
    // Handle late data
  } else {
    // Process normally
  }
}
```

### Watermarks

**Watermark**: Signal that says "no events with timestamp < T will arrive"

```
Event Timeline (Event Time):
───1───3───2───5───4───7───6───9───→

With 5-second watermark lag:

Processing Time | Event | Watermark
─────────────────────────────────────
t=0             | 1     | -∞
t=1             | 3     | -∞
t=2             | 2     | -∞  (out of order, but within watermark)
t=3             | 5     | 0   (watermark = max(seen) - 5 = 5-5 = 0)
t=4             | 4     | 0   (out of order, but > watermark)
t=5             | 7     | 2
t=6             | 6     | 2   (out of order, but > watermark)
t=7             | 9     | 4

Window [0, 5):
  Triggered when watermark >= 5
  Contains: 1, 2, 3, 4 (event time < 5)
```

**Watermark Visualization:**

```
Events:
                    │ 
  1   3   2   5   4 │ 7   6   9
─────────────────────┼──────────────→ Event Time
  0   1   2   3   4 │ 5   6   7   8

Watermark (lag=2):
              ▲
              │ Watermark = 5
              │ (max event time - lag = 7 - 2 = 5)
              │
Window [0,5)  │
closes here   │
```

**Handling Late Data:**

```javascript
// Option 1: Discard late data
class WindowProcessor {
  constructor() {
    this.allowedLatenessMs = 0;  // No lateness allowed
  }
  
  processEvent(event, watermark) {
    const eventTime = new Date(event.timestamp).getTime();
    
    if (eventTime < watermark) {
      // Discard late event
      return null;
    }
    
    // Process event
    return this.process(event);
  }
}

// Option 2: Accept late data up to threshold
class LateDataProcessor {
  constructor() {
    this.allowedLatenessMs = 60 * 1000;  // Accept up to 1 min late
    this.lateDataOutput = [];
  }
  
  processEvent(event, watermark) {
    const eventTime = new Date(event.timestamp).getTime();
    const lateness = watermark - eventTime;
    
    if (lateness > this.allowedLatenessMs) {
      // Very late - send to side output
      this.lateDataOutput.push(event);
      return null;
    }
    
    if (lateness > 0) {
      // Late but within threshold - still process
      return this.processLate(event);
    }
    
    // On time
    return this.process(event);
  }
}

// Option 3: Update window results (retraction)
class RetractionProcessor {
  constructor() {
    this.allowedLatenessMs = 10 * 60 * 1000;  // 10 minutes
    this.windowResults = new Map();
  }
  
  processEvent(event, watermark) {
    const eventTime = new Date(event.timestamp).getTime();
    const windowKey = Math.floor(eventTime / (5 * 60 * 1000));
    
    // Get current window result
    const current = this.windowResults.get(windowKey) || 0;
    
    // Update with new event (even if late)
    this.windowResults.set(windowKey, current + 1);
    
    // Emit updated result
    return {
      window: windowKey,
      count: this.windowResults.get(windowKey),
      isUpdate: eventTime < watermark
    };
  }
}
```

## Part 5: Joins in Stream Processing

### Stream-Stream Join

**Example: Join clicks and purchases (both streams)**

```javascript
// Interval join implementation
class IntervalJoin {
  constructor(lowerBoundMs, upperBoundMs) {
    this.lowerBoundMs = lowerBoundMs;
    this.upperBoundMs = upperBoundMs;
    this.clicksBuffer = [];
    this.purchasesBuffer = [];
  }
  
  processClick(click) {
    this.clicksBuffer.push(click);
    this.cleanup();
    return this.findMatches(click, this.purchasesBuffer);
  }
  
  processPurchase(purchase) {
    this.purchasesBuffer.push(purchase);
    this.cleanup();
    return this.findMatches(purchase, this.clicksBuffer);
  }
  
  findMatches(event, otherBuffer) {
    const matches = [];
    const eventTime = new Date(event.timestamp).getTime();
    
    for (const other of otherBuffer) {
      const otherTime = new Date(other.timestamp).getTime();
      const timeDiff = Math.abs(eventTime - otherTime);
      
      if (event.user_id === other.user_id &&
          timeDiff >= this.lowerBoundMs &&
          timeDiff <= this.upperBoundMs) {
        matches.push({ click: event.event_type === 'click' ? event : other,
                      purchase: event.event_type === 'purchase' ? event : other });
      }
    }
    
    return matches;
  }
  
  cleanup() {
    const now = Date.now();
    const cutoff = now - this.upperBoundMs;
    
    this.clicksBuffer = this.clicksBuffer.filter(c => 
      new Date(c.timestamp).getTime() > cutoff
    );
    this.purchasesBuffer = this.purchasesBuffer.filter(p => 
      new Date(p.timestamp).getTime() > cutoff
    );
  }
}

const joiner = new IntervalJoin(0, 10 * 1000);  // Within 10 seconds

// Process clicks and purchases
const clickMatches = joiner.processClick({
  user_id: 'A',
  event_type: 'click',
  timestamp: '2024-01-15T10:00:00Z'
});

const purchaseMatches = joiner.processPurchase({
  user_id: 'A',
  event_type: 'purchase',
  timestamp: '2024-01-15T10:00:03Z'
});
```

**Join Semantics:**

```
Clicks:
───c1(user=A, t=10:00:00)───c2(user=B, t=10:00:05)───

Purchases:
───p1(user=A, t=10:00:03)───p2(user=A, t=10:00:15)───

Interval Join (within 10 seconds):
  c1 joins p1 ✓ (|10:00:00 - 10:00:03| = 3s < 10s)
  c1 does NOT join p2 ✗ (|10:00:00 - 10:00:15| = 15s > 10s)
```

### Stream-Table Join

**Example: Enrich stream with database lookup**

```javascript
const { Kafka } = require('kafkajs');

// Stream-table join in JavaScript
class StreamTableJoin {
  constructor() {
    this.userTable = new Map();  // Local table (cache)
  }
  
  // Populate table from Kafka compacted topic
  async loadUserTable(kafka) {
    const consumer = kafka.consumer({ groupId: 'table-loader' });
    await consumer.connect();
    await consumer.subscribe({ topic: 'users', fromBeginning: true });
    
    await consumer.run({
      eachMessage: async ({ message }) => {
        const userId = message.key.toString();
        const user = JSON.parse(message.value.toString());
        
        if (user === null) {
          // Tombstone - delete from table
          this.userTable.delete(userId);
        } else {
          // Update table
          this.userTable.set(userId, user);
        }
      }
    });
  }
  
  // Process purchase stream and join with user table
  async processPurchases(kafka) {
    const consumer = kafka.consumer({ groupId: 'purchase-enricher' });
    await consumer.connect();
    await consumer.subscribe({ topic: 'purchases', fromBeginning: false });
    
    await consumer.run({
      eachMessage: async ({ message }) => {
        const purchase = JSON.parse(message.value.toString());
        
        // Join with user table
        const user = this.userTable.get(purchase.user_id);
        
        const enriched = {
          purchase_id: purchase.id,
          amount: purchase.amount,
          product: purchase.product,
          user_id: purchase.user_id,
          user_name: user?.name || 'Unknown',
          user_email: user?.email || 'Unknown',
          timestamp: purchase.timestamp
        };
        
        console.log('Enriched purchase:', enriched);
        return enriched;
      }
    });
  }
}

const joiner = new StreamTableJoin();

// Load user table (from compacted topic)
await joiner.loadUserTable(kafka);

// Process purchases with enrichment
await joiner.processPurchases(kafka);
```

**Performance Consideration:**

```javascript
// Problem: Lookup in external DB (slow!)
async function enrichWithDB(event) {
  const user = await db.query(
    "SELECT * FROM users WHERE id = ?", 
    [event.user_id]
  );
  event.user_name = user.name;
  return event;
}

// One DB query per event - SLOW
for (const event of stream) {
  await enrichWithDB(event);
}

// Solution: Cache in local state
class EnrichmentCache {
  constructor(db) {
    this.db = db;
    this.cache = new Map();
  }
  
  async enrich(event) {
    const userId = event.user_id;
    
    if (!this.cache.has(userId)) {
      // Cache miss - query database
      const user = await this.db.query(
        "SELECT * FROM users WHERE id = ?",
        [userId]
      );
      this.cache.set(userId, user);
    }
    
    // Cache hit - fast
    event.user_name = this.cache.get(userId).name;
    return event;
  }
}

const enricher = new EnrichmentCache(db);

// Much faster with caching! ✓
for (const event of stream) {
  await enricher.enrich(event);
}
```

## Part 6: Fault Tolerance and Exactly-Once Processing

### Checkpointing

**Flink checkpointing**: Periodic snapshots of state.

```
Stream Processing:
Source → Transformation → Sink

Checkpoint 1 (t=10s):
  Source offset: 1000
  State: {user_123: 5, user_456: 3}

Checkpoint 2 (t=20s):
  Source offset: 2000
  State: {user_123: 8, user_456: 5, user_789: 2}

If failure at t=25s:
  → Restart from Checkpoint 2
  → Replay events from offset 2000
  → Recover state {user_123: 8, ...}
```

**Configuration:**

```javascript
// Conceptual checkpoint configuration in JavaScript
class StreamProcessor {
  constructor(config) {
    this.checkpointIntervalMs = config.checkpointIntervalMs || 60000;  // 60 seconds
    this.checkpointStorage = config.checkpointStorage;  // e.g., 'hdfs:///checkpoints'
    this.checkpointMode = config.checkpointMode || 'EXACTLY_ONCE';
    this.state = {};
    this.sourceOffset = 0;
  }
  
  async saveCheckpoint(checkpointId) {
    const checkpoint = {
      id: checkpointId,
      timestamp: Date.now(),
      sourceOffset: this.sourceOffset,
      state: JSON.parse(JSON.stringify(this.state))  // Deep copy
    };
    
    // Save to durable storage
    await this.saveToStorage(checkpoint);
    console.log(`Checkpoint ${checkpointId} saved`);
  }
  
  async restoreFromCheckpoint(checkpointId) {
    const checkpoint = await this.loadFromStorage(checkpointId);
    
    this.sourceOffset = checkpoint.sourceOffset;
    this.state = checkpoint.state;
    
    console.log(`Restored from checkpoint ${checkpointId}`);
  }
  
  startCheckpointTimer() {
    setInterval(() => {
      this.saveCheckpoint(Date.now());
    }, this.checkpointIntervalMs);
  }
}

const processor = new StreamProcessor({
  checkpointIntervalMs: 60000,
  checkpointStorage: 'hdfs:///checkpoints',
  checkpointMode: 'EXACTLY_ONCE'
});

processor.startCheckpointTimer();
```

### Idempotent Operations

**Idempotent**: Applying operation multiple times has same effect as applying once.

```javascript
// NOT idempotent (don't use with at-least-once!) 
async function incrementCounter(userId) {
  await db.execute(
    "UPDATE counters SET count = count + 1 WHERE user_id = ?",
    [userId]
  );
  // If processed twice → counter incremented twice
}

// Idempotent (safe for at-least-once) ✓
async function setLastSeen(userId, timestamp) {
  await db.execute(
    `UPDATE users 
     SET last_seen = ? 
     WHERE user_id = ? AND last_seen < ?`,
    [timestamp, userId, timestamp]
  );
  // If processed twice → no change (timestamp already updated) ✓
}

// Idempotent with message deduplication ✓
const processedMessageIds = new Set();

async function processMessage(message) {
  if (processedMessageIds.has(message.id)) {
    // Already processed - skip
    return;
  }
  
  // Process message
  await doWork(message);
  
  // Mark as processed
  processedMessageIds.add(message.id);
  await db.execute(
    "INSERT INTO processed_messages (id, timestamp) VALUES (?, ?)",
    [message.id, Date.now()]
  );
}
```

**Transactional Writes:**

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'transactional-processor',
  brokers: ['localhost:9092']
});

const producer = kafka.producer({
  transactionalId: 'my-app-txn',
  idempotent: true,  // Required for transactions
  maxInFlightRequests: 1  // Required for idempotence
});

const consumer = kafka.consumer({ groupId: 'my-group' });

await producer.connect();
await consumer.connect();
await consumer.subscribe({ topic: 'input-topic' });

// Process with exactly-once semantics
await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    // Start transaction
    const transaction = await producer.transaction();
    
    try {
      // Process message
      const input = JSON.parse(message.value.toString());
      const result = transform(input);
      
      // Send output within transaction
      await transaction.send({
        topic: 'output-topic',
        messages: [{
          key: message.key,
          value: JSON.stringify(result)
        }]
      });
      
      // Commit consumer offset within transaction
      await transaction.sendOffsets({
        consumerGroupId: 'my-group',
        topics: [{
          topic,
          partitions: [{
            partition,
            offset: (parseInt(message.offset) + 1).toString()
          }]
        }]
      });
      
      // Commit transaction (all or nothing)
      await transaction.commit();
      
    } catch (error) {
      // Abort transaction on error
      await transaction.abort();
      console.error('Transaction aborted:', error);
      throw error;
    }
  }
});

function transform(input) {
  // Your processing logic
  return {
    ...input,
    processed: true,
    timestamp: Date.now()
  };
}
```

## Part 7: Use Cases and Patterns

### Real-Time Analytics

**Use Case**: Process high-volume event streams to provide immediate insights.

**Example: Trending Topics on Twitter**

```javascript
const { DateTime } = require('luxon');
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'trending-topics',
  brokers: ['localhost:9092']
});

const consumer = kafka.consumer({ groupId: 'trending-group' });

// Sliding window for trend detection
class TrendDetector {
  constructor(windowMinutes = 5, slideMinutes = 1) {
    this.windowMs = windowMinutes * 60 * 1000;
    this.slideMs = slideMinutes * 60 * 1000;
    this.hashtagCounts = new Map();  // { hashtag -> [{ timestamp, count }] }
  }
  
  processTweet(tweet) {
    const now = Date.now();
    const hashtags = this.extractHashtags(tweet.text);
    
    for (const hashtag of hashtags) {
      if (!this.hashtagCounts.has(hashtag)) {
        this.hashtagCounts.set(hashtag, []);
      }
      
      const counts = this.hashtagCounts.get(hashtag);
      
      // Add current count
      if (counts.length === 0 || counts[counts.length - 1].timestamp < now - this.slideMs) {
        counts.push({ timestamp: now, count: 1 });
      } else {
        counts[counts.length - 1].count++;
      }
      
      // Clean old entries outside window
      const cutoff = now - this.windowMs;
      while (counts.length > 0 && counts[0].timestamp < cutoff) {
        counts.shift();
      }
    }
  }
  
  getTrending(topN = 10) {
    const trends = [];
    
    for (const [hashtag, counts] of this.hashtagCounts) {
      const totalCount = counts.reduce((sum, c) => sum + c.count, 0);
      
      if (totalCount > 0) {
        trends.push({ hashtag, count: totalCount });
      }
    }
    
    // Sort by count descending
    trends.sort((a, b) => b.count - a.count);
    
    return trends.slice(0, topN);
  }
  
  extractHashtags(text) {
    const regex = /#(\w+)/g;
    const hashtags = [];
    let match;
    
    while ((match = regex.exec(text)) !== null) {
      hashtags.push(match[1].toLowerCase());
    }
    
    return hashtags;
  }
}

// Usage
const detector = new TrendDetector(5, 1);  // 5-minute window, 1-minute slide

await consumer.connect();
await consumer.subscribe({ topic: 'tweets', fromBeginning: false });

await consumer.run({
  eachMessage: async ({ message }) => {
    const tweet = JSON.parse(message.value.toString());
    detector.processTweet(tweet);
  }
});

// Get trending hashtags every minute
setInterval(() => {
  const trending = detector.getTrending(10);
  
  console.log('\nTrending Now:');
  trending.forEach((trend, i) => {
    console.log(`${i + 1}. #${trend.hashtag} (${trend.count} tweets)`);
  });
}, 60 * 1000);

# Write to dashboard
query = trending.writeStream \
    .format("memory") \
    .queryName("trending") \
    .outputMode("complete") \
    .start()
```

### Fraud Detection

**Example: Detect unusual transactions**

```javascript
// Stateful fraud detection
class FraudDetector {
  constructor() {
    this.lastTransactions = new Map();
  }
  
  async processTransaction(transaction) {
    const alerts = [];
    const userId = transaction.user_id;
    const lastTx = this.lastTransactions.get(userId);
    
    if (lastTx) {
      // Check for rapid transactions
      const timeDiff = transaction.timestamp - lastTx.timestamp;
      
      if (timeDiff < 60 * 1000) {  // Less than 1 minute
        const amountDiff = Math.abs(transaction.amount - lastTx.amount);
        
        if (amountDiff > 10000) {  // Large amount difference
          alerts.push({
            type: 'RAPID_LARGE_TRANSACTIONS',
            userId,
            message: `Potential fraud: ${userId}`,
            transactions: [lastTx, transaction],
            timeDiffMs: timeDiff,
            amountDiff
          });
        }
      }
      
      // Check for unusual location
      if (lastTx.location && transaction.location) {
        const distance = calculateDistance(lastTx.location, transaction.location);
        const timeDiffHours = timeDiff / (1000 * 60 * 60);
        const speedKmh = distance / timeDiffHours;
        
        if (speedKmh > 1000) {  // Faster than airplane
          alerts.push({
            type: 'IMPOSSIBLE_TRAVEL',
            userId,
            message: `Impossible travel speed: ${speedKmh.toFixed(0)} km/h`,
            transactions: [lastTx, transaction]
          });
        }
      }
    }
    
    // Update last transaction
    this.lastTransactions.set(userId, transaction);
    
    return alerts;
  }
}

const detector = new FraudDetector();

// Process transaction stream
for await (const transaction of transactionStream) {
  const alerts = await detector.processTransaction(transaction);
  
  for (const alert of alerts) {
    console.log('🚨 FRAUD ALERT:', alert);
    await sendToSecurityTeam(alert);
  }
}

function calculateDistance(loc1, loc2) {
  // Haversine formula for distance between lat/long coordinates
  const R = 6371; // Earth's radius in km
  const dLat = (loc2.lat - loc1.lat) * Math.PI / 180;
  const dLon = (loc2.lon - loc1.lon) * Math.PI / 180;
  const a = Math.sin(dLat/2) * Math.sin(dLat/2) +
            Math.cos(loc1.lat * Math.PI / 180) * Math.cos(loc2.lat * Math.PI / 180) *
            Math.sin(dLon/2) * Math.sin(dLon/2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  return R * c;
}
```

### Event Sourcing

**Pattern**: Store all changes as sequence of events.

```javascript
// Events
const events = [
  { type: 'UserCreated', user_id: 1, name: 'Alice', timestamp: '2024-01-01T10:00:00Z' },
  { type: 'EmailChanged', user_id: 1, email: 'alice@new.com', timestamp: '2024-01-02T10:00:00Z' },
  { type: 'UserDeleted', user_id: 1, timestamp: '2024-01-03T10:00:00Z' }
];

// Rebuild state by replaying events
function replayEvents(events) {
  const users = new Map();
  
  for (const event of events) {
    switch (event.type) {
      case 'UserCreated':
        users.set(event.user_id, { 
          id: event.user_id,
          name: event.name 
        });
        break;
        
      case 'EmailChanged':
        if (users.has(event.user_id)) {
          users.get(event.user_id).email = event.email;
        }
        break;
        
      case 'UserDeleted':
        users.delete(event.user_id);
        break;
    }
  }
  
  return users;
}

// Current state
const users = replayEvents(events);  // Map {}
console.log('Current users:', Array.from(users.values()));

// Time travel: state at any point
const usersAtT2 = replayEvents(events.slice(0, 2));  
// Map { 1 => { id: 1, name: 'Alice', email: 'alice@new.com' } }
console.log('Users at T2:', Array.from(usersAtT2.values()));

// Event store implementation
class EventStore {
  constructor() {
    this.events = [];
  }
  
  append(event) {
    event.sequence = this.events.length;
    event.timestamp = event.timestamp || new Date().toISOString();
    this.events.push(event);
    return event.sequence;
  }
  
  getEvents(fromSequence = 0, toSequence = Infinity) {
    return this.events.slice(fromSequence, toSequence);
  }
  
  getSnapshot(entityId, toSequence = Infinity) {
    const relevantEvents = this.events
      .filter(e => e.user_id === entityId && e.sequence < toSequence);
    return replayEvents(relevantEvents);
  }
}

const eventStore = new EventStore();
eventStore.append({ type: 'UserCreated', user_id: 1, name: 'Alice' });
eventStore.append({ type: 'EmailChanged', user_id: 1, email: 'alice@example.com' });
```

### Change Data Capture (CDC)

**Pattern**: Stream database changes to other systems.

```
PostgreSQL Database:
  INSERT INTO users VALUES (1, 'Alice')
  UPDATE users SET email='new@email.com' WHERE id=1
  DELETE FROM users WHERE id=1

CDC Stream (Debezium):
  {'op': 'c', 'after': {'id': 1, 'name': 'Alice'}}
  {'op': 'u', 'before': {...}, 'after': {'id': 1, 'email': 'new@email.com'}}
  {'op': 'd', 'before': {'id': 1}}

Consumers:
  - Elasticsearch (full-text search)
  - Redis (cache)
  - Data warehouse (analytics)
```

**Debezium Configuration:**

```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "localhost",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "password",
    "database.dbname": "mydb",
    "database.server.name": "myserver",
    "table.include.list": "public.users,public.orders",
    "plugin.name": "pgoutput"
  }
}
```

**JavaScript CDC Consumer:**

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'cdc-consumer',
  brokers: ['localhost:9092']
});

const consumer = kafka.consumer({ groupId: 'cdc-group' });

await consumer.connect();
await consumer.subscribe({ 
  topic: 'myserver.public.users',  // Debezium topic format: {server}.{schema}.{table}
  fromBeginning: true 
});

await consumer.run({
  eachMessage: async ({ topic, partition, message }) => {
    const change = JSON.parse(message.value.toString());
    
    // Debezium event structure
    const { op, before, after, ts_ms } = change;
    
    switch (op) {
      case 'c':  // CREATE (INSERT)
        console.log('New row:', after);
        await elasticsearch.index({ id: after.id, document: after });
        await redis.set(`user:${after.id}`, JSON.stringify(after));
        break;
        
      case 'u':  // UPDATE
        console.log('Updated from:', before, 'to:', after);
        await elasticsearch.update({ id: after.id, doc: after });
        await redis.set(`user:${after.id}`, JSON.stringify(after));
        break;
        
      case 'd':  // DELETE
        console.log('Deleted row:', before);
        await elasticsearch.delete({ id: before.id });
        await redis.del(`user:${before.id}`);
        break;
        
      case 'r':  // READ (initial snapshot)
        console.log('Snapshot row:', after);
        await elasticsearch.index({ id: after.id, document: after });
        break;
    }
    
    console.log(`Processed change at ${new Date(ts_ms).toISOString()}`);
  }
});
```

## Part 8: Stream Processing vs Batch Processing

### Complementary Approaches

**Lambda Architecture**: Combine batch and stream processing.

```
                    Speed Layer (Stream)
                   ┌─────────────────────┐
                   │ Kafka Streams       │
Raw Data ─────────→│ (real-time views)   │─────→ Serving Layer
(Events)           └─────────────────────┘       (Combine results)
     │                                                   ↑
     │             Batch Layer                           │
     └────────────→┌─────────────────────┐              │
                   │ Spark               │              │
                   │ (accurate, complete)│──────────────┘
                   └─────────────────────┘

Real-time: Approximate, fast (low latency)
Batch: Accurate, slow (high latency)
```

**Kappa Architecture**: Stream processing only (reprocess streams for corrections).

```
              ┌─────────────────────┐
Raw Data ─────→ Kafka (message log) │
(Events)      └──────────┬──────────┘
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
       ┌─────────────┐       ┌─────────────┐
       │ Stream App 1│       │ Stream App 2│
       │ (real-time) │       │ (batch-like)│
       └─────────────┘       └─────────────┘

Single pipeline: Stream everything
Reprocessing: Replay Kafka log from any point
```

### When to Use Each

| Use Case | Batch | Stream |
|----------|-------|--------|
| **Real-time dashboards** | ✗ | ✓ |
| **Fraud detection** | ✗ | ✓ |
| **ML model training** | ✓ | ✗ |
| **Daily reports** | ✓ | ✗ |
| **Real-time alerts** | ✗ | ✓ |
| **Historical analysis** | ✓ | Maybe |
| **ETL pipelines** | ✓ | ✓ (both work) |

## Summary

**Key Takeaways:**

1. **Stream processing** handles unbounded data in real-time
   - Events processed as they arrive
   - Low latency (milliseconds to seconds)
   - Stateful computations

2. **Message brokers** (Kafka) decouple producers and consumers
   - Durable, replicated logs
   - Scalable (partitioning)
   - Multiple consumers can read independently

3. **Frameworks** provide high-level APIs
   - **Flink**: True streaming, event time, powerful state
   - **Kafka Streams**: Lightweight, embedded library
   - **Spark Streaming**: Micro-batches, unified batch/stream API

4. **Event time** vs **processing time**
   - Event time: When event happened (source)
   - Processing time: When processed (system)
   - Watermarks handle late/out-of-order data

5. **Windowing** aggregates unbounded data
   - Tumbling: Fixed, non-overlapping
   - Sliding: Overlapping windows
   - Session: Gap-based grouping

6. **Fault tolerance** via checkpointing
   - Periodic snapshots of state
   - Replay from checkpoint on failure
   - Exactly-once semantics with transactions

7. **Common patterns**:
   - Real-time analytics (trending, monitoring)
   - Fraud detection (stateful pattern matching)
   - Event sourcing (append-only log)
   - CDC (streaming database changes)

**Stream Processing Trade-offs:**

```
Latency:        Batch >>> Stream
Throughput:     Stream < Batch (per node)
Complexity:     Stream > Batch
Freshness:      Stream >>> Batch
Correctness:    Batch ≥ Stream (eventual consistency)
```

**Looking Ahead:**
Chapter 12 explores the future of data systems—how to integrate batch and stream processing, ensure correctness, and address emerging challenges like privacy and ethics.
