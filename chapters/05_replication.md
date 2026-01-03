# Chapter 5: Replication

## Introduction: Why Replicate Data?

Imagine you run a photo-sharing app like Instagram. You have one server in California storing all photos. What happens if:
- The server crashes? → All photos are lost! 💥
- A user in Australia requests a photo? → Takes 2 seconds due to distance 🐌
- 10 million users try to access photos simultaneously? → Server overload! 🔥

**Solution**: **Replication** - keeping copies of the same data on multiple machines.

**Three Main Reasons to Replicate**:

1. **High Availability**: If one machine fails, others continue serving requests
2. **Reduced Latency**: Serve requests from nearby servers
3. **Increased Throughput**: Distribute load across multiple machines

```
┌────────────────────────────────────────────────────┐
│           WHY REPLICATE?                           │
├────────────────────────────────────────────────────┤
│                                                    │
│  Single Server:                                    │
│  [Database] → 💥 Fails = System Down              │
│             → 🐌 Far = Slow Response              │
│             → 🔥 Overload = Can't Handle Load     │
│                                                    │
│  Replicated:                                       │
│  [DB-1] [DB-2] [DB-3]                             │
│    ✅    ✅     💥  → Still works (2/3 alive)     │
│    🌎    🌍    🌏   → Serve from nearby copy     │
│    📊    📊    📊   → Split load 3 ways          │
└────────────────────────────────────────────────────┘
```

**The Challenge**: Keeping replicas synchronized is hard! This chapter explores different strategies and their trade-offs.

## Part 1: Leader-Based Replication (Master-Slave)

### How It Works

The most common replication approach uses a **leader** (also called **master** or **primary**) and **followers** (also called **slaves**, **secondaries**, or **read replicas**).

```
┌─────────────────────────────────────────────────────┐
│        LEADER-BASED REPLICATION                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│         📝 WRITES                                   │
│           │                                         │
│           ↓                                         │
│      ┌─────────┐                                   │
│      │ LEADER  │                                    │
│      │ (write) │                                    │
│      └─────────┘                                   │
│           │                                         │
│           ├──── Replication ────┐                  │
│           │                     │                  │
│           ↓                     ↓                  │
│      ┌─────────┐          ┌─────────┐             │
│      │FOLLOWER1│          │FOLLOWER2│             │
│      │ (read)  │          │ (read)  │             │
│      └─────────┘          └─────────┘             │
│           ↑                     ↑                  │
│           │                     │                  │
│         📖 READS             📖 READS              │
└─────────────────────────────────────────────────────┘
```

**Process**:
1. Client sends **write** requests to the leader
2. Leader writes to local storage
3. Leader sends change to all followers (via **replication log**)
4. Followers apply the change
5. Clients can send **read** requests to any replica

**Real-World Example - PostgreSQL Streaming Replication**:

```python
# Application code
import psycopg2

# Write to leader
leader_conn = psycopg2.connect(
    host='db-leader.example.com',
    database='myapp'
)
cursor = leader_conn.cursor()
cursor.execute(
    "INSERT INTO users (name, email) VALUES (%s, %s)",
    ('Alice', 'alice@example.com')
)
leader_conn.commit()

# Read from follower
follower_conn = psycopg2.connect(
    host='db-follower-1.example.com',
    database='myapp'
)
cursor = follower_conn.cursor()
cursor.execute("SELECT * FROM users WHERE email = %s", ('alice@example.com',))
user = cursor.fetchone()
```

**Popular Implementations**:
- **Relational**: PostgreSQL, MySQL, Oracle, SQL Server
- **NoSQL**: MongoDB, RethinkDB, Espresso
- **Distributed**: Kafka (maintains replicated logs)

### Synchronous vs Asynchronous Replication

This is one of the most critical decisions in replication!

#### Synchronous Replication

Leader waits for follower to confirm the write before reporting success.

```
Time: ──→

Client          Leader          Follower 1
  │               │                │
  ├─ WRITE ──→    │                │
  │               ├─ Replicate ──→ │
  │               │                ├─ Write to disk
  │               │    ← ACK ──────┤
  ├  ← SUCCESS ───┤                │
  │               │                │
```

**Advantages**:
- ✅ **Guaranteed durability**: If leader crashes immediately after acknowledging write, follower has the data
- ✅ **Up-to-date followers**: Followers are always consistent with leader

