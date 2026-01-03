# Chapter 7: Transactions

## Introduction: What Could Go Wrong?

Imagine you're building a banking application. A user transfers $100 from their checking account to their savings account. Simple, right?

```javascript
// Transfer $100
let checkingBalance = getBalance("checking");
let savingsBalance = getBalance("savings");

checkingBalance -= 100;  // Deduct from checking
savingsBalance += 100;    // Add to savings

saveBalance("checking", checkingBalance);
saveBalance("savings", savingsBalance);
```

**What could go wrong?**

1. **Power outage** after deducting from checking but before adding to savings → Money disappears
2. **Two transfers** happening simultaneously → Race condition, wrong balances ‍♂️
3. **System crash** in the middle → Partial update, inconsistent state 
4. **Network failure** → One update succeeds, other fails 

**Solution**: **Transactions** - a way to group multiple operations so they succeed or fail together as a single unit.

```
┌────────────────────────────────────────────────┐
│      WITHOUT TRANSACTIONS                      │
├────────────────────────────────────────────────┤
│  Step 1: checking -= 100                     │
│  Step 2: CRASH!                              │
│  Step 3: savings += 100   Never happens      │
│                                                │
│  Result: Money lost!                         │
└────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│      WITH TRANSACTIONS                         │
├────────────────────────────────────────────────┤
│  BEGIN TRANSACTION                             │
│    Step 1: checking -= 100                     │
│    Step 2: CRASH!                            │
│  ROLLBACK (undo everything)                    │
│                                                │
│  Result: No change, money safe!              │
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

**Real-World Analogy**: Atomic bomb - it either explodes completely or not at all. No half-explosion
**The Core Problem**: Atomicity addresses the fundamental challenge that operations aren't truly atomic at the hardware level. Even a simple "insert 10 rows" query actually involves many steps:

1. Parse the query
2. Validate each row
3. Insert row 1 into table
4. Update indexes for row 1
5. Insert row 2 into table
6. Update indexes for row 2
7. ...and so on

**Without atomicity**, if something fails at step 6:
- Half your rows are in the table
- Half your rows never made it
- Your indexes are partially updated
- Your database is in an **inconsistent state**
- You have no idea which rows succeeded
**Example 1: Simple Insert Gone Wrong**

```sql
-- Try to insert 10 rows
INSERT INTO users (username, email) VALUES
  ('alice', 'alice@example.com'),
  ('bob', 'bob@example.com'),
  ('charlie', 'charlie@example.com'),
  ('dave', 'dave@example.com'),
  ('eve', 'eve@example.com'),
  ('frank', 'frank@example.com'),  -- Duplicate! Email already exists
  ('grace', 'grace@example.com'),
  ('henry', 'henry@example.com'),
  ('iris', 'iris@example.com'),
  ('jack', 'jack@example.com');
```

**Without Atomicity**:
```
alice, bob, charlie, dave, eve inserted 
frank fails due to unique constraint violation 
grace, henry, iris, jack never inserted 

Result: Database has 5 rows inserted, 5 missing
You asked for all-or-nothing, got half-and-half
```

**With Atomicity**:
```
Database detects constraint violation on frank 
Automatically ROLLBACK entire transaction
All 10 rows undone → Database unchanged 
Error message: "Duplicate email constraint violation"
```

**Example 2: E-commerce Order (Real-World Complexity)**:

```javascript
async function placeOrder(userId, productId, quantity) {
    try {
        // BEGIN TRANSACTION
        await db.beginTransaction();
        
        // Step 1: Check and update inventory
        const inventory = await db.query("SELECT quantity FROM inventory WHERE product_id = ?", [productId]);
        if (inventory < quantity) {
            throw new Error("Out of stock");
        }
        await db.execute("UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?", [quantity, productId]);
        
        // Step 2: Charge payment
        await db.execute("INSERT INTO charges (user_id, amount) VALUES (?, ?)", [userId, price * quantity]);
        
        // Step 3: Create order
        await db.execute("INSERT INTO orders (user_id, product_id, quantity) VALUES (?, ?, ?)", 
                   [userId, productId, quantity]);
        
        // Step 4: Update user's order history
        await db.execute("UPDATE user_stats SET total_orders = total_orders + 1 WHERE user_id = ?", [userId]);
        
        // Step 5: Add loyalty points
        const points = calculatePoints(price * quantity);
        await db.execute("UPDATE user_points SET points = points + ? WHERE user_id = ?", [points, userId]);
        
        // If we got here, all steps succeeded
        await db.commit();  // Make changes permanent
        return "Order placed successfully";
        
    } catch (e) {
        // Something went wrong at ANY step
        await db.rollback();  // Undo ALL changes (even successful ones)
        return `Order failed: ${e.message}`;
    }
}
```

**What Atomicity Prevents**:
```
Scenario 1: Payment service is down
 Inventory decremented (Step 1) 
 Payment fails (Step 2) 
 Order never created (Step 3) 
Result: Free products! Customer gets item without paying
Scenario 2: Database crash during order
 Inventory decremented (Step 1) 
 Payment charged (Step 2) 
 CRASH HAPPENS 
 Order never created (Step 3) 
Result: Customer charged but no order exists! 😡

Scenario 3: Network interruption
 Inventory decremented (Step 1) 
 Payment charged (Step 2) 
 Order created (Step 3) 
 Network dies (Step 4) 
 Stats never updated, points never awarded
Result: Inconsistent state - order exists but stats are wrong
 With atomicity: Either ALL 5 steps succeed, or ALL are undone
If ANY step fails, it's as if nothing ever happened.
```

**How It Works: The Write-Ahead Log (WAL)**

Databases use a **write-ahead log** to achieve atomicity. This is one of the most elegant solutions in computer science:

```
┌────────────────────────────────────────────────┐
│      WRITE-AHEAD LOG (SEQUENTIAL WRITES)       │
├────────────────────────────────────────────────┤
│  Transaction ID: 12345                         │
│  BEGIN                                         │
│  [12345] UPDATE inventory SET qty = 9          │
│  [12345] INSERT INTO charges VALUES (...)      │
│  [12345] INSERT INTO orders VALUES (...)       │
│  [12345] UPDATE user_stats...                  │
│  [12345] UPDATE user_points...                 │
│  COMMIT ← This is the atomic moment!           │
└────────────────────────────────────────────────┘

  ↓ Only after COMMIT, changes applied to actual tables

