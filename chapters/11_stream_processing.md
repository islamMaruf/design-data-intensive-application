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

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Send message
event = {
    'user_id': 'user_123',
    'action': 'click',
    'url': '/products/laptop',
    'timestamp': '2024-01-15T10:30:45Z'
}

future = producer.send('user_clicks', value=event)

# Block until sent
result = future.get(timeout=10)
print(f"Sent to partition {result.partition}, offset {result.offset}")

# Flush and close
producer.flush()
producer.close()
```

**Partitioning Strategies:**

```python
# 1. Round-robin (default if no key)
producer.send('topic', value=message)

# 2. By key (messages with same key go to same partition)
producer.send('topic', key=user_id.encode(), value=message)
# Ensures ordering per user

# 3. Custom partitioner
def custom_partitioner(key, all_partitions, available_partitions):
    """Route VIP users to dedicated partition"""
    if key and key.startswith(b'vip_'):
        return 0  # Partition 0 for VIPs
    else:
        return hash(key) % (len(all_partitions) - 1) + 1

producer = KafkaProducer(partitioner=custom_partitioner)
```

### Consuming Messages

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'user_clicks',
    bootstrap_servers=['localhost:9092'],
    auto_offset_reset='earliest',  # Start from beginning
    enable_auto_commit=True,
    group_id='analytics_group',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

# Consume messages
for message in consumer:
    event = message.value
    print(f"User {event['user_id']} clicked {event['url']}")
    
    # Process event
    process_click(event)

consumer.close()
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

```python
# Guarantee: All messages for user_123 are ordered
producer.send('topic', key=b'user_123', value=message1)
producer.send('topic', key=b'user_123', value=message2)
# message1 arrives before message2 (same partition)

# No guarantee: messages for different users
producer.send('topic', key=b'user_123', value=message1)
producer.send('topic', key=b'user_456', value=message2)
# Could arrive in any order (different partitions)
```

**Delivery Semantics:**

**1. At-most-once**: Messages may be lost but never redelivered
```python
# Consumer auto-commits offset BEFORE processing
consumer = KafkaConsumer(enable_auto_commit=True, auto_commit_interval_ms=5000)
for msg in consumer:
    try:
        process(msg)
    except Exception:
        pass  # Message lost if processing fails!
```

**2. At-least-once**: Messages never lost but may be redelivered
```python
# Consumer commits offset AFTER processing
consumer = KafkaConsumer(enable_auto_commit=False)
for msg in consumer:
    process(msg)
    consumer.commit()  # Commit only after success
    # If crash before commit → redelivery → duplicate processing
```

**3. Exactly-once**: Messages processed exactly once (Kafka 0.11+)
```python
# Transactional producer + idempotent processing
producer = KafkaProducer(
    transactional_id='my_transactional_id',
    enable_idempotence=True
)

producer.init_transactions()
producer.begin_transaction()
producer.send('topic', value=message)
producer.commit_transaction()

# Combined with idempotent consumer logic
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

## Part 3: Stream Processing Frameworks

### Stream Processing Patterns

**1. Stateless Transformations**

```python
# Filter events
def is_purchase(event):
    return event['event_type'] == 'purchase'

purchases = stream.filter(is_purchase)

# Map/Transform events
def enrich_event(event):
    event['processed_at'] = datetime.now()
    event['category'] = get_product_category(event['product_id'])
    return event

enriched = stream.map(enrich_event)

# FlatMap (one-to-many)
def extract_words(sentence):
    return sentence.split()

words = sentences.flatMap(extract_words)
```

**2. Stateful Transformations**

```python
# Aggregation (requires state)
def count_by_user(stream):
    # Keep running count per user
    counts = {}
    for event in stream:
        user_id = event['user_id']
        counts[user_id] = counts.get(user_id, 0) + 1
        yield (user_id, counts[user_id])
```

### Apache Flink

**Flink** is a true stream processor with powerful windowing and state management.

**Basic Stream Processing:**

```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.datastream.connectors import FlinkKafkaConsumer
from pyflink.common.serialization import SimpleStringSchema

# Set up environment
env = StreamExecutionEnvironment.get_execution_environment()

# Kafka source
kafka_consumer = FlinkKafkaConsumer(
    topics='user_clicks',
    deserialization_schema=SimpleStringSchema(),
    properties={'bootstrap.servers': 'localhost:9092', 'group.id': 'flink_group'}
)

# Create stream
stream = env.add_source(kafka_consumer)

# Transformations
parsed = stream.map(lambda x: json.loads(x))
purchases = parsed.filter(lambda x: x['event_type'] == 'purchase')

# Count purchases per user
purchase_counts = purchases \
    .key_by(lambda x: x['user_id']) \
    .map(lambda x: (x['user_id'], 1)) \
    .key_by(lambda x: x[0]) \
    .sum(1)

# Print results
purchase_counts.print()

# Execute
env.execute("Purchase Counter")
```

