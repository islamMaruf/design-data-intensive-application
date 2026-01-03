# Chapter 9: Consistency and Consensus

## Introduction

In distributed systems, achieving agreement among nodes is one of the hardest problems. This chapter explores the challenges and solutions for ensuring consistency and reaching consensus in the face of:
- **Network delays and partitions**
- **Node failures**  
- **Clock skew**
- **Concurrent operations**

We'll cover fundamental concepts:
- **Linearizability**: The strongest consistency guarantee
- **Ordering guarantees**: Causal consistency and total order
- **Distributed transactions**: Two-phase commit (2PC)
- **Consensus algorithms**: Paxos, Raft, and ZooKeeper
- **Membership and coordination**: How nodes agree on cluster state

Understanding consensus is essential for building correct distributed systems that behave predictably even when things go wrong.

## Part 1: Consistency Guarantees

### What is Consistency?

**Consistency** defines what guarantees a system provides about the state of data, especially during concurrent operations and failures.

**Spectrum of Consistency Models:**

```
Stronger (Harder to scale) ←→ Weaker (Easier to scale)

Linearizability
    │
Causal Consistency
    │
Sequential Consistency
    │
Eventual Consistency
    │
No Guarantees
```

### The Replication Lag Problem

In Chapter 5, we saw that replication lag causes temporary inconsistencies:

```
Write to Leader:
  t=0: SET user_123 = "Alice"

Followers:
  t=0:   Follower 1: user_123 = "Bob" (old value)
  t=0:   Follower 2: user_123 = "Bob" (old value)
  
  t=10ms: Follower 1: user_123 = "Alice" (updated)
  t=50ms: Follower 2: user_123 = "Alice" (updated)

Read from Follower 1 at t=5ms  → Returns "Bob"  (stale read!)
Read from Follower 2 at t=30ms → Returns "Bob"  (stale read!)
Read from Leader at t=0ms      → Returns "Alice" (up-to-date)
```

**Questions:**
- How can we provide stronger guarantees?
- What does it mean for a distributed system to "behave correctly"?

## Part 2: Linearizability

### Definition

**Linearizability** (also called atomic consistency or strong consistency):
> Once a write completes, all subsequent reads must return that value or a newer one.

**Intuition**: The system behaves as if there's only one copy of the data, and all operations are atomic.

### Linearizability Example

**Scenario**: Alice and Bob are both viewing a user profile.

**Timeline:**

```
Alice's operations:                Bob's operations:
    │                                  │
t=0 │ WRITE(user_123, "New Name")     │
    │ (acknowledged)                   │
    │                                  │
t=1 │                                  │ READ(user_123)
    │                                  │ → Must return "New Name"
    │                                  │    (not "Old Name")
```

**Linearizability guarantees**:
1. Once write is acknowledged, all reads see new value
2. Total ordering of operations exists
3. Operations appear to happen atomically

**Visual: Linearizable System**

```
Time →
Client 1: ──WRITE(x=1)─────────────────────────────
                │
                ✓ (acknowledged)
                │
Client 2: ──────────────READ(x)→1────────────────
                         │
                         └─ Must see new value!
```

### Non-Linearizable Behavior

**Example: Split-brain scenario**

```
Network partition:
┌─────────────┐         ┌─────────────┐
│  Node A     │    X    │  Node B     │
│  (Leader)   │  Partition │  (Leader)   │
└─────────────┘         └─────────────┘

Client 1 writes to Node A: x = 1
Client 2 writes to Node B: x = 2

Client 3 reads from Node A: x = 1
Client 4 reads from Node B: x = 2

This violates linearizability!
(No total order exists)
```

### What Makes a System Linearizable?

**Linearizability requires:**
1. **Recency guarantee**: Read returns most recent write
2. **Global ordering**: All operations have a total order
3. **Real-time ordering**: If op1 completes before op2 starts, op1 < op2 in total order

**Non-linearizable example:**

```python
# Thread 1
database.write('x', 1)
database.write('x', 2)

# Thread 2
print(database.read('x'))  # Prints 1
print(database.read('x'))  # Prints 2

# Thread 3
print(database.read('x'))  # Prints 2
print(database.read('x'))  # Prints 1  ← Violation! (went backwards)
```

**Linearizable system**: Cannot observe value going backwards.

### Linearizability vs Serializability

**Serializability** (Chapter 7): Isolation property for transactions
- Transactions execute as if in some serial order
- Order can be different from real-time order

**Linearizability**: Recency guarantee for reads/writes
- Operations reflect a real-time total order
- Single-object operations