**Disadvantages**:
- ❌ **Slower writes**: Must wait for network round-trip to follower
- ❌ **Availability risk**: If follower is unavailable or slow, all writes are blocked

**Real-World Impact**: 
- A database in Virginia with synchronous replication to Oregon adds ~60ms latency to EVERY write
- If Oregon datacenter loses network connectivity, ALL writes fail

#### Asynchronous Replication

Leader doesn't wait for followers; sends replication data but continues immediately.

```
Time: ──→

Client          Leader          Follower 1
  │               │                │
  ├─ WRITE ──→    │                │
  │               ├─ Replicate ──→ │
  ├  ← SUCCESS ───┤                │
  │               │                ├─ Write to disk
  │               │                ├─ ACK (ignored)
```

**Advantages**:
- ✅ **Fast writes**: No waiting for followers
- ✅ **High availability**: Leader continues even if all followers are down

**Disadvantages**:
- ❌ **Possible data loss**: If leader crashes before followers receive data, writes are lost
- ❌ **Stale reads**: Followers may be behind, returning outdated data

**Real-World Example - The GitHub Outage (2012)**:

GitHub used MySQL with asynchronous replication. Due to a configuration change:
1. Leader crashed after accepting writes
2. Followers hadn't received the data yet
3. Several minutes of user data (issues, comments, etc.) were lost

This led them to implement semi-synchronous replication (at least one follower must confirm).

#### Semi-Synchronous Replication

A compromise: leader waits for ONE follower, others are asynchronous.

```
Time: ──→

Client    Leader    Follower 1 (sync)   Follower 2 (async)
  │         │             │                    │
  ├─WRITE─→ │             │                    │
  │         ├─Replicate─→ │                    │
  │         ├─Replicate───────────────────────→│
  │         │             ├─ ACK               │
  ├←SUCCESS─┤             │                    │
  │         │             │                    ├─ ACK (ignored)
```

**Best Practice**: This is what most production systems use!
- **MySQL**: semi-sync replication plugin
- **PostgreSQL**: synchronous_standby_names = '1 (follower1, follower2)'

### Setting Up New Followers

**Challenge**: How do you add a new follower to a running system without downtime?

You can't just copy the data files because the leader is constantly being written to!

**Standard Process**:

1. **Take a snapshot** of leader's database (without locking)
2. **Copy snapshot** to new follower
3. **Connect follower** to leader and request all changes since snapshot
4. **Catch up** to leader
5. **Start serving** read requests

```
┌────────────────────────────────────────────────┐
│      ADDING NEW FOLLOWER                       │
├────────────────────────────────────────────────┤
│                                                │
│ Step 1: Snapshot                               │
│  [Leader] ─→ snapshot@position=1000            │
│                                                │
│ Step 2: Copy                                   │
│  snapshot@1000 ────────→ [New Follower]       │
│                                                │
│ Step 3: Stream Changes                         │
│  [Leader] ─→ changes from 1000 onward          │
│       ↓                      ↓                 │
│  [Old Follower]        [New Follower]          │
│                                                │
│ Step 4: Caught Up                              │
│  [Leader]@5000                                 │
│  [Old Follower]@5000                           │
│  [New Follower]@5000 ← Ready!                  │
└────────────────────────────────────────────────┘
```

**Real-World Example - MongoDB Replica Set**:

```javascript
// Add new member to replica set
rs.add("mongodb-new.example.com:27017")

// MongoDB automatically:
// 1. Takes snapshot from another member
// 2. Copies data to new member
// 3. Streams oplog (operation log) to catch up
// 4. Marks member as SECONDARY when caught up

// Check status
rs.status()
// Output shows new member catching up:
// { 
//   name: "mongodb-new:27017",
//   state: "RECOVERING",  // Initially
//   ...
// }
// 
// After caught up:
// {
//   name: "mongodb-new:27017", 
//   state: "SECONDARY",  // Ready to serve reads!
//   ...
// }
```

### Handling Node Outages

Systems fail. The question is how to handle it gracefully.

#### Follower Failure: Catch-up Recovery

Relatively easy to handle!

```
Timeline:
T0: [Leader]@position=1000 → [Follower] receives
T1: [Leader]@position=1200 → [Follower] CRASHES 💥
T2: [Leader]@position=1400 → (follower down)
T3: [Leader]@position=1600 → (follower down)
T4: [Follower] RESTARTS ✅
T5: [Follower] checks last position: 1000
T6: [Follower] requests changes from 1000 onwards
T7: [Follower]@position=1600 → Caught up!
```

