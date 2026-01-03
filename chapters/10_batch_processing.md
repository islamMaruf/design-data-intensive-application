# Chapter 10: Batch Processing

## Introduction

In the previous chapters, we focused on online systems: databases that respond to user requests as they come in. But many data systems also need to process large amounts of data in **batches**—crunching through terabytes of data to produce reports, indexes, or derived datasets.

**Batch processing** systems process bounded input datasets to produce output datasets, typically running periodically (e.g., daily, hourly). They are characterized by:
- **High throughput**: Processing large volumes efficiently
- **High latency**: Minutes to hours to complete
- **No user interaction**: Automated, scheduled jobs

This chapter explores:
- **MapReduce**: The paradigm that made batch processing scalable
- **Distributed filesystems**: HDFS and object storage
- **Dataflow engines**: Spark, Flink, and modern alternatives
- **Batch processing patterns**: Joins, grouping, and workflows

Understanding batch processing is essential for building analytics pipelines, ETL workflows, and machine learning training pipelines.

## Part 1: Batch Processing with Unix Tools

### The Unix Philosophy

Before diving into distributed systems, let's appreciate the elegance of Unix tools:

**Principles:**
1. **Make each program do one thing well**
2. **Expect the output of every program to become the input to another**
3. **Design for pipes** to connect programs together

**Example: Web Server Log Analysis**

**Task**: Find the 5 most popular URLs from an access log.

**Log format (access.log):**
```
192.168.1.1 - - [01/Jan/2024:10:00:00 +0000] "GET /index.html HTTP/1.1" 200 1234
192.168.1.2 - - [01/Jan/2024:10:00:01 +0000] "GET /about.html HTTP/1.1" 200 5678
192.168.1.3 - - [01/Jan/2024:10:00:02 +0000] "GET /index.html HTTP/1.1" 200 1234
...
```

**Unix pipeline solution:**

```bash
cat access.log |
  awk '{print $7}' |           # Extract URL (7th field)
  sort |                       # Sort URLs
  uniq -c |                    # Count occurrences
  sort -rn |                   # Sort by count (descending)
  head -n 5                    # Take top 5
```

**Output:**
```
  15234 /index.html
   8721 /about.html
   5432 /products.html
   3210 /contact.html
   1987 /blog.html
```

**Why This Works:**
- **Uniform interface**: Text streams
- **Composability**: Combine simple tools
- **Automatic parallelization**: `sort` can use multiple CPUs
- **Lazy evaluation**: Process data in small chunks

**Visual: Unix Pipeline**

```
┌──────────┐    ┌──────┐    ┌──────┐    ┌──────────┐    ┌──────────┐    ┌──────┐
│access.log│ → │ awk  │ → │ sort │ → │ uniq -c  │ → │ sort -rn │ → │ head │
└──────────┘    └──────┘    └──────┘    └──────────┘    └──────────┘    └──────┘
   (input)     (extract)   (sort)      (count)       (sort by    )   (top 5)
                                                       count)
```

### Limitations of Unix Tools

**1. Limited to single machine**
- Can't process more data than fits on one machine
- CPU and memory bounded

**2. No fault tolerance**
- If process crashes, must restart from beginning

**3. Limited data structures**
- Text streams only (no complex types)

**Solution: Distributed batch processing** (MapReduce, Spark)

## Part 2: MapReduce and the Distributed Filesystem

### The Origins and Evolution of MapReduce

Before we dive into the technical details of MapReduce, it's important to understand its historical context and why it became such a transformative technology in the world of data processing.

**The Google Paper (Early 2000s)**

MapReduce originated from a research paper published by Google engineers in the early 2000s. At that time, as one practitioner recalls: "MapReduce was the hot thing when I was up and coming as a dev." Google faced an unprecedented challenge: how to process massive amounts of web data—entire dumps of the internet—to build their search index.

The internet was already a large place two decades ago, with hundreds of thousands (soon millions) of websites containing text, images, and constantly changing content. Google needed a way to:
- Crawl and index this massive, ever-growing dataset
- Process data that didn't fit on a single machine
- Handle failures gracefully (servers crash, disks fail, networks partition)
- Make it accessible to engineers who weren't distributed systems experts

**Why MapReduce Became "The Hot Thing"**

MapReduce solved a fundamental problem: **it democratized large-scale data processing**. Before MapReduce, building distributed data processing systems required:
- Deep expertise in distributed systems
- Custom code for every new analysis task
- Manual handling of failures and load balancing
- Intimate knowledge of the underlying infrastructure

MapReduce changed this by providing a simple programming model: **just write two functions—map and reduce**—and the framework handles all the distributed systems complexity for you.

**The Technology Lifecycle**

As with many technologies, MapReduce has gone through a lifecycle:

1. **Early Days (2004-2010)**: Revolutionary and exciting
   - Open-source Hadoop implementation made it accessible
   - Companies built entire data teams around Hadoop/MapReduce
   - "Big Data" became synonymous with MapReduce

2. **Maturity (2010-2015)**: Widespread adoption
   - Standard tool for batch processing
   - Extensive ecosystem (Pig, Hive, HBase)
   - Best practices and patterns emerged

3. **Evolution (2015-Present)**: Still relevant, but evolved
   - Newer engines (Spark, Flink) improve on limitations
   - Cloud services abstract away infrastructure
   - Stream processing gains prominence
   - MapReduce principles still foundational

As one data engineer notes: "You might say well it's not used as much these days," but the reality is that MapReduce's core concepts—dividing work into independent tasks, processing data where it lives, handling failures automatically—remain fundamental to modern data processing, even if the specific implementation has evolved.

### HDFS: Hadoop Distributed File System

**HDFS** stores large files across many machines, providing:
- **Fault tolerance**: Replication (typically 3x)
- **High throughput**: Parallel reads from multiple machines
- **Large blocks**: 64MB-128MB chunks (vs 4KB in traditional filesystems)

**Architecture:**

```
                    NameNode (Master)
                   ┌──────────────────┐
                   │ Metadata:        │
                   │ - File locations │
                   │ - Block mappings │
                   └──────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
  ┌──────────┐        ┌──────────┐        ┌──────────┐
  │DataNode 1│        │DataNode 2│        │DataNode 3│
  │ Block A1 │        │ Block A2 │        │ Block A3 │
  │ Block B1 │        │ Block A1 │        │ Block B1 │
  │ (replica)│        │ (replica)│        │ (replica)│
  └──────────┘        └──────────┘        └──────────┘
```

**File storage example:**

```
Large file (1GB):
  ┌────────────────────────────────────────┐
  │ Block 1 | Block 2 | Block 3 | ... | Block 16 │
  └────────────────────────────────────────┘
       64MB     64MB      64MB           64MB

Stored across cluster:
  DataNode 1: [Block 1, Block 2, Block 5, ...]
  DataNode 2: [Block 1, Block 3, Block 6, ...]  ← Replicas
  DataNode 3: [Block 2, Block 4, Block 7, ...]
```

**Benefits:**
- **Locality**: Process data where it's stored
- **Fault tolerance**: If DataNode fails, read from replica
- **Scalability**: Add more DataNodes to increase capacity

### MapReduce Paradigm

**MapReduce** is a programming model for processing large datasets with a parallel, distributed algorithm.

**Key Idea**: Split computation into two phases:
1. **Map**: Transform each input record independently
2. **Reduce**: Aggregate records with the same key

#### Understanding Batch Processing vs. Database Queries

Before we go deeper into MapReduce, let's clarify a crucial conceptual distinction: **Why use batch processing instead of just running database queries?**

**Batch Processing is for "Behind-the-Scenes" Work**

MapReduce and batch processing systems are designed for workloads that are **not on the critical path of your application**. Here's what that means:

Imagine you're Wikipedia, and you want to show statistics about your content—things like:
- Total number of articles
- Most frequently used words across all articles
- Most edited articles
- Language distribution statistics

You *could* technically compute these stats using SQL queries on your production database:
```sql
SELECT word, COUNT(*) as count 
FROM (
  SELECT unnest(string_to_array(content, ' ')) as word 
  FROM articles
) subquery 
GROUP BY word 
ORDER BY count DESC 
LIMIT 100;
```

**But this would be a terrible idea!** Here's why:

1. **Too Slow for Users**: If this query takes 30 minutes to run on 100TB of data, you can't make users wait while viewing a stats page

2. **Resource Intensive**: The query would consume massive CPU, memory, and I/O resources, slowing down regular user requests

3. **Not Time-Critical**: These statistics don't need to be real-time. If they're a few hours old, that's perfectly fine

**The Batch Processing Workflow**

Instead, a typical workflow looks like this:

```
┌─────────────────────────────────────────────────────────┐
│                    Nightly Batch Job                    │
│  (Runs at 2 AM when traffic is low)                     │
│                                                          │
│  1. Read all Wikipedia articles from storage            │
│  2. Run MapReduce word count                            │
│  3. Compute statistics (takes 2 hours)                  │
│  4. Store results in simple database table              │
└─────────────────────────────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────┐
│              Production Database (MySQL/Postgres)       │
│                                                          │
│  stats_table:                                           │
│   word          | count                                 │
│  ──────────────────────────                             │
│   "the"         | 15,234,567                            │
│   "wikipedia"   | 8,721,033                             │
│   "article"     | 5,432,910                             │
│   ...                                                   │
└─────────────────────────────────────────────────────────┘
                          │
                          ↓
┌─────────────────────────────────────────────────────────┐
│            User Visits Stats Page                       │
│                                                          │
│  SELECT * FROM stats_table WHERE word = 'the'           │
│  → Returns instantly! (simple lookup)                   │
└─────────────────────────────────────────────────────────┘
```