```
┌─────────────────────────────────────────┐
│ Serializability:                        │
│   Transactions appear in SOME order     │
│   (may not match real-time)             │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ Linearizability:                        │
│   Operations appear in REAL-TIME order  │
│   (recency guarantee)                   │
└─────────────────────────────────────────┘

Both can be combined: "Strict Serializability"
```

### Use Cases Requiring Linearizability

**1. Locking and Leader Election**

```python
# Distributed lock using linearizable storage
def acquire_lock(lock_name, node_id):
    # Only one node can successfully write
    success = cas('lock:' + lock_name, 
                  expected=None, 
                  new_value=node_id)
    return success

# Without linearizability: Multiple nodes could acquire lock!
```

**2. Constraints and Uniqueness**

```sql
-- Enforce unique username across distributed database
INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');

-- Without linearizability: Two nodes could both insert 'alice'
```

**3. Cross-Channel Timing Dependencies**

```
User uploads profile photo:
  1. Upload image to storage service → URL
  2. Write URL to database
  3. Send notification "Photo uploaded"

Without linearizability:
  - Notification arrives
  - User tries to view photo
  - Database read returns old URL (stale!)
  - Photo not found
```

### Implementing Linearizable Systems

**Options:**

**1. Single-Leader Replication (potentially linearizable)**
- Reads from leader: Linearizable
- Reads from followers: NOT linearizable (replication lag)

```python
# Linearizable read (always from leader)
def read_linearizable(key):
    return leader.read(key)

# Non-linearizable read (from any replica)
def read_eventually_consistent(key):
    replica = random.choice(replicas)
    return replica.read(key)
```

**2. Consensus Algorithms (linearizable)**
- Raft, Paxos, ZooKeeper
- Coordinate writes across multiple nodes
- More expensive but truly linearizable

**3. Multi-Leader Replication (NOT linearizable)**
- Concurrent writes to different leaders
- Conflicts resolved asynchronously

**4. Leaderless Replication with Quorums (NOT necessarily linearizable)**
- Even with strict quorums (w + r > n)
- Concurrent writes and network delays cause issues

**Example: Quorum Non-Linearizability**

```
Configuration: 3 nodes, w=2, r=2

t=0: Client 1 writes x=1 to Node A, Node B (w=2, acknowledged)
t=1: Client 2 reads x from Node B, Node C → Returns x=1
t=2: Client 1 writes x=2 to Node A, Node C (w=2, acknowledged)
t=3: Client 2 reads x from Node A, Node B → Returns x=1 (stale!)

Problem: Read at t=3 doesn't see write at t=2, even though:
  - Write was acknowledged
  - Read happened after write
```

### The Cost of Linearizability

**CAP Theorem**:
> In a distributed system with network partitions, you must choose between Consistency (linearizability) and Availability.

```
Network partition:
┌─────────────┐         ┌─────────────┐
│  Node A     │    X    │  Node B     │
└─────────────┘         └─────────────┘

Choice 1: Prioritize Consistency (CP)
  - Reject reads/writes on minority partition
  - System becomes unavailable on minority side
  
Choice 2: Prioritize Availability (AP)
  - Accept reads/writes on both sides
  - Lose linearizability (divergent state)
```

**Real-World Trade-offs:**

| System | Choice | Justification |
|--------|--------|---------------|
| **Bank account balance** | CP (Consistency) | Can't allow overdrafts |
| **Social media likes** | AP (Availability) | Eventual consistency acceptable |
| **Inventory count** | CP | Can't oversell products |
| **News feed** | AP | Stale posts tolerable |

**Performance Impact:**

```python
# Linearizable write (slow: must wait for quorum)
start = time.time()
db.write_linearizable('key', 'value')
print(f"Latency: {time.time() - start:.3f}s")  # 50ms+

# Eventual consistency write (fast: async replication)
start = time.time()
db.write_async('key', 'value')
print(f"Latency: {time.time() - start:.3f}s")  # 1ms
```

## Part 3: Ordering Guarantees

### Causality

**Causal Consistency**: If event A causally depends on event B, all nodes see B before A.

**Example: Social Media Comments**

```
Alice: Posts "Check out this photo!"
  ↓ (causal dependency)
Bob: Comments "Nice photo!"

Without causal consistency:
  Charlie sees Bob's comment first, then Alice's post
  (confusing!)

With causal consistency:
  All users see Alice's post before Bob's comment
```

**Causal Dependencies:**

```
Events:
A: User 1 writes x=1
B: User 2 reads x=1, then writes y=2  (depends on A)
C: User 3 reads y=2, then writes z=3  (depends on B)

Causal order:
  A → B → C

Non-causal events (can be concurrent):
A: User 1 writes x=1
D: User 4 writes w=5  (independent of A)
```

