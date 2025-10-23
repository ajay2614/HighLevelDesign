# Two Phase Locking (2PL) - Interview Notes

## What is Two Phase Locking?

Two Phase Locking (2PL) is a **pessimistic concurrency control protocol** used in database systems to ensure **conflict-serializability** of transactions. It guarantees that concurrent transactions execute in a way that produces the same result as if they were executed one after another.

### Key Concept
2PL divides transaction execution into **two distinct phases**:

1. **Growing Phase (Expanding Phase)**: Transaction can **acquire locks** but **cannot release** any locks
2. **Shrinking Phase**: Transaction can **release locks** but **cannot acquire** any new locks

### Lock Types
- **Shared Lock (S-Lock)**: Allows reading data, multiple transactions can hold shared locks on same data
- **Exclusive Lock (X-Lock)**: Allows reading and writing, only one transaction can hold exclusive lock

### Lock Compatibility
| Lock Type | Shared Lock | Exclusive Lock |
|-----------|-------------|----------------|
| Shared Lock | ✓ Compatible | ✗ Incompatible |
| Exclusive Lock | ✗ Incompatible | ✗ Incompatible |

---

## 1. Basic Two Phase Locking (Basic 2PL)

### How it Works
- Transaction acquires all needed locks during growing phase
- Once first lock is released, transaction enters shrinking phase
- No new locks can be acquired after any lock is released

### Example
```
Transaction T1:
- Lock-X(A)     // Growing phase starts
- Lock-S(B)     // Still growing phase
- Read(A), Write(A)
- Unlock(A)     // Shrinking phase starts
- Read(B)
- Unlock(B)     // Must release all remaining locks
```

### **Problems with Basic 2PL**

#### 1. Cascading Aborts (Cascading Rollbacks)
- **What it is**: When one transaction reads uncommitted data from another transaction, and the first transaction aborts, the second must also abort
- **Example**: 
  - T1 writes to A, releases lock on A
  - T2 reads A (dirty read)
  - T1 aborts - now T2 must also abort because it read invalid data
  - **Chain reaction**: If T3 read from T2, T3 must also abort

#### 2. Dirty Reads
- Transaction can read uncommitted (dirty) data from other transactions
- Leads to inconsistent results if the writing transaction aborts

#### 3. Deadlocks
- Two or more transactions wait for each other to release locks
- **Example**:
  - T1 locks A, T2 locks B
  - T1 requests lock on B (waits for T2)
  - T2 requests lock on A (waits for T1)
  - **Deadlock!** Neither can proceed

### **When to Use Basic 2PL**
- **Good for**: Simple systems where occasional cascading aborts are acceptable
- **Bad for**: Systems requiring high reliability and minimal wasted work

---

## 2. Strict Two Phase Locking (Strict 2PL)

### How it Works
- Follows basic 2PL rules PLUS
- **All exclusive (X) locks are held until transaction commits or aborts**
- Shared locks can be released during shrinking phase
- **No dirty reads possible**

### Example
```
Transaction T1:
- Lock-X(A), Lock-S(B)     // Growing phase
- Read(A), Write(A)
- Read(B)
- Unlock(B)                // Can release shared locks
- Commit                   // Exclusive locks released only here
- Unlock(A)
```

### **Advantages**
1. **Prevents cascading aborts** - no transaction reads uncommitted data
2. **Ensures recoverable schedules** - if T1 reads from T2, T2 commits before T1
3. **Avoids dirty reads**

### **Disadvantages**
1. **Reduced concurrency** - transactions hold exclusive locks longer
2. **Still prone to deadlocks**
3. **Longer wait times** for transactions needing locked resources

### **When to Use Strict 2PL**
- **Good for**: Systems where data consistency is critical and some performance trade-off is acceptable
- **Bad for**: High-throughput systems where maximum concurrency is needed

---

## 3. Strong Strict 2PL (Rigorous 2PL)

### How it Works
- Most restrictive form of 2PL
- **ALL locks (both shared and exclusive) are held until transaction commits or aborts**
- No locks released during shrinking phase
- Also called **Rigorous 2PL**

### Example
```
Transaction T1:
- Lock-X(A), Lock-S(B)     // Growing phase
- Read(A), Write(A)
- Read(B)
- Commit                   // ALL locks released only here
- Unlock(A), Unlock(B)
```

### **Advantages**
1. **Prevents cascading aborts completely**
2. **Ensures strict schedules** - no transaction reads from uncommitted transactions
3. **Guarantees recoverability**
4. **Easier to implement** than other variants
5. **Most commonly used in practice**

### **Disadvantages**
1. **Lowest concurrency** among all 2PL variants
2. **Highest contention** for resources
3. **Still susceptible to deadlocks**
4. **Longest wait times**

### **When to Use Strong Strict 2PL**
- **Good for**: Banking systems, financial transactions, any system where absolute consistency is required
- **Bad for**: Read-heavy workloads, systems requiring maximum throughput

---

## 4. Conservative Two Phase Locking (Static 2PL)

### How it Works
- **Pre-declares all required locks** before transaction begins execution
- **Acquires ALL needed locks at once** or waits until all are available
- **No growing phase** - all locks acquired upfront
- **Only has shrinking phase** - releases locks as needed

