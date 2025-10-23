# Detailed Interview Notes: Concurrency Control in Databases

## Part 1: Database Locking - Fundamental Concepts

### What is Database Locking?

Database locking is a fundamental mechanism used to control concurrent access to data in a database. When multiple transactions try to access and modify the same data simultaneously, locking ensures that these operations happen in a controlled manner, preventing data inconsistencies and corruption.

Think of a database lock like a "Do Not Disturb" sign on a hotel room. When one guest is using the room, a lock is placed on the door. Other guests cannot enter until the lock is removed. Similarly, when a transaction locks a piece of data, other transactions must wait until that lock is released before they can access it.

The core purpose of locking is to provide **mutual exclusion** - ensuring that only one transaction can modify a piece of data at any given time, and preventing other transactions from reading uncommitted (dirty) data.

### Shared Locking (Read Lock)

**Definition:** A shared lock allows multiple transactions to **read** the same data simultaneously, but prevents any of them from modifying it while the lock is held.

**Key Characteristics:**
- Multiple transactions can hold a shared lock on the same resource at the same time
- No transaction holding a shared lock can modify the locked resource
- A transaction with a shared lock can read the data
- Shared locks are compatible with other shared locks but incompatible with exclusive locks

**Real-Life Scenario:**
Imagine a library where multiple people can read the same reference book at the same time. Each reader gets a "reading pass" (shared lock). While these passes are active, nobody can write or modify the book's contents. The book can be read by multiple people simultaneously, but it cannot be altered while anyone is reading it.

**Database Example:**
When Transaction A executes a SELECT query to read a customer's balance, it acquires a shared lock. Transaction B can simultaneously read the same customer's balance (acquiring its own shared lock). However, neither transaction can update the balance while these shared locks are active.

### Exclusive Locking (Write Lock)

**Definition:** An exclusive lock grants a transaction complete and sole access to a resource. Only one transaction can hold an exclusive lock on a resource at any time, and no other transaction can read or write to that resource while the exclusive lock is held.

**Key Characteristics:**
- Only one transaction can hold an exclusive lock on a resource at any given time
- A transaction with an exclusive lock has complete control over the resource
- Other transactions cannot acquire any lock (shared or exclusive) on that resource
- Exclusive locks are incompatible with all other locks

**Real-Life Scenario:**
Imagine a bathroom in a house. When someone enters and locks the door (exclusive lock), nobody else can enter until they leave and unlock the door. The bathroom is completely reserved for that one person during that time. Unlike the library scenario with shared locks, exclusive access means complete isolation.

**Database Example:**
When Transaction A executes an UPDATE statement to modify a customer's balance, it acquires an exclusive lock. No other transaction can read or modify that customer's balance until Transaction A completes and releases the lock. Transaction B must wait if it tries to read or update the same balance.

### Lock Compatibility Matrix

| | Shared Lock | Exclusive Lock |
|---|---|---|
| **Shared Lock** | ✓ Compatible | ✗ Incompatible |
| **Exclusive Lock** | ✗ Incompatible | ✗ Incompatible |

---

## Part 2: Transaction Anomalies - Reading Inconsistent Data

### Overview of Transaction Anomalies

When multiple transactions execute concurrently without proper isolation, three main types of anomalies can occur: dirty reads, non-repeatable reads, and phantom reads. These represent increasingly subtle consistency violations.

### Dirty Read (Uncommitted Dependency)

**Definition:** A dirty read occurs when a transaction reads data that has been modified by another transaction but not yet committed. If the modifying transaction rolls back, the reading transaction has accessed data that never actually existed in the database.

**Why It's Problematic:** The transaction is reading temporary, uncommitted changes that might be reverted, leading to invalid business decisions.

**Real-Life Scenario - Bank Transfer Gone Wrong:**

Imagine two bank tellers:
- Account X has $1000 initially
- Teller A processes a withdrawal and deducts $500 from Account X (balance now shows $500 in memory, not committed)
- Teller B reads Account X's balance and sees $500
- Teller A's transaction is rejected due to fraud check and rolls back
- Now Teller B has based a decision on a balance of $500, but the actual balance is still $1000