**Visual: Causal Relationships**

```
      A (write x=1)
      │
      └──→ B (read x, write y=2)
           │
           └──→ C (read y, write z=3)

D (write w=5) ─────────────────────────
  (concurrent with A, B, C)
```

### The Total Order vs Partial Order

**Total Order**: Every pair of operations is comparable
- Example: Linearizability provides total order
- All operations happen "in a line"

```
Total Order:
  op1 → op2 → op3 → op4 → op5
```

**Partial Order**: Some operations are concurrent (incomparable)
- Example: Causality provides partial order
- Concurrent operations have no defined order

```
Partial Order:
        op1
       ↙   ↘
    op2     op3  (concurrent)
       ↘   ↙
        op4
```

### Sequence Number Ordering

**Idea**: Assign sequence numbers to operations to determine order.

**Lamport Timestamps** (Logical Clocks):

```python
class LamportClock:
    def __init__(self, node_id):
        self.counter = 0
        self.node_id = node_id
    
    def tick(self):
        """Increment on local event"""
        self.counter += 1
        return (self.counter, self.node_id)
    
    def update(self, received_timestamp):
        """Update on receiving message"""
        received_counter, _ = received_timestamp
        self.counter = max(self.counter, received_counter) + 1
        return (self.counter, self.node_id)

# Usage
node1 = LamportClock('node1')
node2 = LamportClock('node2')

# Node 1 events
t1 = node1.tick()  # (1, 'node1')
t2 = node1.tick()  # (2, 'node1')

# Node 2 receives message with timestamp t2
t3 = node2.update(t2)  # (3, 'node2')
t4 = node2.tick()      # (4, 'node2')

# Ordering: t1 < t2 < t3 < t4
```

**Lamport Timestamp Comparison:**

```python
def compare_timestamps(t1, t2):
    """Compare (counter, node_id) tuples"""
    counter1, node1 = t1
    counter2, node2 = t2
    
    if counter1 < counter2:
        return -1  # t1 < t2
    elif counter1 > counter2:
        return 1   # t1 > t2
    else:
        # Break ties with node ID
        return -1 if node1 < node2 else 1
```

**Limitation**: Lamport timestamps provide total order but don't capture causality completely.
- If t1 < t2, we can't conclude that t1 caused t2
- Could be concurrent operations

### Version Vectors (Detecting Concurrency)

**Version vectors** track causality accurately:

```python
class VersionVector:
    def __init__(self, node_id):
        self.node_id = node_id
        self.vector = {}  # node_id → counter
    
    def increment(self):
        """Increment own counter"""
        self.vector[self.node_id] = self.vector.get(self.node_id, 0) + 1
        return self.vector.copy()
    
    def merge(self, other_vector):
        """Merge with received vector"""
        for node, count in other_vector.items():
            self.vector[node] = max(self.vector.get(node, 0), count)
        self.increment()
        return self.vector.copy()
    
    def happens_before(self, other_vector):
        """Check if this happens before other"""
        for node, count in other_vector.items():
            if self.vector.get(node, 0) > count:
                return False  # Not happens-before
        return True
    
    def is_concurrent(self, other_vector):
        """Check if operations are concurrent"""
        return not self.happens_before(other_vector) and \
               not other_happens_before_self(other_vector)

# Example
node1 = VersionVector('node1')
node2 = VersionVector('node2')

v1 = node1.increment()  # {'node1': 1}
v2 = node1.increment()  # {'node1': 2}

v3 = node2.increment()  # {'node2': 1}
node2.merge(v2)         # {'node1': 2, 'node2': 2}

# v1 happens-before v2: True
# v3 concurrent with v1: True
```

**Visual: Version Vectors**

```
Node 1:              Node 2:
{node1: 1}           {node2: 1}
    │                    │
{node1: 2}           {node2: 2}
    │ send to node2      │
    │                    ▼
    │                {node1: 2, node2: 3}
    │ send to node2      │
    ▼                    ▼
{node1: 3, node2: 3}  ...
```

## Part 4: Distributed Transactions and Two-Phase Commit

### The Problem: Atomic Commit Across Multiple Nodes

**Scenario**: Transfer $100 from Account A (Node 1) to Account B (Node 2)

```
Transaction:
  1. Deduct $100 from Account A (Node 1)
  2. Add $100 to Account B (Node 2)

Challenge: Both must succeed or both must fail (atomicity)!
```

**Failure Scenarios:**

```
Scenario 1: Node 1 commits, Node 2 crashes
  → Money deducted but not added (lost $100!)

Scenario 2: Node 1 commits, Node 2 aborts due to constraint violation
  → Money deducted but not added (lost $100!)
```

