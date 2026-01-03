# Chapter 8: The Trouble with Distributed Systems

## Introduction: Welcome to the Real World

You've built a beautiful database system running on one machine. It has perfect ACID transactions, consistent data, and reliable operations. Life is good! 😊

Then your boss says: "We need to scale. Split it across 10 servers."

Suddenly, everything that could go wrong, **will** go wrong:
- 🌐 **Networks fail** (cables unplugged, switches crash, packets dropped)
- ⏰ **Clocks drift** (servers disagree on what time it is)
- 💥 **Machines crash** (power failures, hardware faults)
- 🐌 **Processes pause** (garbage collection, OS suspends threads)
- 📨 **Messages get lost** (network congestion, buffer overflows)
- 🔄 **Messages arrive out of order** (different network paths)
- 🐢 **Operations are slow** (sometimes fast, sometimes slow, unpredictable)

**Welcome to distributed systems** - where Murphy's Law is an understatement!

```
┌────────────────────────────────────────────────┐
│  SINGLE MACHINE vs DISTRIBUTED SYSTEM          │
├────────────────────────────────────────────────┤
│                                                │
│  Single Machine:                               │
│  [CPU] ──fast reliable bus──→ [Memory]        │
│  Either works or crashes completely           │
│  Time is consistent                            │
│  Operations are fast and predictable           │
│                                                │
│  Distributed System:                           │
│  [Server A] ──unreliable network──→ [Server B]│
│  Partial failures (A works, B crashes)         │
│  Clocks disagree                               │
│  Operations are slow and unpredictable         │
│  Messages lost, delayed, duplicated            │
└────────────────────────────────────────────────┘
```

This chapter explores everything that can go wrong in distributed systems, and how to build systems that work despite these problems.

## Part 1: Faults and Partial Failures

### The Fundamental Problem

**In a single computer**:
- Either it works correctly OR
- It fails completely (crash, blue screen)

**In a distributed system**:
- Some parts work correctly AND
- Some parts fail simultaneously
- **Partial failures** are nondeterministic (random, unpredictable)

**Real-World Analogy**:

```
Single Computer = Light Switch
  ON → Everything works ✅
  OFF → Everything fails ❌
  Simple, predictable!

Distributed System = Christmas Lights
  Some bulbs work ✅
  Some bulbs broken ❌
  Working bulbs keep working
  You don't know which are broken until you check
  Complex, unpredictable!
```

### Example: Sending a Request

Simple request from Client to Server. What can go wrong?

```
┌────────────────────────────────────────────────┐
│  REQUEST/RESPONSE SCENARIOS                    │
├────────────────────────────────────────────────┤
│                                                │
│  Scenario 1: Success ✅                        │
│  Client ──request──→ Server                    │
│  Client ←─response── Server                    │
│                                                │
│  Scenario 2: Request lost 📧❌                  │
│  Client ──request──X                           │
│  (Server never receives it)                    │
│                                                │
│  Scenario 3: Server crashes 💥                 │
│  Client ──request──→ Server 💥                 │
│  (No response)                                 │
│                                                │
│  Scenario 4: Response lost 📨❌                 │
│  Client ──request──→ Server                    │
│  Server processes request ✅                   │
│  Client ←──────X (response lost)               │
│                                                │
│  Scenario 5: Response delayed 🐢               │
│  Client ──request──→ Server                    │
│  (Long pause...)                               │
│  Client ←─response── (finally arrives)         │
└────────────────────────────────────────────────┘
```

**The Problem**: From client's perspective, scenarios 2, 3, 4, and 5 all look the same - **no response**!

```python
# Client code
def make_request(server_url, data):
    try:
        response = http.post(server_url, data, timeout=5)
        return response
    except Timeout:
        # What happened?
        # - Request lost? (retry is safe)
        # - Server crashed? (retry is safe)
        # - Response lost? (retry might duplicate! ❌)
        # - Server just slow? (retry might duplicate! ❌)
        # 
        # We can't tell! 🤷
        pass
```

**Key Insight**: In distributed systems, you often can't distinguish between different types of failures. This uncertainty is fundamental and unavoidable.

### Two Philosophies: Cloud Computing vs Supercomputing

**Supercomputing** (HPC - High-Performance Computing):
- Thousands of CPUs, tightly coupled
- Checkpoint entire system state periodically
- If any node fails → **Stop everything**, restore from checkpoint
- Treats partial failure like complete failure