**Process**:
1. Follower keeps a log of which replication position it last processed
2. After restart, follower requests all changes since that position
3. Once caught up, continues normally

**Implementation**:
```sql
-- PostgreSQL: Check replication status
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;

-- sent_lsn: What leader sent
-- replay_lsn: What follower has applied
-- If follower restarts, it resumes from replay_lsn
```

#### Leader Failure: Failover

Much more complex! This is called **failover**.

**Steps**:
1. **Detect failure**: Usually via timeout (e.g., 30 seconds no heartbeat)
2. **Choose new leader**: Usually the most up-to-date follower
3. **Reconfigure system**: Clients and followers must use new leader
4. **Deal with old leader**: If it comes back, make it a follower

```
┌────────────────────────────────────────────────┐
│           FAILOVER PROCESS                     │
├────────────────────────────────────────────────┤
│                                                │
│ BEFORE:                                        │
│  [Leader]                                      │
│     ↓                                          │
│  [F1] [F2] [F3]                               │
│                                                │
│ FAILURE:                                       │
│  [Leader] 💥                                   │
│     ↓                                          │
│  [F1] [F2] [F3]                               │
│        ↑                                       │
│    Most up-to-date                             │
│                                                │
│ AFTER FAILOVER:                                │
│  [F2] ← Now leader!                           │
│     ↓                                          │
│  [F1] [F3] [Old Leader (if recovered)]        │
└────────────────────────────────────────────────┘
```

**Challenges**:

1. **Lost Writes with Asynchronous Replication**

```
T0: Client writes X=5 to leader → Leader stores X=5
T1: Leader sends X=5 to followers (async)
T2: Leader crashes 💥 (before followers receive)
T3: Follower promoted to new leader (still has X=old_value)
T4: Original leader recovers, has X=5
    New leader has X=old_value
    → Conflict! Which is correct?
```

**Solution**: Usually discard old leader's un-replicated writes. This means **data loss**!

**Real-World Disaster - GitHub (2012)**:
- MySQL leader crashed after accepting writes
- Followers hadn't received some writes
- Promoted follower became new leader
- Lost several minutes of user data (issues, pull requests, comments)
- Required complex recovery process

2. **Split Brain**

Two nodes both think they're the leader!

```
Network Partition:
┌─────────────────┐          ┌─────────────────┐
│  [Old Leader]   │          │  [New Leader]   │
│  (thinks I'm    │    ╱╲    │  (thinks I'm    │
│   leader!)      │   ╱  ╲   │   leader!)      │
│       ↓         │  Network │       ↓         │
│  [Client A]     │  Failure │  [Client B]     │
└─────────────────┘          └─────────────────┘

Both accept writes!
Client A: X = 10
Client B: X = 20

When network heals, which value is correct? 🤔
```

**Solution**: Use a mechanism to ensure only one leader (e.g., leader leases, consensus algorithms).

3. **Timeout Tuning**

Too short timeout:
- ❌ False positives: Temporary slow network causes unnecessary failover
- ❌ Unnecessary failovers cause load spikes

Too long timeout:
- ❌ Longer recovery time
- ❌ More downtime

**Real-World Example - Netflix**:
Netflix uses automatic failover with 30-second timeout for detecting failures, but requires manual approval for actual failover (to avoid split-brain scenarios).

### Replication Logs Implementation

How does the leader actually send changes to followers? Several approaches:

#### 1. Statement-Based Replication

Send the actual SQL statements.

```sql
-- Leader executes:
INSERT INTO users (id, name) VALUES (1, 'Alice');
UPDATE accounts SET balance = balance + 100 WHERE user_id = 1;

-- Sends these exact statements to followers
-- Followers execute the same SQL
```

**Problems**:

❌ **Nondeterministic functions**:
```sql
-- Different result on each server!
INSERT INTO logs (timestamp) VALUES (NOW());
INSERT INTO orders (id) VALUES (RAND());
```

❌ **Auto-incrementing columns**: Different servers might generate different IDs

❌ **Side effects**: Triggers, stored procedures might behave differently

**Historical Note**: MySQL used statement-based replication before version 5.1. Led to many subtle bugs!

#### 2. Write-Ahead Log (WAL) Shipping

Send the low-level disk write log.