### Two-Phase Commit (2PC)

**2PC** ensures atomic commit across multiple nodes using a **coordinator**.

**Phase 1: Prepare**
```
Coordinator → Participants: "Prepare to commit"
Participants → Coordinator: "OK" or "Abort"
```

**Phase 2: Commit/Abort**
```
If all participants voted "OK":
  Coordinator → Participants: "Commit"
Else:
  Coordinator → Participants: "Abort"
```

**Visual: 2PC Protocol**

```
Coordinator                 Node 1              Node 2
     │                          │                   │
     │──── Prepare ────────────>│                   │
     │                          │                   │
     │──── Prepare ──────────────────────────────>│
     │                          │                   │
     │<──── OK ─────────────────│                   │
     │                          │                   │
     │<──── OK ──────────────────────────────────│
     │                          │                   │
     │─── Commit ──────────────>│                   │
     │                          │                   │
     │─── Commit ────────────────────────────────>│
     │                          │                   │
     │                       (commit)            (commit)
```

**Implementation:**

```python
class TwoPhaseCommitCoordinator:
    def __init__(self, participants):
        self.participants = participants
        self.transaction_log = []
    
    def execute_transaction(self, transaction):
        transaction_id = generate_id()
        
        # Phase 1: Prepare
        self.transaction_log.append(('PREPARE', transaction_id))
        votes = []
        
        for participant in self.participants:
            try:
                vote = participant.prepare(transaction_id, transaction)
                votes.append(vote)
            except Exception:
                votes.append('ABORT')
        
        # Decision
        if all(vote == 'OK' for vote in votes):
            decision = 'COMMIT'
        else:
            decision = 'ABORT'
        
        self.transaction_log.append((decision, transaction_id))
        
        # Phase 2: Commit or Abort
        for participant in self.participants:
            if decision == 'COMMIT':
                participant.commit(transaction_id)
            else:
                participant.abort(transaction_id)
        
        return decision

class Participant:
    def __init__(self):
        self.prepared_transactions = {}
    
    def prepare(self, transaction_id, transaction):
        """Prepare phase: check if can commit"""
        try:
            # Check constraints, acquire locks, etc.
            self.prepared_transactions[transaction_id] = transaction
            return 'OK'
        except Exception:
            return 'ABORT'
    
    def commit(self, transaction_id):
        """Commit phase: actually apply changes"""
        transaction = self.prepared_transactions.pop(transaction_id)
        # Apply changes to database
        self.apply(transaction)
    
    def abort(self, transaction_id):
        """Abort phase: discard prepared transaction"""
        self.prepared_transactions.pop(transaction_id, None)
```

### Problems with Two-Phase Commit

**1. Coordinator Failure**

```
Coordinator crashes after some participants committed:
┌──────────────┐
│ Coordinator  │  ← CRASH (after sending some COMMIT messages)
└──────────────┘
        │
    ┌───┴───┐
    ▼       ▼
Node 1    Node 2
(committed) (waiting...)

Node 2 is BLOCKED: Can't commit or abort without coordinator!
```

**Solution**: Coordinator writes decision to durable log before sending.
- On recovery, re-send commit/abort messages

**2. Participant Failure**

```
Node 2 crashes after voting OK but before receiving COMMIT:

Coordinator: Sends COMMIT
Node 1: Commits successfully
Node 2: CRASH (in "uncertain" state)

On recovery, Node 2 must ask coordinator for decision.
```

**3. Performance Issues**

```
Latency comparison:
Single-node transaction:  10ms
2PC transaction:          100ms+  (10x slower!)

Reasons:
- Multiple network round-trips
- Synchronous communication
- Locks held across network calls
```

**4. Blocking Protocol**

```
If coordinator crashes, participants are blocked:

Node 1: "I voted OK, but coordinator is gone. Can I commit?"
  → Cannot commit (don't know if Node 2 voted OK)
  → Cannot abort (coordinator might have committed)
  → BLOCKED until coordinator recovers
```

### Three-Phase Commit (3PC)

**Improvement**: Add a "PreCommit" phase to avoid blocking.

**Phases:**
1. **Prepare**: Can you commit?
2. **PreCommit**: All voted OK, prepare to commit
3. **Commit**: Actually commit

**Advantage**: If coordinator fails after PreCommit, participants know everyone voted OK.

**Disadvantage**: Still not practical due to network partitions.

## Part 5: Consensus Algorithms

### What is Consensus?

**Consensus Problem**: Get multiple nodes to agree on a single value.

