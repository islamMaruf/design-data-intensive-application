# Chapter 7: Transactions

## Introduction: What Could Go Wrong?

Imagine you're building a banking application. A user transfers $100 from their checking account to their savings account. Simple, right?

```python
# Transfer $100
checking_balance = get_balance("checking")
savings_balance = get_balance("savings")

checking_balance -= 100  # Deduct from checking
savings_balance += 100   # Add to savings

save_balance("checking", checking_balance)
save_balance("savings", savings_balance)
```

**What could go wrong?**

1. **Power outage** after deducting from checking but before adding to savings → Money disappears! 💸
2. **Two transfers** happening simultaneously → Race condition, wrong balances 🏃‍♂️💥
3. **System crash** in the middle → Partial update, inconsistent state 💥
4. **Network failure** → One update succeeds, other fails 🌐❌

**Solution**: **Transactions** - a way to group multiple operations so they succeed or fail together as a single unit.

```
┌────────────────────────────────────────────────┐
│      WITHOUT TRANSACTIONS                      │
├────────────────────────────────────────────────┤
│  Step 1: checking -= 100  ✅                   │
│  Step 2: CRASH! 💥                             │
│  Step 3: savings += 100  ❌ Never happens      │
│                                                │
│  Result: Money lost! 💸                        │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│      WITH TRANSACTIONS                         │
├────────────────────────────────────────────────┤
│  BEGIN TRANSACTION                             │
│    Step 1: checking -= 100                     │
│    Step 2: CRASH! 💥                           │
│  ROLLBACK (undo everything)                    │
│                                                │
│  Result: No change, money safe! ✅             │
└────────────────────────────────────────────────┘
```

**This chapter explores**:
- What guarantees transactions provide
- Different isolation levels and their trade-offs
- How to achieve serializability (strongest consistency)

## Part 1: ACID - The Promise of Transactions

**ACID** is an acronym describing the guarantees transactions provide:
- **A**tomicity
- **C**onsistency
- **I**solation
- **D**urability

Let's understand each one with real-world examples.

### Atomicity: All or Nothing

**Definition**: Either all operations in a transaction succeed, or none do. No partial updates.

**Real-World Analogy**: Atomic bomb - it either explodes completely or not at all. No half-explosion!

**Example - E-commerce Order**:

```python
def place_order(user_id, product_id, quantity):
    try:
        # BEGIN TRANSACTION
        db.begin_transaction()
        
        # Step 1: Check and update inventory
        inventory = db.query("SELECT quantity FROM inventory WHERE product_id = ?", product_id)
        if inventory < quantity:
            raise Exception("Out of stock")
        db.execute("UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?", quantity, product_id)
        
        # Step 2: Charge payment
        db.execute("INSERT INTO charges (user_id, amount) VALUES (?, ?)", user_id, price * quantity)
        
        # Step 3: Create order
        db.execute("INSERT INTO orders (user_id, product_id, quantity) VALUES (?, ?, ?)", 
                   user_id, product_id, quantity)
        
        # If we got here, all steps succeeded!
        db.commit()  # Make changes permanent
        return "Order placed successfully"
        
    except Exception as e:
        # Something went wrong!
        db.rollback()  # Undo ALL changes
        return f"Order failed: {e}"
```

**What Atomicity Prevents**:
```
❌ Inventory decremented but payment failed → Free products!
❌ Payment charged but inventory not decremented → Overselling!
❌ Order created but payment failed → Unpaid orders!

✅ With atomicity: Either ALL steps succeed, or ALL are undone!
```

**How It Works**:

Databases use a **write-ahead log (WAL)** to track changes:

```
┌────────────────────────────────────────────────┐
│      WRITE-AHEAD LOG                           │
├────────────────────────────────────────────────┤
│  Transaction ID: 12345                         │
│  BEGIN                                         │
│  UPDATE inventory SET qty = 9 WHERE id = 5     │
│  INSERT INTO charges VALUES (...)              │
│  INSERT INTO orders VALUES (...)               │
│  COMMIT                                        │
└────────────────────────────────────────────────┘
```

If crash happens before COMMIT:
- On restart, database reads WAL
- Sees transaction 12345 has no COMMIT
- Undoes all changes from transaction 12345
- **Atomicity preserved!** ✅

### Consistency: Valid State to Valid State

