# Distributed Transactions – System Design Interview Notes

---

## Introduction

**Distributed transactions** refer to transactions that span multiple autonomous systems or databases, synchronizing updates so that either all succeed or all fail. This is harder than standard transactions because each system may have its own rules, databases, failures, and communication delays.

**Why are distributed transactions challenging?** Achieving ACID (Atomicity, Consistency, Isolation, Durability) across networks, services, or data stores can be tough due to issues like network latency, partial failures, and different consistency requirements.

Key distributed transaction patterns:
- **2 Phase Commit (2PC):** Widely used but has blocking issues.
- **3 Phase Commit (3PC):** Designed to solve 2PC’s blocking problem, but also brings its own trade-offs.
- **Saga:** Event-driven and non-blocking, more suitable for microservices.

---

## Two-Phase Commit (2PC)

### Overview:
The 2PC protocol ensures all nodes in a distributed system either *commit* or *roll back* a transaction. One node acts as the **Coordinator**; other nodes are **Participants**.

### Protocol Stages:
1. **Prepare Phase:**
    - The Coordinator sends a “prepare” request to all Participants.
    - Each Participant executes its local part & responds “yes” (ready) or “no” (abort).
    - Each Participant locks resources and writes its outcome to its log.

2. **Commit/Abort Phase:**
    - If all say “yes,” the Coordinator sends “commit” to all. Otherwise, “abort.”
    - All Participants follow the Coordinator’s decision and unlock resources.

### Example:
**Bank Transfer:** Moving money from Account A (Database X) to Account B (Database Y). Both databases are updated only if both agree to commit.

### Advantages:
- Strong consistency (all-or-nothing)
- Simplicity in architecture

### Disadvantages:
- **Blocking Problem:** If Coordinator fails after the Prepare phase, Participants can’t decide and are forced to wait (may block indefinitely).
- Scalability: Each node must lock resources, reducing throughput.
- Single point of failure: If the Coordinator is down, no one can proceed.

### Failure scenarios:
- **Participant failure before voting:** Coordinator treats as "no"
- **Participant failure after voting:** Waits for Coordinator’s outcome
- **Coordinator failure after Prepare, before Commit:** Participants remain blocked until Coordinator recovers.

### When to use
- Small distributed transactions with strict consistency needs (traditional databases, banking)

---

## Three-Phase Commit (3PC)

### Motivation:
3PC improves upon 2PC by reducing blocking (the risk of indefinite waits if the Coordinator fails).

### Protocol Stages:
1. **CanCommit Phase:** Coordinator asks Participants if they are ready to commit.
2. **PreCommit Phase:** If all say “yes,” Coordinator instructs all to prepare (“pre-commit”) and Participants log this state.
3. **DoCommit Phase:** Coordinator instructs Commit. If a crash happens after PreCommit, Participants can use timeouts and logs to decide to commit or abort autonomously.

### Example:
**Multi-service Booking:** Travelers book flights, hotels, and cars. All services confirm readiness, log pre-commit, then finalize booking only if everyone is prepared.

### Advantages:
- Non-blocking under fail-stop model; less risk of indefinite locks
- Better fault tolerance than 2PC

### Disadvantages:
- Not partition-tolerant: Network partitions can still cause blocking
- Requires 3 rounds of messaging; higher latency
- Not widely adopted due to complexity

### When to use
- Distributed systems needing stronger fault tolerance, but synchronous and fail-stop (no network partitions).

---

## Saga Pattern

### What is Saga?
A set of local, independent transactions chained together. If any transaction fails, compensating transactions undo/prevent inconsistency.

### How it works
- Each step is a local transaction in a service.
- If all transactions succeed, the business workflow completes.
- If a transaction fails, *compensating transactions* are triggered to roll back previous actions (not a true rollback–instead, apply an opposite action actively).

### Saga Variants
**Choreography:** Each service emits events, and other services listen/act accordingly (“event-driven chain”–no central controller).
  - Example: Order Service emits OrderPlaced → Inventory Service reserves stock → Payment Service charges customer.
**Orchestration:** Central Saga Orchestrator sends commands to services, maintains sequence and compensations (“central controller”).
  - Example: Orchestrator triggers steps: Place Order, Reserve Inventory, Charge Payment, Ship Order.

### Comparison: Choreography vs Orchestration
| Criterion         | Choreography           | Orchestration           |
|------------------|-----------------------|------------------------|
| Control Flow     | Decentralized          | Centralized            |
| Debug/Journey    | Harder to trace        | Easier to monitor      |
| Complexity       | Event chain complexity | Central logic          |
| Scalability      | High                   | Moderate               |
| Failure Handling | Each service handles   | Orchestrator handles   |

### Example:
**E-commerce Order:** Placing an order involves Inventory, Payment, Shipping. If Payment fails, Saga triggers compensate actions – cancels reservation, refunds payment.

### Advantages:
- Lower latency, no blocking
- Eventual consistency, decoupled services
- Fault tolerance, more scalable

### Disadvantages:
- Doesn’t provide strict ACID guarantees
- Compensating logic can be complex
- Data anomalies may happen (temporarily inconsistent)

### When to use
- Long-running workflows (e.g., order fulfillment)
- Microservices architectures
- Systems valuing availability and scalability over strict consistency

---

## Comparison & Trade-offs
| Feature                | Two-Phase Commit (2PC) | Three-Phase Commit (3PC) | Saga Pattern             |
|-----------------------|------------------------|--------------------------|--------------------------|
| Consistency           | Strict                 | Strict                   | Eventual (with compensation) |
| Scalability           | Low                    | Low/Medium               | High                     |
| Availability          | Low (blocking)         | Medium (non-blocking)    | High (no global locks)   |
| Fault Tolerance       | Weak                   | Medium                   | Strong (recoverable by compensations) |
| Use Case Examples     | Databases, Financial   | Fault-tolerant DBs       | E-commerce, Microservices |
| Failure Handling      | Global rollback        | Better automatic aborts  | Compensating transactions |
| Latency               | High                   | Higher                   | Low                      |
| Implementation        | Simple                 | Complex                  | Medium-Complex           |

---

## Summary
- 2PC: Use for small, tightly coupled systems needing strict consistency. Beware blocking.
- 3PC: Consider if you want improved fault-tolerance and can manage complexity. Not partition tolerant.
- Saga: Recommended for microservices, cloud-native apps, workflows with eventual consistency. Compensating transactions must be well designed.

**Example scenarios**
- **Bank Transfer:** 2PC (both accounts must commit together)
- **E-commerce Order:** Saga (steps across inventory, payment, shipment; each can undo if something fails)
- **Travel Booking:** 3PC may be viable if strict consistency, multiple confirmations, and coordinator reliability are needed.

---