**Requirements:**
1. **Agreement**: All correct nodes decide on the same value
2. **Integrity**: Nodes decide at most once
3. **Validity**: Decided value was proposed by some node
4. **Termination**: All correct nodes eventually decide

**Applications:**
- **Leader election**: Agree on which node is leader
- **Atomic commit**: Agree to commit or abort
- **State machine replication**: Agree on sequence of operations

### FLP Impossibility Result

**Fischer, Lynch, Paterson (1985)**: 
> No deterministic consensus algorithm is guaranteed to terminate in an asynchronous system with even one faulty process.

**Implication**: Perfect consensus is impossible!

**Practical solution**: Use timeouts and retries (non-deterministic).

### Paxos Algorithm

Paxos is the most famous consensus algorithm, invented by Leslie Lamport (1989).

**Roles:**
- **Proposer**: Proposes values
- **Acceptor**: Votes on proposals
- **Learner**: Learns decided value

**Paxos Phases:**

**Phase 1: Prepare**
```
Proposer → Acceptors: "Prepare(n)"
  (n = unique proposal number)

Acceptors → Proposer: "Promise(n, v)"
  v = highest-numbered proposal already accepted (or null)
  Promise not to accept proposals numbered < n
```

**Phase 2: Accept**
```
Proposer → Acceptors: "Accept(n, v)"
  v = value from Phase 1, or proposer's own value

Acceptors → Learners: "Accepted(n, v)"
  Accept proposal if n is still highest seen
```

**Visual: Paxos Execution**

```
Proposer 1          Acceptor A     Acceptor B     Acceptor C
     │                   │              │              │
     │─ Prepare(5) ─────>│              │              │
     │                   │              │              │
     │─ Prepare(5) ──────────────────>│              │
     │                   │              │              │
     │─ Prepare(5) ───────────────────────────────>│
     │                   │              │              │
     │<── Promise(5) ────│              │              │
     │<── Promise(5) ─────────────────│              │
     │<── Promise(5) ──────────────────────────────│
     │                   │              │              │
     │─ Accept(5,"A") ──>│              │              │
     │─ Accept(5,"A") ───────────────>│              │
     │─ Accept(5,"A") ────────────────────────────>│
     │                   │              │              │
     │                (accepted)     (accepted)     (accepted)
```

**Example: Competing Proposers**

```python
# Simplified Paxos implementation
class PaxosAcceptor:
    def __init__(self):
        self.promised_proposal = None
        self.accepted_proposal = None
        self.accepted_value = None
    
    def receive_prepare(self, proposal_number):
        """Phase 1: Prepare"""
        if self.promised_proposal is None or proposal_number > self.promised_proposal:
            self.promised_proposal = proposal_number
            return ('PROMISE', self.accepted_proposal, self.accepted_value)
        else:
            return ('REJECT', None, None)
    
    def receive_accept(self, proposal_number, value):
        """Phase 2: Accept"""
        if self.promised_proposal is None or proposal_number >= self.promised_proposal:
            self.promised_proposal = proposal_number
            self.accepted_proposal = proposal_number
            self.accepted_value = value
            return ('ACCEPTED', value)
        else:
            return ('REJECT', None)

class PaxosProposer:
    def __init__(self, node_id, acceptors):
        self.node_id = node_id
        self.acceptors = acceptors
        self.proposal_number = 0
    
    def propose(self, value):
        """Run Paxos to get value accepted"""
        self.proposal_number += 1
        proposal = (self.proposal_number, self.node_id)  # Unique proposal number
        
        # Phase 1: Prepare
        promises = []
        for acceptor in self.acceptors:
            response = acceptor.receive_prepare(proposal)
            if response[0] == 'PROMISE':
                promises.append(response)
        
        # Need majority
        if len(promises) < len(self.acceptors) // 2 + 1:
            return None  # Failed to get majority
        
        # Check if any acceptor already accepted a value
        accepted_values = [p[2] for p in promises if p[2] is not None]
        if accepted_values:
            # Use highest-numbered accepted value
            value = max(accepted_values, key=lambda v: promises[2][1])
        
        # Phase 2: Accept
        accepts = []
        for acceptor in self.acceptors:
            response = acceptor.receive_accept(proposal, value)
            if response[0] == 'ACCEPTED':
                accepts.append(response)
        
        # Need majority
        if len(accepts) >= len(self.acceptors) // 2 + 1:
            return value  # Consensus reached!
        else:
            return None  # Failed
```

### Raft Algorithm

**Raft** is a consensus algorithm designed to be easier to understand than Paxos.

**Key Idea**: Strong leader
- Leader handles all client requests
- Followers replicate leader's log
- New leader elected if current leader fails

**Raft Components:**

**1. Leader Election**