**Database Timeline:**

```
Time 1: Transaction A reads Account X balance = $1000
Time 2: Transaction A updates Account X balance to $500 (not yet committed)
Time 3: Transaction B reads Account X balance = $500 (DIRTY READ!)
Time 4: Transaction A encounters an error and ROLLBACK
Time 5: Account X balance reverts to $1000, but Transaction B thinks it's $500
```

**Consequence:** Transaction B made decisions based on data that never actually persisted in the database.

### Non-Repeatable Read (Inconsistent Read)

**Definition:** A non-repeatable read occurs when a transaction reads the same row twice and gets different values because another transaction modified and committed that data between the two reads.

**Why It's Problematic:** The transaction cannot trust that the same query will return the same results within its execution, violating consistency expectations.

**Real-Life Scenario - Movie Ticket Availability:**

Imagine an online ticketing system:
- You query available seats in a theater at 2:00 PM and see 50 seats available
- You take some time to decide and consider your options
- Another customer quickly purchases 30 tickets
- You query again at 2:05 PM and now only 20 seats are available
- You cannot perform consistent business logic because the data changed mid-transaction

**Database Timeline:**

```
Time 1: Transaction A reads Seats_Available = 50
Time 2: Transaction B updates Seats_Available to 20 (and commits)
Time 3: Transaction A reads Seats_Available again = 20 (NON-REPEATABLE READ!)
```

**Consequence:** The same SELECT query executed twice within the same transaction returns different results, making the transaction unreliable for critical operations like generating reports or performing calculations.

### Phantom Read

**Definition:** A phantom read occurs when a transaction executes a query twice and gets a different set of rows the second time because another transaction inserted or deleted rows between the two queries (and committed those changes).

**Why It's Problematic:** The transaction cannot trust the result set size, and new data mysteriously appears or disappears within the transaction scope.

**Real-Life Scenario - Restaurant Inventory:**

Imagine a restaurant manager:
- At 5:00 PM, you query "How many customers are currently seated?" and get 12 tables occupied
- More customers arrive and are seated by the host
- At 5:15 PM, you query the same question and now get 15 tables occupied
- You didn't see the intermediate changes, but new rows appeared

**Database Timeline:**

```
Time 1: Transaction A queries "SELECT COUNT(*) FROM Orders WHERE status='pending'" = 5 orders
Time 2: Transaction B inserts 3 new orders and commits
Time 3: Transaction A queries the same "SELECT COUNT(*) FROM Orders WHERE status='pending'" = 8 orders (PHANTOM READ!)
```

**Consequence:** The result set itself changed between queries, not just the values within existing rows. This is particularly problematic for aggregation queries (COUNT, SUM, AVG) and range queries.

### Key Difference Between Non-Repeatable and Phantom Reads

- **Non-Repeatable Read:** Same row changes its values
  - Row still exists, but the **data within it** is different
  - Affects UPDATE and DELETE scenarios

- **Phantom Read:** New rows appear or disappear
  - The **number of rows** in the result set changes
  - Affects INSERT and DELETE scenarios

---

## Part 3: Transaction Isolation Levels

Transaction isolation levels define how much isolation one transaction has from other concurrent transactions. Higher isolation levels prevent more anomalies but generally reduce concurrency and performance.

### READ UNCOMMITTED (Isolation Level 0)

**Definition:** The lowest isolation level. Transactions can read uncommitted (dirty) data from other transactions.

**Anomalies Prevented:** None

**Anomalies Allowed:**
- Dirty reads (✗ can occur)
- Non-repeatable reads (✗ can occur)
- Phantom reads (✗ can occur)

**How It Works:**
- Transactions do not acquire shared locks when reading data
- Transactions acquire exclusive locks only when writing
- Other transactions can read data being modified before it's committed

**Real-World Scenario:**
An analytics team running exploratory queries on production data doesn't want to wait for locks. They accept the risk of reading partially updated data because they're just investigating trends and don't need 100% accuracy for exploration purposes.

**Pros:**
- Highest concurrency and performance
- Minimal locking overhead

**Cons:**
- Risk of reading invalid data
- Not suitable for critical business operations