**Definition**: Database moves from one valid state to another valid state. Invariants are maintained.

**Real-World Analogy**: Conservation of energy in physics - energy transforms but total amount stays constant.

**Example - Double-Entry Bookkeeping**:

```sql
-- Invariant: Total debits = Total credits (always balanced books)

-- Transfer $100 from Account A to Account B
BEGIN TRANSACTION;
  -- Debit Account A
  INSERT INTO transactions (account, type, amount) 
  VALUES ('A', 'DEBIT', 100);
  
  -- Credit Account B
  INSERT INTO transactions (account, type, amount) 
  VALUES ('B', 'CREDIT', 100);
COMMIT;

-- Check invariant:
SELECT 
  SUM(CASE WHEN type = 'DEBIT' THEN amount ELSE 0 END) as total_debits,
  SUM(CASE WHEN type = 'CREDIT' THEN amount ELSE 0 END) as total_credits
FROM transactions;

-- Result: total_debits = total_credits ✅
```

**What Consistency Prevents**:
```
❌ Account balance becomes negative (violates constraint)
❌ Order references non-existent product (violates foreign key)
❌ Duplicate email in users table (violates uniqueness)

✅ With consistency: All constraints are respected!
```

**Important Note**: Consistency is actually the application's responsibility!

```python
# Database ensures constraints (foreign keys, NOT NULL, UNIQUE)
# Application ensures business logic invariants

def transfer_money(from_account, to_account, amount):
    # Application checks business rule
    if amount > 10000:
        raise Exception("Transfer limit exceeded")
    
    # Database ensures constraints
    db.execute("UPDATE accounts SET balance = balance - ? WHERE id = ?", 
               amount, from_account)
    # If balance goes negative, database raises error (CHECK constraint)
```

### Isolation: Concurrent Transactions Don't Interfere

**Definition**: Concurrent transactions execute as if they ran serially (one after another), even though they run simultaneously.

**Real-World Analogy**: Soundproof rooms - multiple meetings happening simultaneously, but each room is isolated.

**The Problem - Without Isolation**:

```
Time: 0s
  Balance: $1000

Time: 1s
  Transaction A: Read balance = $1000
  Transaction B: Read balance = $1000

Time: 2s
  Transaction A: Deposit $100 → New balance = $1100
  Transaction B: Withdraw $50 → New balance = $950

Time: 3s
  Transaction A: Write balance = $1100
  Transaction B: Write balance = $950

Final balance: $950 ❌
Expected: $1050 ($1000 + $100 - $50)

Lost $100! 💸
```

**With Isolation**:

```
Time: 0s
  Balance: $1000

Time: 1s
  Transaction A: Read balance = $1000, LOCK
  Transaction B: Tries to read... BLOCKED (waits for A)

Time: 2s
  Transaction A: Deposit $100 → balance = $1100
  Transaction A: COMMIT, UNLOCK

Time: 3s
  Transaction B: Read balance = $1100
  Transaction B: Withdraw $50 → balance = $1050
  Transaction B: COMMIT

Final balance: $1050 ✅
```

**We'll explore isolation levels in detail later!**

### Durability: Data Survives Crashes

**Definition**: Once a transaction commits, data is permanently stored (survives crashes, power failures).

**Real-World Analogy**: Signed contract - once signed, it's legally binding even if the building burns down (you have copies).

**How Durability Works**:

```
┌────────────────────────────────────────────────┐
│      DURABILITY MECHANISMS                     │
├────────────────────────────────────────────────┤
│                                                │
│  1. Write-Ahead Log (WAL)                      │
│     - Write changes to disk-based log          │
│     - BEFORE updating actual data              │
│     - Log is append-only (fast!)               │
│                                                │
│  2. Fsync                                      │
│     - Force OS to flush to physical disk       │
│     - Bypasses OS cache                        │
│     - Guarantees data on disk                  │
│                                                │
│  3. Replication                                │
│     - Copy to multiple machines                │
│     - Survives hardware failure                │
│                                                │
│  Process:                                      │
│  WRITE → WAL → fsync → COMMIT → Update data   │
└────────────────────────────────────────────────┘
```

**Example**:

```python
def commit_transaction(transaction_id):
    # 1. Write all changes to WAL
    wal.write(transaction_id, changes)
    
    # 2. Force WAL to disk (fsync)
    wal.fsync()  # This is the EXPENSIVE part!
    
    # 3. Mark as committed
    wal.write(transaction_id, "COMMIT")
    wal.fsync()
    
    # NOW safe to tell user transaction succeeded!
    return "SUCCESS"
    
    # 4. Later: Apply changes to actual data files (can be async)
    apply_changes_to_data_files(transaction_id)
```

**Durability Trade-offs**:

```
┌──────────────────────────────────────────────┐
│  DURABILITY LEVELS                           │
├────────────┬─────────────┬──────────────────┤
│   Mode     │ Performance │   Durability     │
├────────────┼─────────────┼──────────────────┤
│ No fsync   │  Very Fast  │  ❌ Data loss    │
│ fsync WAL  │    Fast     │  ✅ Single disk  │
│ Replication│   Medium    │  ✅ Multi-disk   │
│ Sync Repl. │    Slow     │  ✅✅ Multi-DC   │
└────────────┴─────────────┴──────────────────┘
```

**Real-World Example - Redis**:

Redis offers multiple durability options:

```conf
# redis.conf

# Option 1: No persistence (FASTEST, loses all data on crash)
save ""

# Option 2: RDB snapshots (periodic saves)
save 900 1      # Save if 1 key changed in 900 seconds
save 300 10     # Save if 10 keys changed in 300 seconds

# Option 3: AOF (append-only file) - log every write
appendonly yes
appendfsync everysec  # Fsync every second (fast, might lose 1 sec)
# appendfsync always  # Fsync every write (slow, fully durable)
```

## Part 2: Weak Isolation Levels

Perfect isolation (serializability) is expensive. Most databases offer weaker isolation levels for better performance.

### Read Committed

**Two Guarantees**:
1. **No dirty reads**: Only see committed data
2. **No dirty writes**: Only overwrite committed data

#### Problem 1: Dirty Reads

**What is a dirty read?**
Reading data written by a transaction that hasn't committed yet.

```
Timeline:
T0: Balance = $1000

T1: Transaction A begins
    A: UPDATE accounts SET balance = 500  (not committed yet!)

T2: Transaction B begins
    B: SELECT balance FROM accounts
    B: Sees balance = 500 ❌ (DIRTY READ!)
    B: Decides to allow withdrawal

T3: Transaction A: ROLLBACK  (oops, changed mind!)
    Balance returns to $1000

T4: Transaction B commits based on wrong data!
```

**Why dirty reads are bad**:
- Transaction B made decisions based on data that was rolled back
- Data that B saw never actually existed in committed state
- Leads to incorrect business logic

**Solution - Read Committed**:

```
T1: Transaction A: balance = 500 (uncommitted)

T2: Transaction B: SELECT balance
    → Database returns OLD value: $1000 ✅
    (ignores uncommitted changes)

T3: Transaction A: COMMIT
    
T4: Transaction B: SELECT balance
    → Database returns NEW value: $500 ✅
```

**Implementation**: Database keeps both old and new values until commit.

#### Problem 2: Dirty Writes

**What is a dirty write?**
Overwriting data written by an uncommitted transaction.

```
Timeline:
T0: car_owner = "Alice", car_invoice = "Alice"

T1: Transaction A (Alice selling to Bob):
    A: UPDATE cars SET owner = 'Bob'  (not committed)

T2: Transaction B (Alice selling to Charlie):
    B: UPDATE cars SET owner = 'Charlie'  ❌ (DIRTY WRITE!)
    B: UPDATE invoices SET buyer = 'Charlie'
    B: COMMIT

T3: Transaction A:
    A: UPDATE invoices SET buyer = 'Bob'
    A: COMMIT

Final state:
  car_owner = 'Bob'
  car_invoice = 'Bob'
  
Charlie paid but doesn't own the car! 💸
```

**Solution - Read Committed**:

```
T1: Transaction A: UPDATE cars SET owner = 'Bob'
    → Database LOCKS the row

T2: Transaction B: UPDATE cars SET owner = 'Charlie'
    → BLOCKED (waits for A to commit or rollback)

T3: Transaction A: COMMIT
    → Releases lock

T4: Transaction B: Proceeds with UPDATE
```

**Real-World Implementations**:
- **PostgreSQL**: Default isolation level is Read Committed
- **Oracle**: Read Committed by default
- **SQL Server**: Read Committed by default