┌────────────────────────────────────────────────┐
│      ACTUAL DATABASE TABLES                    │
├────────────────────────────────────────────────┤
│  inventory table (updated)                     │
│  charges table (new row)                       │
│  orders table (new row)                        │
│  user_stats table (updated)                    │
│  user_points table (updated)                   │
└────────────────────────────────────────────────┘
```

**The Recovery Process**:

```
Time T0: Transaction 12345 begins
         All changes written to WAL

Time T1: CRASH!  (before COMMIT written)

Time T2: Database restarts
         Reads WAL from disk
         
Recovery Logic:
  for each transaction in WAL:
      if transaction has COMMIT marker:
          replay all changes 
      else:
          transaction incomplete, undo changes 
          
  Transaction 12345: No COMMIT found
  → Undo all changes from 12345
  → Database state = before transaction started
  → Atomicity preserved
```

**Key Insight**: The WAL is **append-only** and written **sequentially** to disk, which is:
1. **Fast** - sequential writes are 100x faster than random writes
2. **Durable** - fsync() ensures data on physical disk
3. **Recoverable** - can replay or undo any transaction

**Real-World Example: PostgreSQL's WAL**

PostgreSQL's WAL files look like this:

```bash
$ ls -lh /var/lib/postgresql/data/pg_wal/
-rw------- 1 postgres postgres 16M Dec  1 10:23 000000010000000000000001
-rw------- 1 postgres postgres 16M Dec  1 10:45 000000010000000000000002
-rw------- 1 postgres postgres 8.2M Dec  1 10:47 000000010000000000000003
```

Each 16MB segment contains thousands of transactions. When PostgreSQL crashes:

```javascript
// Simplified recovery process
function recoverFromCrash() {
    const walFiles = loadWalFilesFromDisk();
    
    for (const walFile of walFiles) {
        for (const transaction of walFile) {
            if (transaction.hasCommit()) {
                // Redo: apply changes to ensure durability
                replayTransaction(transaction);
            } else {
                // Undo: rollback partial transactions
                undoTransaction(transaction);
            }
        }
    }
    
    return "Recovery complete, database consistent!";
}
```

**Atomicity Violations in the Real World**

**Case Study: MongoDB 2.x (Before 2016)**

MongoDB versions before 2.6 didn't have multi-document transactions. This led to real problems:

```javascript
// Try to transfer money between accounts
db.accounts.update({_id: "alice"}, {$inc: {balance: -100}})  // Succeeds 
// CRASH HERE
db.accounts.update({_id: "bob"}, {$inc: {balance: +100}})    // Never happens 

// Result: Alice lost $100, Bob gained nothing
// Money disappeared from the system
```

Engineers had to implement atomicity manually:

```javascript
// Manual two-phase commit (complex!)
function transferMoney(from, to, amount) {
    const txId = generateId();
    
    // Phase 1: Prepare (mark transaction as pending)
    db.transactions.insert({
        _id: txId,
        from: from,
        to: to,
        amount: amount,
        state: "pending"
    });
    
    // Phase 2: Apply changes
    db.accounts.update({_id: from}, {$inc: {balance: -amount}});
    db.accounts.update({_id: to}, {$inc: {balance: +amount}});
    
    // Phase 3: Mark complete
    db.transactions.update({_id: txId}, {$set: {state: "complete"}});
    
    // Recovery process needed to handle crashes between phases
}
```

**Modern MongoDB (4.0+)** now supports ACID transactions, making this much simpler.

**Atomicity Trade-offs**

```
┌──────────────────────────────────────────────┐
│  ATOMICITY MECHANISMS                        │
├──────────────┬───────────────┬───────────────┤
│   Approach   │  Performance  │  Complexity   │
├──────────────┼───────────────┼───────────────┤
│ WAL Only     │  Fast ⚡       │  Low          │
│ Shadow Pages │  Medium       │  Medium       │
│ MVCC         │  Fast ⚡⚡      │  High         │
│ 2PC (distrib)│  Slow        │  Very High    │
└──────────────┴───────────────┴───────────────┘
```

**The Bottom Line**: Atomicity means you never have to worry about partial failures. Your application code can treat complex multi-step operations as a single "all-or-nothing" unit. This is **fundamental** to building reliable systems.

### Consistency: Valid State to Valid State

**Definition**: Database moves from one valid state to another valid state. Invariants are maintained.

**Real-World Analogy**: Conservation of energy in physics - energy transforms but total amount stays constant.

**The Controversial Truth About Consistency**

Here's something that database textbooks often gloss over: **Consistency is NOT solely the database's job**. It's actually **shared responsibility** between:

1. **The Database** (enforces constraints)
2. **The Application** (enforces business logic)
3. **The Developer** (designs correct invariants)

The author Martin Kleppmann and many database experts actually consider the "C" in ACID somewhat misleading. Let's understand why.

**Example 1: Double-Entry Bookkeeping (Classic Invariant)**

In accounting, there's a fundamental rule: **Total Debits = Total Credits** (balanced books)

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

-- Result: total_debits = total_credits 
-- Books are balanced
```

**Question**: Who enforced this invariant?

**Answer**: **The Application!** The database has no idea about accounting rules. It just stored two rows. The application developer knew to insert both a debit and a credit.

**Example 2: Who's Responsible for What?**

```sql
# SCENARIO 1: Database can enforce
# Constraint: Email must be unique
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100) UNIQUE  -- Database enforces this
);

# Try to insert duplicate:
INSERT INTO users (username, email) VALUES ('alice', 'alice@example.com');  #  Succeeds
INSERT INTO users (username, email) VALUES ('alice2', 'alice@example.com'); #  Fails
# Error: duplicate key value violates unique constraint "users_email_key"

#  Database enforced consistency
```

```javascript
// SCENARIO 2: Application must enforce
// Business Rule: User can't have negative balance
// Database can't know this is wrong
async function withdrawMoney(userId, amount) {
    // Application checks business rule
    const balance = await db.query("SELECT balance FROM accounts WHERE user_id = ?", [userId]);
    
    if (balance < amount) {
        // Application enforces consistency
        throw new Error("Insufficient funds");
    }
    
    await db.execute("UPDATE accounts SET balance = balance - ? WHERE user_id = ?", [amount, userId]);
}

// What if we skip the check?
await db.execute("UPDATE accounts SET balance = -500 WHERE user_id = 1");  //  Database allows it
// Database stored bad data because application didn't enforce the rule
```

**Database vs Application Responsibilities**

