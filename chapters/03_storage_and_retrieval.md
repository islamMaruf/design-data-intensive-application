# Chapter 3: Storage and Retrieval

## Introduction

As application developers, we rarely think about how databases store and retrieve data. We simply write data and trust the database to give it back when we ask. But understanding storage engines is crucial for choosing the right database and tuning performance.

On the most fundamental level, a database needs to do two things:
1. **Store data** when you give it to you
2. **Retrieve data** when you ask for it later

This seems simple, but the way databases accomplish these tasks has profound implications for performance, scalability, and the types of workloads they can handle efficiently.

### Why This Chapter Matters

Before diving into the technical details, let's understand **why** learning about storage engines is important:

**Real-World Impact:**
- Choosing the wrong storage engine can make your application 10x or 100x slower
- Understanding indexes can turn a 30-second query into a 30-millisecond query
- Different workloads require fundamentally different storage approaches

**Example: E-Commerce Analytics Performance**

Consider an e-commerce company running Black Friday sales analytics. The choice of storage engine has significant performance implications:

```
Scenario: Black Friday sales analytics

Using OLTP database for analytics:
- Query: "Show total sales by product category for last 24 hours"
- Scans through 100 million transaction records
- Takes 45 minutes to complete
- Locks tables, potentially slowing down customer purchases
- Result: Delayed insights, possible impact on sales operations

Using OLAP database for analytics:
- Same query on column-oriented storage
- Reads only the 3 columns needed (not all 50 columns)
- Compressed data requires less I/O
- Takes 3 seconds to complete
- Result: Real-time insights without impacting sales operations
```

### OLTP vs OLAP: Two Fundamentally Different Worlds

This chapter explores two competing paradigms in database design, each optimized for completely different use cases:

**OLTP: Online Transaction Processing**

**What It Is:** Databases designed for handling **lots of short, quick transactions**

**Think of it as:** The database powering your web application's day-to-day operations

**Examples:**
- MySQL, PostgreSQL, Oracle Database
- SQL Server, SQLite
- Your typical "application database"

**Use Cases:**
```
 User login and authentication
 E-commerce: Adding items to cart, processing orders
 Social media: Posting tweets, liking posts
 Banking: ATM withdrawals, account transfers
 Any "interactive" application where users expect instant responses
```

**Characteristics:**
```
┌─────────────────────────────────────────────┐
│  OLTP DATABASE CHARACTERISTICS              │
├─────────────────────────────────────────────┤
│  Query Pattern:                             │
│    - Small reads/writes (few rows)          │
│    - Indexed lookups (find by ID)           │
│    - CRUD operations                        │
│                                             │
│  Performance Goals:                         │
│    - Sub-millisecond latency                │
│    - High concurrency (1000s queries/sec)   │
│    - Consistency (ACID transactions)        │
│                                             │
│  Data Volume:                               │
│    - Working set: Recent data              │
│    - Total: GB to TB                        │
│                                             │
│  Real-World Example:                        │
│    SELECT * FROM users WHERE user_id = 123; │
│    (Returns 1 row instantly)                │
└─────────────────────────────────────────────┘
```

**OLAP: Online Analytical Processing**

**What It Is:** Databases designed for **analyzing large amounts of data** to extract insights

**Think of it as:** The database your data science team uses for reports and dashboards

**Examples:**
- Apache Druid, ClickHouse
- Amazon Redshift, Google BigQuery
- Snowflake, Apache Cassandra (for some workloads)

**Use Cases:**
```
 Business intelligence reports
 Data warehousing and analytics
 Trend analysis: "What were our top-selling products last quarter?"
 Aggregations: "Average order value by region"
 Machine learning feature extraction
```

**Characteristics:**
```
┌─────────────────────────────────────────────┐
│  OLAP DATABASE CHARACTERISTICS              │
├─────────────────────────────────────────────┤
│  Query Pattern:                             │
│    - Large scans (millions of rows)         │
│    - Aggregations (SUM, AVG, COUNT)         │
│    - Few writes, mostly reads               │
│                                             │
│  Performance Goals:                         │
│    - Throughput over latency                │
│    - Can take seconds or minutes            │
│    - Optimize for scanning efficiency       │
│                                             │
│  Data Volume:                               │
│    - Working set: All historical data       │
│    - Total: TB to PB                        │
│                                             │
│  Real-World Example:                        │
│    SELECT product_category,                 │
│           SUM(sales_amount)                 │
│    FROM orders                              │
│    WHERE order_date >= '2024-01-01'         │
│    GROUP BY product_category;               │
│    (Scans millions of rows, takes seconds)  │
└─────────────────────────────────────────────┘
```

### Why Different Storage Engines?

The fundamental difference between OLTP and OLAP requirements leads to completely different storage engine designs:

**OLTP Requirements:**
```javascript
// Typical OLTP operations
await db.query('UPDATE users SET last_login = NOW() WHERE user_id = ?', [123]);
await db.query('INSERT INTO orders (user_id, total) VALUES (?, ?)', [123, 99.99]);
await db.query('SELECT * FROM products WHERE product_id = ?', [456]);

// Needs:
// - Fast random access to individual rows
// - Efficient updates in place
// - Quick index lookups
// - Transaction support (ACID)
```

**OLAP Requirements:**
```javascript
// Typical OLAP operations
await db.query(`
  SELECT 
    DATE_TRUNC('month', order_date) as month,
    product_category,
    COUNT(*) as order_count,
    SUM(total_amount) as revenue,
    AVG(total_amount) as avg_order_value
  FROM orders
  WHERE order_date >= '2020-01-01'
  GROUP BY month, product_category
  ORDER BY month, revenue DESC
`);

// Needs:
// - Efficient sequential scans of large data
// - Good compression (reduce I/O)
// - Fast aggregations
// - Column-oriented storage (read only needed columns)
```

### Chapter Roadmap

This chapter explores the internal mechanics of storage engines, covering:

1. **Part 1-2: Building Blocks** - Starting simple, understanding the fundamentals
   - Hash indexes: The simplest approach
   - Why append-only logs are so common

2. **Part 3: LSM-Trees** - Write-optimized storage
   - How Cassandra, HBase, and RocksDB achieve high write throughput
   - Trade-offs: Write performance vs read performance

3. **Part 4: B-Trees** - Balanced storage
   - The workhorse behind MySQL, PostgreSQL
   - Why they've dominated for 40+ years

4. **Part 5-6: Advanced Topics**
   - Secondary indexes and their costs
   - Column-oriented storage for analytics (OLAP)
   - When to use what

**Learning Objectives:**

By the end of this chapter, you'll understand:
- Why databases make the trade-offs they do
- How to choose the right database for your workload
- What questions to ask when evaluating performance
- The difference between "fast writes" and "fast reads"

Let's start by building the world's simplest database to understand the fundamental concepts
## Part 1: Data Structures That Power Your Database

### The World's Simplest Database

**Why Start Simple?**

Before diving into complex database internals, let's build intuition by creating the simplest possible database. This isn't just an academic exercise - many production databases are built on these same fundamental principles
**Key Insight:** Even sophisticated databases like Cassandra and Bitcask use variations of what we're about to build.

**The Core Idea: Append-Only Log**

An **append-only log** is exactly what it sounds like:
- **Append**: Only add data to the end of a file
- **Never modify**: Don't update or delete existing data
- **Never seek**: Don't jump around in the file

**Why Append-Only?**

1. **Speed**: Sequential writes to disk are FAST
   ```
   Sequential writes: 100-200 MB/s (HDD)
   Random writes:     1-5 MB/s (HDD)
   
   That's 20-200x faster
   ```

2. **Simplicity**: No complex file management
3. **Crash safety**: Easier to recover from crashes
4. **Concurrency**: Multiple writers can append simultaneously

**Let's Build It:**

```bash
#!/bin/bash

# Store a key-value pair
# This function APPENDS a line to database.txt
# Format: key,value
db_set() {
    echo "$1,$2" >> database.txt
}

# Retrieve a value by key
# This function:
# 1. Searches for lines starting with the key
# 2. Removes the key prefix
# 3. Takes the LAST match (most recent value)
db_get() {
    grep "^$1," database.txt | sed -e "s/^$1,//" | tail -n 1
}
```

**Understanding Each Part:**

```bash
# db_set "user_123" "Alice"
#   ↓
# echo "user_123,Alice" >> database.txt
#                         ↑
#                    >> means "append to file"
#                    (not > which would overwrite)

# db_get "user_123"
#   ↓
# grep "^user_123," database.txt
#      ↑
#      ^ means "starts with" (so we don't match "another_user_123")
#   ↓
# sed -e "s/^user_123,//"
#        ↑
#        Remove the "user_123," prefix, leaving just the value
#   ↓  
# tail -n 1
#      ↑
#      Take only the last line (most recent value)
```

**Usage Example:**

```bash
# Let's simulate a simple user database
$ db_set "user_123" "Alice"
$ db_set "user_456" "Bob"
$ db_set "user_789" "Carol"

# Read some data
$ db_get "user_456"
Bob

# Update a user (creates a new entry, doesn't modify the old one)
$ db_set "user_123" "Alice Smith"  # Updated name

# The old entry is still there, but we get the latest value
$ db_get "user_123"
Alice Smith
```

**What's Happening on Disk:**

```
database.txt contents:
┌─────────────────────────────┐
│ user_123,Alice              │  ← First write (offset 0)
│ user_456,Bob                │  ← Second write (offset 19)
│ user_789,Carol              │  ← Third write (offset 35)
│ user_123,Alice Smith        │  ← Update (offset 53) - newest value
└─────────────────────────────┘

When we call db_get("user_123"):
1. Grep finds TWO lines: "user_123,Alice" and "user_123,Alice Smith"
2. tail -n 1 takes the LAST one: "user_123,Alice Smith"
3. We return: "Alice Smith"
```

**Key Insight: Updates Create New Entries**

Notice that when we "update" user_123, we don't modify the existing entry. Instead, we **append a new entry**. This is a fundamental property of append-only logs:

```
Traditional Update (modify in place):
 Find the old entry
 Seek to that position
 Overwrite with new data
 Slow! (random disk access)

Append-Only Update:
 Just append to the end
 Fast! (sequential disk access)
 Old values preserved (can implement time-travel queries!)
```

**Performance Characteristics:**

```
┌──────────────────────────────────────────┐
│  OPERATION  │  COMPLEXITY │  REAL TIME  │
├──────────────────────────────────────────┤
│  Write      │  O(1)       │  ~1ms       │
│             │  Constant!  │  (append)   │
│                                          │
│  Read       │  O(n)       │  ~1s per    │
│             │  Linear :(  │  100K rows  │
└──────────────────────────────────────────┘

where n = number of entries in database.txt
```

**Why Reads Are Slow:**

Imagine our database has grown to 1 million entries:

```bash
$ wc -l database.txt
1000000 database.txt

$ time db_get "user_999999"  # Last user in file
Alice

real    0m2.450s  # 2.5 seconds to scan 1 million lines
```

**The Problem:**
Every `db_get()` must scan the ENTIRE file from beginning to end. This is called a **full table scan** in database terminology.

**Real-World Analogy:**

Imagine looking up a word in a dictionary that has no alphabetical order - you'd have to read every single page from start to finish. That's what our database is doing
**This is an append-only log** - the fundamental building block of many production databases! But clearly, we need a better way to find data...

### The Problem: How to Find Data Efficiently?

Reading from our simple database requires scanning the entire file. For large datasets, this is unacceptably slow.

**Solution**: Add an **index** - an additional data structure that helps locate data quickly.

**Trade-off**: Indexes speed up reads but slow down writes (must update index on every write).

```
Write without index: O(1)
Write with index:    O(log n) or worse

Read without index:  O(n)
Read with index:     O(log n) or better
```

## Part 2: Hash Indexes

### Hash Map Index

The simplest indexing strategy: keep an in-memory hash map where keys map to byte offsets in the data file.

**Example:**

```
Data file (database.txt):
Offset 0:   user_123,Alice
Offset 16:  user_456,Bob
Offset 31:  user_123,Alice Updated

In-memory hash map:
{
  "user_123": 31,   # Points to latest value
  "user_456": 16
}
```

**Implementation:**