**Cloud Computing**:
- Commodity hardware, loose coupling
- Nodes fail independently
- System continues operating despite failures
- Built for partial failures

```
┌──────────────────────────────────────────────┐
│   FAILURE HANDLING PHILOSOPHIES              │
├──────────────┬───────────────────────────────┤
│  Approach    │  When Node Fails              │
├──────────────┼───────────────────────────────┤
│ Supercompute │  Stop everything              │
│              │  Restore checkpoint           │
│              │  Resume from there            │
│              │  (like save/load game)        │
├──────────────┼───────────────────────────────┤
│ Cloud        │  Failed node marked dead      │
│              │  Other nodes continue         │
│              │  Reroute traffic              │
│              │  (like highway detour)        │
└──────────────┴───────────────────────────────┘
```

**This book focuses on cloud computing philosophy** - build systems that tolerate partial failures.

## Part 2: Unreliable Networks

Networks are the foundation of distributed systems. Unfortunately, they're also the most unreliable component!

### Network Faults in Practice

**Reality Check**: Major companies experience frequent network issues.

**Real-World Data**:

1. **Microsoft Azure (2012 Study)**:
   - 5 network failures per month affecting customer-visible services
   - Average downtime: 59 seconds
   - Max downtime: 2.5 hours

2. **Amazon EC2 (Multiple incidents)**:
   - 2011: Networking event caused widespread outages
   - 2012: Network issue took down Netflix, Pinterest, Instagram
   - 2017: S3 outage due to network partition

3. **GitHub (2012)**:
   - Network partition split database cluster
   - Led to data inconsistency
   - Required manual recovery

**Common Network Problems**:

```
┌────────────────────────────────────────────────┐
│  TYPES OF NETWORK FAILURES                     │
├────────────────────────────────────────────────┤
│                                                │
│  1. Packet Loss                                │
│     [Client] ──X  (packet dropped)             │
│     - Network congestion                       │
│     - Buffer overflow at switch                │
│     - Faulty network card                      │
│                                                │
│  2. Cable Unplugged                            │
│     [Server] ──╳── [Switch]                    │
│     - Someone trips over cable                 │
│     - Maintenance accident                     │
│                                                │
│  3. Network Partition                          │
│     [A] [B] [C] ─╳─ [D] [E] [F]               │
│     - Two groups can communicate internally    │
│     - Cannot communicate across groups         │
│     - "Split brain" scenario                   │
│                                                │
│  4. Slow Network                               │
│     [Client] ────🐌─slow─→ [Server]           │
│     - Network congestion                       │
│     - Overloaded switch                        │
│     - Bad routing                              │
└────────────────────────────────────────────────┘
```

### Detecting Faults: Timeouts

How do you know if a remote node is down?

**Answer**: Use **timeouts**. If no response within X seconds, assume failure.

```python
def call_remote_service(url, timeout=5):
    start_time = time.time()
    try:
        response = http.get(url, timeout=timeout)
        return response
    except Timeout:
        elapsed = time.time() - start_time
        print(f"No response after {elapsed} seconds")
        # Assume service is down
        return None
```

**The Timeout Dilemma**:

```
┌────────────────────────────────────────────────┐
│  CHOOSING TIMEOUT VALUE                        │
├────────────────────────────────────────────────┤
│                                                │
│  Too Short (e.g., 100ms):                      │
│  ❌ False positives (node just slow)           │
│  ❌ Unnecessary failovers                      │
│  ❌ Cascading failures                         │
│                                                │
│  Too Long (e.g., 60s):                         │
│  ❌ Slow failure detection                     │
│  ❌ Users wait a long time                     │
│  ❌ System unavailable longer                  │
│                                                │
│  Just Right (adaptive):                        │
│  ✅ Based on typical response time             │
│  ✅ Add margin for variance                    │
│  ✅ Adjust based on measurements               │
└────────────────────────────────────────────────┘
```

**Adaptive Timeout Example**:

```python
class AdaptiveTimeout:
    def __init__(self):
        self.response_times = []
        self.window_size = 100  # Last 100 requests
    
    def record_response_time(self, duration):
        self.response_times.append(duration)
        if len(self.response_times) > self.window_size:
            self.response_times.pop(0)
    
    def get_timeout(self):
        if not self.response_times:
            return 5.0  # Default 5 seconds
        
        # Calculate based on percentiles
        p99 = np.percentile(self.response_times, 99)
        
        # Timeout = 2x p99 response time
        timeout = 2 * p99
        
        # Clamp between 1s and 30s
        return max(1.0, min(30.0, timeout))

# Usage
timeout_manager = AdaptiveTimeout()

for request in requests:
    timeout = timeout_manager.get_timeout()
    start = time.time()
    try:
        response = call_service(url, timeout=timeout)
        duration = time.time() - start
        timeout_manager.record_response_time(duration)
    except Timeout:
        handle_timeout()
```