```
┌────────────────────────────────────────────────┐
│  CONSISTENCY ENFORCEMENT                       │
├──────────────────┬─────────────────────────────┤
│   DATABASE       │   APPLICATION               │
├──────────────────┼─────────────────────────────┤
│ NOT NULL         │ Balance >= 0                │
│ UNIQUE           │ Age >= 18 for drivers       │
│ PRIMARY KEY      │ Total debits = credits      │
│ FOREIGN KEY      │ Appointment times don't     │
│ CHECK constraints│   overlap                   │
│ Data types       │ Inventory reflects reality  │
└──────────────────┴─────────────────────────────┘
```

**Example 3: Database Can Help with CHECK Constraints**

```sql
-- Database can enforce SOME business rules
CREATE TABLE accounts (
    user_id INT PRIMARY KEY,
    balance DECIMAL(10,2) CHECK (balance >= 0),  -- No negative balances
    status VARCHAR(20) CHECK (status IN ('active', 'suspended', 'closed'))
);

-- Try to violate constraint:
UPDATE accounts SET balance = -100 WHERE user_id = 1;
--  Error: new row violates check constraint "accounts_balance_check"

--  Database enforced consistency
```

**But this has limitations...**

```sql
-- Database CAN'T enforce complex business logic:

-- Rule: Can't transfer more than $10,000 per day
-- Database has no concept of "per day" across multiple transactions
-- Rule: Appointment times can't overlap
-- Would require checking ALL other rows dynamically

-- Rule: If order is "shipped", inventory must be decremented
-- Involves relationships across multiple tables and states
```

**Example 4: Consistency Failures in the Real World**

**Case Study: The Inconsistent E-commerce System**

```javascript
// BAD CODE: No consistency checks
async function processOrder(productId, quantity) {
    // Just blindly subtract inventory
    await db.execute("UPDATE inventory SET stock = stock - ? WHERE product_id = ?", 
               [quantity, productId]);
    
    // Oops, what if stock goes negative?
    // Database stored: stock = -50
    // Real world: You can't have negative physical items! 📦
}
```

**What happens**:
```
Initial state: product_id=5 has stock=10

Customer A orders 15 units
  → stock = 10 - 15 = -5 (stored in database!)
  
Customer B orders 20 more units  
  → stock = -5 - 20 = -25
  
Result: Database says you have -25 items in stock
But physically impossible
Consistency violated because application didn't check invariant
```

**GOOD CODE: Application enforces consistency**

```javascript
async function processOrder(productId, quantity) {
    // Application checks invariant
    const currentStock = await db.query("SELECT stock FROM inventory WHERE product_id = ?", [productId]);
    
    if (currentStock < quantity) {
        throw new Error("Insufficient inventory");  // Maintain invariant
    }
    
    // Only proceed if invariant will be maintained
    await db.execute("UPDATE inventory SET stock = stock - ? WHERE product_id = ?", 
               [quantity, productId]);
}
```

**Example 5: Foreign Keys (Database-Enforced Consistency)**

```sql
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id),  -- Foreign key
    product_id INT REFERENCES products(product_id),
    quantity INT
);

-- Try to create order for non-existent user:
INSERT INTO orders (user_id, product_id, quantity) VALUES (99999, 1, 5);
--  Error: insert or update violates foreign key constraint "orders_user_id_fkey"
-- Key (user_id)=(99999) is not present in table "users"

--  Database enforces referential integrity
```

**Example 6: Application-Level Invariants**

```javascript
// Complex business rule: Appointment scheduling
async function scheduleAppointment(doctorId, startTime, endTime, patientId) {
    // Check invariant: No overlapping appointments
    const overlapping = await db.query(`
        SELECT * FROM appointments 
        WHERE doctor_id = ?
        AND (
            (start_time < ? AND end_time > ?) OR  -- Overlaps start
            (start_time < ? AND end_time > ?) OR  -- Overlaps end
            (start_time >= ? AND end_time <= ?)   -- Completely inside
        )
    `, [doctorId, endTime, startTime, startTime, endTime, startTime, endTime]);
    
    if (overlapping.length > 0) {
        throw new Error("Doctor already has appointment at this time!");
    }
    
    // Invariant maintained
    await db.execute(`
        INSERT INTO appointments (doctor_id, patient_id, start_time, end_time)
        VALUES (?, ?, ?, ?)
    `, [doctorId, patientId, startTime, endTime]);
}
```

**The Database can't enforce this automatically!** It requires:
1. Application logic to check all existing appointments
2. Understanding of time overlap semantics
3. Transaction isolation to prevent race conditions

**Why "Consistency" in ACID is Controversial**

> "Consistency isn't only the job of the relational database. It's equally the job of the engineer who's using the database. If you give it bad data, there's bad data there."

**The Truth**:
- **Atomicity** - Database's job 
- **Isolation** - Database's job 
- **Durability** - Database's job 
- **Consistency** - **Shared responsibility** between database AND application! ⚠️

**What Databases CAN Guarantee**:
```
 Data type constraints (INT, VARCHAR, etc.)
 NOT NULL constraints
 UNIQUE constraints
 PRIMARY KEY constraints
 FOREIGN KEY constraints
 CHECK constraints (simple rules)
```

**What Applications MUST Guarantee**:
```
 Complex business logic
 Multi-table invariants
 Temporal constraints (per day limits, etc.)
 Semantic meaning (balance must be >= 0)
 Real-world validity (can't ship negative items)
```

**Best Practice: Defense in Depth**

```javascript
// Layer 1: Database constraints
// CREATE TABLE accounts (
//     user_id INT PRIMARY KEY,
//     balance DECIMAL(10,2) NOT NULL CHECK (balance >= 0),  -- Basic check
//     created_at TIMESTAMP NOT NULL DEFAULT NOW()
// );

// Layer 2: Application validation
async function withdraw(userId, amount) {
    if (amount <= 0) {
        throw new Error("Amount must be positive");
    }
    
    const balance = await getBalance(userId);
    
    if (balance < amount) {
        throw new Error("Insufficient funds");
    }
    
    // Layer 3: Transaction ensures atomicity
    await db.transaction(async (trx) => {
        await trx.execute("UPDATE accounts SET balance = balance - ? WHERE user_id = ?", 
                   [amount, userId]);
        await trx.execute("INSERT INTO transactions (user_id, type, amount) VALUES (?, 'withdrawal', ?)",
                   [userId, amount]);
    });
}
```

**The Bottom Line**: Consistency requires **partnership** between database and application. The database provides tools (constraints, transactions), but the application must use them correctly to maintain meaningful invariants.
  
  -- Credit Account B
  INSERT INTO transactions (account, type, amount) 
  VALUES ('B', 'CREDIT', 100);
COMMIT;