```javascript
const fs = require('fs');
const readline = require('readline');

class HashIndexDB {
  constructor(dataFile) {
    this.dataFile = dataFile;
    this.index = new Map();  // key -> byte offset
    this._loadIndex();
  }
  
  _loadIndex() {
    // Build index by scanning data file on startup
    if (!fs.existsSync(this.dataFile)) {
      fs.writeFileSync(this.dataFile, '');
      return;
    }
    
    const fileContent = fs.readFileSync(this.dataFile, 'utf8');
    const lines = fileContent.split('\n');
    let offset = 0;
    
    for (const line of lines) {
      if (line.trim()) {
        const key = line.split(',')[0];
        this.index.set(key, offset);
        offset += Buffer.byteLength(line + '\n', 'utf8');
      }
    }
  }
  
  set(key, value) {
    // Append to data file and update index
    const fd = fs.openSync(this.dataFile, 'a');
    const stat = fs.fstatSync(fd);
    const offset = stat.size;
    
    const data = `${key},${value}\n`;
    fs.writeSync(fd, data);
    fs.closeSync(fd);
    
    this.index.set(key, offset);
  }
  
  get(key) {
    // Look up in index, then read from file
    if (!this.index.has(key)) {
      return null;
    }
    
    const offset = this.index.get(key);
    const fd = fs.openSync(this.dataFile, 'r');
    const buffer = Buffer.alloc(1024);
    
    fs.readSync(fd, buffer, 0, 1024, offset);
    fs.closeSync(fd);
    
    const line = buffer.toString('utf8').split('\n')[0];
    const value = line.split(',')[1];
    return value;
  }
}

// Usage
const db = new HashIndexDB('database.txt');
db.set('user_123', 'Alice');
console.log(db.get('user_123'));  // Fast: O(1) lookup + O(1) disk seek
```

**Performance:**
- **Write**: O(1) - Append + hash map update
- **Read**: O(1) - Hash map lookup + single disk seek

### Real-World Example: Bitcask (Riak's Storage Engine)

Bitcask uses this exact approach! It's used in Riak, a distributed database.

**Bitcask Architecture:**

```
┌─────────────────────────────────────┐
│        In-Memory Hash Map           │
│  ┌──────────────────────────────┐   │
│  │ Key → (File ID, Offset, Size)│   │
│  └──────────────────────────────┘   │
└─────────────────────────────────────┘
                  │
                  │ Points to
                  ▼
┌─────────────────────────────────────┐
│          Data Files on Disk         │
│  ┌────────────┬────────────┬─────┐  │
│  │ File 1     │ File 2     │ ... │  │
│  │ (append-   │ (append-   │     │  │
│  │  only)     │  only)     │     │  │
│  └────────────┴────────────┴─────┘  │
└─────────────────────────────────────┘
```

### The Disk Space Problem: Compaction

Appending forever will eventually fill the disk. We need to reclaim space from deleted/updated records.

**Solution: Compaction**

```
Before compaction:
user_123,Alice
user_456,Bob
user_123,Alice Updated
user_456,Bob Updated
user_789,Carol
user_456,Bob Final

After compaction:
user_123,Alice Updated
user_456,Bob Final
user_789,Carol
```

**Compaction Process:**

```
┌─────────────────────────────────────────────┐
│  Step 1: Freeze current file (read-only)    │
│  Step 2: Create new file, write to it       │
│  Step 3: Background thread compacts old file│
│  Step 4: Switch reads to compacted file     │
│  Step 5: Delete old file                    │
└─────────────────────────────────────────────┘
```

**Implementation:**

```javascript
const fs = require('fs');

function compactSegment(oldFile, newFile) {
  // Keep only the latest value for each key
  const latest = new Map();  // key -> value
  
  // Scan old file, keeping track of latest values
  const fileContent = fs.readFileSync(oldFile, 'utf8');
  const lines = fileContent.split('\n');
  
  for (const line of lines) {
    if (line.trim()) {
      const [key, value] = line.trim().split(',');
      latest.set(key, value);
    }
  }
  
  // Write compacted data to new file
  const output = [];
  for (const [key, value] of latest) {
    output.push(`${key},${value}`);
  }
  
  fs.writeFileSync(newFile, output.join('\n') + '\n');
}
```

### Limitations of Hash Indexes

1. **Hash table must fit in memory**: Not suitable for very large datasets
2. **Range queries are inefficient**: Can't scan keys in sorted order
3. **No partial key matches**: Can't search for keys starting with "user_"

**When to use hash indexes:**
- Workload with many updates
- Small number of keys that fit in memory
- Lookups by exact key only

**Examples**: Session stores, caches

## Part 3: SSTables and LSM-Trees

### Sorted String Table (SSTable)

**Improvement over hash index**: Keep key-value pairs sorted by key.

**SSTable Format:**

```
Before (unsorted log):
user_456,Bob
user_123,Alice
user_789,Carol

After (SSTable - sorted by key):
user_123,Alice
user_456,Bob
user_789,Carol
```

**Advantages of Sorting:**

1. **Efficient merging**: Like merge sort, can merge multiple SSTables efficiently
2. **Sparse index**: Don't need to keep all keys in memory
3. **Range queries**: Can scan keys in order
4. **Compression**: Can compress blocks of sorted data

### Sparse Index Example

```
SSTable file (sorted):
┌──────────────────────────────┐
│ Offset 0:   user_100,Data    │
│ Offset 20:  user_150,Data    │
│ Offset 40:  user_200,Data    │
│ Offset 60:  user_250,Data    │
│ ...                          │
└──────────────────────────────┘

Sparse index (in memory):
{
  "user_100": 0,
  "user_200": 40,   # Only every Nth key
  "user_300": 80,
  ...
}

To find "user_175":
1. Look up in index: "user_100" (offset 0) < "user_175" < "user_200" (offset 40)
2. Seek to offset 0
3. Scan forward until "user_175" or next key > "user_175"
```

**Memory savings**: Index only stores 1 in 100 keys, using 1% of memory.

### LSM-Tree (Log-Structured Merge-Tree)

**What is an LSM-Tree?**

An LSM-tree is a data structure optimized for **high write throughput**. Instead of updating data in place (like B-trees), LSM-trees append all writes to an in-memory buffer, then periodically flush sorted data to disk.

**Historical Context: The Original LSM-Tree Paper**

LSM-trees were introduced in the early 1990s in a foundational research paper that remains highly influential today. The original paper is quite extensive and goes into arguably much more depth than most practitioners need for day-to-day work, but it laid the groundwork for modern NoSQL databases like Cassandra, HBase, and RocksDB.

As discussed in technical streams about this chapter, the LSM-tree paper is fascinating but very in-depth. Many engineers working with LSM-based databases haven't read the full original paper, yet they benefit from its principles every day. The paper provides deep theoretical analysis of performance characteristics, including detailed mathematical models of I/O costs and compaction strategies.

**Why LSM-Trees Were Invented: Understanding the Problem**

To truly appreciate LSM-trees, let's understand the problem they solve. Traditional B-trees (which we'll discuss more in Part 4) have limitations when dealing with high-volume sequential writes:

**Real-World Production Databases Using LSM-Trees:**

1. **Apache Cassandra** (Netflix, Apple, Instagram)
   - Handles 1+ million writes per second
   - Use case: Time-series data, sensor data, user activity logs
   - Why LSM: Extremely high write throughput needed

2. **RocksDB** (Facebook/Meta)
   - Powers Facebook's storage for graph data, messaging
   - Embedded database (like SQLite)
   - Use case: High write rate, frequent updates

3. **HBase** (Yahoo, Adobe)
   - Distributed database on HDFS
   - Use case: Large-scale data warehousing
   - Why LSM: Sequential writes optimize HDFS performance

4. **LevelDB** (Google)
   - Originally created for Chrome's IndexedDB
   - Lightweight, embeddable
   - Used in Bitcoin Core, Minecraft servers

5. **ScyllaDB** (Discord uses for messaging)
   - Compatible with Cassandra
   - Use case: 10+ million writes/sec per node

**Why These Databases Chose LSM-Trees:**

```
┌─────────────────────────────────────────┐
│  WORKLOAD CHARACTERISTIC │  LSM ADVANTAGE │
├─────────────────────────────────────────┤
│  Very high write rate    │  Sequential    │
│  (>100K writes/sec)      │  disk writes   │
│                         │  = 10-100x     │
│                         │  faster        │
│                                             │
│  Time-series data        │  Append-only   │
│  (logs, metrics)         │  perfect for   │
│                         │  historical    │
│                         │  data          │
│                                             │
│  Large dataset           │  Excellent     │
│  (TB to PB scale)        │  compression   │
│                         │  (80-90%)      │
│                                             │
│  Tolerate eventual       │  Can optimize  │
│  consistency             │  for writes    │
│                         │  over reads    │
└─────────────────────────────────────────┘
```

**Real-World Example: Instagram's Cassandra Deployment**

Instagram uses Cassandra (LSM-based) to store user feeds and activity:

```javascript
// Instagram's write pattern (simplified)
const cassandra = require('cassandra-driver');
const client = new cassandra.Client({ contactPoints: ['cassandra-cluster'] });

// Writing a user's post to their followers' feeds
async function distributePost(postId, authorId, followerIds) {
  const timestamp = Date.now();
  
  // This generates potentially MILLIONS of writes
  // (if author has millions of followers)
  for (const followerId of followerIds) {
    // Each write is instant (LSM handles high write volume)
    await client.execute(
      'INSERT INTO user_feed (user_id, post_id, timestamp, author_id) VALUES (?, ?, ?, ?)',
      [followerId, postId, timestamp, authorId]
    );
  }
}

// A celebrity with 10 million followers?
// That's 10 million INSERT operations
// LSM-trees handle this easily because writes are:
// 1. Batched in memory (memtable)
// 2. Written sequentially to disk
// 3. Merged asynchronously in background
```

**Performance Numbers (Real-World):**

```
Cassandra with LSM-Trees:
- Write throughput: 150,000 writes/second/node
- Read latency: p99 = 10ms (with proper tuning)
- Write latency: p99 = 2ms

Vs. MySQL with B-Trees on same hardware:
- Write throughput: 15,000 writes/second
- Both read and write latency higher

Trade-off: Cassandra reads can be slower because
           must check multiple SSTables
```

LSM-trees are the storage engine behind Cassandra, HBase, RocksDB, and LevelDB.

**Deep Dive: LSM-Tree Architecture**

Understanding LSM-trees requires understanding how they optimize for sequential writes while still maintaining reasonable read performance. The architecture is built on several key principles that differentiate it from traditional B-trees.

**The Core Concept: In-Memory Buffer + Sequential Writes**

The fundamental idea behind LSM-trees is to defer writes to disk and batch them efficiently. Here's how it works:

```
┌─────────────────────────────────────────────────────────┐
│             LSM-TREE COMPLETE ARCHITECTURE              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  IN MEMORY (RAM):                                       │
│  ┌───────────────────────────────────────────┐          │
│  │  C0 Tree (MemTable)                       │          │
│  │  • Sorted tree structure (Red-Black Tree) │          │
│  │  • Fast insertions: O(log n)              │          │
│  │  • Fast lookups: O(log n)                 │          │
│  │  • Size limit: ~100MB typical             │          │
│  └───────────────────────────────────────────┘          │
│         │                                               │
│         │ When full, flush to disk                      │
│         ↓                                               │
│                                                         │
│  ON DISK (SSD/HDD):                                     │
│  ┌───────────────────────────────────────────┐          │
│  │  C1 Tree (Level 0 SSTables)               │          │
│  │  • Multiple small sorted files            │          │
│  │  • Immutable (never modified)             │          │
│  │  • Size: ~100MB each                      │          │
│  └───────────────────────────────────────────┘          │
│         │                                               │
│         │ Background compaction merges files            │
│         ↓                                               │
│  ┌───────────────────────────────────────────┐          │
│  │  Level 1 SSTables                         │          │
│  │  • Fewer, larger merged files             │          │
│  │  • Size: ~1GB each                        │          │
│  └───────────────────────────────────────────┘          │
│         │                                               │
│         │ Continuous multi-layer compaction             │
│         ↓                                               │
│  ┌───────────────────────────────────────────┐          │
│  │  Level 2+ SSTables                        │          │
│  │  • Even larger, heavily compacted         │          │
│  │  • Size: 10GB+ each                       │          │
│  └───────────────────────────────────────────┘          │
│                                                         │
│  DURABILITY (WAL):                                      │
│  ┌───────────────────────────────────────────┐          │
│  │  Write-Ahead Log                          │          │
│  │  • Append-only log of all writes          │          │
│  │  • Used for crash recovery                │          │
│  │  • Replays to rebuild C0 tree             │          │
│  └───────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

**The C0/C1 Terminology**

In LSM-tree literature and the original paper, you'll see references to C0 and C1 trees:

- **C0 Tree**: The in-memory component (what we often call "MemTable")
- **C1 Tree**: The on-disk component (the SSTables at various levels)

This naming emphasizes that both are tree-like structures, just stored in different media.

**Why This Design is Brilliant for Writes**

Let's compare what happens during a write operation in B-trees vs LSM-trees:

```
┌──────────────────────────────────────────────────────┐
│        B-TREE WRITE OPERATION                        │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. Find correct leaf node:                          │
│     - Traverse from root                             │
│     - May require 3-4 disk seeks                     │
│     - Random I/O (slow on HDDs)                      │
│                                                      │
│  2. Read the page into memory:                       │
│     - Typical page size: 4KB or 8KB                  │
│     - Must read entire page                          │
│                                                      │
│  3. Modify the page:                                 │
│     - Insert new key-value                           │
│     - May need to split if page full                 │
│                                                      │
│  4. Write page back to disk:                         │
│     - Write full 4KB-8KB page                        │
│     - Even if only adding 100 bytes                  │
│     - Random I/O again                               │
│                                                      │
│  5. Update parent pointers if split occurred         │
│                                                      │
│  Total: 2-4 random disk I/Os per write               │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│        LSM-TREE WRITE OPERATION                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. Append to write-ahead log:                       │
│     - Sequential write to log file                   │
│     - Fast! (~1ms)                                   │
│                                                      │
│  2. Insert into C0 tree (in-memory):                 │
│     - Pure memory operation                          │
│     - Ultra fast (~microseconds)                     │
│                                                      │
│  3. Return success to client                         │
│                                                      │
│  Later (asynchronous, in background):                │
│  4. When C0 tree is full (~100MB):                   │
│     - Sort all entries                               │
│     - Write entire sorted run to disk                │
│     - One large sequential write                     │
│     - Fast! (100MB in ~1 second on SSD)              │
│                                                      │
│  Total: 1 sequential I/O per write (amortized)       │
│         Most writes never touch disk immediately!    │
└──────────────────────────────────────────────────────┘
```

**Real-World Impact: The Numbers**

Here's why this matters in production systems:

```javascript
// Scenario: Time-series data ingestion (logs, metrics, IoT sensors)