**Configuration**:

```sql
-- PostgreSQL
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### Snapshot Isolation (Repeatable Read)

**Problem with Read Committed**: Non-repeatable reads

```
Scenario: Backup process reading database

T0: account1 = $1000, account2 = $1000

T1: Backup: Read account1 = $1000 ✅

T2: Transaction: Transfer $500 from account1 to account2
    account1 = $500
    account2 = $1500
    COMMIT

T3: Backup: Read account2 = $1500 ✅

Backup sees: account1 = $1000, account2 = $1500
Total = $2500 ❌
Expected = $2000

Money appeared out of nowhere! 💰
```

**Solution: Snapshot Isolation**

Each transaction sees a **consistent snapshot** of the database from the start of the transaction.

```
T0: account1 = $1000, account2 = $1000
    SNAPSHOT created: version 1

T1: Backup transaction starts
    → Uses SNAPSHOT at version 1

T2: Transfer transaction:
    account1 = $500 (version 2)
    account2 = $1500 (version 2)
    COMMIT

T3: Backup reads:
    account1 → Reads version 1: $1000 ✅
    account2 → Reads version 1: $1000 ✅
    Total = $2000 ✅

Backup sees consistent snapshot from T0!
```

**How It Works: Multi-Version Concurrency Control (MVCC)**

Database keeps multiple versions of each object.

```
┌────────────────────────────────────────────────┐
│      MVCC EXAMPLE                              │
├────────────────────────────────────────────────┤
│  accounts table:                               │
│  ┌──────┬─────────┬─────────┬────────────┐    │
│  │  id  │ balance │ version │ created_by │    │
│  ├──────┼─────────┼─────────┼────────────┤    │
│  │  1   │  1000   │    1    │   tx_10    │    │
│  │  1   │   500   │    2    │   tx_20    │    │
│  │  2   │  1000   │    1    │   tx_10    │    │
│  │  2   │  1500   │    2    │   tx_20    │    │
│  └──────┴─────────┴─────────┴────────────┘    │
│                                                │
│  Transaction reading at version 1:             │
│    → Sees: account1=1000, account2=1000        │
│                                                │
│  Transaction reading at version 2:             │
│    → Sees: account1=500, account2=1500         │
└────────────────────────────────────────────────┘
```

**Real-World Implementations**:
- **PostgreSQL**: Calls it "Repeatable Read" (actually snapshot isolation)
- **MySQL InnoDB**: Calls it "Repeatable Read"
- **Oracle**: Snapshot isolation available

**Usage**:

```sql
-- PostgreSQL
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  -- All reads see consistent snapshot from now
  SELECT * FROM accounts;  -- Snapshot captured
  -- ... long-running analytics ...
  SELECT * FROM accounts;  -- Same snapshot!
COMMIT;
```

### Lost Updates Problem

Even with snapshot isolation, we have issues with **concurrent updates**.

**Scenario: Two users incrementing a counter**

```
T0: counter = 100

T1: User A reads: counter = 100
T2: User B reads: counter = 100

T3: User A: counter = 100 + 1 = 101
T4: User B: counter = 100 + 1 = 101

T5: User A writes: counter = 101
T6: User B writes: counter = 101

Final: counter = 101 ❌
Expected: 102

One increment was lost! 💥
```

**Solution 1: Atomic Operations**

```sql
-- ❌ Bad: Read-modify-write (loses updates)
counter = SELECT count FROM stats WHERE id = 1;
counter += 1;
UPDATE stats SET count = counter WHERE id = 1;

-- ✅ Good: Atomic operation
UPDATE stats SET count = count + 1 WHERE id = 1;
```

Database handles increment atomically. No lost updates!

**Solution 2: Explicit Locking**

```sql
-- PostgreSQL
BEGIN;
  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- LOCK!
  -- Do calculations...
  UPDATE accounts SET balance = ... WHERE id = 1;
COMMIT;  -- UNLOCK
```

Other transactions wait until lock is released.

**Solution 3: Compare-and-Set**

```sql
-- Only update if value hasn't changed
UPDATE accounts 
SET balance = 900 
WHERE id = 1 AND balance = 1000;