-- Check invariant:
SELECT 
  SUM(CASE WHEN type = 'DEBIT' THEN amount ELSE 0 END) as total_debits,
  SUM(CASE WHEN type = 'CREDIT' THEN amount ELSE 0 END) as total_credits
FROM transactions;

-- Result: total_debits = total_credits 
```

**What Consistency Prevents**:
```
 Account balance becomes negative (violates constraint)
 Order references non-existent product (violates foreign key)
 Duplicate email in users table (violates uniqueness)

 With consistency: All constraints are respected
```

**Important Note**: Consistency is actually the application's responsibility
```javascript
// Database ensures constraints (foreign keys, NOT NULL, UNIQUE)
// Application ensures business logic invariants

async function transferMoney(fromAccount, toAccount, amount) {
    // Application checks business rule
    if (amount > 10000) {
        throw new Error("Transfer limit exceeded");
    }
    
    // Database ensures constraints
    await db.execute("UPDATE accounts SET balance = balance - ? WHERE id = ?", 
               [amount, fromAccount]);
    // If balance goes negative, database raises error (CHECK constraint)
}
```

### Isolation: Concurrent Transactions Don't Interfere

**Definition**: Concurrent transactions execute as if they ran serially (one after another), even though they run simultaneously.

**Real-World Analogy**: Soundproof rooms - multiple meetings happening simultaneously, but each room is isolated from the others.

**Why Isolation Matters**

Modern databases like PostgreSQL and MySQL allow **thousands of concurrent connections**. With connection poolers like PgBouncer, you might have:
- 1,000+ application connections
- 100+ actual database connections
- Tens of transactions executing **simultaneously** on multi-core CPUs

All these transactions are reading and writing to the same tables, sometimes the same rows! Without isolation, chaos would ensue.

**The Core Challenge**: We want each client/transaction to interact with the database **as if it's the only one** using it, without having to manually handle coordination with other transactions.

From the SRT:

> "The idea is we do not want like we want a client, your application server, whatever to be able to interact with the database and treat it as if there's nothing else going on and it has the full ability to read rows and to write rows and to update rows and to delete rows without having to handle the coordination between all the other clients that might also be doing things."

**This is the promise of isolation!**

**The Problem - Without Isolation** (Lost Update):

```
Scenario: Bank account with $1000, two simultaneous deposits

Time: 0s
  Balance: $1000 (in database)

Time: 1s
  Transaction A: Read balance = $1000
  Transaction B: Read balance = $1000  (both read same value!)

Time: 2s
  Transaction A: Calculate new balance = $1000 + $100 = $1100
  Transaction B: Calculate new balance = $1000 + $50 = $1050

Time: 3s
  Transaction A: Write balance = $1100  
  Transaction B: Write balance = $1050  

Final balance: $1050 
Expected: $1150 ($1000 + $100 + $50)

Lost $100!  Transaction A's deposit was completely overwritten
```

**Why This Happens**:

Both transactions used a "read-modify-write" pattern:
1. Read current value
2. Modify it (add money)
3. Write it back

But they both read the **same initial value** ($1000), so Transaction B's write completely overwrites A's changes
**With Isolation (Locking)**:

```
Time: 0s
  Balance: $1000

Time: 1s
  Transaction A: Read balance = $1000, LOCK row
  Transaction B: Tries to read... BLOCKED (waits for A)
                                    ↓
Time: 2s                             ↓ (waiting...)
  Transaction A: Deposit $100       ↓
  Transaction A: Write balance = $1100
  Transaction A: COMMIT, UNLOCK row
                                    ↓
Time: 3s                             ↓
  Transaction B: NOW can read! Gets balance = $1100 
  Transaction B: Withdraw $50 → balance = $1050
  Transaction B: COMMIT

Final balance: $1050 
Expected: $1050 ($1100 - $50) - correct
```

**Different Isolation Levels**

The key insight: Perfect isolation is **expensive**! There's a fundamental trade-off:

```
┌─────────────────────────────────────────────────┐
│  ISOLATION LEVEL SPECTRUM                       │
├──────────────────┬──────────────┬───────────────┤
│  Isolation Level │  Guarantees  │  Performance  │
├──────────────────┼──────────────┼───────────────┤
│ Read Uncommitted │  Weakest ⚠️   │  Fastest ⚡⚡⚡  │
│ Read Committed   │  Basic      │  Fast ⚡⚡     │
│ Repeatable Read  │  Strong    │  Medium ⚡    │
│ Serializable     │  Strongest  │  Slowest   │
└──────────────────┴──────────────┴───────────────┘
```

From the SRT, these are the four standard levels, though different databases implement them differently (and sometimes use the same names but with different semantics!).

**Real-World Database Defaults**:

```
PostgreSQL:  Read Committed (default)
MySQL InnoDB: Repeatable Read (default)
Oracle:      Read Committed (default)
SQL Server:  Read Committed (default)
```

**Configuring Isolation Level**:

```sql
-- PostgreSQL
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- MySQL
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**The Isolation Guarantee We Want**

Imagine you're writing parallel, multi-threaded code. Normally you'd have to think about:
- **Locks** - Which variables need locks?
- **Race conditions** - Can two threads modify the same data?
- **Critical sections** - Which code blocks need synchronization?
- **Deadlocks** - Could threads wait for each other forever?

**Databases take this burden away!** With proper isolation, you can write code as if you're the only transaction running, and the database handles all the concurrency complexities for you.

**We'll explore each isolation level in detail next, starting with Read Committed.**

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

```javascript
function commitTransaction(transactionId) {
    // 1. Write all changes to WAL
    wal.write(transactionId, changes);
    
    // 2. Force WAL to disk (fsync)
    wal.fsync();  // This is the EXPENSIVE part
    // 3. Mark as committed
    wal.write(transactionId, "COMMIT");
    wal.fsync();
    
    // NOW safe to tell user transaction succeeded
    return "SUCCESS";
    
    // 4. Later: Apply changes to actual data files (can be async)
    applyChangesToDataFiles(transactionId);
}
```

**Durability Trade-offs**:

```
┌──────────────────────────────────────────────┐
│  DURABILITY LEVELS                           │
├────────────┬─────────────┬──────────────────┤
│   Mode     │ Performance │   Durability     │
├────────────┼─────────────┼──────────────────┤
│ No fsync   │  Very Fast  │   Data loss    │
│ fsync WAL  │    Fast     │   Single disk  │
│ Replication│   Medium    │   Multi-disk   │
│ Sync Repl. │    Slow     │   Multi-DC   │
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
    B: Sees balance = 500  (DIRTY READ!)
    B: Decides to allow withdrawal

T3: Transaction A: ROLLBACK  (oops, changed mind!)
    Balance returns to $1000

T4: Transaction B commits based on wrong data
```

