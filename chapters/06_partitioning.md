# Chapter 6: Partitioning (Sharding)

## Introduction: Breaking Data Into Pieces

### Terminology: Partitioning vs Sharding

Before we dive in, let's clarify the terminology. This chapter is about **partitioning**, which the book uses as the primary term. However, if you're coming from practical database experience, you might be more familiar with the term **sharding**.

**Are they the same thing?**

Yes! They refer to the same concept: **splitting data across multiple machines**. But there's a subtle cultural difference in how people use these terms:

```
┌─────────────────────────────────────────────────────────┐
│  PARTITIONING vs SHARDING - THE TERMINOLOGY DEBATE      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  PARTITIONING (Book's term):                            │
│  • More academic/theoretical term                       │
│  • Used in research papers                              │
│  • Broader meaning - includes physical disk partitions  │
│  • Can refer to splitting data within a single machine  │
│                                                         │
│  SHARDING (Practitioner's term):                        │
│  • More practical/industry term                         │
│  • What developers and DBAs commonly say                │
│  • Specifically means distributing across MULTIPLE     │
│    physical machines                                    │
│  • Popularized by web-scale companies (Google, FB)     │
│                                                         │
│  IN THIS CHAPTER: We'll use both terms interchangeably │
│  When we say "partitioning" we mean distributed sharding│
└─────────────────────────────────────────────────────────┘
```

**Why does this confusion exist?**

The term "partitioning" in databases originally referred to splitting tables into separate physical sections **on the same disk** for performance reasons. For example, you might partition a `sales` table by year:
- `sales_2022` partition
- `sales_2023` partition  
- `sales_2024` partition

All on the same server! This is still called "partitioning."

But when we talk about scaling **across multiple machines**, that's typically called **sharding** (though the book calls it partitioning). Don't let this confuse you - the concepts are the same.

### The Scaling Problem: When One Server Isn't Enough

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

### The Journey to Sharding: Understanding When and Why

Let's start from the beginning and understand the natural evolution that leads companies to implement sharding. This is the practical journey most applications go through.

#### Stage 1: Starting Small - Single Database Architecture

When you first build an application, you start with the simplest possible setup:

```
┌──────────────────────────────────────────────────────┐
│         STAGE 1: SINGLE DATABASE                     │
├──────────────────────────────────────────────────────┤
│                                                      │
│   [App Server 1]                                     │
│   [App Server 2]  ────────┐                         │
│   [App Server 3]          │                         │
│         ...               ▼                         │
│   [App Server N]     [DATABASE]                     │
│                      32 cores                        │
│                      128 GB RAM                      │
│                      2 TB storage                    │
│                                                      │
│   • All inserts/updates/deletes go here             │
│   • All reads come from here (or replicas)          │
│   • Simple architecture                              │
│   • Easy to manage                                   │
└──────────────────────────────────────────────────────┘
```

**What's happening here:**

1. **Multiple application servers**: You might have 1, 10, 100, or even 1000 application servers depending on your traffic. These could be distributed across different availability zones or regions for redundancy.

2. **One database**: All app servers connect to the same database and run their queries (INSERT, UPDATE, DELETE, SELECT).

3. **This actually works for a while!** Don't assume you need sharding immediately. This simple setup can handle:
   - Millions of rows
   - Thousands of queries per second
   - Terabytes of data
   - **Many successful companies run on this for years**

**Important Reality Check:**

> "It's not like oh you need to go to sharding once you have over a terabyte of data or whatever. There's lots of instances where you can have many terabytes of data and tens of thousands of queries per second and if you have a big enough machine that can handle that." - From video transcript

Don't prematurely optimize! Sharding adds significant complexity. Only do it when necessary.

#### Stage 2: Vertical Scaling - Making the Server Bigger

When your single database starts struggling, the first solution is simple: **make it bigger**.