-- If balance changed, UPDATE affects 0 rows
-- Application detects and retries
```

**Solution 4: Automatic Detection**

Some databases (PostgreSQL with REPEATABLE READ) automatically detect lost updates.

```sql
-- PostgreSQL
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SELECT balance FROM accounts WHERE id = 1;  -- balance = 1000
  -- Another transaction modifies this row
  UPDATE accounts SET balance = 900 WHERE id = 1;
  -- ERROR: could not serialize access due to concurrent update
ROLLBACK;
```

Database aborts transaction if concurrent update detected.

**Real-World Example - MongoDB**:

```javascript
// ❌ Lost update risk
const doc = db.users.findOne({_id: 123});
doc.views += 1;
db.users.update({_id: 123}, doc);

// ✅ Atomic increment
db.users.update(
  {_id: 123},
  {$inc: {views: 1}}
);
```

### Write Skew and Phantoms

Even with snapshot isolation, subtle anomalies exist.

**Write Skew Example - On-Call Doctors**

```
Constraint: At least one doctor must be on call at all times

Current state:
  alice: on_call = true
  bob: on_call = true
  
  Both are on call ✅ Constraint satisfied

T1: Alice's transaction:
  SELECT COUNT(*) FROM doctors WHERE on_call = true;
  -- Result: 2 doctors on call
  -- "OK, I can go off call"
  UPDATE doctors SET on_call = false WHERE name = 'Alice';

T2: Bob's transaction (simultaneous):
  SELECT COUNT(*) FROM doctors WHERE on_call = true;
  -- Result: 2 doctors on call
  -- "OK, I can go off call"
  UPDATE doctors SET on_call = false WHERE name = 'Bob';

T3: Both transactions commit

Final state:
  alice: on_call = false
  bob: on_call = false
  
  Zero doctors on call! ❌ Constraint violated!
```

**Why Snapshot Isolation Doesn't Prevent This**:

Each transaction sees a consistent snapshot (2 doctors on call). The constraint is checked within each snapshot. But when both commit, the constraint is violated globally.

**This is called write skew** - a generalization of lost updates.

**Real-World Examples of Write Skew**:

1. **Meeting Room Booking**
```sql
-- Check room is available 2-3pm
SELECT COUNT(*) FROM bookings 
WHERE room_id = 5 AND time_range OVERLAPS '2:00-3:00';
-- Returns 0

-- Book it
INSERT INTO bookings VALUES (5, '2:00-3:00', 'Alice');

-- If two people do this simultaneously:
-- Both see room available
-- Both book it
-- Double-booked room! ❌
```

2. **Username Uniqueness**
```sql
-- Check username available
SELECT COUNT(*) FROM users WHERE username = 'alice123';
-- Returns 0

-- Create user
INSERT INTO users VALUES ('alice123', ...);

-- Two people can both see username available
-- Both insert
-- Duplicate username! ❌
```

**Phantom Reads**

A special case where a transaction reads a set of rows matching a condition, but another transaction adds/removes rows that match that condition.

```
T1: SELECT COUNT(*) FROM doctors WHERE on_call = true;
    -- Returns: 2

T2: Another transaction:
    UPDATE doctors SET on_call = false WHERE name = 'Charlie';
    COMMIT;

T3: SELECT COUNT(*) FROM doctors WHERE on_call = true;
    -- Returns: 1 ❌ (if not using snapshot isolation)
    
    The row "appeared" (phantom!)
```

**Solutions for Write Skew**:

1. **Explicit Locks (Materializing Conflicts)**

```sql
-- Create a lock table
CREATE TABLE on_call_lock (
  lock_id INT PRIMARY KEY DEFAULT 1,
  dummy INT
);
INSERT INTO on_call_lock VALUES (1, 0);

-- Transaction:
BEGIN;
  SELECT * FROM on_call_lock WHERE lock_id = 1 FOR UPDATE;  -- LOCK!
  
  SELECT COUNT(*) FROM doctors WHERE on_call = true;
  -- Safe to check now
  
  UPDATE doctors SET on_call = false WHERE name = 'Alice';