```
Every database uses a WAL for crash recovery:
┌────────────────────────────────────────┐
│  Write-Ahead Log (PostgreSQL)          │
├────────────────────────────────────────┤
│ Position 1000: Write to page 5, offset 100, bytes: 0xF3A2... │
│ Position 1001: Write to page 5, offset 150, bytes: 0x87BC... │
│ Position 1002: Write to page 8, offset 200, bytes: 0x12DE... │
└────────────────────────────────────────┘

Leader ships this exact log to followers!
```

**Used by**: PostgreSQL, Oracle

**Advantages**:
- ✅ Exact replica: Byte-for-byte identical

**Disadvantages**:
- ❌ **Tightly coupled to storage engine**: Log format contains low-level details
- ❌ **Version incompatibility**: Can't replicate between different database versions

**Real-World Pain Point**: PostgreSQL major version upgrades require dump and restore because WAL format changes!

#### 3. Logical (Row-Based) Log Replication

Send logical representation of changes (which rows changed).

```
Logical Log Format:
┌────────────────────────────────────────┐
│ Transaction 1000:                      │
│   INSERT INTO users                    │
│   Row: (id=123, name='Alice', ...)    │
│                                        │
│ Transaction 1001:                      │
│   UPDATE users WHERE id=123           │
│   Old: (id=123, balance=100)          │
│   New: (id=123, balance=200)          │
│                                        │
│ Transaction 1002:                      │
│   DELETE FROM users WHERE id=456      │
│   Row: (id=456, name='Bob', ...)      │
└────────────────────────────────────────┘
```

**Used by**: MySQL binlog (row-based mode), MongoDB oplog

**Advantages**:
- ✅ **Version independent**: Can replicate between different database versions
- ✅ **External applications can parse**: Useful for change data capture (CDC)

**Real-World Use Case - LinkedIn Databus**:

LinkedIn built Databus to stream database changes to other systems:

```
[MySQL] → binlog → [Databus]
                      ↓
         ┌────────────┼────────────┐
         ↓            ↓            ↓
    [Search]    [Analytics]    [Cache]
```

Users update their profile in MySQL → automatically updates search index, analytics system, and cache!

#### 4. Trigger-Based Replication

Use database triggers to record changes in a custom table.

```sql
-- PostgreSQL example
CREATE TABLE replication_log (
    table_name TEXT,
    operation TEXT,  -- INSERT/UPDATE/DELETE
    row_data JSONB,
    timestamp TIMESTAMP
);

CREATE TRIGGER users_replication
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION log_changes();

CREATE FUNCTION log_changes() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO replication_log VALUES (
            TG_TABLE_NAME, 'INSERT', row_to_json(NEW), NOW()
        );
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO replication_log VALUES (
            TG_TABLE_NAME, 'UPDATE', row_to_json(NEW), NOW()
        );
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO replication_log VALUES (
            TG_TABLE_NAME, 'DELETE', row_to_json(OLD), NOW()
        );
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Advantages**:
- ✅ **Flexible**: Can add custom logic
- ✅ **Selective replication**: Only replicate certain tables/columns

**Disadvantages**:
- ❌ **Performance overhead**: Triggers slow down writes
- ❌ **More complex**: More code to maintain

**Used by**: Oracle GoldenGate, Databus for Oracle

## Part 2: Problems with Replication Lag

Asynchronous replication is fast but creates a window where followers are behind the leader. This is called **replication lag**.

```
┌────────────────────────────────────────────────┐
│         REPLICATION LAG                        │
├────────────────────────────────────────────────┤
│ Time: 0s                                       │
│  [Leader]@100                                  │
│  [Follower]@100                                │
│  Lag: 0 seconds ✅                             │
│                                                │
│ Time: 1s (write happens)                       │
│  [Leader]@200                                  │
│  [Follower]@100                                │
│  Lag: 1 second                                 │
│                                                │
│ Time: 5s (heavy load)                          │
│  [Leader]@500                                  │
│  [Follower]@200                                │
│  Lag: 5 seconds ⚠️                             │
│                                                │
│ Time: Eventually                               │
│  [Leader]@500                                  │
│  [Follower]@500                                │
│  Lag: 0 seconds ✅                             │
└────────────────────────────────────────────────┘
```

Usually lag is milliseconds, but can grow to seconds or even minutes under high load. This creates anomalies...

### Problem 1: Reading Your Own Writes

**Scenario**: You update your profile and immediately view it, but see old data!

```
Time: 0s
  Client: [POST /profile] → Leader: name = "Alice"
  Leader: ✅ Saved