**Windowing:**

```python
from pyflink.datastream.window import TumblingEventTimeWindows
from pyflink.common.time import Time

# Tumbling window: non-overlapping, fixed-size
windowed = purchases \
    .key_by(lambda x: x['user_id']) \
    .window(TumblingEventTimeWindows.of(Time.minutes(5))) \
    .reduce(lambda a, b: a + b)

# Sliding window: overlapping windows
from pyflink.datastream.window import SlidingEventTimeWindows

sliding = purchases \
    .key_by(lambda x: x['user_id']) \
    .window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(5))) \
    .reduce(lambda a, b: a + b)
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

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import window, col

spark = SparkSession.builder.appName("StreamExample").getOrCreate()

# Read from Kafka
events = spark.readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "localhost:9092") \
    .option("subscribe", "user_clicks") \
    .load()

# Parse JSON
from pyspark.sql.functions import from_json, schema_of_json

schema = schema_of_json('{"user_id":"","url":"","timestamp":""}')
parsed = events.select(from_json(col("value").cast("string"), schema).alias("data")).select("data.*")

# Windowed aggregation
windowed_counts = parsed \
    .withColumn("timestamp", col("timestamp").cast("timestamp")) \
    .groupBy(
        window(col("timestamp"), "5 minutes"),
        col("url")
    ) \
    .count()

# Write to console
query = windowed_counts.writeStream \
    .outputMode("complete") \
    .format("console") \
    .start()

query.awaitTermination()
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
                 event_a @ 10:00:10  ← Late!
```

**Problem:** Which timestamp to use for windowing?

### Event Time Processing

**Event Time**: When event actually occurred (source timestamp)
**Processing Time**: When event is processed (server timestamp)

```python
# Flink: Use event time
from pyflink.datastream import TimeCharacteristic

env.set_stream_time_characteristic(TimeCharacteristic.EventTime)

# Extract event timestamp
def extract_timestamp(event):
    return event['timestamp']

stream = stream.assign_timestamps_and_watermarks(
    WatermarkStrategy
        .for_bounded_out_of_orderness(Duration.of_seconds(5))
        .with_timestamp_assigner(lambda event, ts: event['timestamp'])
)
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

```python
# Option 1: Discard late data
stream \
    .window(TumblingEventTimeWindows.of(Time.minutes(5))) \
    .allowed_lateness(Time.seconds(0))  # No lateness allowed

# Option 2: Accept late data up to threshold
stream \
    .window(TumblingEventTimeWindows.of(Time.minutes(5))) \
    .allowed_lateness(Time.minutes(1))  # Accept up to 1 min late
    .side_output_late_data(late_data_output_tag)  # Send very late data to side output

# Option 3: Update window results (retraction)
stream \
    .window(TumblingEventTimeWindows.of(Time.minutes(5))) \
    .allowed_lateness(Time.minutes(10))
    # Emit updated results as late data arrives
```

## Part 5: Joins in Stream Processing

### Stream-Stream Join

**Example: Join clicks and purchases (both streams)**

```python
# Flink example
clicks = env.add_source(kafka_clicks) \
    .assign_timestamps_and_watermarks(...)

purchases = env.add_source(kafka_purchases) \
    .assign_timestamps_and_watermarks(...)

# Join within time window
joined = clicks \
    .key_by(lambda c: c['user_id']) \
    .interval_join(purchases.key_by(lambda p: p['user_id'])) \
    .between(Time.seconds(-10), Time.seconds(10)) \
    .process(JoinFunction())
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

```python
# Kafka Streams example
KStream<String, Purchase> purchases = builder.stream("purchases");
KTable<String, User> users = builder.table("users");

// Join stream with table
KStream<String, EnrichedPurchase> enriched = purchases.join(
    users,
    (purchase, user) -> new EnrichedPurchase(purchase, user)
);
```

**Performance Consideration:**