COMMIT;  -- UNLOCK
```

2. **Use Serializable Isolation** (coming next!)

## Part 3: Serializability

**Serializability** is the gold standard - strongest isolation level.

**Definition**: Transactions execute as if they ran serially (one at a time), even though they run concurrently.

```
┌────────────────────────────────────────────────┐
│      SERIALIZABILITY                           │
├────────────────────────────────────────────────┤
│  Concurrent execution:                         │
│  [TxA] ─────────────                           │
│  [TxB] ──────────────────                      │
│  [TxC] ───────────                             │
│                                                │
│  Equivalent to ONE of these serial orders:     │
│  A → B → C                                     │
│  A → C → B                                     │
│  B → A → C                                     │
│  B → C → A                                     │
│  C → A → B                                     │
│  C → B → A                                     │
│                                                │
│  Result is the same as if transactions ran     │
│  one at a time in SOME order                   │
└────────────────────────────────────────────────┘
```

**Three Ways to Achieve Serializability**:

### 1. Actual Serial Execution

**Idea**: Just run transactions one at a time! No concurrency = no concurrency problems!

```python
# Single-threaded transaction executor
transaction_queue = Queue()

while True:
    transaction = transaction_queue.get()
    execute_transaction(transaction)  # Run completely
    # Next transaction
```

**When This Works**:

✅ **Transactions are fast** (milliseconds, not seconds)
✅ **Active dataset fits in memory** (no slow disk I/O)
✅ **Throughput requirement is modest** (one CPU core worth)

**Real-World: Redis**

Redis is single-threaded and executes commands serially.

```python
import redis
r = redis.Redis()

# Lua script executed atomically
script = """
local current = redis.call('GET', KEYS[1])
if current then
  return redis.call('INCRBY', KEYS[1], ARGV[1])
else
  return redis.call('SET', KEYS[1], ARGV[1])
end
"""

# Runs atomically
r.eval(script, 1, 'counter', 10)
```

**Real-World: VoltDB**

VoltDB partitions data and runs transactions serially within each partition.

```
┌────────────────────────────────────────────┐
│  Partition 1        Partition 2            │
│  [Queue]            [Queue]                │
│   ↓                  ↓                     │
│  Serial exec        Serial exec            │
│  (one at a time)    (one at a time)        │
└────────────────────────────────────────────┘

Transactions within partition: Serial
Transactions across partitions: Can run concurrently
```

**Limitations**:
- ❌ Doesn't scale to multiple cores easily
- ❌ Slow transactions block everything
- ❌ Not suitable for interactive transactions

### 2. Two-Phase Locking (2PL)

**Idea**: Use locks to prevent concurrent access.

**Rules**:
1. **Before reading** object, acquire **shared lock** (read lock)
2. **Before writing** object, acquire **exclusive lock** (write lock)
3. **Multiple readers** can hold shared lock simultaneously
4. **Exclusive lock** blocks everyone else
5. **Hold all locks until transaction ends** (two phases: acquiring, releasing)

```
┌────────────────────────────────────────────────┐
│      TWO-PHASE LOCKING                         │
├────────────────────────────────────────────────┤
│                                                │
│  Phase 1: Acquiring locks (growing phase)      │
│  ─────────────────────────────                 │
│  Read A  → Acquire shared lock on A            │
│  Write B → Acquire exclusive lock on B         │
│  Read C  → Acquire shared lock on C            │
│                                                │
│  Phase 2: Releasing locks (shrinking phase)    │
│  ─────────────────────────────────             │
│  COMMIT  → Release ALL locks                   │
│                                                │
│  Can't acquire locks after releasing any!      │
└────────────────────────────────────────────────┘
```

**Lock Compatibility Table**:

```
┌─────────────────────────────────────┐
│     Lock Compatibility              │
├──────────┬──────────┬───────────────┤
│          │  Shared  │  Exclusive    │
├──────────┼──────────┼───────────────┤
│ Shared   │    ✅    │      ❌       │
│ Exclusive│    ❌    │      ❌       │
└──────────┴──────────┴───────────────┘
```

**Example - Preventing Write Skew**:

```sql
-- Alice's transaction
BEGIN;
  SELECT COUNT(*) FROM doctors WHERE on_call = true FOR SHARE;
  -- Acquires SHARED lock on all rows
  
  -- Bob tries to run:
  UPDATE doctors SET on_call = false WHERE name = 'Bob';
  -- Needs EXCLUSIVE lock
  -- BLOCKED! (waits for Alice)
  
  UPDATE doctors SET on_call = false WHERE name = 'Alice';
COMMIT;  -- Releases locks