```
┌──────────────────────────────────────────────────────┐
│         STAGE 2: VERTICAL SCALING                    │
├──────────────────────────────────────────────────────┤
│                                                      │
│   BEFORE:                  AFTER:                    │
│   [DATABASE]              [DATABASE]                 │
│   32 cores         →      64 cores                   │
│   128 GB RAM              512 GB RAM                 │
│   2 TB storage            10 TB storage              │
│                                                      │
│   Getting slow?     Add more resources!             │
│   CPU at 99%?       Double the CPUs!                │
│   Running out       Add more RAM!                   │
│   of memory?                                         │
└──────────────────────────────────────────────────────┘
```

**This is called "Vertical Scaling" (scaling up)**

**Advantages:**
✅ **Simple**: Just resize your server (cloud providers make this easy)
✅ **No application changes**: Your code stays exactly the same
✅ **No complexity**: Still managing one database
✅ **Works surprisingly well**: Can handle A LOT of traffic

**Limitations:**
❌ **Physical limits**: Eventually you hit hardware limits
❌ **Cost**: Very large servers get exponentially expensive
❌ **Single point of bottleneck**: All traffic still flows through one machine

**Real-World Example:**

Stack Overflow (one of the top 100 websites globally) famously ran on a **single SQL Server** for years, handling billions of page views per month. They just kept making the server bigger. Sometimes vertical scaling is all you need!

#### Stage 3: Adding Replication (Still Not Sharding!)

Even with a single database, you should add **replicas** for reliability and read scaling:

```
┌──────────────────────────────────────────────────────┐
│    STAGE 3: SINGLE DATABASE + REPLICATION            │
├──────────────────────────────────────────────────────┤
│                                                      │
│   [App Servers]                                      │
│        │                                             │
│        ├──── WRITES ────→  [PRIMARY]                │
│        │                   DATABASE                  │
│        │                       │                     │
│        │                  Replication                │
│        │                       │                     │
│        │                   ┌───┴───┐                │
│        │                   ▼       ▼                │
│        └──── READS ──→ [REPLICA] [REPLICA]          │
│                        (AZ-1)     (AZ-2)             │
│                                                      │
│   WHY REPLICAS?                                      │
│   1. Failover: If primary dies, promote replica     │
│   2. Read scaling: Send reads to replicas           │
│   3. Availability: Replicas in different zones      │
│                                                      │
│   STILL NOT SHARDING!                                │
│   This is still ONE database, just replicated       │
└──────────────────────────────────────────────────────┘
```

**Key Understanding:**

> "Even if you have a single machine that's serving all of this, you still are going to want replication... having replicas in other availability zones in case your server breaks. You can also send some of your read traffic to replicas... this doesn't mean necessarily literally a single server because then that's a single point of failure." - From video transcript

**Important:** Replication (Chapter 5) and Partitioning (Chapter 6) are **different concepts**:
- **Replication**: Multiple copies of the SAME data
- **Partitioning**: Different SUBSETS of data on different servers

You almost always want BOTH! Replicate each partition for fault tolerance.

#### Stage 4: The Breaking Point - When You Need Sharding

Eventually, even with a massive server and replicas, you hit a wall. You need sharding when:

**Symptom 1: CPU Pegged at 100%**
```
[DATABASE SERVER]
CPU: ▓▓▓▓▓▓▓▓▓▓ 99%  ← All 64 cores maxed out!
RAM: ▓▓▓▓▓▓▓▓░░ 85%
Disk: ▓▓▓░░░░░░░ 30%

Queries waiting in queue: 5,000+
Response time: 2 seconds (was 50ms)
```

**Symptom 2: Network Congestion**
```
                    [DATABASE]
                        ▲
                        │
        ┌───────────────┼───────────────┐
        │               │               │
   10K req/s       10K req/s       10K req/s
        │               │               │
   [App Servers x 1000]

Total: 30,000 queries/second hitting one database!
Network pipe: SATURATED 🔥
```

**Symptom 3: Write Bottleneck**
```
All writes go to PRIMARY → can't parallelize
Even with 100 replicas for reads, writes are serial!

User posts: ──┐
User likes: ───┼──→ [PRIMARY] ← Bottleneck!
User comments: ┘     (One point of contention)
```