**Batch Job Scheduling**

Batch jobs typically run on a schedule based on business needs:

- **Daily (Nightly)**: Most common—run analytics overnight when system load is low
  - Example: E-commerce daily sales reports

- **Weekly**: For expensive computations that don't need frequent updates
  - Example: User behavior cohort analysis

- **Monthly**: For heavy aggregations or historical analysis
  - Example: Month-end financial reports

- **Hourly**: For near-real-time (but not critical) updates
  - Example: Trending topics calculation

- **On-Demand**: Triggered by events or manual requests
  - Example: Machine learning model re-training

The key insight: **Batch processing lets you do expensive computations offline, so user-facing systems stay fast and responsive.**

#### The Distributed Word Count Example: A Deep Dive

Let's walk through the canonical MapReduce example—word counting—but this time, we'll use a realistic scale to understand how distributed processing actually works.

**The Scenario**

Imagine you need to count every word across all Wikipedia articles:
- **Data size**: 100 terabytes (100,000 gigabytes) of HTML files
- **Cluster size**: 1,000 servers
- **Data distribution**: 100 GB of data per server

This is too much data for one machine, so we need to distribute the work.

**Step 1: Data Distribution Across Servers**

The distributed file system (like HDFS) splits the data across servers:

```
Server 1:  [100 GB of articles: art_000001.html to art_005000.html]
Server 2:  [100 GB of articles: art_005001.html to art_010000.html]
Server 3:  [100 GB of articles: art_010001.html to art_015000.html]
...
Server 1000: [100 GB of articles: art_995001.html to art_1000000.html]
```

Each server has its own chunk of the input data, stored locally on its disk.

**Step 2: The Map Phase**

The mapper function runs **independently on each server**, processing only the data stored locally. This is crucial for performance—we avoid network transfer of the raw input data.

**What the Mapper Does:**

The mapper's job is extremely simple:
1. Read input text (in whatever format: raw text, CSV, HTML, SQL dump)
2. Split text into tokens (words)
3. For each word, output a key-value pair: `(word, 1)`

Let's see this in detail for Server 1:

```javascript
// Pseudo-code for the Map function
function mapper(documentId, textContent) {
  // Input: The full text of one document
  // Example: "Hello world hello Hadoop world of data"
  
  const words = textContent.split(/\s+/); // Split on whitespace
  
  // For each word, emit (word, 1)
  for (const word of words) {
    emit(word.toLowerCase(), 1);
  }
}

// Server 1 processing art_000001.html:
// Input text: "Hello world hello Hadoop world of data"
//
// Mapper outputs:
// ("hello", 1)
// ("world", 1)
// ("hello", 1)  ← Notice: same key can appear multiple times!
// ("hadoop", 1)
// ("world", 1)
// ("of", 1)
// ("data", 1)
```

**Important Insight**: The mapper doesn't aggregate anything. It just blindly emits `(word, 1)` for every single word it sees. If "hello" appears 10,000 times in Server 1's data, the mapper will output `("hello", 1)` exactly 10,000 times.

This might seem wasteful, but it's intentional! The simplicity allows perfect parallelization—each mapper runs completely independently with no coordination.

**Step 3: The Shuffle and Sort Phase (The Magic)**

This is where MapReduce's real power comes in. After all mappers finish, the framework performs a **shuffle and sort**:

1. **Grouping by Key**: All key-value pairs with the same key are collected together
2. **Sorting**: Keys are sorted
3. **Partitioning**: Keys are distributed to reducers

Here's what happens to our word counts:

```
All Mappers Output (simplified):
Server 1:   ("hello", 1), ("hello", 1), ("world", 1), ("hadoop", 1), ...
Server 2:   ("hello", 1), ("world", 1), ("python", 1), ...
Server 3:   ("world", 1), ("hello", 1), ("spark", 1), ...
...
Server 1000: ("world", 1), ("data", 1), ("hello", 1), ...

After Shuffle & Sort:
─────────────────────────────────────────────────────────
All "hello" keys go to Reducer 1:
  ("hello", [1, 1, 1, 1, 1, ... ]) ← 5,000 instances

All "world" keys go to Reducer 2:
  ("world", [1, 1, 1, 1, ... ])   ← 3,000 instances

All "hadoop" keys go to Reducer 1:
  ("hadoop", [1, 1, 1, ... ])     ← 1,200 instances

All "postgres" keys go to Reducer 3:
  ("postgres", [1, 1, 1, ... ])   ← 5,000 instances

All "mysql" keys go to Reducer 4:
  ("mysql", [1, 1, 1, ... ])      ← 10,320 instances

All "sqlite" keys go to Reducer 5:
  ("sqlite", [1, 1, 1, ... ])     ← 3,041 instances
...
```

**Critical Point**: The shuffle phase ensures that **ALL instances of the same key go to the SAME reducer**. This is why reducers can perform aggregation—they're guaranteed to see all values for their assigned keys.

**Step 4: The Reduce Phase**

Now each reducer gets a complete list of all values for each of its assigned keys:

```javascript
// Pseudo-code for the Reduce function
function reducer(key, values) {
  // Input: 
  //   key = "hello"
  //   values = [1, 1, 1, 1, 1, ..., 1]  // Iterator of all counts
  
  // Sum up all the counts
  let totalCount = 0;
  for (const count of values) {
    totalCount += count;
  }
  
  // Output: (key, total_count)
  emit(key, totalCount);
}

// Reducer 1 processing "hello":
// Input: ("hello", [1, 1, 1, ... ]) ← 5,000 ones
// Output: ("hello", 5000)

// Reducer 2 processing "world":
// Input: ("world", [1, 1, 1, ... ]) ← 3,000 ones
// Output: ("world", 3000)

// Reducer 3 processing "postgres":
// Input: ("postgres", [1, 1, 1, ... ]) ← 5,000 ones
// Output: ("postgres", 5000)
```

The reducer's job is simple: **count how many instances of each key it receives**.

**Step 5: Output**

Each reducer writes its results to a file in the distributed filesystem:

```
Output Files in HDFS:

/output/part-00000:  (Reducer 1's output)
hello     5000
hadoop    1200
data      8500

/output/part-00001:  (Reducer 2's output)
world     3000
...

/output/part-00002:  (Reducer 3's output)
postgres  5000
mysql     10320
...
```

**Complete Flow Visualization**

```
INPUT (100 TB across 1,000 servers)
│
├─ Server 1 (100 GB)
│  └─ Mapper 1 → (hello,1), (world,1), (hello,1), ...
│
├─ Server 2 (100 GB)
│  └─ Mapper 2 → (hello,1), (python,1), (world,1), ...
│
├─ Server 3 (100 GB)
│  └─ Mapper 3 → (world,1), (spark,1), (hello,1), ...
│
└─ ... 997 more servers

         ↓ ↓ ↓
         
SHUFFLE & SORT PHASE
(Network transfer - group by key)

         ↓ ↓ ↓

REDUCE PHASE
│
├─ Reducer 1
│  ├─ Input:  ("hello", [1,1,1,...])  ← all "hello" from all mappers
│  └─ Output: ("hello", 5000)
│
├─ Reducer 2
│  ├─ Input:  ("world", [1,1,1,...])  ← all "world" from all mappers
│  └─ Output: ("world", 3000)
│
└─ ... more reducers

         ↓ ↓ ↓

OUTPUT (Much smaller - just word counts)
Stored across distributed filesystem in output directory
```

**Why This Design Works**

1. **Parallelism**: 1,000 mappers run simultaneously, each processing 100 GB
2. **Data Locality**: Mappers process data on their local disk (no network I/O for input)
3. **Scalability**: Adding more servers linearly increases processing capacity
4. **Simplicity**: Map and reduce functions are simple, pure functions
5. **Fault Tolerance**: If a mapper fails, just restart it (stateless)

**Performance Numbers**

With 1,000 servers:
- **Map phase**: ~10-30 minutes (reading 100 GB + processing per server)
- **Shuffle phase**: ~10-20 minutes (network transfer of intermediate results)
- **Reduce phase**: ~5-10 minutes (aggregating counts)
- **Total**: ~30-60 minutes for 100 TB of data

Compare this to processing on one machine (if it were even possible), which would take days or weeks!

### Word Count Example (Code)

**Input:**
```
hello world
hello hadoop
world of big data
```

**MapReduce flow:**

```
┌─────────────────────────────────────────────────────────────┐
│                         INPUT                               │
│ "hello world"   "hello hadoop"   "world of big data"        │
└─────────────────────────────────────────────────────────────┘
                         │
                    ┌────┴────┐
                    │  SPLIT  │
                    └────┬────┘
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    ┌────────┐      ┌────────┐      ┌────────┐
    │ Map 1  │      │ Map 2  │      │ Map 3  │
    └────────┘      └────────┘      └────────┘
    (hello,1)       (hello,1)       (world,1)
    (world,1)       (hadoop,1)      (of,1)
                                    (big,1)
                                    (data,1)
         │               │               │
         └───────┬───────┴───────┬───────┘
                 │   SHUFFLE &   │
                 │    SORT       │
                 └───────┬───────┘
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
    ┌────────┐      ┌────────┐      ┌────────┐
    │Reduce 1│      │Reduce 2│      │Reduce 3│
    └────────┘      └────────┘      └────────┘
    (big,1)         (hello,2)       (world,2)
    (data,1)        (of,1)
    (hadoop,1)
         │               │               │
         └───────────────┼───────────────┘
                         ▼
                    ┌────────┐
                    │ OUTPUT │
                    └────────┘
                    big: 1
                    data: 1
                    hadoop: 1
                    hello: 2
                    of: 1
                    world: 2
```

