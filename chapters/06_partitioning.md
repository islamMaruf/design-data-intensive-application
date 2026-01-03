# Chapter 6: Partitioning (Sharding)

## Introduction: Breaking Data Into Pieces

Imagine you're Instagram. You have 2 billion users posting photos every day. Can you store all photos on one database server?

**Reality Check**:
- One server's capacity: ~10 TB storage, ~100K queries/second
- Instagram's needs: Petabytes of storage, millions of queries/second

**Solution**: **Partitioning** (also called **sharding**) - split your data across multiple machines.

```
┌────────────────────────────────────────────────┐
│      SINGLE DATABASE (Can't scale!)            │
├────────────────────────────────────────────────┤
│  [Single Server]                               │
│  - 10 TB storage                               │
│  - 100K queries/sec                            │
│  - All users: A-Z                              │
│                                                │
│  💥 Too much data!                             │
│  💥 Too many queries!                          │
└────────────────────────────────────────────────┘

              ↓ PARTITION ↓

┌────────────────────────────────────────────────┐
│      PARTITIONED DATABASE (Scales!)            │
├────────────────────────────────────────────────┤
│  [Server 1]      [Server 2]      [Server 3]   │
│  Users: A-H      Users: I-P      Users: Q-Z   │
│  3.3 TB          3.3 TB           3.3 TB       │
│  33K queries/s   33K queries/s    33K queries/s│
│                                                │
│  ✅ Total: 10 TB, 100K queries/sec            │
│  ✅ Add more servers = more capacity!         │
└────────────────────────────────────────────────┘
```

**Goals of Partitioning**:
1. **Scalability**: Distribute data and query load across many machines
2. **Performance**: Each node handles a fraction of the data

**Note**: Partitioning is usually combined with replication (covered in Chapter 5).

```
┌──────────────────────────────────────────┐
│  PARTITIONING + REPLICATION              │
├──────────────────────────────────────────┤
│                                          │
│  Partition 1 (Users A-H):                │
│  [Leader] → [Follower] → [Follower]     │
│                                          │
│  Partition 2 (Users I-P):                │
│  [Leader] → [Follower] → [Follower]     │
│                                          │
│  Partition 3 (Users Q-Z):                │
│  [Leader] → [Follower] → [Follower]     │
└──────────────────────────────────────────┘
```

Each partition is replicated for fault tolerance!

## Part 1: Partitioning of Key-Value Data

The fundamental question: **Given a key, which partition should it go to?**

### Strategy 1: Partitioning by Key Range

Assign continuous ranges of keys to each partition, like an encyclopedia.

```
┌────────────────────────────────────────────────┐
│      RANGE PARTITIONING                        │
├────────────────────────────────────────────────┤
│  Partition 1: Keys A to F                      │
│  [Aardvark, Apple, Banana, ..., Fox]           │
│                                                │
│  Partition 2: Keys G to P                      │
│  [Giraffe, House, India, ..., Penguin]         │
│                                                │
│  Partition 3: Keys Q to Z                      │
│  [Queen, Rabbit, Sun, ..., Zebra]              │
└────────────────────────────────────────────────┘
```

**Real-World Example - Google Bigtable**:

Bigtable (used by Google Search, Gmail, Google Maps) partitions by row key ranges:

```
Partition 1: Keys "" to "g"
Partition 2: Keys "g" to "p"
Partition 3: Keys "p" to "~"
```

Each partition is called a **tablet**.

**Advantages**:

✅ **Range queries are efficient**

```sql
-- Find all users whose names start with 'Al'
SELECT * FROM users WHERE name BETWEEN 'Al' AND 'Az';

-- Query goes to ONE partition (A-F)!
```

✅ **Keys are kept sorted**
- Useful for iterating in order
- Good for time-series data

**Disadvantages**:

❌ **Risk of hot spots** (uneven distribution)

**Example - Time-Series Data**:

```python
# Sensor data with timestamp as key
# 2024-01-15-00:00:00: temp=20
# 2024-01-15-00:00:01: temp=21
# 2024-01-15-00:00:02: temp=22
# ...

Partitions by date:
  Partition 1: 2024-01-01 to 2024-01-31
  Partition 2: 2024-02-01 to 2024-02-28
  Partition 3: 2024-03-01 to 2024-03-31
```