```python
# Problem: Lookup in external DB (slow!)
def enrich_with_db(event):
    user = db.query("SELECT * FROM users WHERE id = ?", event['user_id'])
    event['user_name'] = user['name']
    return event

stream.map(enrich_with_db)  # One DB query per event!

# Solution: Cache in local state
def enrich_with_cache(event, state):
    user_id = event['user_id']
    if user_id not in state:
        state[user_id] = db.query("SELECT * FROM users WHERE id = ?", user_id)
    event['user_name'] = state[user_id]['name']
    return event
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

```python
# Flink
env.enable_checkpointing(60000)  # Checkpoint every 60 seconds
env.get_checkpoint_config().set_checkpoint_storage("hdfs:///checkpoints")
env.get_checkpoint_config().set_checkpointing_mode(CheckpointingMode.EXACTLY_ONCE)
```

### Idempotent Operations

**Idempotent**: Applying operation multiple times has same effect as applying once.

```python
# NOT idempotent (don't use with at-least-once!)
def increment_counter(user_id):
    db.execute("UPDATE counters SET count = count + 1 WHERE user_id = ?", user_id)
    # If processed twice → counter incremented twice!

# Idempotent (safe for at-least-once)
def set_last_seen(user_id, timestamp):
    db.execute("UPDATE users SET last_seen = ? WHERE user_id = ? AND last_seen < ?",
               timestamp, user_id, timestamp)
    # If processed twice → no change (timestamp already updated)
```

**Transactional Writes:**

```python
# Kafka Transactions (exactly-once)
from kafka import KafkaProducer

producer = KafkaProducer(
    transactional_id='my_app',
    enable_idempotence=True
)

producer.init_transactions()

def process_message(msg):
    producer.begin_transaction()
    try:
        # Process
        result = transform(msg)
        
        # Produce output
        producer.send('output_topic', result)
        
        # Commit consumer offset
        producer.send_offsets_to_transaction(
            {TopicPartition('input_topic', 0): msg.offset + 1},
            'consumer_group_id'
        )
        
        producer.commit_transaction()
    except Exception:
        producer.abort_transaction()
```

## Part 7: Use Cases and Patterns

### Real-Time Analytics

**Example: Trending Topics on Twitter**

```python
from pyspark.sql.functions import window, desc

# Read tweet stream
tweets = spark.readStream \
    .format("kafka") \
    .option("subscribe", "tweets") \
    .load()

# Extract hashtags
from pyspark.sql.functions import explode, split

hashtags = tweets \
    .select(explode(split(col("text"), " ")).alias("word")) \
    .filter(col("word").startswith("#"))

# Count in 5-minute windows
trending = hashtags \
    .groupBy(
        window(col("timestamp"), "5 minutes", "1 minute"),  # Sliding window
        col("word")
    ) \
    .count() \
    .orderBy(desc("count"))

# Write to dashboard
query = trending.writeStream \
    .format("memory") \
    .queryName("trending") \
    .outputMode("complete") \
    .start()
```

### Fraud Detection

**Example: Detect unusual transactions**

```python
# Flink example
class FraudDetector(KeyedProcessFunction):
    def __init__(self):
        self.last_transaction = None
        self.timer_service = None
    
    def process_element(self, transaction, ctx):
        # Check for rapid transactions
        if self.last_transaction:
            time_diff = transaction.timestamp - self.last_transaction.timestamp
            if time_diff < 60:  # Less than 1 minute
                amount_diff = abs(transaction.amount - self.last_transaction.amount)
                if amount_diff > 10000:  # Large amount difference
                    yield Alert(f"Potential fraud: {transaction.user_id}")
        
        self.last_transaction = transaction

transactions.key_by(lambda t: t.user_id) \
    .process(FraudDetector())
```

### Event Sourcing

**Pattern**: Store all changes as sequence of events.

```python
# Events
events = [
    {'type': 'UserCreated', 'user_id': 1, 'name': 'Alice'},
    {'type': 'EmailChanged', 'user_id': 1, 'email': 'alice@new.com'},
    {'type': 'UserDeleted', 'user_id': 1}
]

# Rebuild state by replaying events
def replay_events(events):
    users = {}
    for event in events:
        if event['type'] == 'UserCreated':
            users[event['user_id']] = {'name': event['name']}
        elif event['type'] == 'EmailChanged':
            users[event['user_id']]['email'] = event['email']
        elif event['type'] == 'UserDeleted':
            del users[event['user_id']]
    return users

# Current state
users = replay_events(events)  # {}

# Time travel: state at any point
users_at_t2 = replay_events(events[:2])  # {1: {'name': 'Alice', 'email': 'alice@new.com'}}
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
    "table.include.list": "public.users,public.orders"
  }
}
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