```
All nodes start as followers:
┌──────────┐  ┌──────────┐  ┌──────────┐
│Follower A│  │Follower B│  │Follower C│
└──────────┘  └──────────┘  └──────────┘

If no heartbeat from leader → election timeout:
┌──────────┐  ┌──────────┐  ┌──────────┐
│Candidate │  │Follower B│  │Follower C│
│   A      │  │          │  │          │
└──────────┘  └──────────┘  └──────────┘
      │           │              │
      │─ Vote ───>│              │
      │─ Vote ────────────────>│
      │           │              │
      │<── OK ────│              │
      │<── OK ─────────────────│
      │           │              │
    (Becomes Leader)
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Leader A │  │Follower B│  │Follower C│
└──────────┘  └──────────┘  └──────────┘
```

**2. Log Replication**

```
Leader receives write:
┌──────────────────────────┐
│ Leader Log:              │
│ [1: x=1] [2: y=2] [3: z=3]│ ← Append entry 3
└──────────────────────────┘
     │            │
     │ Replicate  │
     ▼            ▼
┌────────┐   ┌────────┐
│Follower│   │Follower│
│[1][2]  │   │[1][2]  │
└────────┘   └────────┘
     │            │
     ▼            ▼
┌────────┐   ┌────────┐
│[1][2][3]│   │[1][2][3]│ ← Acknowledge
└────────┘   └────────┘
     │            │
     └────┬───────┘
          │ Majority replicated
          ▼
   Leader commits entry 3
```

**3. Safety**

**Election Safety**: At most one leader per term
**Leader Append-Only**: Leader never deletes/overwrites log entries
**Log Matching**: If two logs contain entry with same index/term, all preceding entries are identical
**Leader Completeness**: If entry committed in term T, it appears in all leaders' logs for terms > T
**State Machine Safety**: If a server applies log entry at given index, no other server applies different entry at that index

**Raft Implementation (Simplified):**

```python
class RaftNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.peers = peers
        self.state = 'FOLLOWER'  # FOLLOWER, CANDIDATE, or LEADER
        self.current_term = 0
        self.voted_for = None
        self.log = []  # List of (term, command) tuples
        self.commit_index = 0
        self.last_heartbeat = time.time()
    
    def start_election(self):
        """Become candidate and request votes"""
        self.state = 'CANDIDATE'
        self.current_term += 1
        self.voted_for = self.node_id
        votes = 1  # Vote for self
        
        for peer in self.peers:
            response = peer.request_vote(
                term=self.current_term,
                candidate_id=self.node_id,
                last_log_index=len(self.log) - 1,
                last_log_term=self.log[-1][0] if self.log else 0
            )
            if response['vote_granted']:
                votes += 1
        
        # Need majority
        if votes > (len(self.peers) + 1) // 2:
            self.become_leader()
    
    def become_leader(self):
        """Transition to leader state"""
        self.state = 'LEADER'
        print(f"Node {self.node_id} became leader for term {self.current_term}")
        self.send_heartbeats()
    
    def append_entry(self, command):
        """Client request to append entry (leader only)"""
        if self.state != 'LEADER':
            return {'success': False, 'leader': self.current_leader}
        
        # Append to own log
        self.log.append((self.current_term, command))
        log_index = len(self.log) - 1
        
        # Replicate to followers
        acks = 1  # Count self
        for peer in self.peers:
            response = peer.append_entries(
                term=self.current_term,
                leader_id=self.node_id,
                prev_log_index=log_index - 1,
                prev_log_term=self.log[log_index - 1][0] if log_index > 0 else 0,
                entries=[(self.current_term, command)],
                leader_commit=self.commit_index
            )
            if response['success']:
                acks += 1
        
        # Commit if majority replicated
        if acks > (len(self.peers) + 1) // 2:
            self.commit_index = log_index
            return {'success': True}
        else:
            return {'success': False}
```

### Raft vs Paxos

| Aspect | Paxos | Raft |
|--------|-------|------|
| **Understandability** | Complex | Simpler |
| **Leadership** | Weak (can have multiple proposers) | Strong (single leader) |
| **Log structure** | Can have holes | No holes (sequential) |
| **Performance** | Potentially higher | Easier to implement efficiently |
| **Adoption** | Chubby (Google) | etcd, Consul, CockroachDB |

## Part 6: Coordination Services: ZooKeeper and etcd

### ZooKeeper

**ZooKeeper** is a distributed coordination service that implements a consensus algorithm (ZAB, similar to Raft/Paxos).

**Use Cases:**
- **Configuration management**: Centralized configuration
- **Service discovery**: Find service instances
- **Distributed locking**: Coordinate access
- **Leader election**: Choose primary node
- **Group membership**: Track live nodes