**Problem**: All writes go to the current month's partition!

```
[Partition 1] (Jan - cold 🥶)
[Partition 2] (Feb - cold 🥶)
[Partition 3] (Mar - HOT 🔥🔥🔥) ← All writes here!
```

**Solution**: Add a prefix to distribute the load

```python
# Instead of: 2024-03-15-10:30:00
# Use: sensor_id:2024-03-15-10:30:00

# Examples:
sensor_123:2024-03-15-10:30:00 → Partition 1
sensor_456:2024-03-15-10:30:00 → Partition 2
sensor_789:2024-03-15-10:30:00 → Partition 3
```

Now writes are distributed across all partitions!

**Real-World Example - HBase**:

HBase (Hadoop database) faced this exact problem. They now recommend prefixing row keys to avoid hot spots.

### Strategy 2: Partitioning by Hash of Key

Apply a hash function to the key, then partition by hash value.

```
┌────────────────────────────────────────────────┐
│      HASH PARTITIONING                         │
├────────────────────────────────────────────────┤
│  hash("Alice") = 0x1A3F → Partition 1          │
│  hash("Bob") = 0x7C2E → Partition 2            │
│  hash("Charlie") = 0x4B91 → Partition 2        │
│  hash("Diana") = 0x2F17 → Partition 1          │
└────────────────────────────────────────────────┘

Process:
1. Hash the key: hash(key) → number
2. Partition: number % num_partitions → partition_id
```

**Example**:

```python
import hashlib

def get_partition(key, num_partitions):
    # Hash the key
    hash_value = int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    # Determine partition
    partition_id = hash_value % num_partitions
    
    return partition_id

# Example usage:
print(get_partition("Alice", 3))    # Output: 2
print(get_partition("Bob", 3))      # Output: 0
print(get_partition("Charlie", 3))  # Output: 1
```

**Advantages**:

✅ **Even distribution**: Hash function distributes keys uniformly

```
Before hashing (by name):
[A-F]: 10000 users
[G-P]: 15000 users
[Q-Z]: 5000 users
❌ Unbalanced!

After hashing:
[Partition 1]: 10000 users
[Partition 2]: 10000 users
[Partition 3]: 10000 users
✅ Balanced!
```

✅ **No hot spots**: Writes are evenly distributed

**Disadvantages**:

❌ **Range queries are impossible**

```sql
-- Want all users with names A-C
SELECT * FROM users WHERE name BETWEEN 'A' AND 'C';

-- With hash partitioning:
hash("A...") could be in ANY partition
hash("B...") could be in ANY partition  
hash("C...") could be in ANY partition

-- Must scan ALL partitions! 💥
```

❌ **Lost key ordering**: Can't iterate through sorted keys

**Hash Functions**:

Good hash functions:
- **MD5**, **SHA-1**: Cryptographic hashes (overkill, slow)
- **Murmur3**, **xxHash**: Fast, non-cryptographic hashes ✅

**Bad idea**: Language built-in hash (Java's `hashCode`, Python's `hash`)
- Not consistent across processes/machines
- May change between language versions

**Real-World Examples**:

- **Cassandra**: Uses Murmur3 hash for partitioning
- **MongoDB**: Uses MD5 hash for sharded collections
- **Redis Cluster**: Uses CRC16 hash

### Strategy 3: Consistent Hashing

**Problem with Simple Hash Partitioning**:

```python
# 3 partitions initially
get_partition("Alice", 3)  # → Partition 0

# Add one more partition (now 4)
get_partition("Alice", 4)  # → Partition 2 ❌

# "Alice"'s data moved! Must rebalance almost ALL data!
```

**Consistent Hashing** minimizes data movement when partitions change.

**How It Works**:

Imagine a ring (circle) of hash values from 0 to 2^32-1.