### READ COMMITTED (Isolation Level 1)

**Definition:** A transaction can only read data that has been committed by other transactions. This is the most common default isolation level in many databases.

**Anomalies Prevented:**
- Dirty reads (✓ prevented)

**Anomalies Allowed:**
- Non-repeatable reads (✗ can occur)
- Phantom reads (✗ can occur)

**How It Works:**
- When reading, transactions acquire shared locks and immediately release them after reading
- When writing, transactions acquire exclusive locks and hold them until commit
- Transactions can only see committed versions of data

**Real-World Scenario:**
A bank's customer service application reads account balances for display purposes. It prevents dirty reads (doesn't show in-progress transfers), but might show a slightly different balance if a transaction commits between two reads - which is acceptable because the customer is just checking their current status.

**Locking Behavior Timeline:**

```
Time 1: Transaction A acquires shared lock, reads Account Balance = $1000, releases lock
Time 2: Transaction B acquires exclusive lock, updates Account Balance to $900
Time 3: Transaction B commits (releases lock)
Time 4: Transaction A acquires shared lock, reads Account Balance = $900, releases lock
        (Non-repeatable read can occur, but no dirty read)
```

**Pros:**
- Prevents dirty reads
- Better concurrency than higher levels
- Good balance for most applications

**Cons:**
- Non-repeatable reads can occur
- Phantom reads can occur
- Not suitable for operations requiring consistent snapshots

### REPEATABLE READ (Isolation Level 2)

**Definition:** A transaction sees a consistent snapshot of committed data. Any rows read during the transaction cannot be modified or deleted by other transactions, but new rows can be inserted (phantom reads possible).

**Anomalies Prevented:**
- Dirty reads (✓ prevented)
- Non-repeatable reads (✓ prevented)

**Anomalies Allowed:**
- Phantom reads (✗ can occur)

**How It Works:**
- Transactions acquire shared locks when reading and hold them until transaction end (commit or rollback)
- Transactions acquire exclusive locks when writing and hold them until transaction end
- Prevents modification or deletion of rows accessed during the transaction
- Cannot prevent new row insertions matching the criteria

**Real-World Scenario:**
A financial reconciliation system processes account transactions. It needs to ensure that the specific transactions it read at the start don't change mid-process. However, new transactions arriving after the initial scan are acceptable as long as the original ones remain consistent.

**Locking Behavior Timeline:**

```
Time 1: Transaction A acquires shared lock, reads Salary = $5000 for Employee ID 101
Time 2: Transaction B tries to update Salary for Employee ID 101 - BLOCKED (Transaction A holds shared lock)
Time 3: Transaction B tries to INSERT new employee record - SUCCESS (not locked by A)
Time 4: Transaction A reads again, Salary = $5000 (REPEATABLE - no change)
Time 5: Transaction A commits, releases all locks
Time 6: Transaction B's update now proceeds
```

**Pros:**
- Eliminates both dirty reads and non-repeatable reads
- Suitable for most critical operations
- Better than serializable for performance

**Cons:**
- Phantom reads can still occur
- More locking overhead than lower levels
- Can cause more lock contention

### SERIALIZABLE (Isolation Level 3)

**Definition:** The highest isolation level. Transactions are executed as if they were running sequentially, one after another, even though they execute concurrently. All anomalies are prevented.

**Anomalies Prevented:**
- Dirty reads (✓ prevented)
- Non-repeatable reads (✓ prevented)
- Phantom reads (✓ prevented)

**Anomalies Allowed:** None

**How It Works:**
- Transactions acquire shared locks on ranges of rows that match query criteria
- Additional exclusive locks prevent insertions into ranges being read by other transactions
- Provides complete isolation as if transactions executed serially

**Real-World Scenario:**
A stock trading system executing a large portfolio transfer. It must ensure that no intermediate inconsistencies occur. The entire transfer operation sees a completely stable view of inventory, prices, and holdings throughout its execution.

**Locking Behavior Timeline:**