-- Now Bob's transaction can proceed
-- Will see updated state
```

**Predicate Locks**

Lock not just existing rows, but all rows matching a condition (including future rows!).

```sql
-- Lock prevents phantoms
BEGIN;
  SELECT * FROM bookings 
  WHERE room_id = 5 AND time_range OVERLAPS '2:00-3:00'
  FOR UPDATE;
  
  -- This creates a PREDICATE LOCK
  -- Blocks any transaction trying to:
  -- - Insert booking for room 5, 2-3pm
  -- - Update booking to be room 5, 2-3pm
  -- - Delete booking for room 5, 2-3pm
```

**Index-Range Locks**

Predicate locks are expensive to check. Most databases use **index-range locks** instead.

```sql
-- Instead of locking "room 5, 2-3pm"
-- Lock range: room 5, time 2:00 to 3:00

-- Implemented as locks on index entries
```

**Performance Problems with 2PL**:

❌ **Slow**: Lots of waiting for locks
❌ **Deadlocks**: Transactions wait for each other in a cycle

```
Deadlock Example:
T1: Lock A, wait for B
T2: Lock B, wait for A

Both stuck forever! 💀

Solution: Deadlock detection
- Database detects cycle
- Aborts one transaction (victim)
- Other proceeds
```

**Real-World Usage**:
- **MySQL** (InnoDB): Uses 2PL with index-range locks for SERIALIZABLE
- **SQL Server**: Uses 2PL
- **DB2**: Uses 2PL

### 3. Serializable Snapshot Isolation (SSI)

**Idea**: Optimistic concurrency control - let transactions run without locks, detect conflicts at commit time.

```
┌────────────────────────────────────────────────┐
│  SERIALIZABLE SNAPSHOT ISOLATION (SSI)         │
├────────────────────────────────────────────────┤
│                                                │
│  Optimistic: Assume no conflicts               │
│                                                │
│  1. Transactions run without blocking          │
│  2. Track reads and writes                     │
│  3. At commit: Check for conflicts             │
│  4. Abort if serialization violation detected  │
└────────────────────────────────────────────────┘
```

**How It Works**:

Detect two types of conflicts:

**Conflict Type 1: Stale Read** (Reading outdated snapshot)

```
T1: Read X = 100 (snapshot)
T2: Write X = 200, COMMIT
T1: Write based on X = 100 (stale!)
T1: ABORT ❌ (detected stale read)
```

**Conflict Type 2: Write Skew** (Phantom reads)

```
T1: Read: SELECT COUNT(*) WHERE on_call = true → 2
T2: Read: SELECT COUNT(*) WHERE on_call = true → 2
T1: Write: UPDATE ... SET on_call = false (based on count = 2)
T2: Write: UPDATE ... SET on_call = false (based on count = 2)

At commit:
Database detects both transactions made decisions based on same snapshot
but modified data that affects each other's read predicates
→ ABORT one transaction ❌
```

**Implementation**:

Database tracks:
1. **Which transactions read which objects**
2. **Which transactions wrote which objects**
3. **Read predicates** (conditions in WHERE clauses)

```python
# Simplified tracking
read_log = {
  tx1: ["doctors WHERE on_call = true"],
  tx2: ["doctors WHERE on_call = true"]
}

write_log = {
  tx1: ["doctors.alice.on_call = false"],
  tx2: ["doctors.bob.on_call = false"]
}

# At commit:
def check_serializable(tx):
  for other_tx in committed_transactions:
    if writes_affect_reads(other_tx, tx):
      # Conflict detected!
      return ABORT
  return COMMIT
```

**Advantages over 2PL**:

✅ **Better performance**: No waiting for locks
✅ **Read-only queries never block**: Always use snapshot
✅ **Less deadlock**: Only abort at commit time

**Disadvantages**:

❌ **More aborts**: Optimistic = sometimes wrong
❌ **Retry logic needed**: Application must retry aborted transactions

**Real-World Usage**:
- **PostgreSQL** 9.1+: SERIALIZABLE isolation uses SSI
- **FoundationDB**: Uses SSI

**Example in PostgreSQL**:

```sql
-- PostgreSQL
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
  SELECT COUNT(*) FROM doctors WHERE on_call = true;
  -- Returns 2
  
  -- Another transaction commits here, changing count
  
  UPDATE doctors SET on_call = false WHERE name = 'Alice';