// B-Tree approach (traditional RDBMS):
const writeLatency_btree = 5;  // ms average per write (includes disk seeks)
const writesPerSecond_btree = 1000 / writeLatency_btree;  // = 200 writes/sec

console.log(`B-Tree: ${writesPerSecond_btree} writes/second`);
// Output: B-Tree: 200 writes/second

// LSM-Tree approach (Cassandra, HBase):
const writeLatency_lsm = 0.5;  // ms average (mostly RAM + WAL)
const writesPerSecond_lsm = 1000 / writeLatency_lsm;  // = 2000 writes/sec

console.log(`LSM-Tree: ${writesPerSecond_lsm} writes/second`);
// Output: LSM-Tree: 2000 writes/second

// LSM is 10x faster for sequential write workloads
```

**The Multi-Layer Compaction Strategy**

As mentioned in discussions about LSM-trees, one of the most fascinating aspects is the multi-layer compaction strategy. This isn't just a single level of merge; it's a hierarchical system:

```
Layer 0 (L0): Newly flushed from memtable
  • Files: sstable_1.db (100MB), sstable_2.db (100MB), ...
  • Characteristics: May have overlapping key ranges
  • Read check: Must check ALL L0 files for a key

Layer 1 (L1): First compaction level
  • Files: sstable_L1_1.db (1GB), sstable_L1_2.db (1GB), ...
  • Characteristics: Non-overlapping key ranges
  • Read check: Binary search to find correct file

Layer 2 (L2): Second compaction level
  • Files: sstable_L2_1.db (10GB), sstable_L2_2.db (10GB), ...
  • Characteristics: Non-overlapping, heavily compacted
  • Read check: Usually find key here if not in L0/L1

Each layer is typically 10x larger than the previous layer.
```

**How Compaction Works Across Layers:**

```javascript
// Simplified compaction algorithm
async function compactLayers(level) {
  console.log(`Starting compaction for Level ${level}...`);
  
  // 1. Select files to compact from this level
  const filesToCompact = selectCompactionCandidates(level);
  console.log(`Selected ${filesToCompact.length} files for compaction`);
  
  // 2. Identify overlapping files in next level
  const nextLevelOverlaps = findOverlappingFiles(filesToCompact, level + 1);
  console.log(`Found ${nextLevelOverlaps.length} overlapping files in Level ${level + 1}`);
  
  // 3. Merge all these files
  const mergedData = await mergeSortedFiles([
    ...filesToCompact,
    ...nextLevelOverlaps
  ]);
  
  // 4. Remove duplicates (keep only latest version of each key)
  const deduplicated = removeDuplicateKeys(mergedData);
  console.log(`Removed ${mergedData.length - deduplicated.length} duplicate entries`);
  
  // 5. Write new compacted files to next level
  const newFiles = await writeSSTables(deduplicated, level + 1);
  console.log(`Created ${newFiles.length} new files at Level ${level + 1}`);
  
  // 6. Delete old files
  await deleteFiles([...filesToCompact, ...nextLevelOverlaps]);
  
  console.log(`Compaction complete! Freed ${calculateFreedSpace()} bytes`);
}

// Example output:
// Starting compaction for Level 0...
// Selected 10 files for compaction
// Found 2 overlapping files in Level 1
// Removed 45,234 duplicate entries
// Created 1 new files at Level 1
// Compaction complete! Freed 234,567,890 bytes
```

**Visualizing LSM-Tree Operations**

One of the challenges with understanding LSM-trees is visualizing how data flows through the system. Let's trace a specific key through its lifecycle:

```
Time T0: User writes key "user_123" → "Alice"
─────────────────────────────────────────────
C0 Tree (RAM):
  user_123 → Alice

Time T1: C0 tree flushes to disk
─────────────────────────────────────────────
Level 0:
  sstable_001.db: {..., user_123 → Alice, ...}

Time T2: User updates key "user_123" → "Alice Smith"
─────────────────────────────────────────────
C0 Tree (RAM):
  user_123 → Alice Smith

Level 0:
  sstable_001.db: {..., user_123 → Alice, ...}

Time T3: C0 tree flushes again
─────────────────────────────────────────────
Level 0:
  sstable_001.db: {..., user_123 → Alice, ...}
  sstable_002.db: {..., user_123 → Alice Smith, ...}  ← newer
Time T4: Level 0 → Level 1 compaction occurs
─────────────────────────────────────────────
Level 0:
  [empty after compaction]

Level 1:
  sstable_L1_001.db: {..., user_123 → Alice Smith, ...}  
  ← Only latest value kept
```

**Reading from LSM-Trees: The Trade-Off**

While writes are fast, reads require checking multiple locations:

```javascript
async function readKey(key) {
  console.log(`Reading key: ${key}`);
  
  // Step 1: Check C0 tree (in-memory)
  console.log('  Checking memtable...');
  if (memtable.has(key)) {
    console.log('  ✓ Found in memtable!');
    return memtable.get(key);
  }
  
  // Step 2: Check Level 0 SSTables (newest to oldest)
  console.log(`  Checking Level 0 (${level0Files.length} files)...`);
  for (const file of level0Files.reverse()) {
    const value = await searchSSTable(file, key);
    if (value !== null) {
      console.log(`  ✓ Found in ${file}!`);
      return value;
    }
  }
  
  // Step 3: Check Level 1 SSTables (use bloom filter to skip files)
  console.log(`  Checking Level 1 (${level1Files.length} files)...`);
  for (const file of level1Files) {
    if (bloomFilter.mightContain(file, key)) {
      const value = await searchSSTable(file, key);
      if (value !== null) {
        console.log(`  ✓ Found in ${file}!`);
        return value;
      }
    }
  }
  
  // Continue for Level 2, 3, etc...
  console.log('  ✗ Key not found');
  return null;
}

// Example output for a key in Level 1:
// Reading key: user_999
//   Checking memtable...
//   Checking Level 0 (5 files)...
//   Checking Level 1 (10 files)...
//   ✓ Found in sstable_L1_003.db
// Total checks: 1 + 5 + 3 = 9 file checks
```

This is why LSM-trees can have slower reads compared to B-trees - the read must potentially check multiple levels. However, optimizations like bloom filters and caching significantly improve read performance in practice.

**Architecture:**

```
                    Writes
                      │
                      ▼
              ┌──────────────┐
              │   MemTable   │  (In-memory, sorted)
              │  (Red-Black  │  
              │    Tree)     │
              └──────────────┘
                      │
                      │ When full, flush to disk
                      ▼
              ┌──────────────┐
              │  SSTable L0  │  (On disk, immutable)
              └──────────────┘
                      │
                      │ Background compaction
                      ▼
              ┌──────────────┐
              │  SSTable L1  │  (Larger, merged files)
              └──────────────┘
                      │
                      ▼
              ┌──────────────┐
              │  SSTable L2  │  (Even larger files)
              └──────────────┘
```

**Write Path:**

```javascript
const fs = require('fs');

class LSMTree {
  constructor() {
    this.memtable = new Map();  // In-memory sorted tree
    this.sstables = [];  // List of SSTable files on disk
    this.wal = fs.openSync('write_ahead_log.txt', 'a');  // For crash recovery
  }
  
  write(key, value) {
    // 1. Append to write-ahead log (for durability)
    fs.writeSync(this.wal, `${key},${value}\n`);
    fs.fsyncSync(this.wal);
    
    // 2. Write to memtable
    this.memtable.set(key, value);
    
    // 3. If memtable is full, flush to disk
    if (this.memtable.size > 10000) {
      this._flushMemtable();
    }
  }
  
  _flushMemtable() {
    // Convert memtable to immutable SSTable on disk
    const filename = `sstable_${Date.now()}.db`;
    
    // Sort keys and write to file
    const sortedKeys = Array.from(this.memtable.keys()).sort();
    const lines = sortedKeys.map(key => `${key},${this.memtable.get(key)}`);
    fs.writeFileSync(filename, lines.join('\n') + '\n');
    
    this.sstables.push(filename);
    this.memtable.clear();
    
    // Trigger background compaction if needed
    if (this.sstables.length > 10) {
      this._compactSSTables();
    }
  }
}
```

**Read Path:**

```javascript
read(key) {
  // 1. Check memtable first (most recent data)
  if (this.memtable.has(key)) {
    return this.memtable.get(key);
  }
  
  // 2. Check SSTables from newest to oldest
  for (let i = this.sstables.length - 1; i >= 0; i--) {
    const value = this._searchSSTable(this.sstables[i], key);
    if (value !== null) {
      return value;
    }
  }
  
  return null;  // Key not found
}

_searchSSTable(sstableFile, key) {
  // Binary search within SSTable
  const fileContent = fs.readFileSync(sstableFile, 'utf8');
  const lines = fileContent.split('\n');
  
  // Use sparse index + binary search (simplified here)
  for (const line of lines) {
    if (!line.trim()) continue;
    
    const [k, v] = line.trim().split(',');
    if (k === key) {
      return v;
    }
    if (k > key) {
      break;  // Key not in this SSTable
    }
  }
  return null;
}
```

### Compaction Strategies

**Why Compaction is Necessary:**

As writes accumulate, we end up with many SSTable files. This creates problems:

1. **Read performance degrades**: Must check many files to find a key
2. **Disk space waste**: Multiple versions of same key exist
3. **Deleted data lingers**: Tombstones (deletion markers) take space

**Example Problem:**

```javascript
// User updates their email 100 times
for (let i = 0; i < 100; i++) {
  await db.update('user_123', { email: `email${i}@example.com` });
}

// Now we have 100 entries for user_123 across multiple SSTables
// Only the latest matters, but reads must check all 100.
```

**Compaction solves this by:**
- Merging multiple SSTables into fewer, larger ones
- Keeping only the latest value for each key
- Removing deleted entries (tombstones)

**Two Main Strategies:**

---

**1. Size-Tiered Compaction (STCS)** - Used by Cassandra (default)

**How it works:**
- Wait until you have N SSTables of similar size
- Merge them into one larger SSTable
- Repeat at each level

**Visual Example:**

```
Time 0: Writes create small SSTables
┌───────────────────────────────────────────────────┐
│ L0:  [10MB]  [10MB]  [10MB]  [10MB]  <- 4 files, time to merge!│
│ L1:  (empty)                                              │
└───────────────────────────────────────────────────┘

Time 1: After first compaction
┌───────────────────────────────────────────────────┐
│ L0:  (empty, accepting new writes)                       │
│ L1:  [40MB]  <- Merged from 4 x 10MB files              │
└───────────────────────────────────────────────────┘

Time 2: More writes, multiple L1 files
┌───────────────────────────────────────────────────┐
│ L0:  [10MB]  [10MB]                                       │
│ L1:  [40MB]  [40MB]  [40MB]  [40MB]  <- Time to merge!  │
│ L2:  (empty)                                             │
└───────────────────────────────────────────────────┘