### Network Congestion and Queueing

Networks don't fail just by breaking - they also fail by getting **slow**.

**Where Delays Happen**:

```
┌────────────────────────────────────────────────┐
│  NETWORK DELAY SOURCES                         │
├────────────────────────────────────────────────┤
│                                                │
│  [Client] → [Queue 1] → [Network] → [Queue 2] → [Server] │
│     ↓          ↓          ↓            ↓         ↓   │
│   App      NIC         Switch        NIC       App  │
│   queue    queue       queue         queue     queue│
│                                                │
│  1. Application Send Queue                     │
│     - TCP send buffer full                     │
│     - OS waiting to send                       │
│                                                │
│  2. Network Interface Card (NIC) Queue         │
│     - Hardware buffer                          │
│     - Waiting for transmission                 │
│                                                │
│  3. Switch Queue                               │
│     - Multiple inputs, one output              │
│     - Congestion here is common                │
│                                                │
│  4. Receiver NIC Queue                         │
│     - Packets arriving faster than processed   │
│                                                │
│  5. Application Receive Queue                  │
│     - TCP receive buffer                       │
│     - Application processing slowly            │
└────────────────────────────────────────────────┘
```

**Queueing Delay Example**:

```python
# Network switch processing

packet_queue = []
QUEUE_SIZE = 1000

def receive_packet(packet):
    if len(packet_queue) < QUEUE_SIZE:
        packet_queue.append(packet)
        packet.queue_time_start = time.time()
    else:
        # Queue full - drop packet! 💥
        drop_packet(packet)

def forward_packets():
    while packet_queue:
        packet = packet_queue.pop(0)
        
        # Calculate queueing delay
        queue_delay = time.time() - packet.queue_time_start
        
        if queue_delay > 0.1:  # 100ms
            print(f"High queue delay: {queue_delay}s")
        
        transmit(packet)
        time.sleep(0.001)  # 1ms per packet transmission
```

**Real-World Impact - Tail Latency**:

```
Normal situation:
  p50: 10ms
  p99: 50ms
  p99.9: 100ms

During congestion:
  p50: 10ms   (median unchanged!)
  p99: 500ms  (10x worse!)
  p99.9: 5s   (50x worse!)

User impact:
  - Most requests: fine
  - 1 in 100 requests: very slow
  - Service feels "glitchy"
```

**Real-World Example - AWS Network Congestion (2020)**:

During AWS outage:
- Network congestion in single availability zone
- Queue delays reached seconds
- Services timed out waiting for responses
- Cascading failures across multiple services

### Unbounded Delays (No Guarantees)

**Key Insight**: In most networks (including the Internet), there are **no guarantees** on message delivery time.

This is called an **asynchronous network** model.

```
┌────────────────────────────────────────────────┐
│  NETWORK TIMING MODELS                         │
├────────────────────────────────────────────────┤
│                                                │
│  Synchronous Network (rare):                   │
│    - Messages delivered within max time d      │
│    - If not delivered by time d, failed        │
│    - Example: Old telephone networks           │
│    - NOT the Internet!                         │
│                                                │
│  Asynchronous Network (reality):               │
│    - Messages may take arbitrarily long        │
│    - No upper bound on delay                   │
│    - Example: Internet, Ethernet               │
│    - What we deal with!                        │
└────────────────────────────────────────────────┘
```

**Why No Guarantees?**

1. **Best-effort delivery**: IP networks don't reserve resources
2. **Shared infrastructure**: Multiple users compete for bandwidth
3. **Variable routing**: Packets take different paths
4. **Economic reasons**: Guaranteed delivery requires expensive infrastructure

**Practical Implication**:

```python
# You CANNOT write this:
def call_service(url):
    response = http.get(url)
    # Assumption: will return within 100ms ❌
    # Reality: might take 10 seconds, or never return!
    return response

# You MUST write this:
def call_service(url, timeout=5, retries=3):
    for attempt in range(retries):
        try:
            response = http.get(url, timeout=timeout)
            return response
        except Timeout:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
                continue
            else:
                raise ServiceUnavailable()
```