**Why dirty reads are bad**:
- Transaction B made decisions based on data that was rolled back
- Data that B saw never actually existed in committed state
- Leads to incorrect business logic

**Solution - Read Committed**:

```
T1: Transaction A: balance = 500 (uncommitted)

T2: Transaction B: SELECT balance
    → Database returns OLD value: $1000 
    (ignores uncommitted changes)

T3: Transaction A: COMMIT
    
T4: Transaction B: SELECT balance
    → Database returns NEW value: $500 
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
    B: UPDATE cars SET owner = 'Charlie'   (DIRTY WRITE!)
    B: UPDATE invoices SET buyer = 'Charlie'
    B: COMMIT

T3: Transaction A:
    A: UPDATE invoices SET buyer = 'Bob'
    A: COMMIT

Final state:
  car_owner = 'Bob'
  car_invoice = 'Bob'
  
Charlie paid but doesn't own the car
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

T1: Backup: Read account1 = $1000 

T2: Transaction: Transfer $500 from account1 to account2
    account1 = $500
    account2 = $1500
    COMMIT

T3: Backup: Read account2 = $1500 

Backup sees: account1 = $1000, account2 = $1500
Total = $2500 
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
    account1 → Reads version 1: $1000 
    account2 → Reads version 1: $1000 
    Total = $2000 

Backup sees consistent snapshot from T0
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
  SELECT * FROM accounts;  -- Same snapshot
COMMIT;
```

## Part 2.5: MVCC Deep Dive - How Databases Actually Do This

The SRT transcript spent significant time explaining how PostgreSQL and MySQL implement snapshot isolation differently using **Multi-Version Concurrency Control (MVCC)**. We will explore into the actual mechanisms
### PostgreSQL's MVCC: Multiple Physical Row Versions

**Core Concept**: PostgreSQL stores **multiple physical copies** of each row on disk, tagged with transaction IDs.

Every row in PostgreSQL has hidden metadata columns:
- `xmin` - Transaction ID that created this row version
- `xmax` - Transaction ID that deleted/replaced this row (NULL if current)

**Example from SRT Transcript**:

```
Initial insert (Transaction ID = 1):
┌────┬──────┬─────────┬────────────┬──────┬──────┐
│ ID │ Name │ Game_ID │ Description│ xmin │ xmax │
├────┼──────┼─────────┼────────────┼──────┼──────┤
│  1 │robot │   222   │ a bot      │  1   │  ?   │
└────┴──────┴─────────┴────────────┴──────┴──────┘

Update by Transaction ID = 100 (changes game_id, description):
┌────┬──────┬─────────┬────────────┬──────┬──────┐
│ ID │ Name │ Game_ID │ Description│ xmin │ xmax │
├────┼──────┼─────────┼────────────┼──────┼──────┤
│  1 │robot │   222   │ a bot      │  1   │ 100  │ ← Old (max set!)
│  1 │robot │   100   │ newbot     │ 100  │  ?   │ ← New version
└────┴──────┴─────────┴────────────┴──────┴──────┘

Update by Transaction ID = 200 (changes name):
┌────┬──────┬─────────┬────────────┬──────┬──────┐
│ ID │ Name │ Game_ID │ Description│ xmin │ xmax │
├────┼──────┼─────────┼────────────┼──────┼──────┤
│  1 │robot │   222   │ a bot      │  1   │ 100  │ ← v1
│  1 │robot │   100   │ newbot     │ 100  │ 200  │ ← v2  
│  1 │monkey│   100   │ newbot     │ 200  │  ?   │ ← v3 (current)
└────┴──────┴─────────┴────────────┴──────┴──────┘
```

**Visibility Rules** (which version does each transaction see?):

```
Transaction 50:  xmin(1) <= 50 < xmax(100) → Sees v1 (robot, 222, "a bot")
Transaction 150: xmin(100) <= 150 < xmax(200) → Sees v2 (robot, 100, "newbot")
Transaction 250: xmin(200) <= 250 → Sees v3 (monkey, 100, "newbot")
```

### The Bloat Problem (PostgreSQL)

From SRT:

> "One of the reasons why you don't want to have long-running transactions is if you have some transactions that run for a really long time then that means you need to keep around more old versions of rows."

**Example**:
- Table starts with 1M rows
- Each row updated 10 times throughout the day
- PostgreSQL keeps all versions until no transaction needs them
- Result: **10 million row versions on disk!** (10x actual data)

### PostgreSQL VACUUM Process

**Auto Vacuum** (from SRT):

```javascript
// What VACUUM does:
function vacuumTable(table) {
    const oldestActiveTx = getOldestRunningTransaction();
    
    for (const rowVersion of table) {
        if (rowVersion.xmax < oldestActiveTx) {
            // No active transaction can see this old version
            markAsReusableSpace(rowVersion);
        }
    }
    
    // Space can be reused by future inserts/updates
}
```

**VACUUM vs VACUUM FULL** (from SRT):

```
VACUUM:
- Marks dead rows as reusable
- Doesn't return disk space to OS
- Table size stays same
- Can run concurrently

VACUUM FULL:
- Rebuilds entire table
- Removes gaps, returns space to OS  
- Requires table lock (blocks all operations!)
- Table size actually shrinks
```

### MySQL's MVCC: Undo Logs

From SRT:

> "MySQL does something different where it does not create a whole new version of the row every time. It actually does the update in place."

**MySQL Approach**:
1. Update row **in-place** in table
2. Store old values in separate **undo log**
3. Old transactions reconstruct previous versions from undo log

**Example from SRT**:

```
Table (always has current version only):
┌────┬──────┬─────────┬────────────┐
│ ID │ Name │ Game_ID │ Description│
├────┼──────┼─────────┼────────────┤
│  1 │monkey│   100   │ bananas    │ ← Current
└────┴──────┴─────────┴────────────┘

Undo Log (stores change history):
┌─────────────────────────────────────────────┐
│ TX 200: ID=1, name: 'robot' → 'monkey'      │
│ TX 300: ID=1, desc: 'newbot' → 'bananas'    │
└─────────────────────────────────────────────┘

Transaction 150 reads:
1. Fetch current row (name='monkey', desc='bananas')
2. Check undo log: TX 150 < 200, need older name
3. Apply undo: name = 'robot'
4. Check undo log: TX 150 < 300, need older desc
5. Apply undo: desc = 'newbot'
6. Return: (robot, 100, newbot) 
```