```
┌────────────────────────────────────────────────┐
│      CONSISTENT HASHING RING                   │
├────────────────────────────────────────────────┤
│                                                │
│                 0 (top)                        │
│                  │                             │
│         [Node A] │                             │
│              ↘   │   ↙ [Node B]               │
│    2^31 ────────────────── 2^30               │
│              ↗       ↖                         │
│         [Node C]                               │
│                  │                             │
│               2^31 (bottom)                    │
└────────────────────────────────────────────────┘

Each node responsible for range from previous node to itself
Node A: hash values from Node C to Node A
Node B: hash values from Node A to Node B  
Node C: hash values from Node B to Node C
```

**Finding Partition**:

```python
def get_partition_consistent(key, nodes):
    key_hash = hash(key)
    
    # Find first node with hash ≥ key_hash (clockwise)
    for node in sorted(nodes, key=lambda n: hash(n)):
        if hash(node) >= key_hash:
            return node
    
    # Wrap around: return first node
    return nodes[0]
```

**Adding a Node**:

```
BEFORE (3 nodes):
  Node A: 0 to 1000
  Node B: 1000 to 2000
  Node C: 2000 to 3000

ADD Node D at position 1500:
  Node A: 0 to 1000      (unchanged ✅)
  Node D: 1000 to 1500   (NEW)
  Node B: 1500 to 2000   (only half moved)
  Node C: 2000 to 3000   (unchanged ✅)

Only 1/4 of data moved! (Instead of 3/4 with simple hashing)
```

**Real-World Usage**:
- **Amazon Dynamo**: Original consistent hashing paper
- **Cassandra**: Uses consistent hashing with virtual nodes
- **Riak**: Consistent hashing
- **CDNs** (Content Delivery Networks): Akamai, CloudFlare

**Virtual Nodes**:

Problem: Physical nodes might not distribute evenly on the ring.

Solution: Each physical node responsible for multiple virtual nodes.

```
Physical Nodes: A, B, C

Virtual Nodes:
Ring position 100: A1
Ring position 300: B1
Ring position 500: C1
Ring position 700: A2
Ring position 900: B2
Ring position 1100: C2
...

Each physical node handles multiple ranges!
More even distribution ✅
```

## Part 2: Partitioning and Secondary Indexes

Secondary indexes make partitioning much more complicated!

**Background**: Secondary indexes let you query by non-key attributes.

```sql
-- Primary key query (easy with partitioning)
SELECT * FROM users WHERE user_id = 12345;
→ Hash(12345) → Partition 2

-- Secondary index query (hard!)
SELECT * FROM users WHERE age = 25;
→ Users with age=25 could be in ANY partition! 💥
```

### Approach 1: Partitioning by Document (Local Indexes)

Each partition maintains its own secondary indexes.

```
┌────────────────────────────────────────────────┐
│    DOCUMENT-PARTITIONED INDEXES                │
├────────────────────────────────────────────────┤
│                                                │
│  Partition 1 (Users 0-999):                    │
│  Primary: [user_id → data]                     │
│  Index: age=25 → [user_123, user_456]          │
│         age=30 → [user_789]                    │
│                                                │
│  Partition 2 (Users 1000-1999):                │
│  Primary: [user_id → data]                     │
│  Index: age=25 → [user_1234, user_1567]        │
│         age=30 → [user_1890]                   │
│                                                │
│  Partition 3 (Users 2000-2999):                │
│  Primary: [user_id → data]                     │
│  Index: age=25 → [user_2345]                   │
│         age=30 → [user_2678, user_2901]        │
└────────────────────────────────────────────────┘
```

**Querying**:

```sql
SELECT * FROM users WHERE age = 25;

-- Must query ALL partitions (scatter/gather):
results = []
for partition in all_partitions:
    results += partition.query("age = 25")
return results
```

```
Query Process:
Client → [Partition 1] → Returns [user_123, user_456]
Client → [Partition 2] → Returns [user_1234, user_1567]
Client → [Partition 3] → Returns [user_2345]

Client merges: [user_123, user_456, user_1234, user_1567, user_2345]
```