**MapReduce Code (Python with MRJob):**

```python
from mrjob.job import MRJob

class WordCount(MRJob):
    
    def mapper(self, _, line):
        """Map phase: emit (word, 1) for each word"""
        for word in line.split():
            yield (word.lower(), 1)
    
    def reducer(self, key, values):
        """Reduce phase: sum counts for each word"""
        yield (key, sum(values))

if __name__ == '__main__':
    WordCount.run()
```

**Run locally:**
```bash
python word_count.py input.txt
```

**Run on Hadoop:**
```bash
python word_count.py -r hadoop hdfs:///input/*.txt
```

### MapReduce Input Flexibility

One of MapReduce's most powerful features is its ability to work with **heterogeneous data sources** in various formats. The mapper function doesn't care what format the input is in—it just needs to know how to parse it.

**Supported Input Formats**

MapReduce can process:
- **Raw text files**: Log files, plain text documents
- **CSV files**: Tabular data exports
- **SQL database dumps**: Exported database tables
- **HTML files**: Web pages, documentation
- **JSON/XML**: Structured data files
- **Binary formats**: Images, videos (with custom InputFormat)
- **Compressed files**: gzip, bzip2, Snappy (automatically decompressed)

**Example: Processing Multiple Data Sources**

Imagine combining data from different systems:

```javascript
// Mapper that handles multiple input formats
function mapper(filename, content) {
  if (filename.endsWith('.csv')) {
    // Parse CSV
    const rows = content.split('\n');
    for (const row of rows) {
      const [userId, action, timestamp] = row.split(',');
      emit(userId, { type: 'action', action, timestamp });
    }
  } 
  else if (filename.endsWith('.sql')) {
    // Parse SQL dump
    const lines = content.split('\n');
    for (const line of lines) {
      if (line.startsWith('INSERT INTO users')) {
        // Extract user data from INSERT statement
        const userData = parseSQLInsert(line);
        emit(userData.id, { type: 'profile', data: userData });
      }
    }
  } 
  else if (filename.endsWith('.json')) {
    // Parse JSON logs
    const logEntry = JSON.parse(content);
    emit(logEntry.userId, { type: 'log', entry: logEntry });
  }
  else {
    // Raw text - just emit words
    const words = content.split(/\s+/);
    for (const word of words) {
      emit(word, 1);
    }
  }
}
```

This flexibility means you can:
1. **Migrate between systems**: Process data from old and new databases simultaneously
2. **Combine sources**: Join log files with database dumps
3. **Handle legacy formats**: Work with data in any format without conversion
4. **Process incrementally**: Add new data types without rewriting existing code

### Real-World Case Study: How Google Used MapReduce for Search

One of the most famous and impactful uses of MapReduce was Google's search index building. This is not just theoretical—this is how Google actually processed the web in the early days (and the principles still apply to modern systems).

**The Challenge**

Google needed to:
1. **Crawl the web**: Download billions of web pages
2. **Extract text and links**: Parse HTML to get content and hyperlinks
3. **Build inverted index**: Map each word to list of pages containing it
4. **Calculate PageRank**: Use link graph to rank page importance
5. **Update regularly**: Re-process as web content changes (pages updated, links added/removed)

And do all this for the **entire internet**—even in the early 2000s, this meant:
- Hundreds of thousands to millions of websites
- Terabytes (soon petabytes) of HTML data
- Constantly changing content (new pages, updated articles, broken links)

**The MapReduce Solution**

Google ran **large batch jobs** (likely hourly, daily, or weekly) to build and update search indexes. Here's a simplified view of how it worked:

**Job 1: Build Inverted Index**

```javascript
// Mapper: Extract words from web pages
function mapPageToWords(url, htmlContent) {
  // Parse HTML and extract text
  const text = extractTextFromHTML(htmlContent);
  const words = tokenize(text);
  
  // For each word, emit (word, url)
  for (const word of words) {
    emit(word.toLowerCase(), { url: url, position: words.indexOf(word) });
  }
}

// Reducer: Build inverted index
function reduceToIndex(word, urlList) {
  // urlList = all URLs containing this word
  // Example: word="database", urlList=[{url: "wiki.org/database", ...}, {url: "postgres.org"}, ...]
  
  // Create posting list (sorted by relevance)
  const postingList = urlList.sort(byRelevance);
  
  // Emit: word → list of URLs
  emit(word, postingList);
}

// Output example:
// "database" → ["https://en.wikipedia.org/wiki/Database", 
//               "https://www.postgresql.org/", 
//               "https://www.mysql.com/", ...]
```

**Job 2: Calculate PageRank**

PageRank is Google's famous algorithm for ranking web pages based on their importance. The key insight: **a page is important if many other important pages link to it**.

```javascript
// Mapper: Extract outgoing links from each page
function mapPageLinks(url, htmlContent) {
  const outgoingLinks = extractLinks(htmlContent);
  const pageRank = getCurrentPageRank(url); // From previous iteration
  
  // Distribute PageRank evenly to all outgoing links
  const rankPerLink = pageRank / outgoingLinks.length;
  
  for (const link of outgoingLinks) {
    emit(link, { incomingRank: rankPerLink, from: url });
  }
}

// Reducer: Calculate new PageRank for each page
function reducePageRank(url, incomingRanks) {
  // Sum up all PageRank contributed by incoming links
  let newRank = 0;
  for (const incoming of incomingRanks) {
    newRank += incoming.incomingRank;
  }
  
  // PageRank formula: 0.15 (base) + 0.85 * (sum of incoming ranks)
  newRank = 0.15 + 0.85 * newRank;
  
  emit(url, newRank);
}
```

**Job 3: Build Search Index**

Combine inverted index + PageRank + other signals:

```javascript
// Reducer: Join index data with PageRank
function buildSearchIndex(word, postingData) {
  // Get all pages containing this word
  const pages = postingData.filter(d => d.type === 'page');
  
  // Get PageRank scores
  const pageRanks = postingData.filter(d => d.type === 'pagerank');
  
  // Combine and sort by relevance
  const rankedResults = pages.map(page => ({
    url: page.url,
    pageRank: pageRanks.find(pr => pr.url === page.url)?.rank || 0,
    position: page.position,  // Where word appears in page
    // Other signals: anchor text, title, meta description...
  }));
  
  // Sort by combined relevance score
  rankedResults.sort((a, b) => 
    computeRelevance(b) - computeRelevance(a)
  );
  
  emit(word, rankedResults);
}
```

**The Batch Processing Workflow**

```
┌─────────────────────────────────────────────────────────────┐
│              Web Crawlers (Continuous)                      │
│  Download pages → Store in distributed file system          │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│        Batch Job 1: Extract Links & Text                    │
│  Input:  HTML files (petabytes)                             │
│  Output: Link graph + text content                          │
│  Schedule: Daily                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│        Batch Job 2: Calculate PageRank                      │
│  Input:  Link graph                                         │
│  Output: PageRank scores for all URLs                       │
│  Schedule: Weekly (iterative, multiple MapReduce jobs)      │
│  Iterations: ~20-30 to converge                             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│        Batch Job 3: Build Inverted Index                    │
│  Input:  Text content + PageRank scores                     │
│  Output: Inverted index (word → ranked list of URLs)        │
│  Schedule: Daily                                            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────┐
│            Search Index (Production)                        │
│  Used to serve real-time search queries                     │
│  Updated with each batch job completion                     │
└─────────────────────────────────────────────────────────────┘
```

**Why Batch Processing Made Sense**

1. **Not Real-Time Critical**: Search results don't need to reflect changes instantly
   - If you publish a new blog post, it's okay if it takes a few hours to appear in Google
   - Users don't expect instant indexing

2. **Eventual Consistency**: The search index eventually reflects the web's current state
   - Some delay is acceptable (and unavoidable at internet scale)
   - Running jobs hourly or daily provides good-enough freshness

3. **Resource Management**: Batch jobs can be scheduled during low-traffic periods
   - Run heavy computations when servers are less busy
   - Don't impact user-facing search queries

4. **Complex Processing**: Building PageRank requires multiple iterations
   - Can't be done in real-time
   - Batch approach allows convergence over multiple passes

**Historical Note**

One interesting observation from practitioners: "Just because you published a new website or a blog, it didn't instantly show up on Google search results." This wasn't always due to ranking—often it was simply because **Google hadn't run the indexing batch job yet**. The job to essentially "keep the entire internet indexed" is so massive that it takes time, even with thousands of servers.