## Part 3: Unreliable Clocks

Time seems simple - what could go wrong? Turns out, a lot!

### Two Types of Clocks

**1. Time-of-Day Clock** (Wall-Clock Time)

```python
import time

# Time-of-day clock
current_time = time.time()
# Returns: 1704067200.123456 (seconds since Unix epoch: Jan 1, 1970)

# Human readable:
datetime.fromtimestamp(current_time)
# Returns: 2024-01-01 00:00:00
```

**Properties**:
- Returns current date and time
- Synchronized with NTP (Network Time Protocol)
- **Can jump backwards** if clock adjusted!
- **Can jump forwards** if clock adjusted!
- Not suitable for measuring durations

**2. Monotonic Clock** (Steady Clock)

```python
import time

# Monotonic clock
start = time.monotonic()
# Do some work...
end = time.monotonic()
duration = end - start  # Always positive, never jumps backwards
```

**Properties**:
- Always moves forward
- Never jumps backwards or forwards
- **Suitable for measuring durations**
- Not comparable across machines
- Doesn't correspond to wall-clock time

```
┌────────────────────────────────────────────────┐
│  CLOCK COMPARISON                              │
├──────────────────┬─────────────────────────────┤
│ Time-of-Day      │ Monotonic                   │
├──────────────────┼─────────────────────────────┤
│ What time is it? │ How long did it take?       │
│ 2:30 PM          │ 5.2 seconds                 │
│ Can jump         │ Always forward              │
│ Comparable across│ Not comparable across       │
│ machines (NTP)   │ machines                    │
│ Use for:         │ Use for:                    │
│ - Timestamps     │ - Timeouts                  │
│ - Logging        │ - Performance measurement   │
│ - Ordering events│ - Durations                 │
└──────────────────┴─────────────────────────────┘
```

### Clock Synchronization Problems

**Problem 1: Clock Drift**

Quartz clocks in computers are not perfect. They drift apart over time.

```
┌────────────────────────────────────────────────┐
│  CLOCK DRIFT OVER TIME                         │
├────────────────────────────────────────────────┤
│                                                │
│  Perfect World:                                │
│  Server A: ──────────→ (1 second per second)   │
│  Server B: ──────────→ (1 second per second)   │
│                                                │
│  Reality:                                      │
│  Server A: ───────────→ (1.0001 sec/sec)       │
│  Server B: ──────────→ (0.9999 sec/sec)        │
│                                                │
│  After 1 day:                                  │
│  Server A: ahead by 8.6 seconds                │
│  Server B: behind by 8.6 seconds               │
│  Difference: 17.2 seconds! 💥                  │
└────────────────────────────────────────────────┘
```

**Typical Clock Drift**:
- Consumer hardware: ±50 ppm (parts per million)
- That's ±4.3 seconds per day!

**Problem 2: NTP Synchronization Issues**

NTP (Network Time Protocol) keeps clocks synchronized, but it's not perfect.

```
┌────────────────────────────────────────────────┐
│  NTP SYNCHRONIZATION                           │
├────────────────────────────────────────────────┤
│                                                │
│  [Server] ──→ "What time is it?" ──→ [NTP]    │
│                                       ↓        │
│                                 Look at atomic │
│                                       clock    │
│                                       ↓        │
│  [Server] ←── "It's 14:30:05.123" ←── [NTP]   │
│       ↓                                        │
│   Adjust local clock                           │
│                                                │
│  Problems:                                     │
│  - Network delay (10-100ms typical)            │
│  - Asymmetric routing (request ≠ response)     │
│  - Server load (NTP server busy)               │
│  - Accuracy: ±35ms on local network            │
│             ±100ms on public internet          │
└────────────────────────────────────────────────┘
```

**Problem 3: Clock Jumps**

When NTP detects large drift, it can **jump** the clock.

```python
# Time-of-day clock can jump backwards!

Time: 14:30:00 → Write record with timestamp 14:30:00
Time: 14:29:55 ← NTP adjusts clock backwards 5 seconds! 💥
Time: 14:30:00 → Write another record with timestamp 14:30:00

# Two records with same timestamp!
# Or worse: later event has earlier timestamp!
```

**Real-World Disaster - Cloudflare (2016)**:

Cloudflare's load balancer used time-of-day clock to expire sessions.
- NTP jumped clock backwards
- Sessions that should be expired were still considered valid
- Security vulnerabilities!

### Relying on Synchronized Clocks

**Dangerous Assumptions**:

```python
# ❌ WRONG: Assuming clocks are synchronized

def get_global_ordering():
    # Server A writes: timestamp = clock_A.now()
    # Server B writes: timestamp = clock_B.now()
    # 
    # If clock_A > clock_B (due to drift):
    # Event on A appears to happen after event on B
    # Even if A happened first in real time!
    pass

# ❌ WRONG: Using timestamps for causality

def is_after(event1, event2):
    return event1.timestamp > event2.timestamp
    # Assumes synchronized clocks! ❌

# ✅ CORRECT: Use logical clocks (Lamport timestamps)

def is_causally_after(event1, event2):
    return event1.logical_clock > event2.logical_clock
    # Doesn't depend on physical time ✅
```

**Timestamp Ordering Example**:

```
Real Order:
  T0: Server A: User clicks "post" (time: 10:00:00.100)
  T1: Server B: User sees post (time: 10:00:00.050)

Wait, what? Effect before cause!?

Reason: Server B's clock is 50ms behind Server A's clock

Result: Posts appear out of order in timeline!
```

**Solutions**:

1. **Don't rely on clock synchronization for ordering**
   - Use sequence numbers
   - Use logical clocks (Lamport timestamps, vector clocks)

2. **Use Google TrueTime-like confidence intervals**
   ```python
   # Instead of: timestamp = 10:00:00.123
   # Use: timestamp = [10:00:00.123 ± 5ms]
   # 
   # Wait until intervals don't overlap before committing
   ```

3. **Combine with other mechanisms**
   - Clocks + version numbers
   - Clocks + consensus algorithms

### Process Pauses

Even if networks and clocks were perfect, processes can pause unexpectedly!

**Causes of Process Pauses**:

```
┌────────────────────────────────────────────────┐
│  WHY PROCESSES PAUSE                           │
├────────────────────────────────────────────────┤
│                                                │
│  1. Garbage Collection (GC)                    │
│     - "Stop the world" GC pause                │
│     - Can be hundreds of milliseconds          │
│     - All threads frozen!                      │
│                                                │
│  2. Virtual Machine Suspension                 │
│     - Hypervisor pauses VM                     │
│     - Seconds or even minutes!                 │
│     - VM doesn't know it was paused            │
│                                                │
│  3. Operating System Context Switching         │
│     - Laptop closed (sleep mode)               │
│     - Process swapped to disk                  │
│                                                │
│  4. Synchronous Disk I/O                       │
│     - Page fault, need to load from disk       │
│     - Slow HDD can pause for 100ms             │
│                                                │
│  5. Other Processes                            │
│     - CPU contention                           │
│     - Another process using all CPU            │
└────────────────────────────────────────────────┘
```

**Real-World Example - GC Pause**:

```java
// Java application

public class LeaderElection {
    private long lastHeartbeat;
    private boolean isLeader = false;
    
    public void runLeaderHeartbeat() {
        while (isLeader) {
            sendHeartbeat();
            lastHeartbeat = System.currentTimeMillis();
            
            // GC PAUSE HERE for 15 seconds! ⏸️
            // (Application frozen, doesn't know!)
            
            Thread.sleep(10000);  // 10 seconds
        }
    }
    
    public void checkLeaderAlive() {
        long now = System.currentTimeMillis();
        if (now - lastHeartbeat > 20000) {  // 20 second timeout
            // Leader hasn't sent heartbeat!
            // Declare leader dead, start election
            startElection();
        }
    }
}

// What happens:
// T0: Leader sends heartbeat
// T1: GC pause for 15 seconds ⏸️
// T2: Other nodes: "Leader dead!" (15s > timeout)
// T3: New leader elected
// T4: GC finishes, old leader resumes
// T5: TWO LEADERS! 💥💥 (split-brain)
```

**Protecting Against Process Pauses**:

```python
# Solution: Fencing tokens

class LeaderWithFencing:
    def __init__(self):
        self.token = 0  # Monotonically increasing token
    
    def become_leader(self):
        # Get token from coordination service (ZooKeeper, etcd)
        self.token = coordination_service.get_next_token()
        # token = 42
        
    def write_data(self, data):
        # Include token with every write
        storage.write(data, fencing_token=self.token)
        
# Storage system:
class Storage:
    def __init__(self):
        self.current_token = 0
    
    def write(self, data, fencing_token):
        if fencing_token < self.current_token:
            # Old leader trying to write!
            raise FencingTokenTooOld()
        
        self.current_token = fencing_token
        self._write(data)

# Timeline:
# T0: Leader A gets token=42
# T1: Leader A writes with token=42 ✅
# T2: Leader A pauses (GC)
# T3: New leader B elected, gets token=43
# T4: Leader B writes with token=43 ✅ (storage accepts, updates current_token=43)
# T5: Leader A resumes, tries to write with token=42
# T6: Storage rejects! (42 < 43) ❌
```

## Part 4: Knowledge, Truth, and Lies

In distributed systems, we face philosophical questions!

### The Truth is Defined by the Majority

**Question**: How do you know if a node is alive or dead?

**Answer**: You can't know for sure. You can only believe based on messages you receive.

**Scenario - Is Node Dead or Just Slow?**

```
┌────────────────────────────────────────────────┐
│  NODE FAILURE DETECTION                        │
├────────────────────────────────────────────────┤
│                                                │
│  [Node A] sends heartbeats                     │
│      ↓                                         │
│  [Node B] receives heartbeats                  │
│      ↓                                         │
│  Time passes... no heartbeat                   │
│      ↓                                         │
│  Question: Is A dead or just slow?             │
│                                                │
│  Possibility 1: A crashed 💥                   │
│  Possibility 2: Network problem 🌐❌            │
│  Possibility 3: A is slow 🐌                   │
│  Possibility 4: B is slow 🐌                   │
│                                                │
│  B cannot distinguish!                         │
│                                                │
│  Solution: Use quorum (majority vote)          │
│                                                │
│  [Node B] "A is dead"                          │
│  [Node C] "A is dead"                          │
│  [Node D] "A is alive"                         │
│      ↓                                         │
│  Majority (2/3) says dead → Declare A dead     │
└────────────────────────────────────────────────┘
```

**Quorum-Based Consensus**:

```python
def is_node_alive(node_id, quorum_size=3):
    votes = []
    
    for checker in cluster_nodes:
        if checker == node_id:
            continue
        
        # Ask if node responds
        if checker.can_reach(node_id):
            votes.append(True)
        else:
            votes.append(False)
    
    # Majority vote
    alive_votes = sum(votes)
    dead_votes = len(votes) - alive_votes
    
    if alive_votes >= quorum_size // 2 + 1:
        return True  # Majority says alive
    else:
        return False  # Majority says dead
```

### Byzantine Faults

**Byzantine Fault**: Node behaves maliciously or arbitrarily (not just failing silently).

```
┌────────────────────────────────────────────────┐
│  BYZANTINE vs NON-BYZANTINE FAULTS             │
├────────────────────────────────────────────────┤
│                                                │
│  Non-Byzantine (Crash fault):                  │
│    Node works correctly OR crashes             │
│    [Node] ✅ → works                           │
│    [Node] 💥 → crashed (silent)                │
│                                                │
│  Byzantine:                                    │
│    Node may send wrong/malicious messages      │
│    [Node] 😈 → sends different messages to     │
│                  different nodes                │
│    [Node] 😈 → claims it received message      │
│                  it didn't receive              │
│    [Node] 😈 → corrupts data                   │
└────────────────────────────────────────────────┘
```

**Examples of Byzantine Faults**:

1. **Malicious Attacker**
   - Hacker compromises node
   - Sends false information

2. **Hardware Corruption**
   - Cosmic ray flips bit in memory
   - CPU calculates wrong result

3. **Software Bug**
   - Nondeterministic behavior
   - Different results on different runs

**Byzantine Fault Tolerance (BFT)**:

Systems that tolerate Byzantine faults are **much more complex** and **much slower**.

```python
# Byzantine consensus (simplified)
# Requires 3f + 1 nodes to tolerate f Byzantine nodes

def byzantine_consensus(nodes, value):
    # Need 2f + 1 agreeing messages to commit
    # (f Byzantine nodes can't forge majority)
    
    messages = []
    for node in nodes:
        msg = node.broadcast(value)
        messages.append(msg)
    
    # Count votes
    vote_counts = {}
    for msg in messages:
        vote_counts[msg.value] = vote_counts.get(msg.value, 0) + 1
    
    # Need supermajority (2f + 1)
    for value, count in vote_counts.items():
        if count >= 2 * max_byzantine_nodes + 1:
            return value
    
    # No consensus
    return None
```