Time 3: After L1 compaction
┌───────────────────────────────────────────────────┐
│ L0:  [10MB]  [10MB]  [10MB]  [10MB]                      │
│ L1:  (empty, recently compacted)                         │
│ L2:  [160MB]  <- Merged from 4 x 40MB files             │
└───────────────────────────────────────────────────┘
```

**Size-Tiered Pros:**
- Simple to implement
- Good write performance
- Works well for time-series data

**Size-Tiered Cons:**
- **Space amplification**: Need 2x temporary space during compaction
- **Read amplification**: May need to check many SSTables
- **Write amplification**: Data rewritten multiple times

**When Cassandra uses it:**
- Time-series workloads (logs, metrics)
- Write-heavy applications
- When disk space is not constrained

---

**2. Leveled Compaction (LCS)** - Used by RocksDB, LevelDB

**How it works:**
- Each level has fixed-size SSTables (e.g., 10MB)
- Level N has 10x more data than Level N-1
- SSTables at each level have non-overlapping key ranges
- When a level is full, merge overlapping SSTables to next level

**Visual Example:**

```
Key ranges: [0-100] [100-200] [200-300] [300-400]

┌──────────────────────────────────────────────────┐
│ L0: [2MB, keys 0-400]   (overlapping OK at L0)      │
│     [2MB, keys 50-350]                              │
│     [2MB, keys 100-300]                             │
├──────────────────────────────────────────────────┤
│ L1: [10MB, 0-100]  [10MB, 100-200]  [10MB, 200-300] │
│     [10MB, 300-400]     (non-overlapping!)          │
├──────────────────────────────────────────────────┤
│ L2: [100MB, 0-100]  [100MB, 100-200]               │
│     [100MB, 200-300]  [100MB, 300-400]             │
└──────────────────────────────────────────────────┘
```

**Compaction Process:**

```
Step 1: L0 file needs compaction
  Input:  L0 [2MB, keys 50-250]
  
  Find overlapping files in L1:
    L1 [10MB, 0-100]     ← overlaps
    L1 [10MB, 100-200]   ← overlaps
    L1 [10MB, 200-300]   ← overlaps
Step 2: Merge L0 + overlapping L1 files
  Inputs:  1 x 2MB (L0) + 3 x 10MB (L1) = 32MB
  Output:  3 x 10MB files in L1 (deduplicated, latest values)
  
  New L1:
    L1 [10MB, 0-100]     ← updated
    L1 [10MB, 100-200]   ← updated
    L1 [10MB, 200-300]   ← updated
    L1 [10MB, 300-400]   ← unchanged
```

**Leveled Compaction Pros:**
- **Better read performance**: Check fewer SSTables (key ranges don't overlap)
- **Less space amplification**: Only 10% overhead
- **More predictable**: No sudden large compactions

**Leveled Compaction Cons:**
- **Higher write amplification**: Rewrite data more frequently
- **More CPU intensive**: More frequent, smaller compactions

**When RocksDB uses it:**
- Read-heavy workloads
- Point lookups common (find specific key)
- Storage space constrained

---

**Comparison: Size-Tiered vs Leveled**

```
┌──────────────────────────────────────────────────────────────────┐
│  METRIC            │  SIZE-TIERED  │  LEVELED          │
├──────────────────────────────────────────────────────────────────┤
│  Write Throughput  │   Excellent │  ⚠️ Good          │
│  Read Performance  │  ⚠️ OK        │   Excellent     │
│  Space Overhead    │   2x (50%)   │   1.1x (10%)    │
│  Write Amplif.     │   Low        │   High          │
│  CPU Usage         │   Low        │  ⚠️ Higher       │
├──────────────────────────────────────────────────────────────────┤
│  Best For:         │  Time-series, │  User profiles,   │
│                   │  logs, heavy  │  catalogs, read-  │
│                   │  writes       │  heavy workloads  │
└──────────────────────────────────────────────────────────────────┘
```

**Real-World Configuration (RocksDB):**

```javascript
const rocksdb = require('rocksdb');

const db = rocksdb('mydb', {
  // Choose compaction strategy
  compaction: 'level',  // or 'universal' (size-tiered variant)
  
  // Level configuration
  writeBufferSize: 64 * 1024 * 1024,  // 64MB memtable
  level0FileNumCompactionTrigger: 4,   // Compact L0 when 4 files
  maxBytesForLevelBase: 256 * 1024 * 1024,  // 256MB at L1
  maxBytesForLevelMultiplier: 10,  // Each level 10x larger
  
  // Tune based on workload:
  // - Writes > Reads: increase writeBufferSize, use universal compaction
  // - Reads > Writes: use level compaction, more aggressive settings
});
```

**Size-Tiered Compaction** (Cassandra):
```
L0: [10MB]  [10MB]  [10MB]  [10MB]
              ↓ Merge when 4 files
L1:         [40MB]              [40MB]
              ↓ Merge when 4 files  
L2:               [160MB]
```

**Leveled Compaction** (RocksDB, LevelDB):
```
L0: [2MB] [2MB] [2MB]
     ↓ Merge overlapping ranges
L1: [10MB] [10MB] [10MB] [10MB]
     ↓ Merge overlapping ranges
L2: [100MB] [100MB] [100MB]
```

**Bloom Filters**: Optimization to avoid reading SSTables

```
Before checking SSTable:
  ┌─────────────────┐
  │  Bloom Filter   │  "Is key X in this SSTable?"
  │  (in memory)    │  → "Definitely not" (skip SSTable)
  └─────────────────┘  → "Maybe" (check SSTable)
```

**Bloom Filter Example:**

```javascript
const crypto = require('crypto');

class BloomFilter {
  constructor(size = 10000, hashCount = 3) {
    this.size = size;
    this.hashCount = hashCount;
    this.bitArray = new Uint8Array(Math.ceil(size / 8));
  }
  
  _hash(key, seed) {
    const hash = crypto.createHash('sha256');
    hash.update(key + seed);
    const digest = hash.digest();
    return digest.readUInt32BE(0) % this.size;
  }
  
  _setBit(index) {
    const byteIndex = Math.floor(index / 8);
    const bitIndex = index % 8;
    this.bitArray[byteIndex] |= (1 << bitIndex);
  }
  
  _getBit(index) {
    const byteIndex = Math.floor(index / 8);
    const bitIndex = index % 8;
    return (this.bitArray[byteIndex] & (1 << bitIndex)) !== 0;
  }
  
  add(key) {
    for (let i = 0; i < this.hashCount; i++) {
      const index = this._hash(key, i);
      this._setBit(index);
    }
  }
  
  mightContain(key) {
    for (let i = 0; i < this.hashCount; i++) {
      const index = this._hash(key, i);
      if (!this._getBit(index)) {
        return false;  // Definitely not present
      }
    }
    return true;  // Might be present (could be false positive)
  }
}
```

### LSM-Tree Performance

**Advantages:**
- **High write throughput**: Sequential writes to disk
- **Good compression**: Can compress SSTables effectively
- **Efficient storage**: Compaction removes dead data

**Disadvantages:**
- **Read amplification**: May need to check multiple SSTables
- **Compaction overhead**: Background CPU/disk usage
- **Write amplification**: Data written multiple times during compaction

**Write Amplification Example:**

```
1 Write request (1 MB)
↓
Write to MemTable + WAL (1 MB)
↓
Flush to L0 SSTable (1 MB)
↓
Compact to L1 (1 MB written again)
↓
Compact to L2 (1 MB written again)

Total disk writes: 4 MB
Write amplification: 4x
```

## Part 4: B-Trees

### What is a B-Tree?

**Definition:**
A B-tree is a **balanced tree data structure** that keeps data sorted and allows for efficient searches, insertions, and deletions. Unlike LSM-trees which append everything sequentially, B-trees modify data **in place**.

**Why "B-Tree"?**
The "B" stands for "balanced" - the tree automatically keeps itself balanced, ensuring all lookups take the same amount of time.

**Real-World Dominance:**

B-trees are the **most widely used indexing structure** in databases. They've been the workhorse of database systems for over 40 years
**Databases Using B-Trees:**

1. **PostgreSQL** (Primary index structure)
   - Used by: Instagram, Spotify, Netflix
   - Why: Excellent read performance, ACID guarantees

2. **MySQL/InnoDB** (Default storage engine)
   - Used by: Facebook, YouTube, Twitter
   - Why: Mature, proven, balanced read/write performance

3. **SQLite** (Most deployed database engine in the world)
   - Used by: Android, iOS, browsers, embedded devices
   - Billions of deployments
   - Why: Simple, reliable, no server needed

4. **MongoDB** (With WiredTiger storage engine)
   - Uses B-tree variant for indexes
   - Why: Efficient document lookups

5. **Oracle Database**
   - Enterprise standard for decades
   - Why: Rock-solid reliability, predictable performance

**Why B-Trees Dominate:**

```
┌────────────────────────────────────────────────┐
│  ADVANTAGE              │  WHY IT MATTERS          │
├────────────────────────────────────────────────┤
│  Predictable reads      │  Always same # of disk │
│                        │  seeks (tree depth)    │
│                                                        │
│  Efficient range scans  │  Keys stored in sorted │
│                        │  order at leaf level   │
│                                                        │
│  Mature & proven        │  40+ years in          │
│                        │  production            │
│                                                        │
│  Good for OLTP          │  Fast point lookups,   │
│                        │  transactions          │
└────────────────────────────────────────────────┘
```

**Real-World Performance (PostgreSQL with B-trees):**

```javascript
// Instagram's PostgreSQL setup
// - Billions of rows
// - B-tree indexes on user_id, post_id

const { Pool } = require('pg');
const pool = new Pool({ /* config */ });

// Fast point lookup (uses B-tree index)
await pool.query(
  'SELECT * FROM posts WHERE post_id = $1',  
  ['post_12345']
);
// Result: 1-2ms average latency
// B-tree: Root -> Interior -> Leaf (3 disk reads max)

// Fast range query (B-tree leaf pages are linked)
await pool.query(
  'SELECT * FROM posts WHERE user_id = $1 ORDER BY created_at DESC LIMIT 20',
  ['user_456']
);
// Result: 5-10ms average latency  
// B-tree: Navigate to start, scan 20 consecutive leaf pages
```

B-trees are the most widely used indexing structure. They power:
- **PostgreSQL, MySQL**: Primary indexes
- **SQLite**: Default storage
- **MongoDB**: Before WiredTiger, used B-trees

### B-Tree Structure

**The Core Concept: Pages**

B-trees organize data into fixed-size blocks called **pages** (also called **blocks** or **nodes**). Think of pages like pages in a phonebook:

- Each page is typically **4 KB** (same as OS page size)
- One page = one disk read/write operation
- Pages contain **sorted keys** and **pointers** to child pages

**Deep Understanding: Why Page-Sized Nodes?**

As discussed in technical deep-dives on this chapter, B-tree nodes are intentionally sized to match disk pages (typically 4KB). This isn't arbitrary - it's a fundamental optimization based on how storage hardware works:

```
┌──────────────────────────────────────────────────────┐
│       WHY B-TREE NODES = DISK PAGE SIZE              │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Disk/SSD reads data in page-sized chunks:           │
│    • Operating system page: 4KB                      │
│    • SSD page: 4KB-8KB                               │
│    • Disk sector cluster: 4KB                        │
│                                                      │
│  Reading less than a page?                           │
│    → Still reads full page (hardware limitation)     │
│                                                      │
│  Reading more than one page?                         │
│    → Multiple I/O operations required                │
│                                                      │
│  Optimal strategy:                                   │
│    → Make B-tree nodes exactly one page              │
│    → One disk I/O = one complete node                │
│    → Maximum efficiency!                             │
│                                                      │
│  Example:                                            │
│    4KB page with 8-byte keys + 8-byte pointers       │
│    = ~250 entries per node                           │
│    = 250-way branching factor                        │
│    = Shallow tree (fewer disk seeks)                 │
└──────────────────────────────────────────────────────┘
```

**The Write Challenge: Understanding B-Tree Write Inefficiencies**

One of the key insights from comparing LSM-trees and B-trees is understanding why B-trees can be less efficient for writes, especially in high-throughput scenarios. Let's explore this in detail:

```
┌──────────────────────────────────────────────────────┐
│     B-TREE WRITE INEFFICIENCY PROBLEMS               │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Problem 1: EMPTY SPACES                             │
│                                                      │
│  B-tree nodes maintain space for future inserts:     │
│                                                      │
│  Node capacity: 250 entries                          │
│  Current usage: [105, 210, 315, _, _, ..., _]        │
│                                   ↑    ↑         ↑    │
│                            Empty slots for future    │
│                                                      │
│  Why: To avoid frequent node splits                  │
│  Cost: Wasted space (~50% utilization typical)       │
│        More nodes = More disk reads                  │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Problem 2: IN-PLACE UPDATES ARE EXPENSIVE           │
│                                                      │
│  To update key 210:                                  │
│    1. Traverse tree to find leaf (random seeks)      │
│    2. Read entire 4KB page into memory               │
│    3. Modify one entry (~16 bytes changed)           │
│    4. Write entire 4KB page back to disk             │
│                                                      │
│  Write amplification: Changed 16 bytes               │
│                      Wrote 4096 bytes                │
│                      = 256x amplification!           │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Problem 3: RANDOM INSERTION ORDER                   │
│                                                      │
│  Inserting keys in random order:                     │
│    Insert 500 → Goes to Node X                       │
│    Insert 150 → Goes to Node Y (different location) │
│    Insert 750 → Goes to Node Z (different again)     │
│                                                      │
│  Each insert = random disk seek                      │
│  With HDDs: ~10ms per seek                           │
│  Max throughput: ~100 random inserts/second          │
│                                                      │
│  Contrast with sequential writes:                    │
│    Sequential: ~100 MB/sec                           │
│    Random: ~1 MB/sec                                 │
│    = 100x slower!                                    │
│                                                      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Problem 4: NODE SPLITS                              │
│                                                      │
│  When node is full:                                  │
│    1. Allocate new node                              │
│    2. Move half the entries                          │
│    3. Update parent pointers                         │
│    4. Write both nodes                               │
│    5. Write parent node                              │
│                                                      │
│  One insert can trigger 3+ disk writes!              │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Visualizing the Write Inefficiency**