Modern systems (like Google's current infrastructure) have evolved to provide faster updates using streaming and incremental processing, but the batch processing principles remain foundational.

### MapReduce Execution

**Job Workflow:**

```
1. Client submits job
     ↓
2. JobTracker (master) receives job
     ↓
3. JobTracker splits input into chunks
     ↓
4. TaskTracker (worker) nodes run map tasks
     ↓
5. Map outputs written to local disk
     ↓
6. Shuffle phase: sort and partition by key
     ↓
7. TaskTracker nodes run reduce tasks
     ↓
8. Reduce outputs written to HDFS
```

#### Understanding the Sorting Phase

The **sorting and shuffling phase** is the secret sauce that makes MapReduce work. Let's understand exactly what happens and why it's necessary.

**Why Sorting is Necessary**

Remember, the map phase produces key-value pairs, but:
1. **Keys are not unique**: The word "hello" might be emitted from hundreds of different mappers
2. **Keys are unsorted**: Mappers process data in arbitrary order
3. **Keys are distributed**: "hello" could appear in output from Mapper 1, Mapper 37, Mapper 542, etc.

The reducer needs **all values for a given key**, but the mapper outputs are scattered across hundreds or thousands of machines!

**The Sorting Solution**

The sorting phase ensures that:
1. All key-value pairs are **sorted by key**
2. All pairs with the **same key are grouped together**
3. Each group is **sent to exactly one reducer**

**Detailed Example: How Keys Get Grouped**

Let's trace what happens to the word "postgres" across the system:

```
MAP PHASE (Distributed)
═══════════════════════

Mapper 1 (Server 1):
  Processing article about databases...
  Outputs: ("postgres", 1), ("postgres", 1), ("mysql", 1), ...

Mapper 2 (Server 2):
  Processing article about SQL...
  Outputs: ("sql", 1), ("postgres", 1), ("oracle", 1), ...

Mapper 3 (Server 3):
  Processing tutorial...
  Outputs: ("postgres", 1), ("postgres", 1), ("postgres", 1), ...

...

Mapper 1000 (Server 1000):
  Processing blog post...
  Outputs: ("postgres", 1), ("python", 1), ...

─────────────────────────────────────────────────────────────

At this point, "postgres" appears in many mapper outputs, 
scattered across 1,000 different servers!

How do we collect them all together for counting?

─────────────────────────────────────────────────────────────

SORTING & SHUFFLE PHASE
════════════════════════

Step 1: Local Sort
Each mapper sorts its own output by key:

Mapper 1 output (sorted):
  [("hadoop", 1), ("mysql", 1), ("postgres", 1), ("postgres", 1), ...]

Mapper 2 output (sorted):
  [("oracle", 1), ("postgres", 1), ("sql", 1), ...]

Step 2: Partition Assignment
MapReduce uses a hash function to decide which reducer handles which keys:

  hash("postgres") % num_reducers = 3
  → All "postgres" keys go to Reducer 3

  hash("mysql") % num_reducers = 7
  → All "mysql" keys go to Reducer 7

  hash("python") % num_reducers = 3
  → All "python" keys go to Reducer 3

Step 3: Network Shuffle
Mapper outputs are transferred to their assigned reducers:

Mapper 1 → Sends ("postgres", 1), ("postgres", 1) → to Reducer 3
Mapper 2 → Sends ("postgres", 1) → to Reducer 3
Mapper 3 → Sends ("postgres", 1), ("postgres", 1), ("postgres", 1) → to Reducer 3
...
Mapper 1000 → Sends ("postgres", 1) → to Reducer 3

Step 4: Merge Sort at Reducer
Reducer 3 receives data from all mappers and merges it:

Input streams:
  Stream from Mapper 1:   [("postgres", 1), ("postgres", 1), ...]
  Stream from Mapper 2:   [("postgres", 1), ...]
  Stream from Mapper 3:   [("postgres", 1), ("postgres", 1), ("postgres", 1), ...]
  ...
  Stream from Mapper 1000: [("postgres", 1), ...]

Merged and sorted:
  [("postgres", 1), ("postgres", 1), ("postgres", 1), ..., ("postgres", 1)]
   ↑──────────────────── All 5,000 instances ───────────────────────────↑

─────────────────────────────────────────────────────────────

REDUCE PHASE
════════════

Reducer 3 receives:
  Key: "postgres"
  Values: [1, 1, 1, ..., 1]  ← Iterator over all 5,000 values

Reducer counts them:
  sum([1, 1, 1, ..., 1]) = 5,000

Reducer 3 outputs:
  ("postgres", 5000)
```

**Critical Insight: Why This Works**

The key guarantee is: **All instances of "postgres" from ALL mappers (1, 2, 3, ..., 1000) are sent to the SAME reducer (Reducer 3)**. 

This is why the reducer can confidently count—it knows it has received **every single instance** of the key from the entire dataset.

If mappers 1, 2, and 3 sent "postgres" to Reducer 1, but mappers 4 and 5 sent it to Reducer 2, the counts would be wrong! The sorting/shuffling phase prevents this.

**Visualizing the Shuffle**

```
                    MAPPERS (1,000 servers)
                    
    Mapper 1         Mapper 2         Mapper 3     ...    Mapper 1000
┌──────────────┐  ┌──────────────┐  ┌──────────────┐    ┌──────────────┐
│ (hello, 1)   │  │ (world, 1)   │  │ (spark, 1)   │    │ (hello, 1)   │
│ (world, 1)   │  │ (hello, 1)   │  │ (hello, 1)   │    │ (data, 1)    │
│ (postgres, 1)│  │ (postgres, 1)│  │ (postgres, 1)│    │ (postgres, 1)│
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘    └──────┬───────┘
       │                 │                 │                     │
       └─────────┬───────┴─────────┬───────┴─────────────┬──────┘
                 │                 │                     │
         ┌───────┴────────┬────────┴───────┐            │
         │                │                │            │
         ▼                ▼                ▼            ▼
    
                    REDUCERS (Variable number)
                    
    Reducer 1        Reducer 2        Reducer 3
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ All "hello"  │  │ All "world"  │  │ All "postgres"│
│ from every   │  │ from every   │  │ from every   │
│ mapper       │  │ mapper       │  │ mapper       │
│              │  │              │  │              │
│ (hello, [1,  │  │ (world, [1,  │  │ (postgres,[1,│
│  1, 1, ...]) │  │  1, 1, ...]) │  │  1, 1, ...]) │
└──────────────┘  └──────────────┘  └──────────────┘
```

### Understanding MapReduce Fault Tolerance

One of MapReduce's key innovations is **automatic fault tolerance**. When processing 100 TB of data across 1,000 servers running for hours, failures are not just possible—they're inevitable.

**Common Failure Scenarios**

1. **Server hardware failure**: Disk crash, memory error, power loss
2. **Network issues**: Switch failure, network partition
3. **Software bugs**: Mapper/reducer code crashes
4. **Resource exhaustion**: Out of memory, disk full

**How MapReduce Handles Failures**

#### Scenario 1: Mapper Task Fails Mid-Execution

```
Initial State (Map Phase in Progress)
═════════════════════════════════════

Mapper 1 (Server 1): ████████ 100% Complete ✓
Mapper 2 (Server 2): ████████ 100% Complete ✓
Mapper 3 (Server 3): ████──── 60% Complete... ✗ CRASH!
Mapper 4 (Server 4): ████████ 100% Complete ✓
Mapper 5 (Server 5): ██────── 30% Complete...
...
```

**What Happens:**

```
1. TaskTracker on Server 3 stops sending heartbeats
     ↓
2. JobTracker detects missing heartbeat (timeout: 10 minutes)
     ↓
3. JobTracker marks Mapper 3 as failed
     ↓
4. JobTracker reschedules Mapper 3 on another available server
     ↓
5. New Mapper 3 restarts from beginning (re-reads input data)
     ↓
6. Job continues once Mapper 3 completes
```

**Why Restart from Beginning?**

- Mapper tasks are **stateless** and **idempotent**
- The input data is immutable in HDFS
- Partial progress is discarded (simpler than checkpointing)

#### Scenario 2: Server Dies During Map Phase

**The Problem:**

When a server dies completely, it affects not just the mapper running on it, but also the **output** that mapper produced.

```
Map Phase Complete, Starting Reduce Phase
══════════════════════════════════════════

Server 1: Mapper 1 output → [temp file on local disk]
Server 2: Mapper 2 output → [temp file on local disk]  ✗ SERVER DIES!
Server 3: Mapper 3 output → [temp file on local disk]
...

Reduce Phase Needs:
  Reducer 1 needs data from: Mapper 1, Mapper 2✗, Mapper 3, ...
  Reducer 2 needs data from: Mapper 1, Mapper 2✗, Mapper 3, ...
  ...
```

**The Cascade Effect:**

When Server 2 dies:
1. Mapper 2's output (stored on Server 2's local disk) is **lost**
2. **ALL reducers** need data from Mapper 2
3. Even if Mapper 2 finished successfully, its output is gone
4. **The entire map phase must be restarted**

```
Why ALL Reducers Are Affected
══════════════════════════════

Mapper 2 produced output like:
  ("hello", 1), ("world", 1), ("python", 1), ...

These keys get partitioned to different reducers:
  "hello"  → Reducer 1
  "world"  → Reducer 2
  "python" → Reducer 3
  ...

So Mapper 2 sent data to ALL reducers!

If any part of Mapper 2's output is lost,
NO reducer can confidently say it has received
all data for its keys.
```

**MapReduce's Solution:**

```
1. JobTracker detects Server 2 failure
     ↓
2. Mark ALL tasks on Server 2 as failed
     ↓
3. Reschedule Mapper 2 on another server
     ↓
4. Wait for new Mapper 2 to complete
     ↓
5. Delete any partial output from original Mapper 2
     ↓
6. Notify ALL reducers to ignore old Mapper 2 data
     ↓
7. Continue reduce phase with new Mapper 2 output
```

