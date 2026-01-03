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

**Word Count Example:**

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

**Detailed View:**

```
┌─────────────────────────────────────────────────────────┐
│                     JobTracker                           │
│ - Schedules tasks                                        │
│ - Monitors progress                                      │
│ - Handles failures                                       │
└─────────────────────────────────────────────────────────┘
                          │
       ┌──────────────────┼──────────────────┐
       ▼                  ▼                  ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│TaskTracker 1│    │TaskTracker 2│    │TaskTracker 3│
├─────────────┤    ├─────────────┤    ├─────────────┤
│ Map Task A  │    │ Map Task B  │    │ Map Task C  │
│    ↓        │    │    ↓        │    │    ↓        │
│ (spill to   │    │ (spill to   │    │ (spill to   │
│  local disk)│    │  local disk)│    │  local disk)│
│    ↓        │    │    ↓        │    │    ↓        │
│ Reduce Task │    │ Reduce Task │    │ Reduce Task │
│    ↓        │    │    ↓        │    │    ↓        │
│ (write HDFS)│    │ (write HDFS)│    │ (write HDFS)│
└─────────────┘    └─────────────┘    └─────────────┘
```

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

### Types of Joins

**1. Reduce-Side Join (Sort-Merge Join)**

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

**2. Map-Side Join (Broadcast Join)**

If one dataset fits in memory, broadcast it to all mappers.

**When to use**: Small dimension table + large fact table.

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

Map-Side Join:
  - No shuffle (cheap)
  - Requires small table fits in memory
  - Time: O(n) for large table
```

**3. Skewed Join (Handling Hot Keys)**

**Problem**: Some keys have many more values (e.g., popular product).

```
Key 123 has 1 million records
→ Single reducer overloaded!

Other keys have ~100 records
→ Other reducers underutilized
```

**Solution: Split hot keys**

```python
def mapper_with_salting(key, value):
    """Add random suffix to hot keys"""
    if is_hot_key(key):
        # Split into 10 partitions
        salt = random.randint(0, 9)
        yield (f"{key}_{salt}", value)
    else:
        yield (key, value)
```

## Part 5: Group By and Aggregation

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

**Looking Ahead:**
Chapter 11 explores **stream processing**—how to process unbounded data in real-time, bridging the gap between batch and streaming paradigms.