```
Time 1: Transaction A acquires shared lock on range [Seats 1-50], reads 50 available seats
Time 2: Transaction B tries to INSERT new seat in range [Seats 1-50] - BLOCKED
Time 3: Transaction B tries to UPDATE any seat in range [Seats 1-50] - BLOCKED
Time 4: Transaction A reads again, still 50 available seats (REPEATABLE, NO PHANTOMS)
Time 5: Transaction A commits, releases all locks
Time 6: Transaction B can now proceed
```

**Pros:**
- Complete protection against all anomalies
- Guaranteed consistency and correctness
- Suitable for critical financial operations

**Cons:**
- Severely reduced concurrency
- High locking overhead
- Can cause significant performance degradation
- More likely to cause deadlocks

### Isolation Level Comparison Table

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Concurrency | Performance |
|---|---|---|---|---|---|
| **Read Uncommitted** | ✗ Possible | ✗ Possible | ✗ Possible | Very High | Very Fast |
| **Read Committed** | ✓ Prevented | ✗ Possible | ✗ Possible | High | Fast |
| **Repeatable Read** | ✓ Prevented | ✓ Prevented | ✗ Possible | Medium | Medium |
| **Serializable** | ✓ Prevented | ✓ Prevented | ✓ Prevented | Low | Slow |

---

## Part 4: Optimistic vs. Pessimistic Concurrency Control

### Pessimistic Concurrency Control (Locking)

**Philosophy:** "Assume the worst will happen - lock resources before accessing them."

Pessimistic concurrency control assumes that conflicts between transactions are likely to occur, so it prevents them proactively by locking resources before accessing them.

**How It Works:**
1. Before reading or writing data, a transaction acquires a lock on that resource
2. The lock is held for the duration of the transaction
3. Other transactions must wait for the lock to be released
4. Only when the lock is released can other transactions access the resource

**Characteristics:**
- Locks are acquired upfront (preventive approach)
- Lock contention can be high in high-concurrency scenarios
- Guarantees that conflicts won't occur because they're prevented
- Generally associated with higher isolation levels (Repeatable Read, Serializable)

**Real-World Scenario - Airline Seat Booking:**
A pessimistic system works like the traditional booking counter. When a customer indicates they want to book Seat 12A, the agent immediately reserves that seat (acquires a lock). Other customers cannot even view or consider that seat while it's locked. The agent then processes the full booking. Only when the booking is complete or the customer leaves does the lock release, allowing other customers to book that seat.

**Locking Timeline:**

```
Time 1: Customer A requests Seat 12A - Agent acquires EXCLUSIVE lock on Seat 12A
Time 2: Customer B tries to request Seat 12A - BLOCKED (waiting for lock)
Time 3: Customer A completes booking - lock released
Time 4: Customer B acquires lock on Seat 12A - booking proceeds
```

**Typical Isolation Level:** Repeatable Read or Serializable
- Repeatable Read: Prevents reading other transactions' modifications mid-transaction
- Serializable: Completely isolates concurrent transactions

**Pros:**
- Simple to understand and implement
- Prevents conflicts entirely (no retries needed)
- Suitable for high-contention scenarios where conflicts are expected
- Good for operations requiring strong consistency guarantees

**Cons:**
- Reduced concurrency and throughput in low-conflict scenarios
- Can waste resources locking data that won't actually be contested
- Potential for deadlocks
- Higher latency for waiting transactions

### Optimistic Concurrency Control

**Philosophy:** "Assume conflicts are rare - read freely, check for conflicts only at commit time."

Optimistic concurrency control assumes that conflicts between transactions are uncommon, so it allows transactions to proceed without locking. At commit time, it checks whether any conflicts occurred during execution.

**How It Works:**
1. Transactions read data without acquiring locks
2. Each transaction maintains a version number or timestamp of the data it read
3. Transactions execute their business logic independently
4. At commit time, before writing changes to the database:
   - The system checks if the data being modified has been changed since it was read
   - If it hasn't changed, the transaction commits successfully
   - If it has changed, the transaction fails and must be retried

**Characteristics:**
- No locks are held during transaction execution (optimistic)
- Conflict detection occurs only at commit time
- If conflicts are detected, the transaction must be retried
- Generally associated with lower isolation levels (typically Read Committed)
- Uses version numbers, timestamps, or MVCC (Multi-Version Concurrency Control)