**When These Happen: Time to Shard**

> "There reaches a point where that is basically not going to be sufficient to handle your traffic... even a very very big server when you're talking about a really high-scale application... where people turn to after this generally is some form of partitioning or sharding." - From video transcript

**The Sharding Decision Matrix:**

```
┌────────────────────────────────────────────────────┐
│  DO YOU NEED SHARDING? DECISION TREE               │
├────────────────────────────────────────────────────┤
│                                                    │
│  Can you vertical scale further?                   │
│  ├─ YES → Do that first (simpler!)               │
│  └─ NO → Continue ↓                               │
│                                                    │
│  Is your database CPU/Memory maxed out?           │
│  ├─ NO → Optimize queries, add indexes           │
│  └─ YES → Continue ↓                              │
│                                                    │
│  Have you added read replicas?                     │
│  ├─ NO → Do that first                            │
│  └─ YES (and still struggling) → Continue ↓      │
│                                                    │
│  Is the PRIMARY write load too high?              │
│  ├─ YES → You need sharding!                      │
│  └─ NO → Maybe caching or async processing       │
│                                                    │
│  Do you have hundreds of terabytes?                │
│  ├─ YES → You probably need sharding              │
│  └─ NO → Maybe you don't (yet!)                   │
└────────────────────────────────────────────────────┘
```

**Real Talk:** Sharding adds MASSIVE complexity:
- Application logic becomes more complex
- Joins across shards become difficult/impossible
- Transactions across shards are hard
- Rebalancing data is risky
- More operational burden

Only shard when you absolutely must!

## Part 1: Partitioning of Key-Value Data

The fundamental question: **Given a key, which partition should it go to?**

### A Practical First Example: Social Media Sharding

Before we dive into the formal strategies, let's walk through a complete, practical example of how you might implement your first sharding setup. This will make the formal strategies easier to understand.

**Scenario:** You're building a social media site (like Twitter/X). You have:
- Users (with usernames, bios, profile pictures)
- Posts (text, images, likes)
- Followers/Following relationships
- Comments and replies

Everything is centered around **users** - users create posts, users follow other users, users comment on posts.

#### The Simplest Sharding: Split by Username Range

Let's say your single database can't handle the load anymore. Here's your first sharding approach:

```
┌────────────────────────────────────────────────────┐
│    BEFORE SHARDING: ONE DATABASE                   │
├────────────────────────────────────────────────────┤
│                                                    │
│  [App Servers 1-1000]                              │
│         │                                          │
│         ▼                                          │
│    [DATABASE]                                      │
│    • All users (A-Z)                               │
│    • All posts                                     │
│    • All followers                                 │
│    • All comments                                  │
│                                                    │
│    🔥 OVERLOADED!                                  │
└────────────────────────────────────────────────────┘

                    ↓ SHARD ↓

┌────────────────────────────────────────────────────┐
│    AFTER SHARDING: TWO DATABASES                   │
├────────────────────────────────────────────────────┤
│                                                    │
│  [App Servers]                                     │
│     │      │                                       │
│     │      └──────────┐                           │
│     │                 │                           │
│     ▼                 ▼                           │
│ [DATABASE 1]      [DATABASE 2]                    │
│ Users: A-O        Users: P-Z                      │
│ + their posts     + their posts                   │
│ + their followers + their followers               │
│ + their comments  + their comments                │
│                                                    │
│ Load: 50%         Load: 50%                       │
│ ✅ MANAGEABLE!    ✅ MANAGEABLE!                   │
└────────────────────────────────────────────────────┘
```

**Implementation Details:**

When a user logs in with username "ben":