Time: 0.5s  
  Leader: name = "Alice"
  Follower-1: name = "Bob" (old value - not replicated yet)

Time: 1s
  Client: [GET /profile] → Follower-1
  Follower-1: Returns name = "Bob" ❌
  
  User sees old data immediately after update! Confusing!
```

**Real-World Example**:
You post a tweet. You refresh. Your tweet isn't there! You think it failed so you post again. Now you have duplicate tweets!

**Solution: Read-After-Write Consistency**

Ensure users can see their own writes.

**Implementation Strategies**:

1. **Read user's own data from leader**

```python
def get_profile(user_id, requesting_user_id):
    # If viewing your own profile, read from leader
    if user_id == requesting_user_id:
        return read_from_leader(user_id)
    else:
        # Others can read from follower
        return read_from_follower(user_id)
```

2. **Track timestamp of last write**

```python
def write_profile(user_id, data):
    leader.write(user_id, data)
    timestamp = leader.get_replication_timestamp()
    # Return timestamp to client
    return {"success": True, "timestamp": timestamp}

def read_profile(user_id, min_timestamp):
    # Only read from replicas that have caught up
    for follower in followers:
        if follower.replication_timestamp() >= min_timestamp:
            return follower.read(user_id)
    # If no follower caught up, read from leader
    return leader.read(user_id)
```

3. **Read from leader for 1 minute after write**

```python
class UserSession:
    last_write_time = None
    
def read_profile(session, user_id):
    if session.last_write_time and (time.now() - session.last_write_time < 60):
        # Recent write - read from leader
        return leader.read(user_id)
    else:
        return follower.read(user_id)
```

**Real-World: Facebook's Solution**

Facebook uses a hybrid approach:
- Profile updates go to leader
- For ~3 seconds after update, reads come from leader
- After 3 seconds, reads come from nearby cache/follower

### Problem 2: Monotonic Reads

**Scenario**: Time goes backward! You see a comment, refresh, and it disappears!

```
Time: 0s
  [Leader]@100: Comments = ["Good!", "Nice!"]
  [Follower-1]@100: Comments = ["Good!", "Nice!"]
  [Follower-2]@90: Comments = ["Good!"] (lagging)

Time: 1s
  Client: [GET /comments] → Load Balancer → Follower-1
  Sees: ["Good!", "Nice!"]

Time: 2s
  Client: [GET /comments] → Load Balancer → Follower-2
  Sees: ["Good!"] ❌
  
  "Nice!" disappeared! User thinks comment was deleted!
```

**Real-World Impact**:
Amazon reviews: You see 50 reviews, refresh, see 48 reviews. Confusing!

**Solution: Monotonic Reads**

Once you've seen data at time T, you never see older data.

**Implementation**:

```python
def get_comments(user_id, last_seen_timestamp):
    # Hash user_id to consistently route to same follower
    follower = followers[hash(user_id) % len(followers)]
    
    # Ensure follower has caught up to last_seen_timestamp
    while follower.replication_timestamp() < last_seen_timestamp:
        time.sleep(0.01)  # Wait for replication
    
    data = follower.read()
    new_timestamp = follower.replication_timestamp()
    return data, new_timestamp
```

**Alternative**: Always route same user to same follower (sticky sessions)

```
User Alice (ID: 123) → Always routes to Follower-1
User Bob (ID: 456) → Always routes to Follower-2
```

### Problem 3: Consistent Prefix Reads

**Scenario**: See effects before causes! Like watching a conversation where answers come before questions!

```
Time: 0s
  [Leader]: "What's the capital of France?" 
  [Leader]: "Paris!"

Replication (different speeds):
  [Follower-1]: Receives answer first: "Paris!"
  [Follower-2]: Receives question first: "What's the capital of France?"

User reads:
  Request 1 → Follower-1: Sees "Paris!" (no question) 🤔
  Request 2 → Follower-2: Sees "What's the capital of France?"
  
  Out of order! Causality violated!
```

**Real-World Example - Social Media**:

```
Alice: "Should I buy the red or blue dress?"
Bob: "Definitely blue!"

Charlie sees (due to replication lag):
Bob: "Definitely blue!" ← Sees this first
Alice: "Should I buy the red or blue dress?" ← Sees this second