**Real-World Usage**:
- **Blockchain**: Bitcoin, Ethereum (Byzantine-tolerant)
- **Most databases**: NOT Byzantine-tolerant (assumes non-malicious failures)

**Why Not Always Use BFT?**
- Much slower (3x+ latency)
- More complex
- Requires more nodes (3f+1 vs 2f+1)
- Most systems assume trusted internal network

### System Models

To reason about distributed systems, we define models that describe what can go wrong.

**Timing Models**:

```
┌────────────────────────────────────────────────┐
│  SYSTEM TIMING MODELS                          │
├────────────────────────────────────────────────┤
│                                                │
│  Synchronous:                                  │
│    - Bounded network delay                     │
│    - Bounded process pause                     │
│    - Bounded clock error                       │
│    Reality: Aircraft systems, hard real-time   │
│                                                │
│  Partially Synchronous (MOST COMMON):          │
│    - Usually behaves like synchronous          │
│    - Sometimes delays unbounded                │
│    Reality: Most data centers, cloud systems   │
│                                                │
│  Asynchronous:                                 │
│    - No timing assumptions at all              │
│    - Arbitrarily long delays                   │
│    Reality: Theoretical model, pessimistic     │
└────────────────────────────────────────────────┘
```

**Node Failure Models**:

```
┌────────────────────────────────────────────────┐
│  NODE FAILURE MODELS                           │
├────────────────────────────────────────────────┤
│                                                │
│  Crash-stop:                                   │
│    - Node works correctly OR crashes           │
│    - Doesn't recover after crash               │
│                                                │
│  Crash-recovery (MOST COMMON):                 │
│    - Node may crash at any time                │
│    - May recover after crash                   │
│    - Stable storage survives crash             │
│                                                │
│  Byzantine:                                    │
│    - Node may behave arbitrarily               │
│    - May send wrong/malicious messages         │
└────────────────────────────────────────────────┘
```

**Most real systems assume**: Partially synchronous + Crash-recovery model

### Safety and Liveness

**Two types of properties**:

**Safety**: "Nothing bad happens"
- Examples:
  - No data loss
  - No data corruption
  - Uniqueness (no duplicate IDs)
- Must ALWAYS be true
- If violated once, can't be undone

**Liveness**: "Something good eventually happens"
- Examples:
  - Request eventually completes
  - Node eventually responds
- Allows delays
- Can retry

```
┌────────────────────────────────────────────────┐
│  SAFETY vs LIVENESS                            │
├────────────────────────────────────────────────┤
│                                                │
│  Safety:                                       │
│    "The system will never return wrong data"   │
│    Must be true at ALL times                   │
│    If violated → permanent damage              │
│                                                │
│  Liveness:                                     │
│    "The system will eventually respond"        │
│    May take time, but will happen              │
│    If violated → retries may help              │
└────────────────────────────────────────────────┘
```

**Trade-off**: In partially synchronous systems with crash-recovery nodes:
- Can guarantee **safety**
- Can guarantee **liveness** (with caveats)
- But difficult to guarantee both simultaneously!

## Summary

**Key Takeaways**:

1. **Partial Failures are Fundamental**
   - Can't avoid them in distributed systems
   - Must design for them from the start

2. **Networks are Unreliable**
   - Packets lost, delayed, duplicated
   - Timeouts are necessary but imperfect
   - No guarantees on delivery time

3. **Clocks are Unreliable**
   - Clocks drift apart
   - NTP synchronization is imperfect
   - Don't rely on synchronized clocks for ordering
   - Process pauses can cause stale data

4. **Truth is Subjective**
   - No node has complete information
   - Majority vote (quorum) determines truth
   - Byzantine faults are rare but serious

5. **System Models Help Reasoning**
   - Partially synchronous + crash-recovery most common
   - Safety vs liveness properties
   - Different guarantees require different assumptions

**Design Principles**:

```
✅ Assume networks will fail
✅ Use timeouts, but choose them carefully
✅ Don't depend on synchronized clocks for correctness
✅ Use logical clocks (Lamport timestamps) for ordering
✅ Use quorums and consensus for important decisions
✅ Design for crash-recovery (stable storage)
✅ Assume processes can pause unexpectedly
✅ Use fencing tokens to prevent split-brain
```

**Next Chapter**: Consistency and Consensus - how to build reliable distributed systems despite all these problems!