**Real-World Scenario - Google Docs Collaborative Editing:**
Multiple people can edit the same document simultaneously without locking sections. Each person sees their own view of the document. When one person saves changes, the system checks if someone else modified the same parts. If they did, it alerts the user and asks them to resolve conflicts or retry. This works well because simultaneous edits to the same exact lines are relatively rare.

**Version Number Timeline:**

```
Time 1: Transaction A reads Customer Record (version=5), Balance=$1000
Time 2: Transaction B reads Customer Record (version=5), Balance=$1000
Time 3: Transaction A calculates new balance=$900 and tries to commit
        System checks: version still = 5 ✓ COMMIT SUCCEEDS
Time 4: Transaction B calculates new balance=$950 and tries to commit
        System checks: version is now 6 (changed by A) ✗ COMMIT FAILS - RETRY
Time 5: Transaction B retries, reads updated Customer Record (version=6), Balance=$900
        Recalculates and commits with new version=7
```

**Typical Isolation Level:** Read Committed
- Allows reading committed data without locks
- Conflicts are detected and handled at commit time

**Pros:**
- Better concurrency in low-conflict scenarios
- No risk of deadlocks (no locks held)
- No waiting for locks, better responsiveness
- Suitable for high-concurrency, low-contention workloads

**Cons:**
- Transaction must be retried if conflicts occur
- Higher rollback overhead
- Not suitable for high-contention scenarios (frequent retries)
- More complex conflict detection logic
- Can waste work if retries are frequent

### Comparison: When to Use Which Approach

| Scenario | Better Choice | Reason |
|---|---|---|
| **High Contention (many users modifying same data)** | Pessimistic | Conflicts are expected; locking prevents expensive retries |
| **Low Contention (few users modify same data)** | Optimistic | Locks are wasted; better to detect rare conflicts at commit |
| **Short Transactions** | Pessimistic | Locks released quickly; minimal blocking |
| **Long-Running Transactions** | Optimistic | Pessimistic locking would block other users for too long |
| **Financial Systems** | Pessimistic | Strong consistency requirements demand prevention over detection |
| **Web Applications** | Optimistic | Stateless nature; user conflicts are rare |
| **Real-Time Systems** | Pessimistic | Predictable performance more important than throughput |
| **High-Throughput Systems** | Optimistic | Maximum concurrency more important than preventing retries |

---

## Part 5: Deadlocks in Pessimistic Concurrency Control

### What is a Deadlock?

A deadlock is a situation where two or more transactions are blocked indefinitely, each waiting for resources that another transaction holds and won't release.

**Key Condition:** For a deadlock to occur, there must be a **circular wait** - a cycle of transactions, where each transaction is waiting for a resource held by the next transaction in the cycle.

**Real-Life Scenario - Rush Hour Traffic:**

Imagine a 4-way intersection during rush hour:
- Car A (heading North) is in the intersection and needs to go to the North exit
- Car B (heading East) is waiting for Car A to move so it can enter
- Car C (heading South) entered the intersection before Car A and is crossing
- Car A is blocked by Car C and cannot proceed
- Car B is blocked by Car A and cannot proceed
- Car C is blocked by someone not yet in our scenario

All three cars are stuck, waiting for each other to move. None can proceed. This is a deadlock.

### Classic Deadlock Scenario in Databases

**Setup:**
- Account X has $100
- Account Y has $100
- Transaction A: Transfer $50 from Account X to Account Y
- Transaction B: Transfer $50 from Account Y to Account X

**Deadlock Timeline:**

```
Time 1: Transaction A acquires EXCLUSIVE lock on Account X (for reading $50)
Time 2: Transaction B acquires EXCLUSIVE lock on Account Y (for reading $50)
Time 3: Transaction A tries to acquire EXCLUSIVE lock on Account Y
        → BLOCKED (Transaction B holds the lock)
Time 4: Transaction B tries to acquire EXCLUSIVE lock on Account X
        → BLOCKED (Transaction A holds the lock)
Time 5: DEADLOCK!
        - Transaction A: "I'll give up X's lock once I get Y's lock"
        - Transaction B: "I'll give up Y's lock once I get X's lock"
        - Both wait forever in a circular dependency
```