```javascript
// In your application server code:
function getDatabase(username) {
    // Determine which database based on first letter
    const firstLetter = username[0].toLowerCase();
    
    if (firstLetter >= 'a' && firstLetter <= 'o') {
        return DATABASE_1; // Connection to Database 1
    } else {
        return DATABASE_2; // Connection to Database 2
    }
}

// User login
async function loginUser(username, password) {
    const db = getDatabase(username); // Route to correct database
    
    const user = await db.query(
        'SELECT * FROM users WHERE username = ?',
        [username]
    );
    
    // Verify password, etc.
    return user;
}

// Posting a message
async function createPost(username, content) {
    const db = getDatabase(username); // Same shard as the user
    
    await db.query(
        'INSERT INTO posts (user_id, content, created_at) VALUES (?, ?, NOW())',
        [username, content]
    );
}
```

**Key Insight:**

> "In your application servers you actually would need to sort of have logic in there that you know you get a user decides to log in and let's say their username is Ben right so your application code is basically going to have to know okay their username is Ben, that's in the first shard so I need to go to the first database." - From video transcript

**ALL USER-RELATED DATA GOES TO THE SAME SHARD:**

This is crucial! For user "ben":
- User profile → Database 1
- Ben's posts → Database 1
- Ben's followers → Database 1
- Ben's comments → Database 1

Everything related to "ben" lives together. This is called **co-location** and it's essential for performance.

#### Why Co-Location Matters

**BAD: Data scattered across shards**
```
Database 1: User "ben" profile
Database 2: Ben's posts
Database 1: Ben's followers
Database 2: Ben's comments

To show Ben's profile page:
1. Query Database 1 for profile
2. Query Database 2 for posts
3. Query Database 1 for followers
4. Query Database 2 for comments

= 4 network round trips! 🐌
```

**GOOD: Data co-located on same shard**
```
Database 1: 
  • User "ben" profile
  • Ben's posts
  • Ben's followers
  • Ben's comments

To show Ben's profile page:
1. Query Database 1 for everything

= 1 network round trip! ⚡
```

#### The Cross-Shard Query Problem

Now here's where it gets tricky. What if Alice (Database 1) wants to follow Bob (Database 2)?

```
┌────────────────────────────────────────────────────┐
│         THE CROSS-SHARD FOLLOW PROBLEM             │
├────────────────────────────────────────────────────┤
│                                                    │
│   [Database 1]              [Database 2]           │
│   Users: A-O                Users: P-Z             │
│                                                    │
│   Alice (username: alice)   Bob (username: bob)    │
│   • profile                 • profile              │
│   • posts                   • posts                │
│   • following: [bob, charlie] • followers: [alice]│
│                             ▲                      │
│                             │                      │
│                     This is a cross-shard         │
│                     relationship!                  │
└────────────────────────────────────────────────────┘
```

**Two Approaches:**

**Approach 1: Store relationship on both sides (duplication)**
```javascript
// When alice follows bob:
async function follow(follower, followee) {
    const db1 = getDatabase(follower);  // alice → Database 1
    const db2 = getDatabase(followee);  // bob → Database 2
    
    // Store in BOTH databases
    await db1.query(
        'INSERT INTO following (user, follows) VALUES (?, ?)',
        [follower, followee]
    );
    
    await db2.query(
        'INSERT INTO followers (user, followed_by) VALUES (?, ?)',
        [followee, follower]
    );
}

// To show "who alice follows": Query Database 1 (fast!)
// To show "bob's followers": Query Database 2 (fast!)
```

✅ Fast reads (single database query)  
❌ Data duplication  
❌ Consistency issues (what if one write fails?)

**Approach 2: Store in one place, accept cross-shard queries**
```javascript
// Store following relationship only in follower's shard
async function follow(follower, followee) {
    const db = getDatabase(follower);
    
    await db.query(
        'INSERT INTO following (user, follows) VALUES (?, ?)',
        [follower, followee]
    );
}

// To show "who alice follows": Query Database 1 (fast!)
// To show "bob's followers": Query BOTH databases (slow!)
async function getFollowers(username) {
    // Need to query ALL shards to find who follows this user
    const followers = [];
    
    for (const db of [DATABASE_1, DATABASE_2]) {
        const results = await db.query(
            'SELECT user FROM following WHERE follows = ?',
            [username]
        );
        followers.push(...results);
    }
    
    return followers;
}
```

✅ No data duplication  
❌ Some queries require scanning multiple shards (scatter/gather)