Let's see this in action with a concrete example:

```javascript
// Simulating B-tree write overhead

// Scenario: High-frequency sensor data ingestion
const sensorReadings = [
  { timestamp: 1001, value: 23.5 },
  { timestamp: 998, value: 24.1 },   // Out of order
  { timestamp: 1002, value: 23.8 },
  { timestamp: 995, value: 24.5 },   // Far out of order
  // ... thousands more
];

// B-Tree write pattern:
class BTreeWrite {
  async insert(key, value) {
    // 1. Find the correct leaf page (random seek)
    const leafPage = await this.findLeafPage(key);
    console.log(`Seek to page ${leafPage.pageId} for key ${key}`);
    
    // 2. Read entire 4KB page
    const pageData = await disk.readPage(leafPage.pageId);  // 4KB read
    console.log(`Read 4KB page for 16-byte insert`);
    
    // 3. Check if page is full
    if (pageData.entryCount >= pageData.maxEntries) {
      console.log(`Page full! Triggering split...`);
      await this.splitPage(leafPage);  // Expensive
      // Split requires:
      // - Allocate new page
      // - Redistribute entries
      // - Update parent
      // - Write 3+ pages
    }
    
    // 4. Insert entry into page
    pageData.insert(key, value);
    
    // 5. Write entire 4KB page back (even though only 16 bytes changed)
    await disk.writePage(leafPage.pageId, pageData);  // 4KB write
    console.log(`Wrote 4KB page for 16-byte change`);
    
    // Total for one insert: 
    // - 1-3 random disk seeks (find path to leaf)
    // - 1-2 page reads (4KB each)
    // - 1-3 page writes (4KB each)
    // - Total time on HDD: ~10-30ms per insert
  }
}

// Throughput analysis:
console.log('B-Tree insert throughput:');
console.log('  Time per insert: 10-30ms');
console.log('  Max throughput: 33-100 inserts/second');
console.log('  For 10,000 sensors at 1 reading/sec:');
console.log('  Required: 10,000 inserts/sec');
console.log('  Can handle: 100 inserts/sec');
console.log('   B-tree is 100x too slow!');
```

**Why B-Trees Still Work Well for OLTP**

Despite these write inefficiencies, B-trees remain the dominant choice for general-purpose OLTP (Online Transaction Processing) databases. Here's why:

```
┌──────────────────────────────────────────────────────┐
│     WHY B-TREES EXCEL FOR OLTP DESPITE WRITES       │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. READS ARE PREDICTABLE                            │
│     • Always log₂₅₀(N) disk seeks                    │
│     • For 1 billion rows: 4 seeks max                │
│     • Very consistent latency                        │
│                                                      │
│  2. EXCELLENT CACHING                                │
│     • Hot nodes stay in memory                       │
│     • Root + top interior nodes almost always cached │
│     • Only leaf nodes hit disk                       │
│     • Effective depth: 1-2 disk reads                │
│                                                      │
│  3. EFFICIENT RANGE SCANS                            │
│     • Leaf nodes are linked                          │
│     • Sequential page reads                          │
│     • Perfect for "get last 100 orders"              │
│                                                      │
│  4. MATURE OPTIMIZATIONS                             │
│     • 40+ years of refinement                        │
│     • Buffer pool management                         │
│     • Write-ahead logging                            │
│     • Multi-version concurrency control (MVCC)       │
│                                                      │
│  5. TYPICAL OLTP WRITE PATTERNS                      │
│     • Not continuous streaming writes                │
│     • Bursty, interactive writes                     │
│     • Lots of reads mixed with writes                │
│     • Transactions need immediate consistency        │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**When B-Tree Write Inefficiency Doesn't Matter:**

```javascript
// Typical e-commerce application with B-tree (PostgreSQL)

// User browsing products (mostly reads):
await db.query('SELECT * FROM products WHERE category = $1', ['electronics']);
// B-tree: Fast! Root cached, 1-2 disk reads to leaf

await db.query('SELECT * FROM products WHERE price BETWEEN $1 AND $2', [100, 500]);
// B-tree: Fast! Range scan of linked leaf pages

// User adds to cart (occasional writes):
await db.query('INSERT INTO cart_items (user_id, product_id, quantity) VALUES ($1, $2, $3)',
               [userId, productId, 1]);
// B-tree: 10-30ms write latency - perfectly acceptable
// Ratio: 95% reads, 5% writes
// Conclusion: B-tree is optimal choice
```

**When B-Tree Write Inefficiency DOES Matter:**

```javascript
// Time-series metrics system (high write volume)

// Thousands of servers sending metrics every second:
for (const server of servers) {
  for (const metric of metrics) {
    await db.query(
      'INSERT INTO metrics (timestamp, server_id, metric_name, value) VALUES ($1, $2, $3, $4)',
      [Date.now(), server.id, metric.name, metric.value]
    );
  }
}

// 10,000 servers × 100 metrics/server = 1,000,000 inserts/second
// B-tree max: ~1000 inserts/second (with write buffering)
// Ratio: 99% writes, 1% reads
// Conclusion:  B-tree is wrong choice! Use LSM-tree instead
```

**The Empty Space Problem Visualized:**

One particularly interesting aspect mentioned in discussions is how B-trees maintain empty spaces. Let's visualize this:

```
┌───────────────────────────────────────────────────────┐
│  B-TREE NODE AFTER RANDOM INSERTIONS                  │
├───────────────────────────────────────────────────────┤
│                                                       │
│  Node capacity: 10 entries (simplified)               │
│  Current state after inserting: 50, 150, 250, 350     │
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ [50] [ ] [ ] [150] [ ] [250] [ ] [ ] [350] [ ] │  │
│  └─────────────────────────────────────────────────┘  │
│     ↑    ↑    ↑     ↑    ↑     ↑    ↑   ↑     ↑    ↑   │
│   Used Empty Empty Used Empty Used Empty Empty Used E │
│                                                       │
│  Utilization: 4/10 = 40%                              │
│  Wasted space: 60%                                    │
│                                                       │
│  Why leave empty spaces?                              │
│    • Future inserts (e.g., 75, 200) can fit          │
│    • Avoids expensive node splits                     │
│    • Trade-off: Space for speed                       │
│                                                       │
│  But this means:                                      │
│    • Tree is not fully compacted                      │
│    • More nodes needed for same data                  │
│    • More disk reads during traversal                 │
│                                                       │
├───────────────────────────────────────────────────────┤
│  CONTRAST: LSM-TREE SSTABLE (FULLY COMPACTED)        │
├───────────────────────────────────────────────────────┤
│                                                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │ [50][150][250][350][450][550][650][750][850][950]│  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  Utilization: 10/10 = 100%                            │
│  Wasted space: 0%                                     │
│                                                       │
│  Why no empty spaces?                                 │
│    • Immutable files (no future inserts)             │
│    • New data goes to new SSTable                     │
│    • Compaction removes duplicates                    │
│    • Optimized for sequential scans                   │
│                                                       │
└───────────────────────────────────────────────────────┘
```

**Real-World Impact of Empty Spaces:**

```javascript
// Database size comparison for same data

// B-Tree storage (PostgreSQL):
const bTreeStats = {
  logicalDataSize: 1000000,      // 1 MB of actual data
  indexOverhead: 0.3,             // 30% for pointers
  emptySpaceFactor: 0.5,          // 50% average fill factor
  physicalSize: 1000000 * 1.3 / 0.5
};

console.log('B-Tree physical size:', bTreeStats.physicalSize / 1000000, 'MB');
// Output: B-Tree physical size: 2.6 MB

// LSM-Tree storage (Cassandra):
const lsmStats = {
  logicalDataSize: 1000000,       // 1 MB of actual data
  compressionRatio: 0.3,          // 70% compression
  emptySpaceFactor: 0.95,         // 95% fill factor (tightly packed)
  physicalSize: 1000000 * 0.3 / 0.95
};

console.log('LSM physical size:', lsmStats.physicalSize / 1000000, 'MB');
// Output: LSM physical size: 0.32 MB

console.log('Storage efficiency: LSM is', 
            (bTreeStats.physicalSize / lsmStats.physicalSize).toFixed(1), 
            'x more space efficient');
// Output: Storage efficiency: LSM is 8.1x more space efficient
```

**Real-World Analogy: Library Card Catalog**

Imagine a library with millions of books:

```
Floor 1 (Root): Cabinet with drawers labeled "A-M" and "N-Z"
  ↓
Floor 2 (Interior): Drawers with labels "A-C", "D-F", "G-I", etc.
  ↓
Floor 3 (Leaves): Individual cards with book details
```

To find "Database Systems":
1. Floor 1: "Database" starts with D, goes to "A-M" drawer
2. Floor 2: Opens "D-F" drawer
3. Floor 3: Finds card for "Database Systems"

**That's exactly how a B-tree works!**

B-trees break the database into fixed-size **pages** (typically 4 KB), and organize them into a tree.

**Example B-Tree (simplified):**

```
                    Root Page
             [100, 200, 300, 400]
            /    |    |    |     \
           /     |    |    |      \
     [10,50] [110,150] [210,250] [310,350] [410,450]
        |      |         |         |         |
        ↓      ↓         ↓         ↓         ↓
     Data    Data      Data      Data      Data
```

**Detailed Structure Breakdown:**

```
┌────────────────────────────────────────────────────────┐
│  Root Page (In memory most of the time)             │
│  ┌──────────────────────────────────────┐  │
│  │ Keys:  [100, 200, 300, 400]             │  │
│  │ Ptrs:  [P1, P2, P3, P4, P5]            │  │
│  │                                        │  │
│  │ Meaning:                             │  │
│  │   P1: Keys < 100                     │  │
│  │   P2: Keys 100-200                   │  │
│  │   P3: Keys 200-300                   │  │
│  │   P4: Keys 300-400                   │  │
│  │   P5: Keys > 400                     │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
          │       │       │       │       │
          ↓       ↓       ↓       ↓       ↓
       ┌──────  (Interior Pages on Disk)  ──────┐
       │                                              │
┌──────├────────────────────────────────────────┤──────┐
│ Page P2│        Page P3         │ Page P4│
│ [110,   │        [210,            │ [310,   │
│  150]   │         250]            │  350]   │
└────────└─────────────────────────────────────────┘────────┘
    │   │       │   │                │   │
    ↓   ↓       ↓   ↓                ↓   ↓
┌──────── (Leaf Pages - contain actual data) ────────┐
│                                                        │
│  Leaf 1:  Leaf 2:  Leaf 3:  Leaf 4:  Leaf 5:  Leaf 6: │
│  [110:    [120:    [210:    [220:    [310:    [320:   │
│   Alice]  Bob]     Carol]  David]   Eve]     Frank]  │
│                                                        │
│  (Leaf pages are linked for range scans)              │
│  Leaf1 → Leaf2 → Leaf3 → Leaf4 → Leaf5 → Leaf6    │
└──────────────────────────────────────────────────────┘
```

**Properties:**
- Each page contains up to **N keys** and **N+1 child pointers**
- Keys are sorted within each page
- Tree is balanced: all leaf pages at same depth
- **Branching factor** (N): typically hundreds

**Why Hundreds of Keys Per Page?**

```
Page size: 4 KB = 4096 bytes

If each entry is:
  Key: 8 bytes (integer)
  Pointer: 8 bytes (page offset)
  = 16 bytes per entry

Entries per page: 4096 / 16 = 256 entries

With 256-way branching:
  Depth 1: 256 keys
  Depth 2: 256 × 256 = 65,536 keys
  Depth 3: 256 × 256 × 256 = 16,777,216 keys
  Depth 4: 256⁴ = 4,294,967,296 keys (4 billion!)