Confusing!
```

**Solution: Consistent Prefix Reads**

Causally related writes appear in correct order.

**Implementation for Partitioned Databases**:

```python
# Track causal dependencies
def write_with_causality(data, causal_dependency=None):
    write_id = generate_id()
    
    if causal_dependency:
        # Wait for dependency to be replicated everywhere
        wait_for_replication(causal_dependency)
    
    leader.write(data, write_id, causal_dependency)
    return write_id

# Usage
question_id = write_with_causality("What's the capital of France?")
answer_id = write_with_causality(
    "Paris!", 
    causal_dependency=question_id  # Answer depends on question
)
```

## Part 3: Multi-Leader Replication

### What and Why?

Instead of one leader, have multiple leaders that accept writes.

```
┌────────────────────────────────────────────────┐
│      MULTI-LEADER REPLICATION                  │
├────────────────────────────────────────────────┤
│                                                │
│  Datacenter 1           Datacenter 2           │
│  ┌──────────┐          ┌──────────┐           │
│  │ Leader 1 │◄────────►│ Leader 2 │           │
│  └────┬─────┘          └────┬─────┘           │
│       │                     │                  │
│       ↓                     ↓                  │
│  [Followers]            [Followers]            │
└────────────────────────────────────────────────┘
```

### Use Cases

#### 1. Multi-Datacenter Operation

**Single-Leader**:
```
 USA Datacenter              Europe Datacenter
 [Leader]                    [Follower]
    ↑                             ↑
    │                             │
 [Users in USA]              [Users in Europe]
    └── Fast write               └── SLOW write (must go to USA!)
```

**Multi-Leader**:
```
 USA Datacenter              Europe Datacenter
 [Leader 1] ◄────────────►  [Leader 2]
    ↑                             ↑
    │                             │
 [Users in USA]              [Users in Europe]
    └── Fast write               └── Fast write!
```

**Benefits**:
- ✅ Performance: Each region writes to local leader (low latency)
- ✅ Availability: If datacenter connection fails, each can operate independently
- ✅ Network tolerance: No cross-datacenter writes in the critical path

**Real-World Example**: CouchDB's multi-datacenter deployment

#### 2. Clients with Offline Operation

**Example**: Note-taking apps (Evernote, Notion, Google Docs)

```
┌────────────────────────────────────────────┐
│  OFFLINE-FIRST APPLICATION                 │
├────────────────────────────────────────────┤
│                                            │
│ Your Phone (offline)                       │
│ [Local Database] ← You edit notes          │
│                                            │
│ Your Laptop (offline)                      │
│ [Local Database] ← You edit notes          │
│                                            │
│ When online:                               │
│ [Phone]  ───────┐                          │
│                 ↓                          │
│ [Server] ← Synchronize                     │
│                 ↑                          │
│ [Laptop] ───────┘                          │
└────────────────────────────────────────────┘
```

Each device is effectively a "leader" - accepts writes even when offline!

### The Big Problem: Write Conflicts

**Scenario**: Two users edit the same document simultaneously

```
Time: 0s
  Document: "Distributed Systems are cool"

Time: 1s
  User A (USA): Changes to "Distributed Systems are awesome"
  User B (Europe): Changes to "Distributed Systems are amazing"

  Both writes succeed at their local datacenter!

Time: 2s (replication happens)
  USA datacenter receives: "amazing" from Europe
  Europe datacenter receives: "awesome" from USA
  
  CONFLICT! Which version is correct? 🤔
```

#### Conflict Avoidance

**Strategy**: Ensure writes for particular record always go to same datacenter.

```python
# Route based on user location
def get_leader_for_user(user_id):
    user = get_user(user_id)
    if user.region == "USA":
        return usa_leader
    elif user.region == "Europe":
        return europe_leader
```

**Works well unless**:
- User travels (USA → Europe)
- Datacenter fails (must reroute to different datacenter)

#### Converging Toward Consistent State

All replicas must eventually have the same data. How?

**1. Last Write Wins (LWW)**

Give each write a timestamp. Highest timestamp wins.

```python
def merge_writes(write_a, write_b):
    if write_a.timestamp > write_b.timestamp:
        return write_a.value
    else:
        return write_b.value

# Example:
write_a = {"value": "awesome", "timestamp": 1000}
write_b = {"value": "amazing", "timestamp": 1001}

result = merge_writes(write_a, write_b)  # "amazing" (higher timestamp)
```

**Problem**: Data loss! "awesome" is discarded even though it was a valid edit.

**Real-World**: Cassandra, Riak use LWW as default

**2. Replica with Higher ID Wins**

Arbitrary but deterministic.

```python
write_a = {"value": "awesome", "replica_id": 1}
write_b = {"value": "amazing", "replica_id": 2}