**Visual Representation:**

```
Transaction A          Transaction B
     |                     |
     v                     v
  Lock X              Lock Y
     |                     |
     v                     v
  Need Y <--- WAIT ----> Need X
     |                     |
     +-----DEADLOCK!-------+
```

### Four Conditions Required for Deadlock

All four conditions must be present simultaneously for a deadlock to occur:

**1. Mutual Exclusion:**
Resources cannot be shared. Only one transaction can hold an exclusive lock at a time.
- In our scenario: Only one transaction can modify Account X at a time

**2. Hold and Wait (Hold-While-Waiting):**
A transaction can hold one resource and wait for another.
- In our scenario: Transaction A holds a lock on Account X while waiting for Account Y

**3. No Preemption:**
A lock cannot be forcefully taken from a transaction. The transaction must voluntarily release it.
- In our scenario: Transaction A cannot forcefully take Transaction B's lock on Account Y

**4. Circular Wait:**
There exists a circular chain of transactions, each waiting for a resource held by the next.
- In our scenario: A waits for B, B waits for A (circular dependency)

### Real-World Example - Order Processing System

**Scenario:**
An e-commerce system processes orders and inventory updates.

**Business Logic:**
- Process_Order_A: Lock Order Table, lock Product Table
- Process_Order_B: Lock Product Table, lock Order Table

**Execution Timeline:**

```
Time 1: Order_A thread acquires lock on Order Table (to insert new order)
Time 2: Order_B thread acquires lock on Product Table (to update inventory)
Time 3: Order_A thread tries to lock Product Table
        → BLOCKED (Order_B thread holds it)
Time 4: Order_B thread tries to lock Order Table
        → BLOCKED (Order_A thread holds it)
Time 5: SYSTEM DEADLOCK - both threads frozen indefinitely
```

**Consequence:** Orders cannot be processed, customers experience timeouts, and the system appears hung.

### How Databases Detect Deadlocks

**Wait-for Graph:**
Databases maintain a "wait-for graph" that tracks which transactions are waiting for locks held by other transactions.

- **Nodes** represent transactions
- **Edges** represent wait relationships (Transaction A waiting for Transaction B)

A deadlock exists when there is a **cycle** in this graph.

**Example Wait-for Graph in Deadlock:**

```
Transaction A -----> Transaction B
    ^                    |
    |                    v
    +-------- CYCLE ------+
```

**Detection Process:**
1. Database continuously monitors lock requests and grants
2. When a transaction waits for a lock, an edge is added to the graph
3. Database checks for cycles in the graph
4. If a cycle is detected, a deadlock is identified
5. The database picks one transaction as the "victim" to roll back

### Deadlock Resolution Strategies

#### 1. **Deadlock Victim Selection (Rollback)**

When a deadlock is detected, the database must choose one transaction to terminate (rollback). The victim transaction releases all its locks, allowing others to proceed.

**Victim Selection Criteria:**
- Transaction with least locks held
- Transaction with shortest execution time
- Transaction with least work done
- Transaction with lowest priority
- Random selection

**In Our Example:**
```
Time 1-4: Deadlock detected
Time 5: Database chooses Transaction A as victim (arbitrary decision)
Time 6: Transaction A is rolled back
        - All locks held by A are released
        - Account X lock is freed
Time 7: Transaction B acquires lock on Account X and proceeds
Time 8: Transaction B completes
Time 9: Transaction A is retried by the application
        - Now succeeds because no contention
```

**Disadvantage:** The victim transaction's work is completely wasted and must be retried.

#### 2. **Deadlock Prevention**

Instead of detecting and resolving deadlocks, some systems prevent them proactively by enforcing an ordering on resource allocation.

**Ordered Resource Allocation:**
All transactions must acquire locks in the same predetermined order.

**Before (Vulnerable to Deadlock):**
- Transaction A: Lock X then Lock Y
- Transaction B: Lock Y then Lock X

**After (Deadlock Prevention):**
- All transactions: Lock X first, then Lock Y (or a predefined order)

