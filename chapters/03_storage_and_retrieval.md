# Chapter 3: Storage and Retrieval

## Introduction

As application developers, we rarely think about how databases store and retrieve data. We simply write data and trust the database to give it back when we ask. But understanding storage engines is crucial for choosing the right database and tuning performance.

On the most fundamental level, a database needs to do two things:
1. **Store data** when you give it to you
2. **Retrieve data** when you ask for it later

This chapter explores the internal mechanics of storage engines, covering:
- **Log-structured storage** (LSM-trees): Optimized for writes
- **Page-oriented storage** (B-trees): Balanced read/write performance
- **OLTP vs OLAP**: Different workload patterns
- **Column-oriented storage**: Analytics optimization
- **Indexes**: Data structures that speed up reads

Understanding these fundamentals will help you make informed decisions about database selection and optimization.

## Part 1: Data Structures That Power Your Database

### The World's Simplest Database

Let's start with the simplest possible database implementation:

```bash
#!/bin/bash

# Store a key-value pair
db_set() {
    echo "$1,$2" >> database.txt
}

# Retrieve a value by key
db_get() {
    grep "^$1," database.txt | sed -e "s/^$1,//" | tail -n 1
}
```

**Usage:**

```bash
$ db_set "user_123" "Alice"
$ db_set "user_456" "Bob"
$ db_set "user_123" "Alice Updated"  # Update
$ db_get "user_123"
Alice Updated
```

**How it works:**

```
database.txt contents:
user_123,Alice
user_456,Bob
user_123,Alice Updated
```

**Performance Characteristics:**
- **Write**: O(1) - Append to end of file
- **Read**: O(n) - Scan entire file to find key

This is an **append-only log** - the fundamental building block of many databases!

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

```python
class HashIndexDB:
    def __init__(self, data_file):
        self.data_file = data_file
        self.index = {}  # key -> byte offset
        self._load_index()
    
    def _load_index(self):
        """Build index by scanning data file on startup"""
        with open(self.data_file, 'rb') as f:
            offset = 0
            while True:
                line = f.readline()
                if not line:
                    break
                key = line.split(b',')[0].decode()
                self.index[key] = offset
                offset = f.tell()
    
    def set(self, key, value):
        """Append to data file and update index"""
        with open(self.data_file, 'ab') as f:
            offset = f.tell()
            data = f"{key},{value}\n".encode()
            f.write(data)
            self.index[key] = offset
    
    def get(self, key):
        """Look up in index, then read from file"""
        if key not in self.index:
            return None
        
        with open(self.data_file, 'rb') as f:
            f.seek(self.index[key])
            line = f.readline()
            value = line.split(b',')[1].strip().decode()
            return value

# Usage
db = HashIndexDB('database.txt')
db.set('user_123', 'Alice')
print(db.get('user_123'))  # Fast: O(1) lookup + O(1) disk seek
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

```python
def compact_segment(old_file, new_file):
    """Keep only the latest value for each key"""
    latest = {}  # key -> (offset, value)
    
    # Scan old file, keeping track of latest values
    with open(old_file, 'r') as f:
        for line in f:
            key, value = line.strip().split(',')
            latest[key] = value
    
    # Write compacted data to new file
    with open(new_file, 'w') as f:
        for key, value in latest.items():
            f.write(f"{key},{value}\n")
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

LSM-trees are the storage engine behind Cassandra, HBase, RocksDB, and LevelDB.

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

```python
class LSMTree:
    def __init__(self):
        self.memtable = {}  # In-memory sorted tree
        self.sstables = []  # List of SSTable files on disk
        self.wal = open('write_ahead_log.txt', 'a')  # For crash recovery
    
    def write(self, key, value):
        # 1. Append to write-ahead log (for durability)
        self.wal.write(f"{key},{value}\n")
        self.wal.flush()
        
        # 2. Write to memtable
        self.memtable[key] = value
        
        # 3. If memtable is full, flush to disk
        if len(self.memtable) > 10000:
            self._flush_memtable()
    
    def _flush_memtable(self):
        """Convert memtable to immutable SSTable on disk"""
        filename = f"sstable_{time.time()}.db"
        with open(filename, 'w') as f:
            for key in sorted(self.memtable.keys()):
                f.write(f"{key},{self.memtable[key]}\n")
        
        self.sstables.append(filename)
        self.memtable.clear()
        
        # Trigger background compaction if needed
        if len(self.sstables) > 10:
            self._compact_sstables()
```

**Read Path:**