COMMIT;
-- ERROR: could not serialize access due to read/write dependencies
-- Transaction aborted, must retry
```

## Part 4: Real-World Transaction Examples

### Example 1: Bank Transfer (Classic)

```python
def transfer(from_account, to_account, amount):
    try:
        db.begin(isolation_level='SERIALIZABLE')
        
        # Check balance
        balance = db.query(
            "SELECT balance FROM accounts WHERE id = ? FOR UPDATE",
            from_account
        )[0]
        
        if balance < amount:
            raise InsufficientFunds()
        
        # Deduct from source
        db.execute(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            amount, from_account
        )
        
        # Add to destination
        db.execute(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            amount, to_account
        )
        
        # Record transaction
        db.execute(
            "INSERT INTO transfers (from, to, amount) VALUES (?, ?, ?)",
            from_account, to_account, amount
        )
        
        db.commit()
        return "Success"
        
    except Exception as e:
        db.rollback()
        return f"Failed: {e}"
```

### Example 2: Ticket Booking System

```python
def book_ticket(user_id, event_id, num_tickets):
    try:
        db.begin(isolation_level='REPEATABLE READ')
        
        # Lock event row
        event = db.query(
            "SELECT available_tickets FROM events WHERE id = ? FOR UPDATE",
            event_id
        )[0]
        
        if event['available_tickets'] < num_tickets:
            raise SoldOut()
        
        # Update available count
        db.execute(
            "UPDATE events SET available_tickets = available_tickets - ? WHERE id = ?",
            num_tickets, event_id
        )
        
        # Create booking
        db.execute(
            "INSERT INTO bookings (user_id, event_id, num_tickets) VALUES (?, ?, ?)",
            user_id, event_id, num_tickets
        )
        
        # Charge user
        payment_id = charge_payment(user_id, event['price'] * num_tickets)
        
        db.execute(
            "UPDATE bookings SET payment_id = ? WHERE id = LAST_INSERT_ID()",
            payment_id
        )
        
        db.commit()
        return "Booked successfully"
        
    except Exception as e:
        db.rollback()
        return f"Booking failed: {e}"
```

### Example 3: Distributed Transaction (2PC)

When transaction spans multiple databases:

```python
def transfer_across_banks(from_bank_db, to_bank_db, amount):
    transaction_coordinator = TransactionCoordinator()
    
    try:
        # Phase 1: PREPARE
        # Ask each database if it can commit
        from_bank_db.prepare("UPDATE accounts SET balance = balance - ?", amount)
        to_bank_db.prepare("UPDATE accounts SET balance = balance + ?", amount)
        
        # Both said YES
        # Phase 2: COMMIT
        from_bank_db.commit()
        to_bank_db.commit()
        
        return "Success"
        
    except Exception as e:
        # If ANY database says NO or fails
        # ABORT on all
        from_bank_db.rollback()
        to_bank_db.rollback()
        
        return f"Failed: {e}"
```

## Summary

**Key Takeaways**:

1. **ACID Properties**
   - **Atomicity**: All or nothing
   - **Consistency**: Valid state to valid state
   - **Isolation**: Concurrent transactions don't interfere
   - **Durability**: Data survives crashes

2. **Isolation Levels** (weakest to strongest)
   - **Read Uncommitted**: No guarantees (almost never used)
   - **Read Committed**: No dirty reads/writes (most common default)
   - **Snapshot Isolation**: Repeatable read, no phantoms in practice
   - **Serializable**: Full isolation (expensive but correct)

3. **Serializability Approaches**
   - **Serial execution**: Simple, limited throughput
   - **2PL**: Pessimistic locking, slow but works
   - **SSI**: Optimistic, better performance, may abort

4. **Real-World Wisdom**
   - Most apps use Read Committed (good enough, performant)
   - Use Serializable for critical operations (money, inventory)
   - Atomic operations (UPDATE counter = counter + 1) prevent many issues
   - Test with concurrent load (race conditions only appear under load)
   - Monitor transaction abort rates (high = consider weaker isolation)

5. **Trade-offs**
   ```
   Stronger Isolation → More Correctness → Lower Performance
   Weaker Isolation → Less Correctness → Higher Performance
   ```

**Next Chapter**: The Trouble with Distributed Systems - what happens when transactions span multiple machines and networks can fail!