# Replica 2 > Replica 1, so "amazing" wins
```

**Problem**: Still data loss!

**3. Merge Values**

Concatenate or combine conflicting writes.

```python
write_a = {"value": "awesome"}
write_b = {"value": "amazing"}

# Merge:
result = "awesome/amazing"  # Both preserved!
```

**Problem**: Messy and may not make sense semantically

**4. Store Conflict, Let Application Decide**

Preserve both versions, prompt user to resolve.

```python
def read_document(doc_id):
    versions = database.get_all_versions(doc_id)
    if len(versions) > 1:
        # Return conflict to application
        return {
            "conflict": True,
            "versions": versions
        }
    else:
        return versions[0]
```

**Real-World Example - CouchDB**:

CouchDB returns all conflicting versions:

```json
{
  "_id": "doc123",
  "_conflicts": ["2-abc...", "2-def..."],
  "text": "awesome"  // Current version
}
```

Application must resolve by choosing or merging versions.

**Real-World Example - Google Docs**:

Google Docs uses **Operational Transformation (OT)** to automatically resolve conflicts:

```
User A types: "Hello"
User B types: "World" at same time

OT algorithm transforms operations:
- User A's "Hello" at position 0
- User B's "World" at position 0
  → Transformed to position 5 (after "Hello")

Result: "HelloWorld" ✅ (not "WorldHello")
```

### Multi-Leader Replication Topologies

How do leaders communicate?

#### All-to-All (Most Common)

```
     [Leader 1]
       ↗  ↑  ↖
      ↙   │   ↘
[Leader 2]─┼─[Leader 3]
      ↘   │   ↗
       ↖  ↓  ↙
     [Leader 4]
```

Every leader sends changes to every other leader.

**Advantages**: ✅ Fault-tolerant (many paths)

**Disadvantages**: ❌ Ordering issues (consistent prefix reads problem)

#### Circular

```
[Leader 1] → [Leader 2] → [Leader 3] → [Leader 1]
```

**Advantages**: ✅ Simple

**Disadvantages**: 
- ❌ If one node fails, circle breaks
- ❌ Higher latency (must travel through all nodes)

#### Star

```
        [Leader 1]
         ↙ ↓ ↘
[Leader 2][Leader 3][Leader 4]
```

Leader 1 is the hub.

**Advantages**: ✅ Simple

**Disadvantages**: ❌ If Leader 1 fails, topology breaks

## Part 4: Leaderless Replication

### What is Leaderless Replication?

No designated leader. Client sends writes to multiple replicas simultaneously.

```
┌────────────────────────────────────────────┐
│      LEADERLESS REPLICATION                │
├────────────────────────────────────────────┤
│                                            │
│           Client                           │
│             │                              │
│      ┌──────┼──────┐                      │
│      ↓      ↓      ↓                      │
│  [Node 1][Node 2][Node 3]                 │
│             ↑                              │
│             │                              │
│      Client reads from                     │
│      multiple nodes                        │
└────────────────────────────────────────────┘
```

**Used by**: Amazon Dynamo, Cassandra, Riak, Voldemort

### Quorum Reads and Writes

**Key Idea**: Don't require ALL nodes to acknowledge. Use a quorum.

**Parameters**:
- `n` = number of replicas
- `w` = number of nodes that must acknowledge write
- `r` = number of nodes we read from

**Rule**: If `w + r > n`, we're guaranteed to read up-to-date data.

**Example**: n=3, w=2, r=2

```
Write Process:
Client writes X=5
  ↓
Sends to all 3 nodes
  ├→ Node 1: ✅ ACK
  ├→ Node 2: ✅ ACK
  └→ Node 3: ❌ Down

Client receives 2 ACKs (w=2) → Write successful!
```

```
Read Process:
Client reads X
  ↓
Sends read to all 3 nodes
  ├→ Node 1: Returns X=5 (timestamp: 1000)
  ├→ Node 2: Returns X=5 (timestamp: 1000)
  └→ Node 3: Returns X=3 (timestamp: 900) - stale!

Client got 2 reads (r=2)
Picks latest: X=5 ✅
```

**Why w+r > n works**:

```
w=2 nodes have latest write
r=2 nodes in read

Since 2+2 > 3, at least one node overlaps!
  