**Trade-offs**:

```
PostgreSQL MVCC:
 Fast reads (just scan for correct version)
 Table bloat (multiple row versions)
 VACUUM overhead

MySQL MVCC:
 Less table bloat (one row version)
 No table vacuum needed
 Undo log can grow huge
 Must reconstruct old versions (CPU cost)
```

### Long-Running Transaction Problem (Both!)

From SRT, this was **heavily emphasized**:

```
PostgreSQL with long transaction:
- Can't vacuum old rows (transaction still needs them)
- Table size: 1GB → 5GB → 20GB → 100GB! 💾
- Queries slow down (scanning dead rows)

MySQL with long transaction:
- Undo log: 100MB → 1GB → 10GB
- Reconstructing versions gets slower
- Disk fills up

Both databases:
 Never run hour-long analytics in OLTP database
 Keep transactions under seconds
 Use separate analytics database for long queries
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

Final: counter = 101 
Expected: 102

One increment was lost
```

**Solution 1: Atomic Operations**

```sql
--  Bad: Read-modify-write (loses updates)
counter = SELECT count FROM stats WHERE id = 1;
counter += 1;
UPDATE stats SET count = counter WHERE id = 1;

--  Good: Atomic operation
UPDATE stats SET count = count + 1 WHERE id = 1;
```

Database handles increment atomically. No lost updates
**Solution 2: Explicit Locking**

```sql
-- PostgreSQL
BEGIN;
  SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- LOCK
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
//  Lost update risk
const doc = db.users.findOne({_id: 123});
doc.views += 1;
db.users.update({_id: 123}, doc);

//  Atomic increment
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
  
  Both are on call  Constraint satisfied

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
  
  Zero doctors on call!  Constraint violated
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
-- Double-booked room
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
-- Duplicate username
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
    -- Returns: 1  (if not using snapshot isolation)
    
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
  SELECT * FROM on_call_lock WHERE lock_id = 1 FOR UPDATE;  -- LOCK
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

**Idea**: Just run transactions one at a time! No concurrency = no concurrency problems
```javascript
// Single-threaded transaction executor
const transactionQueue = new Queue();

while (true) {
    const transaction = transactionQueue.get();
    executeTransaction(transaction);  // Run completely
    // Next transaction
}
```

**When This Works**:

 **Transactions are fast** (milliseconds, not seconds)
 **Active dataset fits in memory** (no slow disk I/O)
 **Throughput requirement is modest** (one CPU core worth)

**Real-World: Redis**

Redis is single-threaded and executes commands serially.

```javascript
const redis = require('redis');
const client = redis.createClient();

// Lua script executed atomically
const script = `
local current = redis.call('GET', KEYS[1])
if current then
  return redis.call('INCRBY', KEYS[1], ARGV[1])
else
  return redis.call('SET', KEYS[1], ARGV[1])
end
`;

// Runs atomically
await client.eval(script, {
    keys: ['counter'],
    arguments: ['10']
});
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
-  Doesn't scale to multiple cores easily
-  Slow transactions block everything
-  Not suitable for interactive transactions

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
│ Shared   │        │             │
│ Exclusive│        │             │
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

 **Slow**: Lots of waiting for locks
 **Deadlocks**: Transactions wait for each other in a cycle

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
T1: ABORT  (detected stale read)
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
→ ABORT one transaction 
```

**Implementation**:

Database tracks:
1. **Which transactions read which objects**
2. **Which transactions wrote which objects**
3. **Read predicates** (conditions in WHERE clauses)

```javascript
// Simplified tracking
const readLog = {
  tx1: ["doctors WHERE on_call = true"],
  tx2: ["doctors WHERE on_call = true"]
};

const writeLog = {
  tx1: ["doctors.alice.on_call = false"],
  tx2: ["doctors.bob.on_call = false"]
};

// At commit:
function checkSerializable(tx) {
  for (const otherTx of committedTransactions) {
    if (writesAffectReads(otherTx, tx)) {
      // Conflict detected
      return 'ABORT';
    }
  }
  return 'COMMIT';
}
```

**Advantages over 2PL**:

 **Better performance**: No waiting for locks
 **Read-only queries never block**: Always use snapshot
 **Less deadlock**: Only abort at commit time

**Disadvantages**:

 **More aborts**: Optimistic = sometimes wrong
 **Retry logic needed**: Application must retry aborted transactions

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

## Part 3.5: Read Uncommitted - The Weakest Isolation

**Definition**: Transactions can see **uncommitted changes** from other transactions. No isolation at all
### What Read Uncommitted Allows (From SRT)

> "Read uncommitted literally means I can start a transaction and I can start reading data and if there's another transaction that is making writes underneath me to the same data that I'm working on, I can actually see all of those writes taking place."

**Example from SRT**:

```
Time T0: Row has name='robot', game_id=100

Time T1: Transaction A begins
         Transaction B begins (READ UNCOMMITTED)

Time T2: Transaction A: UPDATE row SET name='monkey'
         (NOT COMMITTED YET!)

Time T3: Transaction B: SELECT * FROM table WHERE id=1
         → Sees name='monkey' 👀 (uncommitted!)

Time T4: Transaction B: ...makes decisions based on 'monkey'...

Time T5: Transaction A: ROLLBACK (changed mind!)
         → name reverts to 'robot'

Time T6: Transaction B: SELECT * FROM table WHERE id=1
         → Now sees name='robot' 🤯

Transaction B saw data that NEVER ACTUALLY EXISTED in committed state
```

### Why This is Dangerous

**Scenario**: Bank transfer with Read Uncommitted

```
Initial: Alice balance=$1000, Bob balance=$500

Transaction A (transfer $100 from Alice to Bob):
  UPDATE accounts SET balance = 900 WHERE name='Alice'  (uncommitted!)

Transaction B (READ UNCOMMITTED):
  SELECT balance FROM accounts WHERE name='Alice'
  → Sees $900
  → Calculates: "Alice has $900, she can afford $800 loan"
  → Approves loan
Transaction A: ROLLBACK! (transfer canceled)
  → Alice balance back to $1000

Transaction B approved loan based on balance that never existed
```

### Performance vs. Correctness Trade-off (From SRT)

> "The nice thing from a performance perspective is if I'm doing this, I don't have to handle locking rows...all the queries can just zoom as fast as they can. But you can end up in weird states where you have an inconsistent view."

**What You Get**:

```
 No locks
 No waiting  
 Maximum parallelism
 Fastest possible reads

 Dirty reads (see uncommitted data)
 Non-repeatable reads
 Phantom reads
 Lost updates
 Completely inconsistent view