**Data Model: Hierarchical namespace (like filesystem)**

```
/
├── /config
│   ├── /config/database
│   │   └── connection_string="..."
│   └── /config/cache
│       └── ttl=300
├── /services
│   ├── /services/api
│   │   ├── /services/api/node1  (ephemeral)
│   │   └── /services/api/node2  (ephemeral)
│   └── /services/worker
│       └── /services/worker/node3  (ephemeral)
└── /locks
    └── /locks/critical_section
```

**Node Types:**
- **Persistent**: Survive until explicitly deleted
- **Ephemeral**: Deleted when client session ends
- **Sequential**: Appended with monotonic counter

**ZooKeeper Operations:**

```python
from kazoo.client import KazooClient

zk = KazooClient(hosts='localhost:2181')
zk.start()

# Create node
zk.create('/config/database', b'postgresql://...')

# Read node
data, stat = zk.get('/config/database')
print(data.decode())  # postgresql://...

# Watch for changes
@zk.DataWatch('/config/database')
def watch_database_config(data, stat):
    print(f"Config changed: {data.decode()}")

# Leader election
def become_leader():
    # Create ephemeral sequential node
    path = zk.create('/election/node_', ephemeral=True, sequence=True)
    
    # Get all election nodes
    children = zk.get_children('/election')
    children.sort()
    
    # If we have the smallest sequence number, we're the leader
    if path.endswith(children[0]):
        print("I am the leader!")
        return True
    else:
        # Watch the node before us
        previous_node = children[children.index(path.split('/')[-1]) - 1]
        zk.exists(f'/election/{previous_node}', watch=on_previous_node_deleted)
        return False

def on_previous_node_deleted(event):
    # Previous leader died, try to become leader
    become_leader()
```

**Distributed Lock with ZooKeeper:**

```python
from kazoo.recipe.lock import Lock

# Distributed lock
lock = Lock(zk, '/locks/my_resource')

# Acquire lock
with lock:
    # Critical section
    print("I have the lock!")
    # Perform operations...
# Lock automatically released
```

### etcd

**etcd** is a distributed key-value store that uses the Raft consensus algorithm.

**Key Features:**
- **Strongly consistent**: Linearizable reads/writes
- **Watch API**: Notifications on key changes
- **Lease mechanism**: TTL for keys
- **Transactions**: Multi-key atomic operations

**Example: Service Registration**

```python
import etcd3

etcd = etcd3.client()

# Register service instance
def register_service(service_name, instance_id, address):
    lease = etcd.lease(ttl=30)  # 30-second lease
    
    key = f'/services/{service_name}/{instance_id}'
    value = json.dumps({'address': address, 'registered_at': time.time()})
    
    etcd.put(key, value, lease=lease)
    
    # Keep-alive heartbeat
    while True:
        time.sleep(10)
        lease.refresh()

# Discover service instances
def discover_services(service_name):
    prefix = f'/services/{service_name}/'
    instances = []
    
    for value, metadata in etcd.get_prefix(prefix):
        instance = json.loads(value.decode())
        instances.append(instance)
    
    return instances

# Watch for service changes
def watch_services(service_name):
    prefix = f'/services/{service_name}/'
    
    watch_id = etcd.add_watch_prefix_callback(prefix, on_service_change)
    
def on_service_change(event):
    if isinstance(event, etcd3.events.PutEvent):
        print(f"Service added: {event.key.decode()}")
    elif isinstance(event, etcd3.events.DeleteEvent):
        print(f"Service removed: {event.key.decode()}")
```

**etcd Transactions (MVCC):**

```python
# Atomic compare-and-set
success = etcd.transaction(
    compare=[
        etcd.transactions.value('/counter') == b'5'
    ],
    success=[
        etcd.transactions.put('/counter', b'6')
    ],
    failure=[
        etcd.transactions.get('/counter')
    ]
)
```

## Part 7: Membership and Failure Detection

### Failure Detection

**Challenge**: Distinguish between slow nodes and crashed nodes.

**Heartbeat Protocol:**

```python
class FailureDetector:
    def __init__(self, timeout=5.0):
        self.timeout = timeout
        self.last_heartbeat = {}
    
    def heartbeat_received(self, node_id):
        """Record heartbeat from node"""
        self.last_heartbeat[node_id] = time.time()
    
    def is_alive(self, node_id):
        """Check if node is alive"""
        if node_id not in self.last_heartbeat:
            return False
        
        elapsed = time.time() - self.last_heartbeat[node_id]
        return elapsed < self.timeout
    
    def get_live_nodes(self):
        """Return list of live nodes"""
        return [node for node in self.last_heartbeat if self.is_alive(node)]
```