### Example
```
Transaction T1:
- Declare: Need X-Lock on A, S-Lock on B    // Pre-declaration
- Acquire all locks: Lock-X(A), Lock-S(B)   // All at once or wait
- Read(A), Write(A)
- Unlock(A)                                  // Can release anytime
- Read(B)
- Unlock(B)
```

### **Advantages**
1. **Deadlock-free** - no circular wait possible since all locks acquired upfront
2. **Maximum concurrency potential** - can release locks early when done
3. **No cascading aborts** if implemented carefully

### **Disadvantages**
1. **Difficult to implement** - requires knowing all data needs in advance
2. **Reduced concurrency** - may hold locks longer than needed
3. **Not practical** for complex transactions
4. **Doesn't guarantee recoverability by itself**
5. **Potential for starvation** if high-demand resources are always locked

### **When to Use Conservative 2PL**
- **Good for**: Simple, predictable transactions where data access patterns are known
- **Bad for**: Complex applications, dynamic queries, most real-world scenarios

---

## Comparison Table

| Feature | Basic 2PL | Strict 2PL | Strong Strict 2PL | Conservative 2PL |
|---------|-----------|------------|-------------------|------------------|
| **Cascading Aborts** | ❌ Possible | ✅ Prevented | ✅ Prevented | ⚠️ Depends on implementation |
| **Dirty Reads** | ❌ Possible | ✅ Prevented | ✅ Prevented | ⚠️ Possible |
| **Deadlocks** | ❌ Possible | ❌ Possible | ❌ Possible | ✅ Prevented |
| **Concurrency** | High | Medium | Low | Medium |
| **Implementation** | Easy | Medium | Easy | Hard |
| **Practical Use** | Rare | Common | Most Common | Rare |

---

## Major Issues with 2PL and Solutions

### Issue 1: Deadlocks

#### What Causes Deadlocks?
- **Circular waiting**: T1 waits for T2, T2 waits for T1
- **Resource competition**: Multiple transactions need same resources in different orders

#### **Deadlock Prevention Strategies**

##### 1. Wait-Die Scheme (Non-preemptive)
- **Rule**: Older transaction waits, younger transaction dies
- **Example**: If T1(timestamp=5) wants resource held by T2(timestamp=10):
  - T1 is older → T1 waits for T2
  - If T2 wanted T1's resource → T2 would abort and restart

##### 2. Wound-Wait Scheme (Preemptive)  
- **Rule**: Older transaction wounds (aborts) younger, younger waits for older
- **Example**: If T1(timestamp=5) wants resource held by T2(timestamp=10):
  - T1 is older → T2 is aborted (wounded), T1 gets resource
  - If T2 wanted T1's resource → T2 waits for T1

#### **Deadlock Detection**
- **Wait-for graph**: Build graph of transactions waiting for each other
- **Cycle detection**: If cycle exists, deadlock detected
- **Resolution**: Abort one transaction in the cycle

#### **Which is Better?**
- **Prevention**: Proactive but may abort transactions unnecessarily
- **Detection**: Reactive, only aborts when deadlock actually occurs
- **Most systems use detection** due to lower unnecessary aborts

### Issue 2: Cascading Aborts

#### What Causes Cascading Aborts?
- Transaction reads uncommitted data from another transaction
- First transaction aborts, causing chain reaction of aborts

#### **Solutions**
1. **Use Strict 2PL or Strong Strict 2PL** - prevents reading uncommitted data
2. **Maintain dependency graphs** - track which transactions read from which
3. **Careful abort handling** - ensure all dependent transactions are properly aborted

### Issue 3: Reduced Concurrency

#### What Causes Reduced Concurrency?
- Long lock holding periods
- Conservative lock acquisition
- Blocking due to incompatible locks

#### **Solutions**
1. **Lock granularity optimization** - use finer-grained locks when possible
2. **Lock escalation** - start with fine locks, escalate to coarse when needed
3. **Consider alternative protocols** - like timestamp ordering or optimistic concurrency control

---

## Interview Tips

### **Key Points to Remember**
1. **2PL guarantees conflict-serializability** but may not guarantee view-serializability
2. **Strict variants prevent cascading aborts** by avoiding dirty reads
3. **Conservative 2PL is the only variant that prevents deadlocks**
4. **Trade-off exists**: More restrictive = fewer problems but less concurrency
5. **Most production systems use Strong Strict 2PL** (Rigorous 2PL)

### **Common Interview Questions**
1. **"Why is 2PL important?"** - Ensures serializable execution of concurrent transactions
2. **"What's the difference between Strict and Strong Strict 2PL?"** - Strong Strict holds ALL locks until commit, Strict only holds exclusive locks
3. **"How do you handle deadlocks in 2PL?"** - Prevention (Wait-Die/Wound-Wait) or Detection (Wait-for graph)
4. **"What are cascading aborts and how to prevent them?"** - Use Strict or Strong Strict 2PL to prevent dirty reads

### **Real-world Examples**
- **Banking**: Strong Strict 2PL for account transfers (absolute consistency required)
- **Inventory**: Strict 2PL for stock updates (prevent overselling)
- **Reporting**: Basic 2PL might be acceptable (if occasional inconsistency is tolerable)

Remember: The choice of 2PL variant depends on the specific requirements of your system - consistency needs, performance requirements, and tolerance for complexity.