**Advantages**:
- ✅ Writes are fast (only update one partition's index)

**Disadvantages**:
- ❌ Reads are slow (must query all partitions)
- ❌ Called "scatter/gather" - expensive!

**Real-World Usage**:
- **MongoDB**: Local secondary indexes
- **Cassandra**: Local indexes
- **Elasticsearch**: Each shard has its own index

**Real-World Example - Elasticsearch**:

```python
# Elasticsearch: 3 shards, searching for "python tutorial"
GET /articles/_search
{
  "query": {
    "match": {"content": "python tutorial"}
  }
}

# Process:
# 1. Query sent to all 3 shards
# 2. Each shard searches its local index
# 3. Results merged and scored
# 4. Top results returned

# If you have 100 shards → 100 queries per search! 💥
```

### Approach 2: Partitioning by Term (Global Indexes)

Create a global secondary index, partitioned separately from the primary data.

```
┌────────────────────────────────────────────────┐
│    TERM-PARTITIONED INDEXES                    │
├────────────────────────────────────────────────┤
│                                                │
│  Data Partitions (by user_id):                 │
│  [Partition 1]: Users 0-999                    │
│  [Partition 2]: Users 1000-1999                │
│  [Partition 3]: Users 2000-2999                │
│                                                │
│  Index Partitions (by age):                    │
│  [Index A]: ages 0-33                          │
│    age=25 → [user_123, user_456, user_1234,   │
│               user_1567, user_2345]            │
│  [Index B]: ages 34-66                         │
│    age=50 → [user_789, user_1890, ...]        │
│  [Index C]: ages 67-99                         │
│    age=80 → [user_901, ...]                    │
└────────────────────────────────────────────────┘
```

**Querying**:

```sql
SELECT * FROM users WHERE age = 25;

-- Query ONE index partition:
1. hash(25) → Index partition A
2. Index A returns: [user_123, user_456, user_1234, user_1567, user_2345]
3. Fetch from data partitions (multiple queries)
```

**Advantages**:
- ✅ Reads are faster (query one index partition, not all)
- ✅ More efficient for queries

**Disadvantages**:
- ❌ Writes are slower (must update index partition separately)
- ❌ Asynchronous updates: Index might be slightly stale

**Write Process**:

```
1. Client writes: user_id=1234, age=25
2. Write to data partition (hash(1234) → Partition 2)
3. Write to index partition (hash(25) → Index A)
   ↑
   This is often ASYNCHRONOUS!
```

**Real-World Usage**:
- **DynamoDB**: Global secondary indexes
- **Riak**: Global indexes with async updates

**Real-World Example - Amazon DynamoDB**:

```python
# DynamoDB table partitioned by user_id
# Global Secondary Index on email

# Write user
dynamodb.put_item(
    TableName='Users',
    Item={
        'user_id': '12345',
        'name': 'Alice',
        'email': 'alice@example.com'
    }
)

# Query by email (uses global index)
response = dynamodb.query(
    TableName='Users',
    IndexName='email-index',  # Uses global secondary index
    KeyConditionExpression='email = :email',
    ExpressionAttributeValues={
        ':email': 'alice@example.com'
    }
)

# Behind the scenes:
# 1. Query goes to one partition of the email-index
# 2. Index returns user_id=12345
# 3. Fetch from main table using user_id
```

**Warning**: Global indexes in DynamoDB are eventually consistent! After a write, the index might take milliseconds to update.

## Part 3: Rebalancing Partitions

**Rebalancing**: Moving data between nodes when you add/remove nodes.

**Requirements**:
1. Load should be shared fairly across nodes
2. Database should continue accepting reads/writes during rebalancing
3. Minimize data movement (expensive!)

### Why Rebalance?

**Scenario 1: Add More Nodes**
```
Before: 3 nodes, 30 TB total
[Node 1: 10 TB] [Node 2: 10 TB] [Node 3: 10 TB]

Add Node 4:
Want: 4 nodes, 7.5 TB each
[Node 1: 7.5 TB] [Node 2: 7.5 TB] [Node 3: 7.5 TB] [Node 4: 7.5 TB]

Must move 7.5 TB to Node 4!
```

**Scenario 2: Remove Failed Node**
```
Before: 4 nodes
[Node 1] [Node 2] [Node 3] [Node 4: 💥 Failed]

After:
[Node 1] [Node 2] [Node 3]
Must redistribute Node 4's data!
```

**Scenario 3: Uneven Load**
```
[Node 1: 5 TB, 10K qps]   ← Underutilized
[Node 2: 15 TB, 50K qps]  ← Overloaded! 🔥
[Node 3: 10 TB, 20K qps]  ← Normal

Rebalance to distribute load evenly
```

### Strategy 1: Fixed Number of Partitions

Create many more partitions than nodes from the start.

```
┌────────────────────────────────────────────────┐
│    FIXED PARTITIONS                            │
├────────────────────────────────────────────────┤
│                                                │
│  Configuration: 12 partitions, 3 nodes         │
│                                                │
│  Initial:                                      │
│  Node 1: [P1, P2, P3, P4]                     │
│  Node 2: [P5, P6, P7, P8]                     │
│  Node 3: [P9, P10, P11, P12]                  │
│                                                │
│  Add Node 4:                                   │
│  Node 1: [P1, P2, P3]      ← Moved P4         │
│  Node 2: [P5, P6, P7]      ← Moved P8         │
│  Node 3: [P9, P10, P11]    ← Moved P12        │
│  Node 4: [P4, P8, P12]     ← NEW              │
│                                                │
│  Only moved 3 partitions!                      │
└────────────────────────────────────────────────┘
```

**Process**:
1. Pick partitions to move (usually from high-load nodes)
2. Copy data to new node
3. Switch traffic to new node
4. Delete old copy

**Choosing Number of Partitions**:

Rule of thumb: 10-100 partitions per node

```
10 nodes: 100-1000 partitions
100 nodes: 1000-10000 partitions
```

**Too few partitions**: Can't rebalance effectively
**Too many partitions**: Overhead from managing many partitions

**Real-World Example - Riak**:

Riak uses 64 partitions (called vnodes) by default per physical node.

```python
# 3 physical nodes:
Node A: vnodes 0-63
Node B: vnodes 64-127
Node C: vnodes 128-191

# Total: 192 partitions

# Add Node D:
Node A: vnodes 0-47       (kept 48, gave 16)
Node B: vnodes 64-111     (kept 48, gave 16)
Node C: vnodes 128-175    (kept 48, gave 16)
Node D: vnodes 48-63, 112-127, 176-191  (received 48)
```

### Strategy 2: Dynamic Partitioning

Start with small number of partitions, split when they get too large.

```
┌────────────────────────────────────────────────┐
│    DYNAMIC PARTITIONING                        │
├────────────────────────────────────────────────┤
│                                                │
│  Initial: 1 partition                          │
│  [Partition 1: 0 GB]                           │
│                                                │
│  Partition grows:                              │
│  [Partition 1: 10 GB]                          │
│                                                │
│  Split at 10 GB threshold:                     │
│  [Partition 1: 5 GB] [Partition 2: 5 GB]      │
│                                                │
│  Partition 1 grows again:                      │
│  [Partition 1: 10 GB] [Partition 2: 5 GB]     │
│                                                │
│  Split again:                                  │
│  [P1: 5GB] [P3: 5GB] [P2: 5GB]                │
└────────────────────────────────────────────────┘
```

**Advantages**:
- ✅ Adapts to data volume automatically
- ✅ No need to choose partition count upfront

**Disadvantages**:
- ❌ Empty database has only 1 partition (can't distribute initial load)

**Solution: Pre-splitting**

```python
# HBase example: Create table with pre-split regions
create 'users', 'info', SPLITS => ['100', '200', '300', '400']

# Creates 5 regions:
# Region 1: '' to '100'
# Region 2: '100' to '200'
# Region 3: '200' to '300'
# Region 4: '300' to '400'
# Region 5: '400' to ''
```

**Real-World Usage**:
- **HBase**: Dynamic region splitting
- **RethinkDB**: Automatic sharding

**Real-World Example - MongoDB Auto-Sharding**:

```javascript
// MongoDB: Enable sharding
sh.enableSharding("mydb")

// Shard collection by user_id
sh.shardCollection("mydb.users", {user_id: 1})

// Initially: 1 chunk (partition)
// As data grows:
//   Chunk 1: user_id [-∞ to 1000] (2 GB)
//   → Split into:
//      Chunk 1: [-∞ to 500] (1 GB)
//      Chunk 2: [500 to 1000] (1 GB)

// MongoDB automatically splits chunks > 64 MB
// and balances chunks across shards
```

### Strategy 3: Partitioning Proportional to Nodes

Fix the number of partitions per node.

```
┌────────────────────────────────────────────────┐
│    PROPORTIONAL PARTITIONING                   │
├────────────────────────────────────────────────┤
│  Rule: 10 partitions per node                  │
│                                                │
│  3 nodes:                                      │
│  30 partitions total                           │
│  Node 1: [P1...P10]                            │
│  Node 2: [P11...P20]                           │
│  Node 3: [P21...P30]                           │
│                                                │
│  Add Node 4:                                   │
│  40 partitions total                           │
│  - Create 10 new partitions                    │
│  - Randomly split existing partitions          │
│  - Move half to new node                       │
│                                                │
│  Node 1: [P1...P10]                            │
│  Node 2: [P11...P20]                           │
│  Node 3: [P21...P30]                           │
│  Node 4: [P31...P40] ← New partitions          │
└────────────────────────────────────────────────┘
```

**Advantage**:
- ✅ Partition size remains stable as cluster grows

**Used by**: Cassandra 3.0+

### Automatic vs Manual Rebalancing

**Automatic Rebalancing**:
- System decides when and how to move data
- Convenient but risky

**Risk**: Rebalancing is expensive (network, disk I/O). Automatic rebalancing during peak traffic can make things worse!

**Real-World Disaster**:

```
Scenario: E-commerce site during Black Friday

11:00 AM: High traffic, servers at 80% CPU
11:15 AM: One node slows down (garbage collection pause)
11:16 AM: Auto-rebalancer detects slow node, starts moving data
11:17 AM: Network saturated with rebalancing traffic
11:18 AM: All nodes slow down due to network contention
11:19 AM: System cascade failure 💥💥💥

Customers can't check out!
Millions in lost revenue!
```

**Manual Rebalancing**:
- Operator manually triggers rebalancing
- More work but safer

**Best Practice**: Use semi-automatic
- System suggests rebalancing
- Operator approves and schedules it during low-traffic period

**Real-World Example - Couchbase**:

```
Couchbase UI:
  "Cluster is unbalanced. 
   Node 1: 15% of data
   Node 2: 45% of data
   Node 3: 40% of data
   
   [Rebalance] button
   
   Note: Rebalancing may impact performance.
   Schedule during maintenance window."
```

## Part 4: Request Routing (Service Discovery)

**Problem**: Client wants to read `user_id=12345`. Which node should it connect to?

```
┌────────────────────────────────────────────────┐
│    SERVICE DISCOVERY PROBLEM                   │
├────────────────────────────────────────────────┤
│                                                │
│  Client: "Where is user_id=12345?"             │
│                                                │
│  Cluster:                                      │
│  Node 1: user_ids 0-999                        │
│  Node 2: user_ids 1000-1999                    │
│  Node 3: user_ids 2000-2999                    │
│  Node 4: user_ids 3000-3999                    │
│                                                │
│  Answer: Node 4!                               │
│  But how does client find out?                 │
└────────────────────────────────────────────────┘
```

### Approach 1: Allow Clients to Contact Any Node

Client connects to any node. If it's the wrong node, forward the request.

```
┌────────────────────────────────────────────────┐
│  REQUEST FORWARDING                            │
├────────────────────────────────────────────────┤
│                                                │
│  Step 1: Client → Node 2                       │
│  "Get user_id=12345"                           │
│                                                │
│  Step 2: Node 2 checks                         │
│  "12345 is in Node 4's range, not mine"        │
│                                                │
│  Step 3: Node 2 → Node 4                       │
│  "Get user_id=12345"                           │
│                                                │
│  Step 4: Node 4 → Node 2 → Client              │
│  Returns data                                  │
└────────────────────────────────────────────────┘
```

**Pros**:
- ✅ Simple for clients (connect to any node)

**Cons**:
- ❌ Extra network hop (latency)

**Used by**: Cassandra, Riak

**Cassandra Implementation**:

Every Cassandra node knows about all other nodes (gossip protocol).

```python
# Client connects to any node
client = Cluster(['node1.example.com']).connect()

# Query
result = client.execute("SELECT * FROM users WHERE user_id = 12345")

# Behind the scenes:
# 1. node1 receives request
# 2. node1 knows user_id=12345 is on node4 (via gossip)
# 3. node1 forwards to node4
# 4. node4 returns data to node1
# 5. node1 returns to client

# Future requests: Client learns node4 has that data,
# connects directly to node4 (optimization)
```

### Approach 2: Routing Tier

Dedicated load balancer routes requests to correct partition.

```
┌────────────────────────────────────────────────┐
│  ROUTING TIER                                  │
├────────────────────────────────────────────────┤
│                                                │
│  Client → [Load Balancer/Router]               │
│              ↓         ↓        ↓              │
│           [Node 1] [Node 2] [Node 3]           │
│                                                │
│  Router knows partition mapping:               │
│  user_id 0-999 → Node 1                        │
│  user_id 1000-1999 → Node 2                    │
│  user_id 2000-2999 → Node 3                    │
└────────────────────────────────────────────────┘
```

**Pros**:
- ✅ No extra hop (router sends to correct node)
- ✅ Clients are simple (always connect to router)

**Cons**:
- ❌ Router is single point of failure
- ❌ Router can become bottleneck

**Used by**: Many systems with HAProxy, nginx

**Real-World Example - MongoDB**:

```
┌────────────────────────────────────────────────┐
│  MONGODB SHARDED CLUSTER                       │
├────────────────────────────────────────────────┤
│                                                │
│  Application → [mongos] (query router)         │
│                    ↓                           │
│         ┌──────────┼──────────┐               │
│         ↓          ↓          ↓               │
│    [Shard 1]  [Shard 2]  [Shard 3]            │
│                                                │
│  mongos maintains routing table:               │
│  Collection: users                             │
│    {user_id: MinKey} → {user_id: 1000} → Shard 1 │
│    {user_id: 1000} → {user_id: 2000} → Shard 2   │
│    {user_id: 2000} → {user_id: MaxKey} → Shard 3 │
└────────────────────────────────────────────────┘
```

Application connects to `mongos`, which routes queries to the right shard.

### Approach 3: Partition-Aware Clients

Client itself knows partition mapping.

```
┌────────────────────────────────────────────────┐
│  PARTITION-AWARE CLIENT                        │
├────────────────────────────────────────────────┤
│                                                │
│  Client has routing table:                     │
│  user_id 0-999 → node1.example.com             │
│  user_id 1000-1999 → node2.example.com         │
│  user_id 2000-2999 → node3.example.com         │
│                                                │
│  Client calculates:                            │
│  user_id=12345 → node4.example.com             │
│                                                │
│  Client directly connects to correct node!     │
└────────────────────────────────────────────────┘
```

**Pros**:
- ✅ Lowest latency (direct connection)
- ✅ No routing tier needed

**Cons**:
- ❌ Complex client library
- ❌ Client must track partition changes

**Used by**: Some NoSQL drivers (Cassandra, Riak client libraries)

### Coordination Services: ZooKeeper

**Problem**: How do nodes/routers/clients know the current partition mapping?

**Solution**: Use a coordination service like **ZooKeeper** or **etcd**.

```
┌────────────────────────────────────────────────┐
│  ZOOKEEPER-BASED ROUTING                       │
├────────────────────────────────────────────────┤
│                                                │
│         [ZooKeeper Cluster]                    │
│         Stores: Partition mapping              │
│                  │                             │
│       ┌──────────┼──────────┐                 │
│       ↓          ↓          ↓                  │
│   [Node 1]   [Node 2]   [Node 3]              │
│       ↑          ↑          ↑                  │
│       └──────────┼──────────┘                 │
│                  │                             │
│            Notify changes                      │
│                  │                             │
│       [Routing Tier / Client]                  │
│       Subscribes to changes                    │
└────────────────────────────────────────────────┘
```

**ZooKeeper Stores**:
- Which partitions are on which nodes
- Which node is the leader for each partition
- When nodes join/leave

**Process**:

1. **Nodes register with ZooKeeper**
```python
# Node 1 starts up
zk.create("/nodes/node1", "alive", ephemeral=True)
zk.create("/partitions/p1", "node1")
zk.create("/partitions/p2", "node1")
```

2. **Router/Client watches ZooKeeper**
```python
def on_partition_change(partitions):
    # Update local routing table
    routing_table = build_routing_table(partitions)

zk.watch("/partitions", on_partition_change)
```

3. **Partition moves**
```python
# Rebalancing: Move partition 5 from node2 to node4
zk.set("/partitions/p5", "node4")
# ↓
# ZooKeeper notifies all watchers
# ↓  
# Routers/clients update their routing tables
```

**Real-World Usage**:
- **HBase**: Uses ZooKeeper to track region servers
- **Kafka**: Uses ZooKeeper for topic partition assignment (moving to internal metadata)
- **MongoDB**: Uses its own internal version (config servers)

## Part 5: Real-World Examples

### Example 1: Instagram's Sharding Journey

**Initial (2010)**: Single PostgreSQL database

**Problem (2011)**: 10 million users, database overloaded

**Solution**: Sharded by user_id
```
hash(user_id) % 1000 → Shard ID

Initially: 1000 logical shards, 100 physical servers
Each server: 10 logical shards
```

**Why 1000 logical shards, not 100?**
- Can split physical servers easily
- Server 1 with shards [0-9] → Split to two servers: [0-4] and [5-9]

**User ID Generation**:
```python
# Instagram snowflake-style IDs
# |--- 41 bits: timestamp ---|-- 13 bits: shard_id --|-- 10 bits: sequence --|

user_id = (timestamp << 23) | (shard_id << 10) | sequence
```

User ID encodes which shard it's on! Makes lookup O(1).

**Result**: Scaled to 1 billion users by 2018

### Example 2: Discord's Message Storage

**Challenge**: Billions of messages, need fast access to recent messages in each channel.

**Partitioning Strategy**: By channel_id and timestamp

```python
# Partition key: (channel_id, bucket)
# bucket = timestamp // BUCKET_SIZE

message_id = (timestamp << 22) | (shard_id << 12) | sequence
partition_key = (channel_id, message_id // BUCKET_SIZE)
```

**Benefits**:
- Recent messages in same channel are co-located
- Range queries are efficient: "Get last 50 messages in channel"

**Technology**: Cassandra with custom partitioning

### Example 3: Uber's Schemaless (Docstore)

**Challenge**: Different cities have different access patterns

**Partitioning**: Multi-level
1. First level: By city
2. Second level: By entity type (users, trips, payments)
3. Third level: By entity_id

```
Partition: city + entity_type + hash(entity_id)

Examples:
SF + users + hash(user_123) → Partition A
NYC + trips + hash(trip_456) → Partition B
```

**Benefits**:
- Can scale each city independently
- Can prioritize high-value cities (more resources)
- Data locality (city data stays together)

**Result**: Supports 18 million trips per day across 600+ cities

## Summary

**Key Takeaways**:

1. **Partitioning Strategies**
   - **Range partitioning**: Good for range queries, risk of hot spots
   - **Hash partitioning**: Even distribution, can't do range queries
   - **Consistent hashing**: Minimizes data movement during rebalancing

2. **Secondary Indexes**
   - **Document-partitioned**: Fast writes, slow reads (scatter/gather)
   - **Term-partitioned**: Fast reads, slower writes (async updates)

3. **Rebalancing**
   - **Fixed partitions**: Simple, need to choose count upfront
   - **Dynamic**: Adapts automatically, empty database is slow
   - **Proportional**: Stable partition sizes

4. **Request Routing**
   - **Contact any node**: Simple, extra hop
   - **Routing tier**: Clean separation, potential bottleneck
   - **Partition-aware clients**: Fastest, complex clients
   - **ZooKeeper**: Coordination for tracking partition changes

5. **Real-World Wisdom**
   - Start simple (range or hash partitioning)
   - Monitor hot spots constantly
   - Be very careful with automatic rebalancing
   - Over-provision partitions (easier to rebalance)
   - Test rebalancing in staging first!

**Next Chapter**: Transactions - how to ensure correctness when multiple operations must succeed or fail together.