**Real-World Choice:**

Most social networks choose Approach 1 (duplication) because:
1. Showing followers is a common operation (must be fast)
2. Storage is cheap
3. Following/unfollowing is relatively rare (acceptable to write twice)

#### The "Hot Spot" Problem with Range Partitioning

Our simple A-O / P-Z split has a problem: what if more users have names starting with A-O?

```
[Database 1]              [Database 2]
Users: A-O                Users: P-Z
70% of users              30% of users
CPU: 90%                  CPU: 40%
Storage: 7 TB             Storage: 3 TB

🔥 Unbalanced!            😎 Underutilized
```

**Why does this happen?**

In English-speaking countries, names starting with A, B, C, D, E are very common:
- Alice, Andrew, Amy, Adam, Alex...
- Brian, Bob, Beth, Brandon...
- Chris, Christina, Charles, Carol...

But names starting with X, Y, Z are rare!

**Solution Options:**

1. **Adjust the ranges manually:**
   ```
   Database 1: A-G (33% of users)
   Database 2: H-O (33% of users)
   Database 3: P-Z (33% of users)
   ```

2. **Use hash partitioning instead** (covered next!)

3. **Monitor and rebalance** (we'll cover this in Part 3)

This example shows why sharding is **complex** - you have to think about:
- Routing logic in your application
- Data co-location
- Cross-shard queries
- Load balancing
- Consistency across shards

Let's now look at the formal strategies...

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

```javascript
// Sensor data with timestamp as key
// 2024-01-15-00:00:00: temp=20
// 2024-01-15-00:00:01: temp=21
// 2024-01-15-00:00:02: temp=22
// ...

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

```javascript
// Instead of: 2024-03-15-10:30:00
// Use: sensor_id:2024-03-15-10:30:00

// Examples:
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

```javascript
const crypto = require('crypto');

function getPartition(key, numPartitions) {
    // Hash the key
    const hash = crypto.createHash('md5').update(key).digest('hex');
    const hashValue = parseInt(hash, 16);
    
    // Determine partition
    const partitionId = hashValue % numPartitions;
    
    return partitionId;
}

// Example usage:
console.log(getPartition("Alice", 3));    // Output: 2
console.log(getPartition("Bob", 3));      // Output: 0
console.log(getPartition("Charlie", 3));  // Output: 1
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

```javascript
// 3 partitions initially
getPartition("Alice", 3);  // → Partition 0

// Add one more partition (now 4)
getPartition("Alice", 4);  // → Partition 2 ❌

// "Alice"'s data moved! Must rebalance almost ALL data!
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

```javascript
function getPartitionConsistent(key, nodes) {
    const keyHash = hash(key);
    
    // Find first node with hash ≥ keyHash (clockwise)
    const sortedNodes = nodes.sort((a, b) => hash(a) - hash(b));
    
    for (const node of sortedNodes) {
        if (hash(node) >= keyHash) {
            return node;
        }
    }
    
    // Wrap around: return first node
    return nodes[0];
}
```

**Adding a Node**:

```
Before (3 nodes):
  Node A: 0 to 1,000,000,000
  Node B: 1,000,000,001 to 2,000,000,000
  Node C: 2,000,000,001 to 4,294,967,295

After adding Node D at position 1,500,000,000:
  Node A: 0 to 1,000,000,000 (unchanged)
  Node D: 1,000,000,001 to 1,500,000,000 (NEW!)
  Node B: 1,500,000,001 to 2,000,000,000 (shrank)
  Node C: 2,000,000,001 to 4,294,967,295 (unchanged)

Only data in range 1,000,000,001 to 1,500,000,000 moves!
(roughly 25% of data, not 75%!)
```

**Virtual Nodes (vnodes)**:

Problem: Nodes might not be evenly distributed on ring.

Solution: Each physical node gets **multiple** positions on ring.

```
Physical Setup: 3 servers

Virtual nodes: Each server has 256 positions on ring
  Server 1: vnodes 0, 3, 6, 9, ..., 765
  Server 2: vnodes 1, 4, 7, 10, ..., 766
  Server 3: vnodes 2, 5, 8, 11, ..., 767

Total: 768 virtual nodes on ring (256 × 3)

Benefit: Much more even distribution!
```

**Why Virtual Nodes Matter**:

```
Without vnodes (3 physical nodes):
  Node A gets: 45% of data 😞
  Node B gets: 32% of data
  Node C gets: 23% of data

With 256 vnodes per node:
  Node A gets: 33.2% of data ✓
  Node B gets: 33.5% of data ✓
  Node C gets: 33.3% of data ✓

Much more balanced!
```

**Used by**:
- **DynamoDB**: Consistent hashing with vnodes
- **Cassandra**: Uses vnodes (256 per node by default)
- **Riak**: Uses vnodes (64 per node)

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

```javascript
// Elasticsearch: 3 shards, searching for "python tutorial"
GET /articles/_search
{
  "query": {
    "match": {"content": "python tutorial"}
  }
}

// Process:
// 1. Query sent to all 3 shards
// 2. Each shard searches its local index
// 3. Results merged and scored
// 4. Top results returned

// If you have 100 shards → 100 queries per search! 💥
```

**The Cost of Scatter-Gather at Scale**:

Let's calculate the real cost:

```
Small setup (10 shards):
  1 user search query = 10 shard queries
  100 searches/sec = 1,000 queries/sec at database layer
  Manageable!

Medium setup (100 shards):
  1 user search query = 100 shard queries
  100 searches/sec = 10,000 queries/sec at database layer
  CPU usage: High but tolerable

Large setup (1000 shards):
  1 user search query = 1,000 shard queries
  100 searches/sec = 100,000 queries/sec at database layer
  CPU usage: 🔥 MASSIVE 🔥
  
  Cost analysis:
    Each shard query: 5ms CPU time
    100,000 queries/sec × 5ms = 500 CPU-seconds per second
    Need 500 CPU cores just for searching!
```

**Why Companies Still Do This**:

Despite the cost, scatter-gather is necessary for:
1. **Full-text search** (Elasticsearch, Solr)
2. **Analytics** (count, sum, average across all data)
3. **Global queries** (leaderboards, trending topics)

**Optimization Strategies**:

```javascript
// Strategy 1: Limit scope with filters
// Bad: Search across all 1000 shards
GET /posts/_search {"query": {"match": {"content": "python"}}}

// Good: Search only recent data (maybe 10 shards)
GET /posts/_search {
  "query": {
    "bool": {
      "must": [{"match": {"content": "python"}}],
      "filter": [{"range": {"date": {"gte": "2024-01-01"}}}]
    }
  }
}

// Strategy 2: Cache popular queries
// Cache key: "search:python tutorial"
// TTL: 5 minutes
// Hit rate: 80% for popular queries
// Result: 80% fewer scatter-gather queries!

// Strategy 3: Use routing
// Route queries to specific shards when possible
GET /posts/_search?routing=user_123
// Only queries shards where user_123's data lives
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

```javascript
// DynamoDB table partitioned by user_id
// Global Secondary Index on email

const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();

// Write user
await dynamodb.put({
    TableName: 'Users',
    Item: {
        user_id: '12345',
        name: 'Alice',
        email: 'alice@example.com'
    }
}).promise();

// Query by email (uses global index)
const response = await dynamodb.query({
    TableName: 'Users',
    IndexName: 'email-index',  // Uses global secondary index
    KeyConditionExpression: 'email = :email',
    ExpressionAttributeValues: {
        ':email': 'alice@example.com'
    }
}).promise();

// Behind the scenes:
// 1. Query goes to one partition of the email-index
// 2. Index returns user_id=12345
// 3. Fetch from main table using user_id
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

```javascript
// 3 physical nodes:
Node A: vnodes 0-63
Node B: vnodes 64-127
Node C: vnodes 128-191

// Total: 192 partitions

// Add Node D:
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

```javascript
// HBase example: Create table with pre-split regions
create 'users', 'info', SPLITS => ['100', '200', '300', '400']

// Creates 5 regions:
// Region 1: '' to '100'
// Region 2: '100' to '200'
// Region 3: '200' to '300'
// Region 4: '300' to '400'
// Region 5: '400' to ''
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

**Real-World Example - Riak**:

Riak uses 64 partitions (called vnodes) by default per physical node.

```javascript
// 3 physical nodes:
Node A: vnodes 0-63
Node B: vnodes 64-127
Node C: vnodes 128-191

// Total: 192 partitions

// Add Node D:
Node A: vnodes 0-47       (kept 48, gave 16)
Node B: vnodes 64-111     (kept 48, gave 16)
Node C: vnodes 128-175    (kept 48, gave 16)
Node D: vnodes 48-63, 112-127, 176-191  (received 48)
```

**Why this works well**:
- Adding a node only requires 25% of data to move (48 vnodes out of 192)
- Each existing node gives up ~25% of its data
- Load distribution is automatic and fair

### The Resharding Process in Detail

**Scenario**: Your sharded system needs to grow from 2 shards to 4 shards

**Step-by-Step Resharding (Vitess-style approach)**:

```
Current state (2 shards):
  Shard 1: Users A-M (600 GB)
  Shard 2: Users N-Z (800 GB)

Problem: Shard 2 is getting overloaded!

Step 1: Create NEW shard configuration (4 shards)
  Shard 1-new: Users A-F
  Shard 2-new: Users G-M  
  Shard 3-new: Users N-T
  Shard 4-new: Users U-Z

Step 2: Start background copy
  While OLD shards still serve traffic:
    Copy A-F from Shard 1 → Shard 1-new
    Copy G-M from Shard 1 → Shard 2-new
    Copy N-T from Shard 2 → Shard 3-new
    Copy U-Z from Shard 2 → Shard 4-new

Step 3: Keep a change log
  All writes during copy are logged:
    User "alice" updates profile → Log to replay later
    User "zach" posts message → Log to replay later

Step 4: Catch up phase
  When initial copy is done (maybe hours later):
    Replay logged changes to new shards
    Keep doing this until lag is minimal (< 1 second)

Step 5: Switch traffic (CRITICAL MOMENT!)
  5a. Stop writes for 1-2 seconds
  5b. Replay final changes to new shards
  5c. Update routing table to point to new shards
  5d. Resume writes (now going to new shards)

Step 6: Verify
  Monitor for errors
  Check data consistency
  Compare row counts between old and new

Step 7: Clean up
  After 1-2 days of new shards being stable:
    Delete old Shard 1 and Shard 2
```

**Why This Approach Works**:

1. **Minimal Downtime**: Only 1-2 seconds of write blocking
2. **Safe**: Old shards remain available during transition
3. **Reversible**: Can switch back to old shards if problems
4. **Gradual**: Copying happens in background

**Trade-off**: The process can take DAYS for large datasets (which is fine! Better slow and safe than fast and broken)

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

```javascript
const cassandra = require('cassandra-driver');

// Client connects to any node
const client = new cassandra.Client({
    contactPoints: ['node1.example.com']
});
await client.connect();

// Query
const result = await client.execute(
    "SELECT * FROM users WHERE user_id = 12345"
);

// Behind the scenes:
// 1. node1 receives request
// 2. node1 knows user_id=12345 is on node4 (via gossip)
// 3. node1 forwards to node4
// 4. node4 returns data to node1
// 5. node1 returns to client

// Future requests: Client learns node4 has that data,
// connects directly to node4 (optimization)
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
```javascript
const zookeeper = require('node-zookeeper-client');

// Node 1 starts up
await zk.create("/nodes/node1", Buffer.from("alive"), 
    zookeeper.CreateMode.EPHEMERAL);
await zk.create("/partitions/p1", Buffer.from("node1"));
await zk.create("/partitions/p2", Buffer.from("node1"));
```

2. **Router/Client watches ZooKeeper**
```javascript
function onPartitionChange(partitions) {
    // Update local routing table
    const routingTable = buildRoutingTable(partitions);
}

zk.getChildren("/partitions", onPartitionChange, (error, children) => {
    // Watch callback
});
```

3. **Partition moves**
```javascript
// Rebalancing: Move partition 5 from node2 to node4
await zk.setData("/partitions/p5", Buffer.from("node4"));
// ↓
// ZooKeeper notifies all watchers
// ↓  
// Routers/clients update their routing tables
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
**Why Multi-Level Partitioning?**

Let's understand Uber's reasoning:

```
Challenge 1: Cities have different sizes
  San Francisco: 100,000 trips/day
  New York: 500,000 trips/day  
  Small town: 100 trips/day

Challenge 2: Cities have different patterns
  Las Vegas: Surge on weekends
  Financial districts: Surge on weekdays
  
Challenge 3: Different latency requirements
  Premium cities (SF, NYC): Need < 100ms response
  Smaller cities: Can tolerate 200ms

Solution: City-level partitioning!

San Francisco region:
  Shard SF-1: SF users (0-999999)
  Shard SF-2: SF users (1000000-1999999)
  Shard SF-3: SF trips
  Shard SF-4: SF payments

New York region (bigger, needs more):
  Shard NY-1: NY users (0-499999)
  Shard NY-2: NY users (500000-999999)
  Shard NY-3: NY users (1000000-1499999)
  Shard NY-4: NY trips (0-999999)
  Shard NY-5: NY trips (1000000-1999999)
  Shard NY-6: NY payments

Small Town region:
  Shard ST-1: All data for 10 small cities
```

**Operational Benefits**:

```
Scenario: New York having issues

Impact: Only New York users affected
Action: Scale up NY shards independently
Cost: Only pay for NY capacity increase

Compare to single partition scheme:
Impact: Potentially all users affected  
Action: Scale entire cluster
Cost: Much more expensive
```

**Performance Benefits**:

```
Query: "Get my last trip in San Francisco"

Multi-level partitioning:
  1. Route to SF region (based on city in request)
  2. Route to trips shard
  3. Hash user_id to find exact partition
  4. ONE database query
  Latency: 50ms ✓

Simple hash partitioning:
  1. Hash user_id (but which city?)
  2. Might need to query multiple cities
  3. Scatter-gather if city not specified
  Latency: 200ms+ ✘
```

**Real Implementation Details**:

Uber's Schemaless system uses:
- **Primary key**: `(city, entity_type, entity_id)`
- **Storage**: MySQL shards (thousands of them)
- **Routing**: Custom routing layer
- **Replication**: 3x per shard (primary + 2 replicas)
- **Failover**: Automatic via Mesos/Kubernetes

**Lessons Learned**:

1. **Geography matters**: Partition by region when possible
2. **Entity types have different scales**: Don't mix hot and cold data
3. **Operational flexibility**: Being able to scale parts independently is valuable
4. **Cost optimization**: Only scale what needs scaling

```

**Why 1000 logical shards, not 100?**
- Can split physical servers easily
- Server 1 with shards [0-9] → Split to two servers: [0-4] and [5-9]

**User ID Generation**:
```javascript
// Instagram snowflake-style IDs
// |--- 41 bits: timestamp ---|-- 13 bits: shard_id --|-- 10 bits: sequence --|

const userId = (BigInt(timestamp) << 23n) | (BigInt(shardId) << 10n) | BigInt(sequence);
```

User ID encodes which shard it's on! Makes lookup O(1).

**Result**: Scaled to 1 billion users by 2018

### Example 2: Discord's Message Storage

**Challenge**: Billions of messages, need fast access to recent messages in each channel.

**Partitioning Strategy**: By channel_id and timestamp

```javascript
// Partition key: (channel_id, bucket)
// bucket = timestamp // BUCKET_SIZE

const messageId = (BigInt(timestamp) << 22n) | (BigInt(shardId) << 12n) | BigInt(sequence);
const partitionKey = [channelId, messageId / BigInt(BUCKET_SIZE)];
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