```python
def read(self, key):
    # 1. Check memtable first (most recent data)
    if key in self.memtable:
        return self.memtable[key]
    
    # 2. Check SSTables from newest to oldest
    for sstable in reversed(self.sstables):
        value = self._search_sstable(sstable, key)
        if value is not None:
            return value
    
    return None  # Key not found

def _search_sstable(self, sstable_file, key):
    """Binary search within SSTable"""
    with open(sstable_file, 'r') as f:
        # Use sparse index + binary search (simplified here)
        for line in f:
            k, v = line.strip().split(',')
            if k == key:
                return v
            if k > key:
                break  # Key not in this SSTable
    return None
```

### Compaction Strategies

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

```python
from bitarray import bitarray
import mmh3

class BloomFilter:
    def __init__(self, size=10000, hash_count=3):
        self.size = size
        self.hash_count = hash_count
        self.bit_array = bitarray(size)
        self.bit_array.setall(0)
    
    def add(self, key):
        for i in range(self.hash_count):
            index = mmh3.hash(key, i) % self.size
            self.bit_array[index] = 1
    
    def might_contain(self, key):
        for i in range(self.hash_count):
            index = mmh3.hash(key, i) % self.size
            if not self.bit_array[index]:
                return False  # Definitely not present
        return True  # Might be present (could be false positive)
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

B-trees are the most widely used indexing structure. They power:
- **PostgreSQL, MySQL**: Primary indexes
- **SQLite**: Default storage
- **MongoDB**: Before WiredTiger, used B-trees

### B-Tree Structure

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

**Properties:**
- Each page contains up to **N keys** and **N+1 child pointers**
- Keys are sorted within each page
- Tree is balanced: all leaf pages at same depth
- **Branching factor** (N): typically hundreds

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
   Found 220!
```

**Search Performance**: O(log_N k) where N = branching factor, k = number of keys

With branching factor 500 and 4 levels, can store 500^4 = 62.5 billion keys!

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

```python
def delete(key):
    # 1. Find and remove key from leaf page
    page = find_leaf_page(key)
    page.remove(key)
    
    # 2. If page is too empty, rebalance
    if page.too_empty():
        # Option A: Borrow from sibling
        if sibling.has_extra_keys():
            borrow_from_sibling(page, sibling)
        # Option B: Merge with sibling
        else:
            merge_with_sibling(page, sibling)
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

```python
def insert_with_wal(key, value):
    # 1. Write to WAL first
    wal_offset = wal.append(f"INSERT {key} {value}")
    wal.flush()  # Ensure on disk
    
    # 2. Modify B-tree
    page = find_leaf_page(key)
    page.insert(key, value)
    page.write_to_disk()
    
    # 3. Mark WAL entry as committed
    wal.mark_committed(wal_offset)
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

```python
import time

# Benchmark writes
def benchmark_writes(db, num_writes=100000):
    start = time.time()
    for i in range(num_writes):
        db.write(f"key_{i}", f"value_{i}")
    elapsed = time.time() - start
    print(f"Writes: {num_writes / elapsed:.0f} ops/sec")

# Benchmark reads
def benchmark_reads(db, num_reads=100000):
    keys = [f"key_{i % 10000}" for i in range(num_reads)]
    start = time.time()
    for key in keys:
        db.read(key)
    elapsed = time.time() - start
    print(f"Reads: {num_reads / elapsed:.0f} ops/sec")

# Results (example):
# LSM-Tree: Writes: 50,000 ops/sec, Reads: 20,000 ops/sec
# B-Tree:   Writes: 10,000 ops/sec, Reads: 40,000 ops/sec
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

```python
import redis

r = redis.Redis(host='localhost', port=6379)

# Simple key-value
r.set('user:123:name', 'Alice')
print(r.get('user:123:name'))  # b'Alice'

# Data structures (not possible in disk-based KV stores)
r.lpush('queue', 'task1', 'task2', 'task3')
r.rpop('queue')  # Process tasks from queue

r.hset('user:123', mapping={
    'name': 'Alice',
    'age': 30,
    'email': 'alice@example.com'
})
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

**Problem**: Must read ALL columns from disk, even though only `product` and `revenue` are needed!

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

**Performance improvement**: 10x-100x for analytical queries!

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

```python
# Traditional (scalar) processing
def sum_revenue_scalar(revenues):
    total = 0
    for revenue in revenues:
        total += revenue
    return total

# Vectorized processing (SIMD)
import numpy as np

def sum_revenue_vectorized(revenues):
    # Process 4-8 values per CPU instruction!
    return np.sum(revenues)

# Performance difference:
# Scalar:     100M rows → 1.5 seconds
# Vectorized: 100M rows → 0.2 seconds (7.5x faster)
```

**Modern OLAP databases** (ClickHouse, DuckDB) exploit SIMD extensively.

### Writing to Column-Oriented Storage

**Problem**: Inserting one row requires updating ALL column files!

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