That's why lookups are so fast: Just 4 disk reads for billions of keys
```

**Search Example:** Find key 220

```
Step 1: Start at root
  [100, 200, 300, 400]
   220 is between 200 and 300
   → Follow 3rd pointer

Step 2: Interior page
  [210, 250]
   220 is between 210 and 250
   → Follow 2nd pointer

Step 3: Leaf page
  [215, 220, 230, 240]
   Found 220
```

**Search Performance**: O(log_N k) where N = branching factor, k = number of keys

With branching factor 500 and 4 levels, can store 500^4 = 62.5 billion keys
### B-Tree Operations

**Insert:**

```python
def insert(key, value):
    # 1. Find correct leaf page
    page = find_leaf_page(key)
    
    # 2. Insert into leaf page
    if page.has_space():
        page.insert(key, value)
    else:
        # Page is full: split into two pages
        split_page(page, key, value)

def split_page(page, key, value):
    """Split full page into two pages"""
    # 1. Create new page
    new_page = Page()
    
    # 2. Move half the keys to new page
    midpoint = len(page.keys) // 2
    new_page.keys = page.keys[midpoint:]
    page.keys = page.keys[:midpoint]
    
    # 3. Insert new key into appropriate page
    if key < page.keys[-1]:
        page.insert(key, value)
    else:
        new_page.insert(key, value)
    
    # 4. Update parent page to point to new page
    parent = page.parent
    parent.insert(midpoint_key, pointer_to_new_page)
    
    # 5. If parent is full, recursively split
    if parent.is_full():
        split_page(parent, ...)
```

**Visual: Page Split**

```
Before insert (page full):
┌─────────────────────────────────┐
│ [10, 20, 30, 40]  ← Full!       │
└─────────────────────────────────┘

Insert 25:

After split:
┌──────────────────┐  ┌──────────────────┐
│ [10, 20]         │  │ [25, 30, 40]     │
└──────────────────┘  └──────────────────┘
         ↑                      ↑
         └──────┬───────────────┘
                │
        ┌───────────────┐
        │ Parent: [25]  │
        └───────────────┘
```

**Delete:**

```javascript
function deleteKey(key) {
  // 1. Find and remove key from leaf page
  const page = findLeafPage(key);
  page.remove(key);
  
  // 2. If page is too empty, rebalance
  if (page.tooEmpty()) {
    // Option A: Borrow from sibling
    if (sibling.hasExtraKeys()) {
      borrowFromSibling(page, sibling);
    }
    // Option B: Merge with sibling
    else {
      mergeWithSibling(page, sibling);
    }
  }
}
```

### B-Tree vs LSM-Tree

| Aspect | B-Tree | LSM-Tree |
|--------|--------|----------|
| **Write** | Random writes (update in place) | Sequential writes (append-only) |
| **Read** | Predictable (single tree traversal) | Variable (check multiple levels) |
| **Space** | Can have fragmentation | Compact via compaction |
| **Concurrency** | Complex (page-level locking) | Simpler (immutable SSTables) |
| **Write amplification** | Lower | Higher (due to compaction) |
| **Read amplification** | Lower | Higher (multiple SSTables) |

**Rule of thumb:**
- **Write-heavy workload**: LSM-tree (Cassandra, HBase)
- **Read-heavy workload**: B-tree (PostgreSQL, MySQL)
- **Balanced workload**: Either (both have optimizations)

### Write-Ahead Log (WAL) in B-Trees

B-trees also use a write-ahead log for crash recovery:

```
Every write operation:
1. Append to WAL (sequential write)
2. Modify B-tree pages (random write)
3. Mark WAL entry as committed

On crash recovery:
1. Replay WAL from last checkpoint
2. Reconstruct B-tree to consistent state
```

**Example:**

```javascript
const fs = require('fs');

function insertWithWAL(key, value) {
  // 1. Write to WAL first
  const walEntry = `INSERT ${key} ${value}\n`;
  const walOffset = fs.appendFileSync('wal.log', walEntry);
  fs.fsyncSync('wal.log');  // Ensure on disk
  
  // 2. Modify B-tree
  const page = findLeafPage(key);
  page.insert(key, value);
  page.writeToDisk();
  
  // 3. Mark WAL entry as committed
  markCommitted(walOffset);
}
```

### Copy-on-Write (Alternative to WAL)

Some databases (LMDB, CouchDB) use copy-on-write instead of in-place updates:

```
Before write:
      Root
       │
       ▼
     Page A
       │
       ▼
     Page B (contains key X)

After write (modify key X):
      New Root
       │
       ▼
     New Page A
       │
       ▼
     New Page B (modified key X)

Old pages:
  Root → Page A → Page B  (can be garbage collected)
```

**Benefits:**
- No WAL needed (new tree atomic)
- Supports MVCC snapshots easily
- No overwrite (good for flash SSDs)

**Drawbacks:**
- Write amplification (copy entire path)
- Fragmentation over time

## Part 5: Comparing Storage Engines

### Benchmarking Example

```javascript
// Benchmark writes
function benchmarkWrites(db, numWrites = 100000) {
  const start = Date.now();
  
  for (let i = 0; i < numWrites; i++) {
    db.set(`key_${i}`, `value_${i}`);
  }
  
  const duration = (Date.now() - start) / 1000;
  const throughput = numWrites / duration;
  
  console.log(`Writes: ${throughput.toFixed(0)} ops/sec`);
  return throughput;
}

// Benchmark reads
function benchmarkReads(db, numReads = 100000) {
  const start = Date.now();
  
  for (let i = 0; i < numReads; i++) {
    db.get(`key_${Math.floor(Math.random() * numReads)}`);
  }
  
  const duration = (Date.now() - start) / 1000;
  const throughput = numReads / duration;
  
  console.log(`Reads: ${throughput.toFixed(0)} ops/sec`);
  return throughput;
}

// Results comparison
const results = {
  lsmTree: {
    writes: benchmarkWrites(lsmDB),  // ~100,000 ops/sec
    reads: benchmarkReads(lsmDB)     // ~20,000 ops/sec
  },
  bTree: {
    writes: benchmarkWrites(btreeDB), // ~30,000 ops/sec
    reads: benchmarkReads(btreeDB)    // ~50,000 ops/sec
  }
};
```

**Typical Results:**

```
LSM-Tree Database:
  Writes: 100,000 ops/sec   Excellent
  Reads:   20,000 ops/sec  ⚠️  Good but slower

B-Tree Database:
  Writes:  30,000 ops/sec  ⚠️  Good
  Reads:   50,000 ops/sec   Excellent
```

## Part 6: Other Indexing Structures

### Secondary Indexes

Primary index: unique, determines data location
Secondary index: non-unique, points to primary key or data

**Example:**

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,           -- Primary index (clustered)
    email VARCHAR(100),
    username VARCHAR(50),
    age INT
);

CREATE INDEX idx_email ON users(email);      -- Secondary index
CREATE INDEX idx_username ON users(username);-- Secondary index
```

**Secondary Index Structure (B-tree):**

```
idx_email tree:
  "alice@example.com" → [user_id: 1]
  "bob@example.com"   → [user_id: 2]
  "carol@example.com" → [user_id: 3]

idx_username tree:
  "alice"  → [user_id: 1]
  "bob"    → [user_id: 2]
  "carol"  → [user_id: 3]
```

**Clustered vs Non-Clustered:**

```
Clustered Index (data stored with index):
┌────────────────────────────────────┐
│  B-tree leaf pages                 │
│  ┌──────────────────────┐          │
│  │ user_id=1, data=...  │          │
│  │ user_id=2, data=...  │          │
│  └──────────────────────┘          │
└────────────────────────────────────┘

Non-Clustered Index (points to data):
┌────────────────────────────────────┐
│  B-tree leaf pages                 │
│  ┌──────────────────────┐          │
│  │ email → pointer to row│          │
│  └──────────────────────┘          │
└────────────────────────────────────┘
             │
             ▼
    ┌─────────────────┐
    │  Heap file      │
    │  (actual data)  │
    └─────────────────┘
```

### Multi-Column Indexes

**Concatenated Index:**

```sql
CREATE INDEX idx_city_age ON users(city, age);
```

**Index structure:**

```
("New York", 25) → [user_id: 1]
("New York", 30) → [user_id: 2]
("San Francisco", 25) → [user_id: 3]
```

**Query optimization:**

```sql
-- Uses index (city is first column)
SELECT * FROM users WHERE city = 'New York';

-- Uses index fully
SELECT * FROM users WHERE city = 'New York' AND age = 25;

-- Does NOT use index efficiently (age is second column)
SELECT * FROM users WHERE age = 25;
```

### Full-Text Search Indexes

For text search, specialized indexes are needed:

**Inverted Index Example:**

```
Documents:
Doc 1: "The quick brown fox"
Doc 2: "The lazy dog"
Doc 3: "Quick thinking"

Inverted Index:
"brown"    → [Doc 1]
"dog"      → [Doc 2]
"fox"      → [Doc 1]
"lazy"     → [Doc 2]
"quick"    → [Doc 1, Doc 3]  (case-insensitive)
"the"      → [Doc 1, Doc 2]
"thinking" → [Doc 3]
```

**PostgreSQL Full-Text Search:**

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    content_tsv TSVECTOR  -- Full-text search column
);

-- Create index
CREATE INDEX idx_fts ON documents USING GIN(content_tsv);

-- Populate tsvector
UPDATE documents 
SET content_tsv = to_tsvector('english', content);

-- Search query
SELECT * FROM documents 
WHERE content_tsv @@ to_tsquery('english', 'fox & quick');
```

### Keeping Everything in Memory

**In-Memory Databases** (Redis, Memcached, VoltDB):

**Advantages:**
- No disk I/O overhead
- Can use simpler data structures (no serialization)
- Predictable latency

**Disadvantages:**
- Limited by RAM size
- Data loss on crash (without persistence)
- More expensive per GB

**Redis Example:**

```javascript
const redis = require('redis');

const client = redis.createClient({
  host: 'localhost',
  port: 6379
});

await client.connect();

// Simple key-value
await client.set('user:123:name', 'Alice');
console.log(await client.get('user:123:name'));  // 'Alice'

// Data structures (not possible in disk-based KV stores)
await client.lPush('queue', ['task1', 'task2', 'task3']);
await client.rPop('queue');  // Process tasks from queue

await client.hSet('user:123', {
  name: 'Alice',
  age: '30',
  email: 'alice@example.com'
});
```

**Persistence Strategies:**

1. **Snapshots** (RDB): Periodically dump memory to disk
2. **Append-only log** (AOF): Log every write operation
3. **Hybrid**: Use both for faster recovery

## Part 7: Transaction Processing vs Analytics

### OLTP vs OLAP

**OLTP** (Online Transaction Processing):
- User-facing queries
- Small number of rows per query
- Random access, low latency
- Examples: E-commerce orders, bank transactions

**OLAP** (Online Analytical Processing):
- Business intelligence queries
- Aggregate large number of rows
- Sequential scans, high throughput
- Examples: Monthly revenue, user cohorts

**Comparison:**

| Property | OLTP | OLAP |
|----------|------|------|
| **Read pattern** | Small # of rows by key | Aggregate millions of rows |
| **Write pattern** | Random writes, user input | Bulk import (ETL) |
| **Used by** | End users (web app) | Analysts (internal) |
| **Data** | Latest state | History of events |
| **Size** | GB to TB | TB to PB |
| **Example** | Get product details | Total sales by region |

**OLTP Query:**

```sql
SELECT * FROM orders 
WHERE order_id = 12345;
```

**OLAP Query:**

```sql
SELECT 
    DATE_TRUNC('month', order_date) as month,
    product_category,
    SUM(revenue) as total_revenue
FROM orders
WHERE order_date >= '2023-01-01'
GROUP BY month, product_category
ORDER BY month, total_revenue DESC;
```

### Data Warehousing

**Architecture:**

```
┌──────────────────────────────────────────────┐
│         OLTP Databases (Sources)             │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│   │PostgreSQL│  │  MySQL   │  │ MongoDB  │  │
│   └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────────────────────────────┘
                    │
                    │ ETL (Extract, Transform, Load)
                    ▼
┌──────────────────────────────────────────────┐
│         Data Warehouse (Analytics)           │
│   ┌────────────────────────────────────┐     │
│   │  Redshift / BigQuery / Snowflake   │     │
│   │  (Optimized for OLAP)              │     │
│   └────────────────────────────────────┘     │
└──────────────────────────────────────────────┘
                    │
                    │ Business Intelligence Tools
                    ▼