#### Scenario 3: Reducer Failure

Reducer failures are simpler to handle because reducer output goes to HDFS (which is replicated):

```
Reducer 1: Processing... 100% → Output written to HDFS ✓
Reducer 2: Processing... 60% → ✗ CRASH!
Reducer 3: Processing... 100% → Output written to HDFS ✓

Recovery:
─────────
1. JobTracker reschedules Reducer 2
2. New Reducer 2 re-processes its assigned keys
3. Overwrites output in HDFS (safe because idempotent)
4. Job completes
```

**No Side Effects Design**

MapReduce is designed to have **no side effects**:
- Mappers only read input, write output
- Reducers only read input, write output
- No database updates, no external API calls
- This makes tasks **safely restartable**

If mappers/reducers had side effects (e.g., "update database row"), restarting tasks would cause problems (double updates). MapReduce avoids this by keeping all work purely functional.

**Speculative Execution: Handling Stragglers**

Sometimes a task isn't failing—it's just slow (a "straggler"):

```
Mapper 1: ████████ Complete (10 min)
Mapper 2: ████████ Complete (10 min)
Mapper 3: ████████ Complete (10 min)
Mapper 4: ████──── Still running... (30 min and counting)
             ↑ Slow disk? Overloaded CPU? Bad hardware?
```

**Solution: Launch Duplicate Task**

```
1. JobTracker notices Mapper 4 is slow
     ↓
2. Launch duplicate Mapper 4 on different server
     ↓
3. Whichever finishes first wins
     ↓
4. Kill the slower one
```

This **speculative execution** prevents a single slow machine from bottlenecking the entire job.

### Combiner Function (Optimization)

**Problem**: Shuffle phase transfers large amounts of data.

**Solution**: Run a local reduce before shuffle.

```python
class WordCountOptimized(MRJob):
    
    def mapper(self, _, line):
        for word in line.split():
            yield (word.lower(), 1)
    
    def combiner(self, key, values):
        """Local aggregation (runs on map node)"""
        yield (key, sum(values))
    
    def reducer(self, key, values):
        """Final aggregation"""
        yield (key, sum(values))
```

**Without combiner:**
```
Map outputs: (hello,1), (hello,1), (hello,1), (world,1), (world,1)
Shuffle transfers: 5 key-value pairs
```

**With combiner:**
```
Map outputs: (hello,1), (hello,1), (hello,1), (world,1), (world,1)
After combiner: (hello,3), (world,2)
Shuffle transfers: 2 key-value pairs  ← 60% reduction!
```

## Part 3: Beyond MapReduce: Distributed Dataflow Engines

### Limitations of MapReduce

**1. Materialization of Intermediate State**

Every MapReduce job writes output to HDFS. For multi-stage jobs:

```
Job 1: Read from HDFS → Map → Reduce → Write to HDFS
Job 2: Read from HDFS → Map → Reduce → Write to HDFS
Job 3: Read from HDFS → Map → Reduce → Write to HDFS

Problems:
- Excessive I/O (write + read intermediate results)
- High latency (wait for each job to finish)
- Wasted resources (redundant serialization/deserialization)
```

**2. Inflexible Programming Model**

Only Map and Reduce operations. Complex algorithms require chaining many jobs.

**3. Developer Experience: Power vs. Complexity**

While MapReduce is powerful and flexible, it comes with a significant learning curve and cognitive overhead.

#### The Flexibility Trade-Off

MapReduce gives you **immense flexibility**:
- Write arbitrary map and reduce functions in any language
- Process any data format
- Implement any algorithm
- Leverage thousands of machines automatically

But this flexibility comes at a cost: **you have to think in the MapReduce paradigm**.

**The Challenge**

For developers, MapReduce requires fundamentally rethinking problems:

```
Traditional Approach (SQL):
──────────────────────────
SELECT category, AVG(price), COUNT(*)
FROM products
GROUP BY category
HAVING COUNT(*) > 100

"Show me average price per category for categories with >100 products"

Clear, declarative, expresses WHAT you want.
```

```
MapReduce Approach:
──────────────────
function mapper(product) {
  emit(product.category, {price: product.price, count: 1});
}

function combiner(category, values) {
  let totalPrice = 0, totalCount = 0;
  for (const v of values) {
    totalPrice += v.price * v.count;
    totalCount += v.count;
  }
  emit(category, {price: totalPrice, count: totalCount});
}

function reducer(category, values) {
  let totalPrice = 0, totalCount = 0;
  for (const v of values) {
    totalPrice += v.price;
    totalCount += v.count;
  }
  
  if (totalCount > 100) {
    emit(category, totalPrice / totalCount);
  }
}

Must think: "How do I break this into map and reduce steps?"
```

**As one practitioner describes it:**

> "If you just handed me an arbitrary problem right now and said, 'Hey, here's this arbitrarily complex problem that I want to solve in my big data set that I have,' it would take me a while to sit down and think through: How do I write the proper mapping? How do I write the proper reduction?"

Even experienced engineers need time to decompose problems into the map-reduce pattern.

**Common Mental Hurdles**

1. **Thinking in Key-Value Pairs**
   - Everything must be expressed as (key, value) emissions
   - Choosing the right key is critical (determines grouping)
   - Sometimes requires creative key design

2. **State Limitation**
   - Mappers can't share state
   - Reducers can't access external data easily
   - Must encode everything in keys/values

3. **Multi-Stage Pipelines**
   - Complex algorithms need multiple MapReduce jobs
   - Must manually chain jobs together
   - Intermediate data serialized/deserialized repeatedly

**Example: Complexity in Practice**

**Problem**: Find users who bought products from at least 3 different categories

**SQL (Simple)**:
```sql
SELECT user_id
FROM purchases p
JOIN products pr ON p.product_id = pr.id
GROUP BY user_id
HAVING COUNT(DISTINCT pr.category) >= 3
```

**MapReduce (Complex)**:
```javascript
// Job 1: Join purchases with products
function mapPurchases(purchase) {
  emit(purchase.product_id, {type: 'purchase', user_id: purchase.user_id});
}

function mapProducts(product) {
  emit(product.id, {type: 'product', category: product.category});
}

function reduceJoin(product_id, records) {
  let purchases = [], category = null;
  for (const r of records) {
    if (r.type === 'purchase') purchases.push(r);
    else category = r.category;
  }
  for (const p of purchases) {
    emit(p.user_id, category);
  }
}

// Job 2: Count distinct categories per user
function mapCategories(user_id, category) {
  emit(user_id, category);
}

function reduceCountCategories(user_id, categories) {
  const unique = new Set(categories);
  if (unique.size >= 3) {
    emit(user_id, unique.size);
  }
}
```

Requires **two separate MapReduce jobs**, explicit join logic, and careful state management. SQL handles this in one declarative query.

#### SQL-Like Layers: Abstracting Complexity

The community recognized this usability challenge and built higher-level abstractions on top of MapReduce:

**Apache Hive** (SQL on Hadoop):
```sql
-- Write familiar SQL
SELECT category, AVG(price)
FROM products
GROUP BY category;

-- Hive compiler translates to MapReduce jobs behind the scenes
```

**Apache Pig** (Dataflow scripting):
```
products = LOAD 'products.csv' USING PigStorage(',');
grouped = GROUP products BY category;
averages = FOREACH grouped GENERATE 
  group AS category,
  AVG(products.price) AS avg_price;
STORE averages INTO 'output';

-- Pig translates to optimized MapReduce jobs
```

**The Trade-Off**

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Maximum Flexibility                                   │
│   (Raw MapReduce)                                       │
│   ↕                                                     │
│   High Complexity                                       │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Moderate Flexibility                                  │
│   (Pig, Cascading)                                      │
│   ↕                                                     │
│   Moderate Complexity                                   │
│                                                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Limited Flexibility                                   │
│   (Hive SQL)                                            │
│   ↕                                                     │
│   Low Complexity                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

- **Raw MapReduce**: Can express any algorithm, but requires expert-level thinking
- **Pig/Cascading**: Balanced—more expressive than SQL, less complex than raw MapReduce
- **Hive**: Easy for SQL users, but limited to SQL-expressible operations

**When to Use What?**

Use **raw MapReduce** when:
- Implementing novel algorithms
- Needing fine-grained control over performance
- Working on problems that don't fit declarative models

Use **Hive/Pig** when:
- Doing standard data transformations
- Team familiar with SQL/scripting
- Rapid development more important than optimal performance

Modern systems like Spark provide the best of both worlds: high-level APIs (DataFrames, SQL) with the ability to drop down to low-level RDDs when needed.

### Apache Spark

**Spark** improves upon MapReduce with:
- **In-memory processing**: Avoid disk writes
- **Rich API**: Transformations beyond Map/Reduce
- **Lazy evaluation**: Optimize entire workflow
- **Interactive queries**: REPL for exploratory analysis

**Core Abstraction: RDD (Resilient Distributed Dataset)**

```python
from pyspark import SparkContext

sc = SparkContext("local", "WordCount")

# Read input
lines = sc.textFile("input.txt")

# Transformations (lazy)
words = lines.flatMap(lambda line: line.split())
word_pairs = words.map(lambda word: (word, 1))
word_counts = word_pairs.reduceByKey(lambda a, b: a + b)

# Action (triggers execution)
results = word_counts.collect()

for word, count in results:
    print(f"{word}: {count}")
```