**Phi Accrual Failure Detector** (used in Cassandra):
- Instead of binary alive/dead, compute suspicion level (Φ)
- Higher Φ = more suspicious
- Adapt to network conditions

```python
import math
from collections import deque

class PhiAccrualFailureDetector:
    def __init__(self, threshold=8.0, window_size=100):
        self.threshold = threshold
        self.window_size = window_size
        self.intervals = deque(maxlen=window_size)
        self.last_heartbeat = None
    
    def heartbeat(self):
        """Record heartbeat"""
        now = time.time()
        if self.last_heartbeat is not None:
            interval = now - self.last_heartbeat
            self.intervals.append(interval)
        self.last_heartbeat = now
    
    def phi(self):
        """Calculate suspicion level"""
        if not self.intervals or self.last_heartbeat is None:
            return 0.0
        
        # Time since last heartbeat
        now = time.time()
        elapsed = now - self.last_heartbeat
        
        # Calculate mean and std dev of intervals
        mean = sum(self.intervals) / len(self.intervals)
        variance = sum((x - mean) ** 2 for x in self.intervals) / len(self.intervals)
        std_dev = math.sqrt(variance)
        
        # Phi value (higher = more suspicious)
        if std_dev == 0:
            return 0.0
        
        phi = -math.log10(1 - normal_cdf((elapsed - mean) / std_dev))
        return phi
    
    def is_available(self):
        """Check if node is available"""
        return self.phi() < self.threshold
```

### Gossip Protocols

**Gossip** (epidemic protocol): Nodes randomly exchange information.

**Example: SWIM (Scalable Weakly-consistent Infection-style Process Group Membership)**

```python
class GossipProtocol:
    def __init__(self, node_id, all_nodes):
        self.node_id = node_id
        self.all_nodes = all_nodes
        self.state = {}  # node_id → (state, timestamp)
    
    def update_local_state(self, key, value):
        """Update local state"""
        self.state[key] = (value, time.time())
    
    def gossip_round(self):
        """Periodically gossip with random node"""
        peer = random.choice([n for n in self.all_nodes if n != self.node_id])
        
        # Send our state to peer
        peer_state = peer.receive_gossip(self.state)
        
        # Merge peer's state into ours
        self.merge_state(peer_state)
    
    def receive_gossip(self, peer_state):
        """Receive gossip from peer"""
        self.merge_state(peer_state)
        return self.state
    
    def merge_state(self, peer_state):
        """Merge peer state (keep more recent values)"""
        for key, (value, timestamp) in peer_state.items():
            if key not in self.state or timestamp > self.state[key][1]:
                self.state[key] = (value, timestamp)
```

**Gossip Convergence:**

```
Time 0: Node A has update X
┌───┐ ┌───┐ ┌───┐ ┌───┐
│ X │ │   │ │   │ │   │
└───┘ └───┘ └───┘ └───┘

Time 1: A gossips to B
┌───┐ ┌───┐ ┌───┐ ┌───┐
│ X │ │ X │ │   │ │   │
└───┘ └───┘ └───┘ └───┘

Time 2: A→C, B→D
┌───┐ ┌───┐ ┌───┐ ┌───┐
│ X │ │ X │ │ X │ │ X │
└───┘ └───┘ └───┘ └───┘

All nodes converge to X
```

## Summary

**Key Takeaways:**

1. **Linearizability** is the strongest consistency model but expensive
   - Requires coordination between nodes
   - Incompatible with high availability during partitions (CAP)

2. **Causality** provides weaker but more scalable consistency
   - Tracks happens-before relationships
   - Can be implemented without coordination

3. **Consensus** is fundamental to distributed systems
   - Required for leader election, atomic commit, state machine replication
   - Paxos and Raft are practical consensus algorithms
   - FLP impossibility: perfect consensus is impossible in asynchronous systems

4. **Coordination services** (ZooKeeper, etcd) provide:
   - Linearizable operations
   - Failure detection
   - Membership management
   - Distributed locking

5. **Trade-offs**:
   - Stronger consistency → Lower performance/availability
   - Weaker consistency → Higher performance but complex application logic

**Consistency Spectrum:**

```
Linearizability (Strongest)
    ↓ (Less coordination required)
Causal Consistency
    ↓
Sequential Consistency
    ↓
Eventual Consistency
    ↓
No Guarantees (Weakest)
```

**Looking Ahead:**
In Chapters 10-12, we'll explore how these consistency and consensus principles apply to batch and stream processing systems, and how to build correct, scalable data systems that integrate multiple technologies.