**Timeline with Ordered Locking:**

```
Time 1: Transaction A tries to lock X (lower order) → SUCCESS
Time 2: Transaction B tries to lock X (same order) → BLOCKED (A has it)
Time 3: Transaction A locks Y → SUCCESS (no contention, A got X first)
Time 4: Transaction A completes and releases both locks
Time 5: Transaction B acquires X and Y in order → SUCCESS
```

**Advantage:** Deadlock never occurs because circular waits are impossible.

**Disadvantage:** Requires knowing all resources upfront and enforcing strict ordering everywhere.

### Common Deadlock Patterns in Real Applications

**Pattern 1: Nested Table Updates**
```
Transaction A updates Table1, then updates Table2
Transaction B updates Table2, then updates Table1
→ Likely deadlock if timing aligns
```

**Pattern 2: Bidirectional Foreign Key Updates**
```
Update Customer and their associated Orders in different sequences
→ Can cause circular locks
```

**Pattern 3: Different Transaction Isolation Levels**
```
Some transactions use Repeatable Read (aggressive locking)
Others use Read Committed (minimal locking)
→ Inconsistent lock ordering
```

### How to Avoid Deadlocks

**1. Always acquire locks in a consistent order**
- Establish a global ordering of resources
- Train developers to follow the ordering

**2. Minimize transaction scope**
- Keep transactions short and focused
- Release locks quickly

**3. Use lower isolation levels when possible**
- Read Committed instead of Serializable
- Reduces lock contention

**4. Use optimistic concurrency control**
- No locks held during transaction execution
- Impossible to deadlock (no locks to wait for)

**5. Implement timeout mechanisms**
- If a transaction waits longer than X seconds, abort and retry
- Prevents indefinite waiting

**6. Monitor and log deadlock events**
- Identify problematic access patterns
- Optimize queries and business logic accordingly

**7. Use connection pooling effectively**
- Limit number of concurrent connections
- Reduce resource contention at system level

### Performance Impact of Deadlocks

Even with deadlock detection and resolution, deadlocks have significant costs:

**Direct Costs:**
- Victim transaction work is completely wasted
- Resources consumed during victim's execution are forfeited
- Application must detect failure and retry

**Indirect Costs:**
- Transaction latency increases (waiting for locks)
- System throughput decreases
- User experience degradation (timeouts, errors)
- Database logs fill up with deadlock events

**Example Calculation:**
- If a transaction takes 10 seconds to execute
- Deadlock victim loses all 10 seconds of work
- Application retries after 1-2 second delay
- 15 second delay for user waiting for operation
- Repeated deadlocks multiply this impact

---

## Summary: Choosing the Right Approach

### Decision Framework

**For Financial Systems:**
- Use Pessimistic Concurrency + Serializable isolation
- Deadlock risk acceptable for correctness guarantee
- Example: Bank transfers, stock trading

**For Web Applications:**
- Use Optimistic Concurrency + Read Committed isolation
- Deadlock risk eliminated (no locks)
- Retries acceptable for scalability
- Example: Social media, collaborative tools

**For Inventory/Warehouse Systems:**
- Use Pessimistic Concurrency + Repeatable Read isolation
- Balance between consistency and concurrency
- Deadlock prevention through resource ordering
- Example: Order fulfillment, stock management

**For Reporting/Analytics:**
- Use Read Uncommitted or Read Committed
- High concurrency priority over isolation
- No writes, so deadlock/anomaly risk minimal
- Example: Data warehouses, dashboards

---

## Key Takeaways

1. **Locking** is fundamental to preventing data corruption in concurrent environments
2. **Shared locks** enable multiple readers; **exclusive locks** prevent all access
3. **Transaction anomalies** (dirty reads, non-repeatable reads, phantom reads) are real risks without proper isolation
4. **Isolation levels** provide increasing protection at the cost of concurrency
5. **Pessimistic control** prevents conflicts upfront; **optimistic control** detects them at commit
6. **Deadlocks** are an inherent risk of pessimistic locking but preventable with proper design
7. The right approach depends on your specific workload, consistency requirements, and expected contention levels