**RDD Operations:**

**Transformations** (lazy, return new RDD):
- `map()`, `flatMap()`, `filter()`
- `reduceByKey()`, `groupByKey()`, `sortByKey()`
- `join()`, `union()`, `intersection()`

**Actions** (trigger execution):
- `collect()`, `count()`, `first()`, `take(n)`
- `reduce()`, `fold()`, `saveAsTextFile()`

**Visual: Spark Execution**

```
┌──────────────────────────────────────────────────┐
│              Driver Program                      │
│  lines = sc.textFile("input")                    │
│  words = lines.flatMap(...)                      │
│  pairs = words.map(...)                          │
│  counts = pairs.reduceByKey(...)                 │
│  counts.saveAsTextFile("output")                 │
└──────────────────────────────────────────────────┘
                      │
                      │ Creates DAG (Directed Acyclic Graph)
                      ▼
┌──────────────────────────────────────────────────┐
│               DAG Scheduler                      │
│  Stage 1: textFile → flatMap → map              │
│  Stage 2: reduceByKey → saveAsTextFile          │
└──────────────────────────────────────────────────┘
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
┌───────────┐  ┌───────────┐  ┌───────────┐
│ Worker 1  │  │ Worker 2  │  │ Worker 3  │
│ Executor  │  │ Executor  │  │ Executor  │
│ Tasks     │  │ Tasks     │  │ Tasks     │
│ Cache     │  │ Cache     │  │ Cache     │
└───────────┘  └───────────┘  └───────────┘
```

### Spark SQL and DataFrames

**DataFrame**: Structured data with named columns (like SQL table).

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("Example").getOrCreate()

# Read JSON data
users = spark.read.json("users.json")

# Schema
users.printSchema()
# root
#  |-- name: string
#  |-- age: long
#  |-- email: string

# SQL-like operations
users.select("name", "age").show()

# Filter
adults = users.filter(users.age >= 18)

# Group by and aggregate
age_groups = users.groupBy("age").count()

# SQL queries
users.createOrReplaceTempView("users")
result = spark.sql("SELECT age, COUNT(*) FROM users GROUP BY age")
result.show()
```

**Optimizations (Catalyst Optimizer):**

```
Logical Plan:
  SELECT name, age FROM users WHERE age > 18

Optimized Physical Plan:
  1. Scan JSON (push down filter: age > 18)
  2. Project (name, age)  ← Only read needed columns

Optimizations:
- Predicate pushdown
- Column pruning
- Constant folding
- Join reordering
```

### Apache Flink: Stream and Batch Unification

**Flink** treats batch as a special case of streaming.

**Key Features:**
- **True streaming**: Process events as they arrive (not micro-batches)
- **Event time processing**: Handle out-of-order events correctly
- **State management**: Fault-tolerant stateful computations
- **Exactly-once semantics**: No data loss or duplication

**Batch Processing with Flink:**

```python
from flink import ExecutionEnvironment

env = ExecutionEnvironment.get_execution_environment()

# Read input
lines = env.read_text_file("input.txt")

# Word count
word_counts = lines \
    .flat_map(lambda line: line.split()) \
    .map(lambda word: (word, 1)) \
    .group_by(0) \
    .reduce(lambda a, b: (a[0], a[1] + b[1]))

# Write output
word_counts.write_text("output.txt")

# Execute
env.execute("Batch Word Count")
```

## Part 4: Joins in Batch Processing

Joins are one of the most important—and most complex—operations in batch processing. When working with large datasets distributed across hundreds of machines, performing joins efficiently requires careful strategy.

### Types of Joins

**1. Reduce-Side Join (Sort-Merge Join)**

This is the most general-purpose join strategy, but also the most expensive because it requires shuffling data over the network.

**When to use**: 
- Both datasets are large
- Data is not pre-sorted
- Most common case

Both datasets are sorted and merged by reducer.

**Example: Join users and purchases**

```
Users:
user_id, name
1, Alice
2, Bob
3, Carol

Purchases:
purchase_id, user_id, amount
101, 1, 50
102, 2, 30
103, 1, 25
```

**How Reduce-Side Joins Work**

The key insight: **use the join key as the MapReduce key** so that matching records end up at the same reducer.

```
STEP 1: Map Phase (Tag Records)
════════════════════════════════

User Mapper:
  Input: (1, "Alice")
  Output: emit(1, {type: "USER", name: "Alice"})
  
  Input: (2, "Bob")
  Output: emit(2, {type: "USER", name: "Bob"})

Purchase Mapper:
  Input: (101, 1, 50)
  Output: emit(1, {type: "PURCHASE", purchase_id: 101, amount: 50})
  
  Input: (102, 2, 30)
  Output: emit(2, {type: "PURCHASE", purchase_id: 102, amount: 30})
  
  Input: (103, 1, 25)
  Output: emit(1, {type: "PURCHASE", purchase_id: 103, amount: 25})

STEP 2: Shuffle & Sort
═══════════════════════

All records with user_id=1 go to same reducer:
  Reducer 1: 
    key=1, values=[
      {type: "USER", name: "Alice"},
      {type: "PURCHASE", purchase_id: 101, amount: 50},
      {type: "PURCHASE", purchase_id: 103, amount: 25}
    ]

All records with user_id=2 go to same reducer:
  Reducer 2:
    key=2, values=[
      {type: "USER", name: "Bob"},
      {type: "PURCHASE", purchase_id: 102, amount: 30}
    ]

STEP 3: Reduce Phase (Perform Join)
════════════════════════════════════

Reducer 1 (user_id=1):
  - Finds 1 USER record: "Alice"
  - Finds 2 PURCHASE records: [101, $50], [103, $25]
  - Joins them:
    Output: (1, "Alice", 101, 50)
    Output: (1, "Alice", 103, 25)

Reducer 2 (user_id=2):
  - Finds 1 USER record: "Bob"
  - Finds 1 PURCHASE record: [102, $30]
  - Joins them:
    Output: (2, "Bob", 102, 30)
```

**MapReduce Implementation:**

```python
class JoinMapper(MRJob):
    
    def mapper(self, _, line):
        """Emit (user_id, record) for both datasets"""
        parts = line.split(',')
        if len(parts) == 2:  # User record
            user_id, name = parts
            yield (user_id, ('USER', name))
        elif len(parts) == 3:  # Purchase record
            purchase_id, user_id, amount = parts
            yield (user_id, ('PURCHASE', purchase_id, amount))
    
    def reducer(self, user_id, records):
        """Join records with same user_id"""
        user_name = None
        purchases = []
        
        for record in records:
            if record[0] == 'USER':
                user_name = record[1]
            else:  # PURCHASE
                purchases.append(record[1:])
        
        # Emit joined records
        for purchase_id, amount in purchases:
            yield (user_id, f"{user_name},{purchase_id},{amount}")
```

**Execution Flow:**

```
Map Phase:
User file → (1,'USER','Alice'), (2,'USER','Bob'), ...
Purchase file → (1,'PURCHASE',101,50), (1,'PURCHASE',103,25), ...

Shuffle & Sort (by user_id):
Reducer 1: (1, ['USER','Alice'], ['PURCHASE',101,50], ['PURCHASE',103,25])
Reducer 2: (2, ['USER','Bob'], ['PURCHASE',102,30])
...

Reduce Phase:
Reducer 1 outputs:
  (1, 'Alice,101,50')
  (1, 'Alice,103,25')
Reducer 2 outputs:
  (2, 'Bob,102,30')
```

**Performance Considerations**

Reduce-side joins require **shuffling ALL data** over the network:

```
Network Traffic:
══════════════
Users table: 1 GB → All sent to reducers
Purchases table: 10 GB → All sent to reducers
Total shuffle: 11 GB of network transfer

For large datasets (terabytes), this is expensive!
```

**2. Map-Side Join (Broadcast Join)**

When one dataset is **small enough to fit in memory**, we can avoid the shuffle entirely by broadcasting the small dataset to all mappers.

**When to use**: 
- One dataset is small (fits in memory, typically <100MB-1GB)
- Other dataset is large
- Called "broadcast join" or "replicated join"

If one dataset fits in memory, broadcast it to all mappers.

**When to use**: Small dimension table + large fact table.

**Example Scenario**

```
Users table (small): 1,000 users = ~100 MB
Purchases table (large): 100 million purchases = ~10 GB

Instead of shuffling 10 GB, broadcast 100 MB to all mappers!
```

**How Map-Side Joins Work**

```
SETUP: Load Small Table Into Memory
════════════════════════════════════

Each mapper loads users.txt into a hash map:

users_map = {
  1: "Alice",
  2: "Bob",
  3: "Carol",
  ...
}

MAP PHASE: Join On-the-Fly
═══════════════════════════

Mapper 1 (processing part of purchases):
  
  Input: (101, 1, 50)
  Lookup: users_map[1] = "Alice"
  Output: (1, "Alice", 101, 50)  ← Joined!
  
  Input: (102, 2, 30)
  Lookup: users_map[2] = "Bob"
  Output: (2, "Bob", 102, 30)  ← Joined!

Mapper 2 (processing another part of purchases):
  
  Input: (103, 1, 25)
  Lookup: users_map[1] = "Alice"
  Output: (1, "Alice", 103, 25)  ← Joined!

NO SHUFFLE PHASE NEEDED!
NO REDUCE PHASE NEEDED!