```

### When to Use Read Uncommitted? (From SRT)

> "In what real cases read uncommitted can be useful? I would probably just not use it. There's a reason why it's not the default."

The presenter's guidance:

```
Maybe use for:
- Approximate statistics (view counts, page visits)
- Non-critical dashboards
- "Good enough" estimates

Never use for:
- Financial transactions 
- Inventory management 
- User data 
- Anything requiring consistency 
```

**Even for statistics, probably better to use Read Committed:**

```javascript
// Instead of Read Uncommitted for stats:
await db.setIsolation('READ COMMITTED');  // Still fast, but consistent
const count = await db.query("SELECT COUNT(*) FROM page_views");
```

### Configuration

```sql
-- PostgreSQL (rarely used!)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- MySQL
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
  SELECT * FROM accounts;  -- Can see uncommitted changes
COMMIT;
```

**Bottom Line** (from SRT): Read Uncommitted is almost never the right choice. Even when you think you don't need consistency, you probably do. Use Read Committed instead.

## Part 4: Real-World Transaction Examples

### Example 1: Bank Transfer (Classic)

```javascript
async function transfer(fromAccount, toAccount, amount) {
    try {
        await db.begin({ isolationLevel: 'SERIALIZABLE' });
        
        // Check balance
        const result = await db.query(
            "SELECT balance FROM accounts WHERE id = ? FOR UPDATE",
            [fromAccount]
        );
        const balance = result[0].balance;
        
        if (balance < amount) {
            throw new Error('InsufficientFunds');
        }
        
        // Deduct from source
        await db.execute(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            [amount, fromAccount]
        );
        
        // Add to destination
        await db.execute(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            [amount, toAccount]
        );
        
        // Record transaction
        await db.execute(
            "INSERT INTO transfers (from_account, to_account, amount) VALUES (?, ?, ?)",
            [fromAccount, toAccount, amount]
        );
        
        await db.commit();
        return "Success";
        
    } catch (e) {
        await db.rollback();
        return `Failed: ${e.message}`;
    }
}
```

### Example 2: Ticket Booking System

```javascript
async function bookTicket(userId, eventId, numTickets) {
    try {
        await db.begin({ isolationLevel: 'REPEATABLE READ' });
        
        // Lock event row
        const result = await db.query(
            "SELECT available_tickets, price FROM events WHERE id = ? FOR UPDATE",
            [eventId]
        );
        const event = result[0];
        
        if (event.available_tickets < numTickets) {
            throw new Error('SoldOut');
        }
        
        // Update available count
        await db.execute(
            "UPDATE events SET available_tickets = available_tickets - ? WHERE id = ?",
            [numTickets, eventId]
        );
        
        // Create booking
        const bookingResult = await db.execute(
            "INSERT INTO bookings (user_id, event_id, num_tickets) VALUES (?, ?, ?)",
            [userId, eventId, numTickets]
        );
        
        // Charge user
        const paymentId = await chargePayment(userId, event.price * numTickets);
        
        await db.execute(
            "UPDATE bookings SET payment_id = ? WHERE id = ?",
            [paymentId, bookingResult.insertId]
        );
        
        await db.commit();
        return "Booked successfully";
        
    } catch (e) {
        await db.rollback();
        return `Booking failed: ${e.message}`;
    }
}
```

### Example 3: Distributed Transaction (2PC)

When transaction spans multiple databases:

```javascript
async function transferAcrossBanks(fromBankDb, toBankDb, amount) {
    const transactionCoordinator = new TransactionCoordinator();
    
    try {
        // Phase 1: PREPARE
        // Ask each database if it can commit
        await fromBankDb.prepare("UPDATE accounts SET balance = balance - ?", [amount]);
        await toBankDb.prepare("UPDATE accounts SET balance = balance + ?", [amount]);
        
        // Both said YES
        // Phase 2: COMMIT
        await fromBankDb.commit();
        await toBankDb.commit();
        
        return "Success";
        
    } catch (e) {
        // If ANY database says NO or fails
        // ABORT on all
        await fromBankDb.rollback();
        await toBankDb.rollback();
        
        return `Failed: ${e.message}`;
    }
}
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

## Part 6: PostgreSQL vs MySQL - The Transaction Trade-offs

### The Popularity Misconception (From SRT)

> "PostgreSQL is very popular right now...But there are some people who get this perspective of like oh PostgreSQL is popular...therefore it must just be better than MySQL...there are some places where MySQL is arguably better than PostgreSQL."

### MVCC Comparison

```
┌──────────────────────────────────────────────────────────┐
│  PostgreSQL vs MySQL MVCC                                │
├────────────────┬────────────────┬────────────────────────┤
│    Aspect      │  PostgreSQL    │      MySQL InnoDB      │
├────────────────┼────────────────┼────────────────────────┤
│ Row versions   │ Multiple in    │ One in table,          │
│                │ table itself   │ old in undo log        │
├────────────────┼────────────────┼────────────────────────┤
│ Updates        │ New row        │ In-place with          │
│                │ version        │ undo log entry         │
├────────────────┼────────────────┼────────────────────────┤
│ Table bloat    │ Can be severe  │ Less prone to bloat    │
├────────────────┼────────────────┼────────────────────────┤
│ Maintenance    │ VACUUM needed  │ Undo log cleanup       │
├────────────────┼────────────────┼────────────────────────┤
│ Write-heavy    │ Can struggle   │ Better for updates     │
│ workloads      │ with bloat     │                        │
├────────────────┼────────────────┼────────────────────────┤
│ Long txns      │ Prevent VACUUM,│ Huge undo logs,        │
│                │ massive bloat  │ slow reconstruction    │
└────────────────┴────────────────┴────────────────────────┘
```

### When PostgreSQL Excels

From the SRT context:

```
 Read-heavy workloads
 Complex queries and analytics
 Rich extension ecosystem (PostGIS, pg_trgm, etc.)
 Periodic low-traffic windows (for VACUUM)
 JSONB and advanced data types
 Excellent query optimizer
 Strong community and cloud provider support
```

### When MySQL Excels

From the SRT discussion about table bloat:

> "For some people this whole thing of making many versions of rows and having to vacuum, this is actually a huge problem...especially if you have like a lot of writes and you do a lot of overwriting existing rows or writing new rows and expiring deleting old rows, this architecture can actually be a really big problem."

```
 High-volume writes (updates/deletes)
 Workloads with frequent row updates
 24/7 high-traffic (can't schedule VACUUM windows)
 Simpler operational model
 Predictable disk usage
 Better for write-heavy OLTP
```