┌──────────────────────────────────────────────┐
│      Tableau / Looker / Power BI             │
└──────────────────────────────────────────────┘
```

**Star Schema Example (Dimensional Modeling):**

```
              ┌──────────────┐
              │ Fact Table:  │
              │   Sales      │
              ├──────────────┤
              │ date_id      │───────┐
              │ product_id   │────┐  │
              │ store_id     │──┐ │  │
              │ customer_id  │─┐│ │  │
              │ quantity     ││ │ │  │
              │ revenue      ││ │ │  │
              └──────────────┘│ │ │  │
                              │ │ │  │
        ┌─────────────────────┘ │ │  │
        │         ┌─────────────┘ │  │
        │         │         ┌─────┘  │
        │         │         │     ┌──┘
        ▼         ▼         ▼     ▼
  ┌─────────┐ ┌────────┐ ┌──────┐ ┌───────┐
  │Customer │ │ Store  │ │Product│ │ Date  │
  │  Dim    │ │  Dim   │ │  Dim  │ │  Dim  │
  └─────────┘ └────────┘ └──────┘ └───────┘
  Dimension   Dimension  Dimension  Dimension
   Tables      Tables     Tables     Tables
```

**Fact table**: Measurements/events (sales, clicks, etc.)
**Dimension tables**: Context (who, what, where, when)

```sql
-- Example analytical query on star schema
SELECT 
    d.year,
    d.quarter,
    p.category,
    s.region,
    SUM(f.revenue) as total_revenue,
    COUNT(*) as num_transactions
FROM sales_fact f
JOIN date_dim d ON f.date_id = d.date_id
JOIN product_dim p ON f.product_id = p.product_id
JOIN store_dim s ON f.store_id = s.store_id
WHERE d.year = 2024 AND s.region = 'West'
GROUP BY d.year, d.quarter, p.category, s.region;
```

## Part 8: Column-Oriented Storage

### The Problem with Row-Oriented Storage

**Row-oriented** (OLTP databases):

```
Row 1: [date=2024-01-01, product=Laptop, store=NYC, revenue=1200]
Row 2: [date=2024-01-01, product=Mouse, store=LA, revenue=25]
Row 3: [date=2024-01-02, product=Laptop, store=SF, revenue=1200]
...
```

**OLAP query** (sum revenue by product):

```sql
SELECT product, SUM(revenue) FROM sales GROUP BY product;
```

**Problem**: Must read ALL columns from disk, even though only `product` and `revenue` are needed
### Column-Oriented Storage

**Column-oriented** (OLAP databases):

```
date column:     [2024-01-01, 2024-01-01, 2024-01-02, ...]
product column:  [Laptop, Mouse, Laptop, ...]
store column:    [NYC, LA, SF, ...]
revenue column:  [1200, 25, 1200, ...]
```

**Same query now reads only 2 columns**:
- product column
- revenue column

**Performance improvement**: 10x-100x for analytical queries
**Visual: Row vs Column Storage**

```
Row-Oriented:
┌──────────────────────────────────────────────────┐
│ Row 1: [date | product | store | revenue]        │
│ Row 2: [date | product | store | revenue]        │
│ Row 3: [date | product | store | revenue]        │
└──────────────────────────────────────────────────┘
     ↑     Read entire row even if only need 1 column

Column-Oriented:
┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ date column │ │product      │ │store column │ │revenue      │
│ [...]       │ │column [...]  │ │ [...]       │ │column [...]  │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
                      ↑                                  ↑
                   Read only needed columns
```

### Column Compression

Columns often have many repeated values, enabling aggressive compression.

**Example: `product` column**

```
Before compression:
[Laptop, Laptop, Mouse, Laptop, Keyboard, Mouse, Laptop, Laptop, ...]

After compression (run-length encoding):
[Laptop: 2, Mouse: 1, Laptop: 1, Keyboard: 1, Mouse: 1, Laptop: 2, ...]

Or bitmap encoding:
Laptop:    [1, 1, 0, 1, 0, 0, 1, 1, ...]
Mouse:     [0, 0, 1, 0, 0, 1, 0, 0, ...]
Keyboard:  [0, 0, 0, 0, 1, 0, 0, 0, ...]
```

**Bitmap Index Example:**

```
Product column (10 rows):
[Laptop, Mouse, Laptop, Keyboard, Laptop, Mouse, Laptop, Keyboard, Mouse, Laptop]

Bitmap for "Laptop":
[1, 0, 1, 0, 1, 0, 1, 0, 0, 1]

Bitmap for "Mouse":
[0, 1, 0, 0, 0, 1, 0, 0, 1, 0]

Query: COUNT(*) WHERE product = 'Laptop' OR product = 'Mouse'
Answer: Bitwise OR + count 1s
  [1,0,1,0,1,0,1,0,0,1]  (Laptop)
| [0,1,0,0,0,1,0,0,1,0]  (Mouse)
= [1,1,1,0,1,1,1,0,1,1]  → Count = 8
```

**Compression Ratios:**
- Typical compression: 10:1 to 100:1
- Enables fitting more data in memory
- Faster queries due to less I/O

### Vectorized Processing

**SIMD** (Single Instruction Multiple Data): Process multiple values in parallel.

```javascript
// Traditional (scalar) processing
function sumRevenueScalar(revenues) {
  let total = 0;
  for (const revenue of revenues) {
    total += revenue;
  }
  return total;
}

// Vectorized processing (using typed arrays for performance)
function sumRevenueVectorized(revenues) {
  // JavaScript typed arrays enable better CPU optimization
  // Modern JS engines can use SIMD internally
  const typedArray = new Float64Array(revenues);
  
  // Reduce is optimized by JIT compilers to use SIMD
  return typedArray.reduce((sum, val) => sum + val, 0);
}

// Performance difference:
// Scalar:     100M rows → 1.5 seconds
// Vectorized: 100M rows → 0.2 seconds (7.5x faster)
```

**Modern OLAP databases** (ClickHouse, DuckDB) exploit SIMD extensively.

### Writing to Column-Oriented Storage

**Problem**: Inserting one row requires updating ALL column files
**Solution: LSM-tree approach**

```
Recent writes (row-oriented, in memory):
┌──────────────────────────────────┐
│ MemTable (row format)            │
└──────────────────────────────────┘
         │
         ▼ Periodically flush
┌──────────────────────────────────┐
│ Column files on disk             │
│ [date col] [product col] [...]   │
└──────────────────────────────────┘
```

**Read query**:
1. Check MemTable (recent writes)
2. Read column files (older data)
3. Merge results

## Part 9: Aggregation: Data Cubes and Materialized Views

### Materialized Views

**Problem**: OLAP queries often aggregate the same data repeatedly.

**Solution**: Pre-compute and cache aggregations.

```sql
-- Regular view (computed on demand)
CREATE VIEW monthly_sales AS
SELECT 
    DATE_TRUNC('month', date) as month,
    product_category,
    SUM(revenue) as total_revenue
FROM sales
GROUP BY month, product_category;

-- Materialized view (pre-computed, stored on disk)
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT 
    DATE_TRUNC('month', date) as month,
    product_category,
    SUM(revenue) as total_revenue
FROM sales
GROUP BY month, product_category;

-- Refresh periodically
REFRESH MATERIALIZED VIEW monthly_sales;
```

**Query performance:**
- Without materialized view: Scan millions of rows → 10 seconds
- With materialized view: Read pre-computed results → 0.1 seconds

### Data Cubes (OLAP Cubes)

**Concept**: Pre-compute aggregates along multiple dimensions.

```
Example: Sales data with dimensions [Date, Product, Store]

1D aggregate: SUM(revenue) GROUP BY date
2D aggregate: SUM(revenue) GROUP BY date, product
3D aggregate: SUM(revenue) GROUP BY date, product, store
```

**Visualization (2D cube):**

```
         Product
        │ Laptop│ Mouse │Keyboard│ Total │
────────┼───────┼───────┼────────┼───────┤
Jan     │ 10K   │ 5K    │ 3K     │ 18K   │
Date Feb│ 12K   │ 6K    │ 4K     │ 22K   │
    Mar │ 15K   │ 7K    │ 5K     │ 27K   │
────────┼───────┼───────┼────────┼───────┤
Total   │ 37K   │ 18K   │ 12K    │ 67K   │
```

**All cells are pre-computed!**

**Trade-offs:**
- **Faster queries**: Direct lookup instead of aggregation
- **Storage overhead**: Store all combinations
- **Flexibility**: Can only query pre-computed dimensions

## Part 7: Choosing the Right Storage Engine - A Practical Guide

### Understanding Your Workload

The single most important factor in choosing a storage engine is understanding your **read/write pattern**. As extensively discussed in analysis of this chapter, LSM-trees and B-trees are fundamentally optimized for different kinds of read and write traffic.

Let's develop a systematic approach to making this decision:

**The Core Trade-Off: Reads vs Writes**

```
┌──────────────────────────────────────────────────────┐
│        THE FUNDAMENTAL STORAGE ENGINE TRADE-OFF      │
├──────────────────────────────────────────────────────┤
│                                                      │
│  B-TREES: Optimize for READ performance              │
│     Predictable read latency                       │
│     Efficient range scans                          │
│     Good for mixed workloads                       │
│     Write amplification (update 16B → write 4KB)   │
│     Random I/O for writes                          │
│     Empty spaces waste storage                     │
│                                                      │
│  LSM-TREES: Optimize for WRITE performance           │
│     Sequential writes (very fast)                  │
│     No write amplification                         │
│     Excellent compression                          │
│     Read amplification (check multiple SSTables)   │
│     Compaction overhead                            │
│     Variable read latency                          │
│                                                      │
│  The Decision:                                       │
│    High write volume + can tolerate read latency     │
│      → LSM-tree                                      │
│                                                      │
│    High read volume + need predictable latency       │
│      → B-tree                                        │
│                                                      │
│    Mixed workload + need transactions                │
│      → B-tree (usually)                              │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### Decision Framework

**Step 1: Measure Your Write Rate**

```javascript
// Analyze your workload over time

function analyzeWorkload(metrics) {
  const writesPerSecond = metrics.writes / metrics.duration;
  const readsPerSecond = metrics.reads / metrics.duration;
  const writeRatio = writesPerSecond / (writesPerSecond + readsPerSecond);
  
  console.log('Workload Analysis:');
  console.log('─'.repeat(50));
  console.log(`Writes/sec: ${writesPerSecond.toFixed(0)}`);
  console.log(`Reads/sec: ${readsPerSecond.toFixed(0)}`);
  console.log(`Write ratio: ${(writeRatio * 100).toFixed(1)}%`);
  console.log('');
  
  if (writeRatio > 0.7) {
    console.log('Recommendation: LSM-tree (high write volume)');
    console.log('  Databases: Cassandra, HBase, ScyllaDB');
  } else if (writeRatio < 0.3) {
    console.log('Recommendation: B-tree (read-heavy)');
    console.log('  Databases: PostgreSQL, MySQL');
  } else {
    console.log('Recommendation: B-tree (balanced workload)');
    console.log('  Databases: PostgreSQL, MySQL');
    console.log('  Note: B-trees handle mixed workloads well');
  }
}

// Example: Time-series metrics system
analyzeWorkload({
  writes: 500000,
  reads: 50000,
  duration: 60  // 60 seconds
});

// Output:
// Workload Analysis:
// ──────────────────────────────────────────────────
// Writes/sec: 8333
// Reads/sec: 833
// Write ratio: 90.9%
//
// Recommendation: LSM-tree (high write volume)
//   Databases: Cassandra, HBase, ScyllaDB
```

**Step 2: Consider Data Access Patterns**

```
┌───────────────────────────────────────────────────────┐
│  ACCESS PATTERN                    │  BEST ENGINE     │
├───────────────────────────────────────────────────────┤
│  Point lookups (SELECT WHERE id=?) │  B-tree or       │
│                                    │  LSM with cache  │
│                                                        │
│  Range scans (time-series)         │  Both good       │
│                                    │  (sorted storage)│
│                                                        │
│  Sequential writes (logging)       │  LSM-tree        │
│                                    │  (append-only)   │
│                                                        │
│  Random updates (user profiles)    │  B-tree          │
│                                    │  (in-place)      │
│                                                        │
│  Bulk imports                      │  LSM-tree        │
│                                    │  (fast ingestion)│
│                                                        │
│  Complex transactions              │  B-tree          │
│                                    │  (ACID support)  │
│                                                        │
│  Analytics (aggregations)          │  Column-store    │
│                                    │  (neither!)      │
└───────────────────────────────────────────────────────┘
```

### Real-World Scenarios with Recommendations

**Scenario 1: E-Commerce Application**