Final output is already joined.
```

**Code Implementation:**

```python
class BroadcastJoin(MRJob):
    
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.users = {}  # Load small table into memory
    
    def mapper_init(self):
        """Load user data (small table) into memory"""
        with open('users.txt') as f:
            for line in f:
                user_id, name = line.strip().split(',')
                self.users[user_id] = name
    
    def mapper(self, _, line):
        """Join on the fly (no shuffle needed!)"""
        purchase_id, user_id, amount = line.split(',')
        user_name = self.users.get(user_id, 'Unknown')
        yield (user_id, f"{user_name},{purchase_id},{amount}")
    
    # No reducer needed!
```

**Performance Comparison:**

```
Reduce-Side Join:
  - Shuffle ALL data (expensive)
  - Suitable for any dataset sizes
  - Time: O(n log n) for sorting
  - Network: 100% of data transferred

Map-Side Join:
  - No shuffle (cheap!)
  - Requires small table fits in memory
  - Time: O(n) for large table (hash lookup is O(1))
  - Network: Only small table transferred once
```

**Real-World Example: Analytics Query**

```
Scenario: Compute total revenue per country

Tables:
  users (1 GB):     user_id, country
  purchases (1 TB): user_id, amount

Map-Side Join Approach:
  1. Broadcast users table (1 GB) to all mappers
  2. Each mapper joins purchases with users (in memory)
  3. Emit: (country, amount)
  4. Reduce: Sum amounts per country
  
Network traffic: 1 GB (broadcast) + minimal shuffle
vs.
Reduce-Side Join: 1 TB + 1 GB shuffled = 1,000x more expensive!
```

**3. Map-Side Join with Pre-Sorted Data**

An even more efficient variant: if **both datasets are already sorted and partitioned by join key**, we can join them during the map phase without any shuffle.

**Requirements:**
- Both datasets sorted by join key
- Both datasets partitioned identically
- Same number of partitions

**How It Works:**

```
Dataset A Partition 1: [user_ids 1-1000, sorted]
Dataset B Partition 1: [user_ids 1-1000, sorted]
↓
Mapper 1: Merge-joins both partitions locally

Dataset A Partition 2: [user_ids 1001-2000, sorted]
Dataset B Partition 2: [user_ids 1001-2000, sorted]
↓
Mapper 2: Merge-joins both partitions locally

No shuffle needed—data already co-located and sorted!
```

This is the **fastest possible join**, but requires pre-processing data into the right format (which may itself require MapReduce jobs).

**4. Handling Data Skew in Joins**

**The Problem:**

Some keys have many more records than others ("hot keys"):

```
Users:
  user_id 999: "John" (famous celebrity)
  
Purchases:
  user_id 999: 10,000,000 purchase records!
  Other users: ~10 purchases each

In reduce-side join:
  Reducer assigned to user_id 999: Processing 10M records
  Other reducers: Processing ~10 records each
  
Result: One reducer becomes a bottleneck, slowing entire job!
```

**Solution: Salting**

Add a random "salt" to hot keys to distribute them across multiple reducers:

```javascript
// For hot keys, add random suffix
function mapper_with_salting(record) {
  const key = record.join_key;
  
  if (is_hot_key(key)) {
    // Split across 10 reducers
    const salt = Math.floor(Math.random() * 10);
    emit(`${key}_${salt}`, record);
  } else {
    emit(key, record);
  }
}
```

For small table, replicate it:

```javascript
// Small table: replicate with all salt values
function mapper_small_table(record) {
  const key = record.join_key;
  
  if (is_hot_key(key)) {
    // Send copy to all 10 reducers
    for (let salt = 0; salt < 10; salt++) {
      emit(`${key}_${salt}`, record);
    }
  } else {
    emit(key, record);
  }
}
```

Now the hot key is processed by 10 reducers in parallel instead of one!

### Performance Comparison:

### Basic Aggregation

```python
# Spark example: Average purchase by user
purchases = spark.read.csv("purchases.csv")

avg_per_user = purchases \
    .groupBy("user_id") \
    .agg({"amount": "avg"})

avg_per_user.show()
```

**Execution:**

```
Input:
(user_id, amount)
(1, 50), (1, 25), (2, 30), (2, 40), (2, 20)

After groupBy:
1 → [50, 25]
2 → [30, 40, 20]

After agg (avg):
1 → 37.5
2 → 30.0
```

### Windowing Functions

**Window functions** compute over a sliding window.

```python
from pyspark.sql.window import Window
from pyspark.sql.functions import row_number, rank, avg

# Rank products by sales within each category
window = Window.partitionBy("category").orderBy(desc("sales"))

ranked = products \
    .withColumn("rank", row_number().over(window)) \
    .filter(col("rank") <= 10)  # Top 10 per category

# Moving average
window = Window.partitionBy("user_id").orderBy("date").rowsBetween(-7, 0)

moving_avg = purchases \
    .withColumn("avg_7d", avg("amount").over(window))
```

## Part 6: Handling Data Skew

### Problem: Data Skew

```
Ideal (balanced):
Worker 1: ████████ (10k records)
Worker 2: ████████ (10k records)
Worker 3: ████████ (10k records)
Time: 10 minutes

Skewed:
Worker 1: ████████████████████████ (50k records) ← Bottleneck!
Worker 2: ████ (5k records) ← Idle
Worker 3: ████ (5k records) ← Idle
Time: 30 minutes (3x slower!)
```

### Solutions

**1. Salting (for joins)**

```python
def skew_resistant_join(large_df, small_df):
    """Add salt to distribute skewed keys"""
    # Replicate small df with salts
    salts = [i for i in range(10)]
    small_df_salted = small_df \
        .withColumn("salt", explode(array([lit(s) for s in salts]))) \
        .withColumn("join_key_salted", concat(col("join_key"), lit("_"), col("salt")))
    
    # Add salt to large df
    large_df_salted = large_df \
        .withColumn("salt", (rand() * 10).cast("int")) \
        .withColumn("join_key_salted", concat(col("join_key"), lit("_"), col("salt")))
    
    # Join on salted key
    result = large_df_salted.join(small_df_salted, "join_key_salted")
    
    return result
```

**2. Adaptive Query Execution (Spark 3.0+)**

Spark automatically detects and handles skew:

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")

# Spark will automatically split skewed partitions!
```

## Part 7: Workflows and Dependencies

### Workflow Orchestration

**Batch jobs** often have complex dependencies:

```
Example: E-commerce analytics pipeline

    ┌─────────────────┐
    │  Raw Logs       │
    └────────┬────────┘
             │
             ├──────────────┬──────────────┐
             ▼              ▼              ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │Parse Clickst.│ │Parse Purchases│ │Parse User Reg│
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
           └────────┬───────┴────────────────┘
                    ▼
          ┌──────────────────┐
          │  Join All Data   │
          └────────┬─────────┘
                   │
           ┌───────┴───────┐
           ▼               ▼
    ┌──────────────┐ ┌──────────────┐
    │User Cohorts  │ │Product Rank  │
    └──────┬───────┘ └──────┬───────┘
           │                │
           └────────┬───────┘
                    ▼
          ┌──────────────────┐
          │  Export to DB    │
          └──────────────────┘
```

### Apache Airflow

**Airflow** is a workflow orchestration platform.

**DAG (Directed Acyclic Graph) Definition:**

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'ecommerce_analytics',
    default_args=default_args,
    schedule_interval='@daily',  # Run once per day
    catchup=False
)

# Task 1: Parse raw logs
parse_clicks = BashOperator(
    task_id='parse_clicks',
    bash_command='spark-submit parse_clicks.py --date {{ ds }}',
    dag=dag
)

parse_purchases = BashOperator(
    task_id='parse_purchases',
    bash_command='spark-submit parse_purchases.py --date {{ ds }}',
    dag=dag
)

parse_users = BashOperator(
    task_id='parse_users',
    bash_command='spark-submit parse_users.py --date {{ ds }}',
    dag=dag
)

# Task 2: Join data
join_data = BashOperator(
    task_id='join_data',
    bash_command='spark-submit join_data.py --date {{ ds }}',
    dag=dag
)

# Task 3: Compute aggregates
compute_cohorts = PythonOperator(
    task_id='compute_cohorts',
    python_callable=compute_user_cohorts,
    dag=dag
)

rank_products = PythonOperator(
    task_id='rank_products',
    python_callable=rank_products_by_sales,
    dag=dag
)

# Task 4: Export results
export_to_db = BashOperator(
    task_id='export_to_db',
    bash_command='python export.py --date {{ ds }}',
    dag=dag
)

# Define dependencies
parse_clicks >> join_data
parse_purchases >> join_data
parse_users >> join_data

join_data >> compute_cohorts >> export_to_db
join_data >> rank_products >> export_to_db
```

**Airflow UI:**

```
DAG View:
┌────────────────────────────────────────────────┐
│ ecommerce_analytics                           │
│ Schedule: @daily                              │
│ Last Run: 2024-01-15 00:00:00                 │
│ Next Run: 2024-01-16 00:00:00                 │
├────────────────────────────────────────────────┤
│ Tasks:                                        │
│  ✓ parse_clicks       (success)              │
│  ✓ parse_purchases    (success)              │
│  ✓ parse_users        (success)              │
│  ✓ join_data          (success)              │
│  ⟳ compute_cohorts    (running...)           │
│  ⧗ rank_products      (queued)               │
│  ⧗ export_to_db       (queued)               │
└────────────────────────────────────────────────┘
```

### Luigi (Alternative to Airflow)

```python
import luigi