### Default Isolation Levels (From SRT)

```
PostgreSQL:  Read Committed (default)
MySQL:       Repeatable Read (default)

Why the difference?
- PostgreSQL: Read Committed avoids many anomalies, good performance
- MySQL: Repeatable Read with Gap Locks prevents more anomalies
```

### Real-World Scenarios

**Scenario 1: E-commerce (frequent inventory updates)**

```javascript
// Many concurrent updates to same products
// UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
// UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;
// UPDATE inventory SET stock = stock - 1 WHERE product_id = 123;

// PostgreSQL: Creates many row versions → bloat
// MySQL: Updates in-place → less bloat
// Winner: MySQL (for this pattern)
```

**Scenario 2: Analytics Dashboard (mostly reads)**

```javascript
// Long-running analytical queries
// SELECT AVG(order_total), COUNT(*), DATE(created_at)
// FROM orders
// WHERE created_at > NOW() - INTERVAL 30 DAY
// GROUP BY DATE(created_at);

// PostgreSQL: Snapshot isolation, no reconstruction needed
// MySQL: Must consult undo log for old versions
// Winner: PostgreSQL (for analytical workloads)
```

**Scenario 3: High-Frequency Trading (write-heavy, no downtime)**

```javascript
// Continuous writes 24/7, can't afford maintenance windows
// INSERT INTO trades VALUES (...);  // 10,000/sec
// UPDATE positions SET quantity = quantity + ...;  // 5,000/sec

// PostgreSQL: VACUUM struggles to keep up, bloat grows
// MySQL: Undo log manageable, in-place updates
// Winner: MySQL (for extreme write workloads)
```

### The Take-Away (From SRT)

> "Say what you will about it's actually not an older database...say what you will about oh it doesn't have as good of a plugin ecosystem...MySQL is really good for a lot of things."

**Key Insight**: Choose based on your workload, not popularity:

```
If your app:
- Mostly reads: Consider PostgreSQL
- Heavy writes to same rows: Consider MySQL  
- Complex queries: PostgreSQL
- Simple CRUD at scale: MySQL
- Needs extensions: PostgreSQL
- Predictable operations: MySQL
```

**Both are excellent databases!** The "right" choice depends on your specific use case.

## Part 7: Practical Decision Framework

### Choosing an Isolation Level

```
┌────────────────────────────────────────────────────────┐
│  ISOLATION LEVEL DECISION TREE                         │
└────────────────────────────────────────────────────────┘

START: What are you building?

Financial transactions (money, payments)?
├─ YES → Use SERIALIZABLE
│         - Worth the performance cost
│         - Correctness is critical
│         - Example: Bank transfers, payment processing
│
└─ NO → Continue...

High-concurrency reads with occasional writes?
├─ YES → Use READ COMMITTED
│         - Default for PostgreSQL, Oracle
│         - Good balance of performance/consistency
│         - Example: Social media feeds, content sites
│
└─ NO → Continue...

Need consistent snapshots (analytics, backups)?
├─ YES → Use REPEATABLE READ
│         - Default for MySQL
│         - Consistent view throughout transaction
│         - Example: Report generation, backups
│         - WARNING: Keep transactions SHORT
│
└─ NO → Continue...

Just counting stats, don't care about precision?
├─ YES → Use READ COMMITTED (not READ UNCOMMITTED!)
│         - Still fast, but avoids dirty reads
│         - Example: Dashboard metrics, approximate counts
│
└─ NO → Use READ COMMITTED (safe default)
```

### Transaction Size Guidelines

From the SRT emphasis on short transactions:

```
┌────────────────────────────────────────────────────────┐
│  TRANSACTION DURATION GUIDELINES                       │
├───────────────────┬────────────────────────────────────┤
│  Duration         │  Guidance                          │
├───────────────────┼────────────────────────────────────┤
│  < 100ms          │   Perfect! Most OLTP should be   │
│                   │     this fast                      │
├───────────────────┼────────────────────────────────────┤
│  100ms - 1s       │   Acceptable for complex ops     │
├───────────────────┼────────────────────────────────────┤
│  1s - 10s         │  ⚠️  Getting long, consider        │
│                   │     breaking up                    │
├───────────────────┼────────────────────────────────────┤
│  10s - 1min       │   Too long! Will cause bloat     │
│                   │     and performance issues         │
├───────────────────┼────────────────────────────────────┤
│  > 1min           │   NEVER do this in OLTP!         │
│                   │     Use separate analytics DB      │
└───────────────────┴────────────────────────────────────┘
```

### Monitoring Checklist

```javascript
// PostgreSQL monitoring
async function monitorPostgreSQL() {
    const checks = {
        'table_bloat': "SELECT * FROM pgstattuple('table_name')",
        'vacuum_status': "SELECT * FROM pg_stat_user_tables",
        'long_transactions': `
            SELECT pid, now() - xact_start as duration, query
            FROM pg_stat_activity
            WHERE xact_start < now() - interval '1 minute'
        `,
        'transaction_id_wraparound': "SELECT age(datfrozenxid) FROM pg_database"
    };
    
    for (const [check, query] of Object.entries(checks)) {
        const result = await runQuery(query);
        if (isProblematic(result)) {
            alert(`PostgreSQL ${check} issue: ${result}`);
        }
    }
}

// MySQL monitoring
async function monitorMySQL() {
    const checks = {
        'undo_log_size': "SHOW ENGINE INNODB STATUS",  // Check "History list length"
        'long_transactions': `
            SELECT * FROM information_schema.innodb_trx
            WHERE trx_started < NOW() - INTERVAL 1 MINUTE
        `,
        'locks': "SELECT * FROM performance_schema.data_locks"
    };
    
    for (const [check, query] of Object.entries(checks)) {
        const result = await runQuery(query);
        if (isProblematic(result)) {
            alert(`MySQL ${check} issue: ${result}`);
        }
    }
}
```

### Common Mistakes to Avoid

```
 Running analytics queries in OLTP database with REPEATABLE READ
   → Use separate analytics database or Read Committed

 Leaving transactions open during user interaction
   → Begin transaction only when ready to commit

 Using Serializable for everything
   → Overkill for most workloads, use Read Committed

 Ignoring abort/retry logic with Serializable
   → Must handle serialization failures gracefully

 Not monitoring table bloat (PostgreSQL)
   → Set up alerts, schedule VACUUM

 Assuming same isolation level works everywhere
   → Test with production-like concurrency
```

**Next Chapter**: The Trouble with Distributed Systems - what happens when transactions span multiple machines and networks can fail