```javascript
// Typical e-commerce database requirements

const ecommerceWorkload = {
  reads: [
    'Product browsing (high volume)',
    'User profile lookups',
    'Order history queries',
    'Inventory checks',
    'Search results'
  ],
  writes: [
    'New orders (moderate)',
    'Inventory updates (frequent)',
    'User profile changes (rare)',
    'Product reviews (occasional)'
  ],
  transactions: true,
  consistency: 'strong',
  readWriteRatio: '80:20'
};

console.log('E-Commerce Recommendation:');
console.log('══════════════════════════════════════');
console.log('');
console.log('PRIMARY DATABASE: PostgreSQL (B-tree)');
console.log('');
console.log('Reasoning:');
console.log('   Read-heavy workload (80% reads)');
console.log('   Need ACID transactions for orders');
console.log('   Complex queries (JOINs for cart, orders)');
console.log('   Strong consistency required');
console.log('   Moderate write volume is acceptable');
console.log('');
console.log('Optimization:');
console.log('  • Create B-tree indexes on:');
console.log('    - user_id, product_id, order_id');
console.log('  • Use connection pooling');
console.log('  • Cache hot products in Redis');
console.log('  • Partition large tables by date');
```

**Scenario 2: IoT Sensor Platform**

```javascript
// IoT platform collecting sensor data

const iotWorkload = {
  writes: [
    '10,000 sensors × 1 reading/sec = 10K writes/sec',
    'Continuous 24/7',
    'Mostly time-series insertions',
    'No updates (historical data immutable)'
  ],
  reads: [
    'Recent data queries (last hour)',
    'Dashboards and monitoring',
    'Occasional historical analysis',
    'Alert threshold checks'
  ],
  transactions: false,
  consistency: 'eventual_ok',
  readWriteRatio: '5:95'
};

console.log('IoT Platform Recommendation:');
console.log('══════════════════════════════════════');
console.log('');
console.log('PRIMARY DATABASE: Cassandra (LSM-tree)');
console.log('');
console.log('Reasoning:');
console.log('   Extremely write-heavy (95% writes)');
console.log('   Time-series data (append-only)');
console.log('   No transactions needed');
console.log('   Can tolerate eventual consistency');
console.log('   LSM excels at sequential writes');
console.log('');
console.log('Architecture:');
console.log('  • Cassandra cluster for raw sensor data');
console.log('  • Partition by sensor_id + time');
console.log('  • Downsample old data with compaction');
console.log('  • Use time-window compaction strategy');
console.log('');
console.log('Performance Expected:');
console.log('  • Write latency: <2ms p99');
console.log('  • Write throughput: 150K writes/sec/node');
console.log('  • Read latency: 10-50ms p99');
```

**Scenario 3: Social Media Application**

```javascript
// Social media platform requirements

const socialWorkload = {
  features: {
    posts: 'High write volume',
    likes: 'Very high write volume',
    follows: 'Moderate writes',
    feeds: 'High read volume (user timelines)',
    profiles: 'Moderate reads, low writes'
  },
  scale: 'Millions of users',
  consistency: 'Eventual consistency OK for likes/follows',
  transactions: 'Needed for some operations'
};

console.log('Social Media Recommendation:');
console.log('══════════════════════════════════════');
console.log('');
console.log('HYBRID ARCHITECTURE:');
console.log('');
console.log('1. USER PROFILES: PostgreSQL (B-tree)');
console.log('   • Critical data needs transactions');
console.log('   • User authentication, settings');
console.log('   • Relatively low volume');
console.log('');
console.log('2. POSTS & FEEDS: Cassandra (LSM-tree)');
console.log('   • High write volume (posts, likes)');
console.log('   • Distribute user timelines');
console.log('   • Fast fan-out writes');
console.log('   • Partition by user_id');
console.log('');
console.log('3. MEDIA FILES: Object Storage (S3)');
console.log('   • Images, videos');
console.log('   • CDN distribution');
console.log('');
console.log('4. SEARCH: Elasticsearch (inverted index)');
console.log('   • Full-text search of posts');
console.log('   • User/hashtag search');
console.log('');
console.log('Data Flow:');
console.log('  User posts → Cassandra (fast write)');
console.log('             → Elasticsearch (async indexing)');
console.log('  User feeds ← Cassandra (distributed reads)');
console.log('  User auth  ← PostgreSQL (transactional)');
```

**Scenario 4: Banking/Financial System**

```javascript
// Financial system requirements

const bankingWorkload = {
  requirements: {
    consistency: 'CRITICAL - Strong ACID required',
    transactions: 'Multi-table transactions essential',
    auditability: 'Every change must be traceable',
    uptime: '99.99% required',
    dataIntegrity: 'Zero tolerance for data loss'
  },
  operations: {
    accountChecks: 'High read volume',
    transfers: 'Moderate writes (must be atomic)',
    balanceUpdates: 'Must be transactional',
    reporting: 'Complex queries with JOINs'
  }
};

console.log('Banking System Recommendation:');
console.log('══════════════════════════════════════');
console.log('');
console.log('PRIMARY DATABASE: PostgreSQL or Oracle (B-tree)');
console.log('');
console.log('Reasoning:');
console.log('   ACID transactions are NON-NEGOTIABLE');
console.log('   B-trees provide strong consistency');
console.log('   Mature, battle-tested systems');
console.log('   SQL for complex reporting');
console.log('   Foreign key constraints for integrity');
console.log('');
console.log('Why NOT LSM-tree (e.g., Cassandra):');
console.log('   Eventual consistency unacceptable');
console.log('   No multi-row transactions');
console.log('   Banking requires immediate consistency');
console.log('   Complex transactions need ACID guarantees');
console.log('');
console.log('Architecture:');
console.log('  • Primary-replica setup for reads');
console.log('  • Write-ahead logging for durability');
console.log('  • Point-in-time recovery');
console.log('  • Regular backups with transaction logs');
console.log('  • Synchronous replication for critical data');
```

### Performance Testing: Know Before You Deploy

```javascript
// Simple benchmark to test different storage engines

async function benchmarkStorageEngine(db, workload) {
  console.log(`Testing: ${db.name}`);
  console.log('═'.repeat(50));
  
  // Warm-up phase
  console.log('Warming up...');
  for (let i = 0; i < 1000; i++) {
    await db.insert({ id: i, data: `warmup_${i}` });
  }
  await db.flush();
  
  // Write benchmark
  console.log('\nWrite Benchmark:');
  const writeStart = Date.now();
  for (let i = 0; i < workload.writeCount; i++) {
    await db.insert({ 
      id: i, 
      timestamp: Date.now(),
      data: generateRandomData(100)  // 100 bytes
    });
  }
  const writeTime = Date.now() - writeStart;
  const writesPerSec = workload.writeCount / (writeTime / 1000);
  
  console.log(`  Writes: ${workload.writeCount}`);
  console.log(`  Time: ${writeTime}ms`);
  console.log(`  Throughput: ${writesPerSec.toFixed(0)} writes/sec`);
  console.log(`  Avg latency: ${(writeTime / workload.writeCount).toFixed(2)}ms`);
  
  // Read benchmark
  console.log('\nRead Benchmark (random keys):');
  const readStart = Date.now();
  for (let i = 0; i < workload.readCount; i++) {
    const randomId = Math.floor(Math.random() * workload.writeCount);
    await db.get(randomId);
  }
  const readTime = Date.now() - readStart;
  const readsPerSec = workload.readCount / (readTime / 1000);
  
  console.log(`  Reads: ${workload.readCount}`);
  console.log(`  Time: ${readTime}ms`);
  console.log(`  Throughput: ${readsPerSec.toFixed(0)} reads/sec`);
  console.log(`  Avg latency: ${(readTime / workload.readCount).toFixed(2)}ms`);
  
  // Storage efficiency
  const dbSize = await db.getSize();
  const dataSize = workload.writeCount * 100;  // 100 bytes per record
  const overhead = ((dbSize / dataSize) - 1) * 100;
  
  console.log('\nStorage:');
  console.log(`  Logical data: ${(dataSize / 1024 / 1024).toFixed(2)} MB`);
  console.log(`  Physical size: ${(dbSize / 1024 / 1024).toFixed(2)} MB`);
  console.log(`  Overhead: ${overhead.toFixed(1)}%`);
  
  console.log('\n');
}

// Run benchmarks
const workload = {
  writeCount: 100000,
  readCount: 10000
};

// Results might look like:
//
// Testing: PostgreSQL (B-tree)
// ══════════════════════════════════════════════════
// Write Benchmark:
//   Writes: 100000
//   Time: 45000ms
//   Throughput: 2222 writes/sec
//   Avg latency: 0.45ms
//
// Read Benchmark (random keys):
//   Reads: 10000
//   Time: 850ms
//   Throughput: 11765 reads/sec
//   Avg latency: 0.09ms
//
// Storage:
//   Logical data: 9.54 MB
//   Physical size: 24.80 MB
//   Overhead: 160.0%
//
//
// Testing: Cassandra (LSM-tree)
// ══════════════════════════════════════════════════
// Write Benchmark:
//   Writes: 100000
//   Time: 15000ms
//   Throughput: 6667 writes/sec
//   Avg latency: 0.15ms
//
// Read Benchmark (random keys):
//   Reads: 10000
//   Time: 1200ms
//   Throughput: 8333 reads/sec
//   Avg latency: 0.12ms
//
// Storage:
//   Logical data: 9.54 MB
//   Physical size: 12.10 MB
//   Overhead: 26.8%
```

### Migration Strategies

**When to Migrate from B-tree to LSM-tree:**

```javascript
// Red flags indicating need for LSM-tree

function shouldMigrate(metrics) {
  const warnings = [];
  
  // Check 1: Write throughput hitting limits
  if (metrics.writeLatencyP99 > 50) {  // ms
    warnings.push('⚠️  High write latency (P99 > 50ms)');
  }
  
  // Check 2: Write amplification
  if (metrics.diskWritesPerLogicalWrite > 5) {
    warnings.push('⚠️  Excessive write amplification');
  }
  
  // Check 3: Hot partition problems
  if (metrics.diskIOUtilization > 80) {
    warnings.push('⚠️  Disk I/O saturated');
  }
  
  // Check 4: Write ratio increasing
  if (metrics.writeRatio > 0.6) {
    warnings.push('⚠️  Write-heavy workload (>60%)');
  }
  
  if (warnings.length >= 2) {
    console.log('MIGRATION RECOMMENDED');
    console.log('');
    warnings.forEach(w => console.log(w));
    console.log('');
    console.log('Consider migrating to:');
    console.log('  • Cassandra (general purpose LSM)');
    console.log('  • ScyllaDB (C++ Cassandra, higher performance)');
    console.log('  • HBase (if in Hadoop ecosystem)');
  }
  
  return warnings.length >= 2;
}

// Example:
shouldMigrate({
  writeLatencyP99: 75,
  diskWritesPerLogicalWrite: 8,
  diskIOUtilization: 95,
  writeRatio: 0.7
});

// Output:
// MIGRATION RECOMMENDED
//
// ⚠️  High write latency (P99 > 50ms)
// ⚠️  Excessive write amplification
// ⚠️  Disk I/O saturated
// ⚠️  Write-heavy workload (>60%)
//
// Consider migrating to:
//   • Cassandra (general purpose LSM)
//   • ScyllaDB (C++ Cassandra, higher performance)
//   • HBase (if in Hadoop ecosystem)
```

## Summary

**Storage Engine Selection Guide:**

| Workload | Engine | Examples |
|----------|--------|----------|
| **Write-heavy, time-series** | LSM-tree | Cassandra, HBase, InfluxDB |
| **Balanced OLTP** | B-tree | PostgreSQL, MySQL, MongoDB |
| **In-memory, cache** | Hash index | Redis, Memcached |
| **Analytics (OLAP)** | Column-store | Redshift, BigQuery, ClickHouse |
| **Graph queries** | Graph DB | Neo4j, TigerGraph |

**Key Concepts:**
1. **Append-only logs** are simple and fast for writes
2. **Indexes** trade write performance for read performance
3. **LSM-trees** optimize for writes via sequential I/O
4. **B-trees** balance reads and writes with in-place updates
5. **Column storage** is essential for analytics workloads
6. **OLTP vs OLAP** require different storage optimizations
7. **Compression** is crucial for analytical databases

**Performance Characteristics:**

```
Hash Index:
  Write: O(1) + disk append
  Read:  O(1) hash lookup + disk seek
  Range: Not supported

B-Tree:
  Write: O(log n) + random disk write
  Read:  O(log n) tree traversal
  Range: Efficient (sorted scan)

LSM-Tree:
  Write: O(1) + sequential disk write
  Read:  O(log n) × number of SSTables
  Range: Efficient (sorted scan within SSTables)

Column Store:
  Write: Batch updates only
  Read (single row):   Slow (read all columns)
  Read (analytics):    Very fast (read only needed columns)
```

**Looking Ahead:**
We've covered how individual database nodes store data. In Chapter 4, we'll explore how data is encoded for transmission and storage, enabling schema evolution. Then in Chapters 5-9, we'll see how these storage techniques scale across multiple machines in distributed systems.