class ParseClicks(luigi.Task):
    date = luigi.DateParameter()
    
    def output(self):
        return luigi.LocalTarget(f'data/clicks_{self.date}.parquet')
    
    def run(self):
        # Parse logs
        parsed = parse_click_logs(self.date)
        parsed.to_parquet(self.output().path)

class JoinData(luigi.Task):
    date = luigi.DateParameter()
    
    def requires(self):
        return [
            ParseClicks(self.date),
            ParsePurchases(self.date),
            ParseUsers(self.date)
        ]
    
    def output(self):
        return luigi.LocalTarget(f'data/joined_{self.date}.parquet')
    
    def run(self):
        # Load inputs
        clicks = pd.read_parquet(self.input()[0].path)
        purchases = pd.read_parquet(self.input()[1].path)
        users = pd.read_parquet(self.input()[2].path)
        
        # Join
        joined = join_all_data(clicks, purchases, users)
        joined.to_parquet(self.output().path)
```

## Part 8: Fault Tolerance in Batch Processing

### MapReduce Fault Tolerance

**Task Failure:**
```
TaskTracker reports failure to JobTracker
→ JobTracker reschedules task on another node
→ Task re-runs from beginning
```

**Node Failure:**
```
JobTracker detects missing heartbeat
→ All tasks on that node marked as failed
→ Tasks rescheduled on other nodes
```

**Speculative Execution:**
```
If task is slow (straggler):
  → JobTracker launches duplicate task on another node
  → Use result from whichever finishes first
```

### Spark Fault Tolerance (RDD Lineage)

**RDD Lineage**: Remember how RDD was computed.

```python
# Original RDD
lines = sc.textFile("input.txt")

# Transformation creates lineage
words = lines.flatMap(lambda line: line.split())
pairs = words.map(lambda word: (word, 1))
counts = pairs.reduceByKey(lambda a, b: a + b)

# Lineage graph:
#   input.txt → lines → words → pairs → counts

# If partition of 'counts' is lost:
#   → Recompute from input using lineage
```

**Checkpointing**: Save RDD to disk periodically.

```python
# Checkpoint after expensive computation
pairs.checkpoint()

# If failure occurs after checkpoint:
#   → Recover from checkpoint (faster than recomputing)
```

## Part 9: Real-World Case Studies

### Case Study 1: Netflix Recommendation Engine

**Pipeline:**

```
1. Collect viewing data (Kafka → S3)
   ↓
2. Daily batch job (Spark on EMR)
   - Join user data + viewing history + ratings
   - Train recommendation model (ALS algorithm)
   ↓
3. Generate top-N recommendations per user
   ↓
4. Load into Cassandra for serving
```

**Spark Job:**

```python
# Load data
views = spark.read.parquet("s3://netflix-data/views/")
ratings = spark.read.parquet("s3://netflix-data/ratings/")

# Collaborative filtering (ALS)
from pyspark.ml.recommendation import ALS

als = ALS(
    userCol="user_id",
    itemCol="movie_id",
    ratingCol="rating",
    coldStartStrategy="drop"
)

model = als.fit(ratings)

# Generate recommendations
users = ratings.select("user_id").distinct()
recommendations = model.recommendForUserSubset(users, 50)

# Save to Cassandra
recommendations.write \
    .format("org.apache.spark.sql.cassandra") \
    .options(table="recommendations", keyspace="netflix") \
    .mode("overwrite") \
    .save()
```

### Case Study 2: Spotify User Segmentation

**Goal**: Cluster users based on listening behavior.

```python
# Feature engineering
user_features = spark.sql("""
    SELECT 
        user_id,
        COUNT(DISTINCT artist) as artist_diversity,
        COUNT(*) as total_plays,
        AVG(song_duration) as avg_song_duration,
        COUNT(DISTINCT CAST(play_time AS DATE)) as days_active,
        SUM(CASE WHEN hour(play_time) BETWEEN 22 AND 6 THEN 1 ELSE 0 END) as night_plays
    FROM listening_history
    WHERE date >= current_date() - INTERVAL 30 DAYS
    GROUP BY user_id
""")

# K-Means clustering
from pyspark.ml.clustering import KMeans
from pyspark.ml.feature import VectorAssembler

assembler = VectorAssembler(
    inputCols=["artist_diversity", "total_plays", "avg_song_duration", 
               "days_active", "night_plays"],
    outputCol="features"
)

feature_vectors = assembler.transform(user_features)

kmeans = KMeans(k=5, seed=42)
model = kmeans.fit(feature_vectors)

segments = model.transform(feature_vectors)

# Analyze segments
segments.groupBy("prediction").agg(
    avg("total_plays").alias("avg_plays"),
    avg("artist_diversity").alias("avg_diversity")
).show()
```

## Summary

**Key Takeaways:**

1. **Unix philosophy** influences batch processing design
   - Simple, composable operations
   - Text streams as uniform interface

2. **MapReduce** democratized large-scale batch processing
   - Map: Independent transformations
   - Reduce: Aggregations
   - Automatic parallelization and fault tolerance

3. **HDFS** provides scalable, fault-tolerant storage
   - Large blocks (64MB-128MB)
   - Replication for durability
   - Data locality for performance

4. **Modern dataflow engines** improve on MapReduce
   - **Spark**: In-memory processing, rich API
   - **Flink**: True streaming + batch
   - **Dataflow optimizations**: Push down filters, avoid shuffles

5. **Joins** are critical batch operation
   - Reduce-side join: General purpose
   - Map-side join: When one side is small
   - Handle skew with salting or adaptive execution

6. **Workflow orchestration** manages complex pipelines
   - DAG-based (Airflow, Luigi)
   - Scheduling, retries, monitoring

7. **Fault tolerance** via lineage or checkpointing
   - Recompute lost partitions from source
   - Trade-off: Recovery speed vs overhead

**Batch vs Streaming:**

| Aspect | Batch | Streaming |
|--------|-------|-----------|
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Input** | Bounded (files) | Unbounded (streams) |
| **Use Case** | Reports, ML training | Real-time alerts, dashboards |
| **Complexity** | Simpler | More complex (windowing, state) |

### From Batch to Stream Processing

As we've seen throughout this chapter, batch processing is powerful for handling large volumes of data offline. But what if you need results faster than "tonight's batch job"?

**The Evolution of Data Processing**

```
1990s-2000s: Databases
─────────────────────
- Process data as it arrives
- Immediate results
- Limited to single-machine scale

2000s-2010s: Batch Processing (MapReduce)
──────────────────────────────────────────
- Process petabytes of data
- High latency (hours)
- Great for analytics, ETL, ML training

2010s-Present: Stream Processing
─────────────────────────────────
- Process data as it arrives (like databases)
- At any scale (like MapReduce)
- Low latency (seconds to minutes)
- Best of both worlds!
```

**Why Streaming Matters**

As one practitioner notes: "I feel like if anything, stream processing technologies have just continued to get more and more important." Modern applications increasingly need:

1. **Real-time analytics**: Track metrics as events happen
   - Website traffic dashboards
   - Fraud detection
   - System monitoring

2. **Continuous data pipelines**: Keep data flowing
   - Log aggregation
   - Metrics collection
   - CDC (Change Data Capture) from databases

3. **Event-driven architectures**: React to events immediately
   - User notifications
   - Automated workflows
   - Alerting systems

**Batch vs. Stream: Complementary, Not Competing**

```
Use Batch Processing When:
──────────────────────────
✓ Processing historical data
✓ Complex, multi-stage computations
✓ Results can wait (hours/days)
✓ Example: Monthly financial reports, ML model training

Use Stream Processing When:
────────────────────────────
✓ Need real-time or near-real-time results
✓ Continuous monitoring
✓ Event-driven reactions
✓ Example: Fraud detection, real-time recommendations

Use Both Together:
──────────────────
✓ Stream for real-time, batch for comprehensive analysis
✓ Example: Stream processes recent data for dashboards,
           nightly batch job computes detailed analytics
```

**Preview of Chapter 11**

Chapter 11 explores **stream processing**—how to process unbounded data in real-time. We'll cover:

- **Event streams**: Treating data as an infinite, ordered sequence
- **Stream processing frameworks**: Apache Kafka, Apache Flink, Apache Storm
- **Windowing**: How to compute aggregates over unbounded streams
- **State management**: Maintaining state in stream processors
- **Time semantics**: Event time vs. processing time
- **Exactly-once processing**: Guaranteeing correctness in streams

The journey from batch to streaming represents the evolution of data systems toward handling the "always-on" nature of modern applications, where **data never stops flowing** and insights need to be delivered in real-time.

---

**End of Chapter 10**

You now understand:
- How batch processing democratized large-scale data processing
- The MapReduce paradigm and its historical significance (from Google's early 2000s innovation to today)
- Why batch processing complements (rather than replaces) databases
- How to scale word counting from single machine to 100TB across 1,000 servers
- Real-world applications (Google search indexing, Netflix recommendations)
- Modern improvements (Spark, Flink) and why they matter
- How to think about distributed joins, aggregations, and fault tolerance
- The developer experience trade-offs (flexibility vs. complexity)
- The trade-offs between batch and stream processing

Next, we'll see how stream processing brings the power of batch systems to real-time data!