[N1] [N2] [N3]
 ✅   ✅   ❌   ← Write went here (w=2)
 ✅   ✅         ← Read from here (r=2)
 
N1 and N2 are in both write and read → guaranteed to see latest!
```

**Common Configurations**:

```
┌──────────────────────────────────────────────┐
│  QUORUM CONFIGURATIONS                       │
├─────┬─────┬─────┬──────────────────────────┤
│  n  │  w  │  r  │     Trade-off            │
├─────┼─────┼─────┼──────────────────────────┤
│  3  │  2  │  2  │  Balanced                │
│  3  │  3  │  1  │  Fast reads, slow writes │
│  3  │  1  │  3  │  Fast writes, slow reads │
│  5  │  3  │  3  │  Can tolerate 2 failures │
│  5  │  4  │  2  │  Strong consistency      │
└─────┴─────┴─────┴──────────────────────────┘
```

### Handling Node Outages

**Scenario**: Node is down during write, comes back later with stale data.

#### Read Repair

When reading, if client detects stale data, update it.

```
Read Process:
Client reads
  ├→ Node 1: X=5 (timestamp: 1000)
  ├→ Node 2: X=5 (timestamp: 1000)
  └→ Node 3: X=3 (timestamp: 900) ← Stale!

Client detects Node 3 is stale
  → Sends X=5 to Node 3 to update it

Node 3 now has X=5 ✅
```

**Limitation**: Only fixes data that is read. Rarely-read data stays stale!

#### Anti-Entropy Process

Background process compares replicas and repairs differences.

```
Background Process (runs periodically):
  Compare Node 1 vs Node 2:
    → If different, sync to latest
  
  Compare Node 1 vs Node 3:
    → If different, sync to latest
    
  Compare Node 2 vs Node 3:
    → If different, sync to latest
```

**Real-World**: Cassandra's "nodetool repair" command

### Detecting Concurrent Writes

**Problem**: Two clients write to same key simultaneously

```
Time: 0s (Both read X=1)
  Client A reads: X=1
  Client B reads: X=1

Time: 1s (Both write)
  Client A writes: X=2
  Client B writes: X=3

Which is correct? 🤔
```

#### Last Write Wins (LWW)

Attach timestamp, keep latest.

```python
def write(key, value):
    timestamp = time.now()
    for node in nodes:
        node.write(key, value, timestamp)

def read(key):
    values = [node.read(key) for node in nodes]
    # Return value with highest timestamp
    return max(values, key=lambda v: v.timestamp).value
```

**Problem**: Not truly last write in wall-clock time! Clocks can drift.

**Real-World Disaster - Amazon Cart**:

Amazon had a bug where items deleted from cart reappeared due to LWW and clock skew between servers!

#### Version Vectors

Track causal history to detect conflicts.

```python
# Version vector: [NodeID: Version]
# Example:

Time: 0
  Node 1 writes X=A: version = {1:1}
  
Time: 1
  Node 2 reads X=A (version {1:1})
  Node 2 writes X=B: version = {1:1, 2:1}
  
Time: 2 (Concurrent writes)
  Node 1 writes X=C: version = {1:2}
  Node 2 writes X=D: version = {1:1, 2:2}
  
  {1:2} vs {1:1, 2:2} → Not comparable → CONFLICT!
```

**Resolution**: Application decides how to merge.

**Real-World**: Riak uses version vectors to detect conflicts

## Summary

**Key Takeaways**:

1. **Leader-Based Replication**
   - ✅ Simple, widely used
   - ⚠️ Choose synchronous vs asynchronous carefully
   - ⚠️ Failover is complex and risky

2. **Replication Lag Problems**
   - Read-after-write consistency
   - Monotonic reads
   - Consistent prefix reads
   - All solvable but require careful design

3. **Multi-Leader Replication**
   - ✅ Better performance, availability
   - ❌ Write conflicts are inevitable
   - Need conflict resolution strategy

4. **Leaderless Replication**
   - ✅ High availability, fault tolerance
   - ⚠️ Eventual consistency
   - Quorums provide tunable consistency

**Real-World Wisdom**:
- Start with single-leader (simplest)
- Use multi-leader only if you need it (multi-datacenter, offline clients)
- Leaderless for highest availability (Cassandra-style)
- Always measure replication lag in production
- Have monitoring and alerting for failover scenarios

**Next Chapter**: Partitioning - how to split data across multiple machines to scale beyond a single machine's capacity.
