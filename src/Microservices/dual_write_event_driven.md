# System Design Notes: Dual Write Problem & Event-Driven Microservices

## Table of Contents
1. [The Dual Write Problem](#the-dual-write-problem)
2. [Why It Matters](#why-it-matters)
3. [Solutions to the Dual Write Problem](#solutions-to-the-dual-write-problem)
4. [Event-Driven Microservices Pattern](#event-driven-microservices-pattern)
5. [Combining Patterns](#combining-patterns)
6. [Trade-offs & Interview Tips](#trade-offs--interview-tips)

---

## The Dual Write Problem

### What Is It?

The dual write problem occurs when a service needs to **write data to two separate systems atomically** without being able to use a traditional distributed transaction.

Think of it like this: You're ordering pizza from two different pizzerias, and you pay one first, then the other. But what if the second pizzeria closes before you pay them? You've already paid the first, but now you can't complete the order. Your system is **inconsistent**.

### Classic Example: Payment Service

Imagine a payment service that processes a money transfer:

**What needs to happen:**
1. Update the database to record the transaction
2. Publish a `TransferCompleted` event to a message broker (like Kafka) so other services know about it

**The Problem:**
- If the database update succeeds but publishing the event fails → Your database shows the transfer, but other services don't know about it
- If the event publishes successfully but the database fails → The event exists, but there's no record in your primary database
- If your application crashes between these two operations → You're stuck in an inconsistent state

### Why Traditional Transactions Don't Work

In monolithic systems, we could use **2-Phase Commit (2PC)** to ensure atomicity across multiple resources. But 2PC has serious problems in distributed systems:

- **Performance overhead**: Blocks resources while waiting for agreement
- **Lack of support**: Modern message brokers like Kafka don't support 2PC
- **Availability risk**: If any participant fails, the entire transaction blocks
- **Scalability issues**: Doesn't work well across geographic regions

So in microservices, we can't rely on 2PC. We need a different approach.

### When Does It Happen?

The dual write problem appears whenever you need to **update two or more independent systems** as part of a single business operation:

- Database + Message Broker (most common)
- Database + Cache + Message Broker
- Database + Search Index
- Database + External Service
- Database + Authorization System

---

## Why It Matters

### Real-World Impact

**Silent Inconsistencies**: Unlike a dramatic system crash, data inconsistencies can hide. A service has updated data, but downstream services are unaware. This leads to:

- Incorrect business logic execution
- Lost transactions
- Audit trail gaps
- Cascading failures

**Example: E-Commerce Order**
1. Order Service creates an order and sends `OrderCreated` event
2. Inventory Service listens and decrements stock
3. Event is lost → Inventory never decremented → Overselling happens

**Production Nightmares**: These bugs are notoriously hard to detect and debug because they're intermittent—they only occur when failures happen at specific moments.

---

## Solutions to the Dual Write Problem

### Solution 1: Transactional Outbox Pattern

#### How It Works

Instead of writing to the database and message broker separately, write both to the **same database**:

1. Create a special `outbox` table in your database
2. When processing a business operation, write the business data **and** the event to the outbox table in a **single database transaction**
3. Because both writes are in the same transaction, they're atomic—either both succeed or both fail
4. A separate background process reads the outbox table and publishes events to the message broker
5. Once published successfully, mark the event as processed (or delete it)

#### Step-by-Step Flow

```
Client Request
    ↓
Service receives command (e.g., "transfer money")
    ↓
Start Database Transaction
    ├─ Execute business logic
    ├─ Update accounts table (money transferred)
    ├─ Write event to outbox table (TransferCompleted event)
    ↓
Commit Transaction (atomic—both writes succeed together)
    ↓
Outbox Relay Process (separate, independent)
    ├─ Poll outbox table for unprocessed events
    ├─ Publish to Kafka
    ├─ Mark as processed/delete
    ↓
Event reaches Kafka
    ↓
Other services consume and process
```

#### Key Characteristics

**Pros:**
- **Strong consistency guarantee**: Event is written if and only if the database transaction commits
- **Simple to understand**: No complex coordination needed
- **Atomic within boundaries**: Single database ensures atomicity for both operations
- **Works with any message broker**: The publishing process is independent

**Cons:**
- **Additional database writes**: Every operation includes an outbox write (small overhead)
- **Requires polling or CDC**: The outbox relay process must continuously check the outbox table
- **Potential duplicates**: If the relay process fails after publishing but before marking processed, you'll resend the same event
- **Cleanup complexity**: You need to manage when to delete from outbox (performance vs. audit trail)

#### Interview Talking Points

"With the Transactional Outbox, we leverage the database's ACID properties to solve the dual write problem. We can't span a transaction across database and message broker, but we can keep both writes within the database."

### Solution 2: Event Sourcing

#### How It Works

Instead of storing just the **current state** of an entity, store the **entire history of events** that led to that state.

**Traditional approach:** Update the database directly
```
Initial: Account balance = $1000
Update: Transfer $100
Result: Account balance = $900
```

**Event Sourcing approach:** Record the event, not the result
```
Account Created → Balance = $0
Money Deposited → +$1000
Money Transferred → -$100
Current State (derived from all events): Balance = $900
```

#### How It Solves Dual Write

With Event Sourcing, the problem naturally dissolves:

1. The **source of truth is the event stream** (usually stored in a database or event store)
2. Write the event to the event store → This IS the database update
3. Publish that same event to the message broker
4. If publishing to the broker fails, downstream services replay the event stream when they catch up
5. The event store is immutable and complete

#### Step-by-Step Flow

```
Client Command
    ↓
Validate Command
    ↓
Generate Events from command
    ↓
Append Events to Event Store (single write)
    ↓
Derive Current State from Event History
    ↓
Publish Events to Message Broker
    ├─ If fails: No problem—events are already in the event store
    ├─ Consumer services will catch up by reading event history
    ↓
Update Read Models (projections)
```

#### Key Characteristics

**Pros:**
- **Complete audit trail**: Every change is recorded forever
- **Time travel debugging**: Replay events to see system state at any point
- **Natural event publishing**: Events are primary, not secondary
- **Inherent idempotency support**: Events can be replayed safely
- **Temporal queries**: "What was the balance on this date?" → Replay up to that date

**Cons:**
- **Significant architectural shift**: Requires rethinking how you store and query data
- **Complexity in implementation**: Event upcasting, versioning, projections
- **Storage overhead**: Storing every event can use substantial disk space
- **Query complexity**: Can't query directly; need to build projections (read models)
- **Learning curve**: Team needs to understand event-driven thinking

#### Interview Talking Points

"Event Sourcing treats events as the primary source of truth. The current state is just a projection of all historical events. This naturally aligns with event-driven systems and solves dual write by making events the canonical record."

### Solution 3: Change Data Capture (CDC)

#### How It Works

Use the database's **transaction log** to automatically detect and capture all changes, then publish them as events to downstream systems.

Most databases maintain internal transaction logs for crash recovery and replication. CDC leverages these:

1. A CDC tool (like Debezium) monitors the database transaction log
2. Whenever data changes (insert, update, delete), CDC detects it
3. CDC publishes this change as an event to a message broker
4. No application code changes needed

#### How It Solves Dual Write

1. You only write to the database (no dual write anymore)
2. CDC automatically captures that write from the log
3. CDC reliably publishes it to the message broker
4. If CDC publishing fails, it retries (it has the full log history)

#### Step-by-Step Flow

```
Client Request
    ↓
Service writes to database only
    ↓
Database transaction commits
    ↓
CDC Monitor (separate process)
    ├─ Reads database transaction log
    ├─ Detects change
    ├─ Publishes to Kafka
    ↓
Event reaches downstream services
```

#### Key Characteristics

**Pros:**
- **No application code changes**: Existing services don't need modification
- **Non-intrusive**: Works with legacy systems
- **Automatic**: CDC handles everything independently
- **At-least-once guarantee**: Won't lose events (full log history retained)

**Cons:**
- **Lost context**: The event contains only database data, not business intent
- **Tool dependency**: Need external CDC tool (adds operational complexity)
- **Limited to database changes**: Can't know *why* the change happened
- **Performance overhead**: Database transaction log parsing can be resource-intensive
- **Latency**: CDC lag means downstream services aren't immediately aware

#### Interview Talking Points

"CDC is non-intrusive but loses business context. The event only contains 'customer name changed' but not 'why.' The Outbox Pattern is better when you need rich, contextual events."

---

## Event-Driven Microservices Pattern

### Core Concept

Event-driven microservices is an architectural style where:
- Services **communicate asynchronously** through events
- Services are **loosely coupled** (don't know about each other)
- Events are the **primary communication mechanism**
- Each service owns its data and publishes domain events

### Key Components

#### 1. Event Producers
Services that emit events when something important happens.

Example: `OrderService` produces `OrderCreated`, `OrderCancelled`, `OrderShipped` events

#### 2. Event Router/Message Broker
Central hub that receives events and routes them to interested consumers.

Examples: Apache Kafka, RabbitMQ, AWS EventBridge, Azure Event Hub

**Key property**: Acts as an **elastic buffer** between producers and consumers, decoupling them in time and space.

#### 3. Event Consumers
Services that listen for events and react to them.

Example: `NotificationService` listens for `OrderCreated` and sends email confirmation

#### 4. Event Schema
Definition of event structure and content.

```
Event: UserRegistered
{
  user_id: UUID
  email: string
  registration_timestamp: datetime
  source_service: string
}
```

### Communication Model: Pub/Sub vs. Event Streaming

#### Publish-Subscribe (Pub/Sub)
- Events are published to a topic
- Subscribers must be subscribed **before** the event is published
- Event is typically transient—once delivered, it's gone
- New subscribers miss historical events
- **Use when**: You only care about current events, not history

#### Event Streaming
- Events are stored in a **durable log** (with ordering guarantees)
- Consumers read from the stream at their own pace
- Historical events can be replayed
- New consumers can catch up from the beginning
- **Use when**: You need event history, replaying, auditing

### Example: E-Commerce System with Event-Driven Architecture

#### Scenario: Customer Places Order

```
Customer places order
    ↓
OrderService (Producer)
├─ Creates order in database
├─ Publishes: OrderCreated event
    ↓
Message Broker receives event
    ├─ Routes to InventoryService
    ├─ Routes to PaymentService
    ├─ Routes to NotificationService
    ├─ Routes to AnalyticsService
    ↓
InventoryService (Consumer)
├─ Listens for OrderCreated
├─ Decrements stock
├─ Publishes: StockReserved event
    ↓
PaymentService (Consumer)
├─ Listens for OrderCreated
├─ Charges payment
├─ Publishes: PaymentProcessed event
    ↓
NotificationService (Consumer)
├─ Listens for StockReserved + PaymentProcessed
├─ Sends order confirmation email
    ↓
AnalyticsService (Consumer)
├─ Listens for all events
├─ Updates dashboards and reports
```

### Benefits of Event-Driven Architecture

#### 1. Loose Coupling
- Services don't know about each other
- New services can be added as consumers without modifying existing services
- Changes to one service don't affect others

**Example**: Add a `FraudDetectionService` without touching `OrderService`

#### 2. Scalability
- Services scale independently based on their load
- The message broker acts as an elastic buffer
- High-volume producers don't block slow consumers

**Example**: If `AnalyticsService` is slow, it doesn't slow down `OrderService`

#### 3. Asynchronous Processing
- Services don't wait for responses
- Operations are non-blocking
- System processes multiple operations concurrently

**Example**: Order placement completes immediately, payments/inventory process in parallel

#### 4. Resilience
- Failure in one service doesn't cascade to others
- If `NotificationService` crashes, orders still process
- Message broker buffers events until consumer recovers

**Example**: Power outage at `PaymentService` → Orders queue up → Processing resumes when service recovers

#### 5. Real-Time Insights
- Events flow continuously → Opportunities for real-time analytics
- Can detect fraud, anomalies, patterns as they happen
- Data is more current than batch-processing systems

#### 6. Natural Audit Trail
- Events represent the complete history of what happened
- Useful for compliance, debugging, and investigations

---

## Combining Patterns

### When to Use Each

#### Use Transactional Outbox When:
- You want **fine-grained control** over which events to publish
- Events need **rich business context** (not just database data)
- You need **strong consistency** within your service boundary
- Working with **new or refactored systems**
- **Example**: Banking service needs to publish `TransactionApproved` with business rules applied

#### Use Event Sourcing When:
- **Audit trails are critical** (healthcare, finance, legal)
- You need **temporal queries** (what was the state on date X?)
- You want **natural support for event-driven architecture**
- You can afford the architectural shift
- **Example**: Trading system where every price change, order, execution must be traceable

#### Use CDC When:
- Working with **legacy systems** you can't modify
- Multiple **independent applications** write to the same database
- You want **non-intrusive** event extraction
- **Example**: Migrating from monolith to microservices without rewriting existing code

### Hybrid Approach

**Best practice**: Combine based on your needs

```
New Critical Systems
├─ Use Event Sourcing + Event-Driven
├─ Rich events with full audit
├─ Complete control

Legacy Systems
├─ Use CDC
├─ No code changes required
├─ Automatic event extraction

Mixed Approach
├─ Critical operations: Outbox Pattern (in new services)
├─ Supporting data: CDC (from legacy systems)
├─ Unified event stream: Both feed into Kafka
```

---

## Trade-offs & Interview Tips

### Consistency vs. Complexity

**Synchronous/Strong Consistency:**
- Immediate feedback (user waits for response)
- Simple to reason about
- Blocking, slow, cascading failures

**Asynchronous/Eventual Consistency:**
- Non-blocking, fast, resilient
- Complex to reason about (what if consumer never processes the event?)
- Need idempotency, deduplication, reconciliation

### Key Decisions

#### 1. How Quickly Do Consumers Need Updates?

| Requirement | Approach |
|---|---|
| Immediate (< 1 second) | Synchronous API calls, consider trade-offs |
| Near real-time (seconds) | Event streaming with fast brokers |
| Eventually consistent (minutes) | Event streaming is fine |

#### 2. Do You Need Event History?

| Requirement | Approach |
|---|---|
| Full audit trail | Event Sourcing |
| Recent events only | Outbox + Kafka with retention |
| No history needed | Simple outbox with cleanup |

#### 3. How Much Can You Modify Existing Systems?

| Situation | Approach |
|---|---|
| Greenfield (new system) | Event Sourcing + Outbox |
| Legacy monolith (can't change) | CDC |
| Some services new, some old | Hybrid: Outbox for new, CDC for legacy |

### Common Interview Questions

**Q: "How would you handle duplicate events?"**
A: Make consumers **idempotent**. Track processed event IDs, check for duplicates before processing. If event is already processed, skip silently. Use database unique constraints or deduplication windows.

**Q: "What if a service crashes after publishing an event but before updating internal state?"**
A: With Outbox: Event is already in outbox table, will be retried. With CDC: Change already in log. With Event Sourcing: Event already persisted. The key is: write first to your system of truth, then eventually publish.

**Q: "How do you handle out-of-order events?"**
A: 
- Use **event versioning** and **sequence numbers**
- **Reorder at consumer**: Buffer events, process in order
- **Idempotency**: Even if processed out of order, final state is correct
- **Example**: Stock price events—processing $10 then $8 is wrong, but if idempotent, retrying in correct order fixes it

**Q: "How do you know if an event was successfully processed by all consumers?"**
A: You don't, and that's okay (eventual consistency). You can:
- Track consumer lag in message broker
- Implement distributed tracing
- Build reconciliation jobs to detect inconsistencies
- Monitor dead-letter queues

**Q: "Isn't the Outbox just moving the dual-write problem to the relay process?"**
A: No, because:
- Event is already atomically committed in database
- Relay process can retry indefinitely—it has source of truth (outbox table)
- Unlike original dual-write, relay can recover from failures without data loss

**Q: "How do you handle versioning of events?"**
A: 
- Add version field to events
- New consumers can handle old event versions
- Use adapter pattern: convert old events to new format
- **Example**: 
  ```
  UserCreated (v1): { user_id, name, email }
  UserCreated (v2): { user_id, name, email, country, phone }
  
  New consumer accepts both versions and fills in defaults
  ```

**Q: "Should services ever make synchronous calls to other services?"**
A: Generally avoid, but exceptions exist:
- **Read operations**: Service A needs data from Service B → Sync call (no dual-write risk)
- **Low-latency requirements**: Need response immediately
- **Tight coupling acceptable**: Within a bounded context

Better: Use event-driven for writes, sync for reads.

---

## Decision Tree: Which Pattern Should I Use?

```
Do you need to write to database + message broker atomically?
    ├─ YES
    │   ├─ Can you modify application code?
    │   │   ├─ YES
    │   │   │   ├─ Do you need full audit trail?
    │   │   │   │   ├─ YES → Event Sourcing
    │   │   │   │   └─ NO → Transactional Outbox
    │   │   │
    │   │   └─ NO → Change Data Capture (CDC)
    │
    └─ NO → Use event-driven architecture with outbox/sourcing for inter-service communication
```

---

## Quick Recap

| Pattern | Problem Solved | Best For | Trade-offs |
|---|---|---|---|
| **Transactional Outbox** | Dual write (DB + message broker) | New services, strong consistency needs | Extra DB writes, relay process complexity |
| **Event Sourcing** | Dual write + audit trail + temporal queries | Finance, healthcare, full audit needed | Architectural shift, storage overhead, query complexity |
| **CDC** | Dual write in legacy systems | Legacy monoliths, non-intrusive | Loss of business context, external tool dependency |
| **Event-Driven Architecture** | Service coupling, scalability, resilience | Microservices in general | Eventual consistency, complex debugging |

---

## Interview Reminders

✅ **Always ask clarifying questions**: "How critical is consistency?" "What's the scale?" "Are these legacy or new systems?"

✅ **Mention trade-offs explicitly**: Don't just say "use Event Sourcing." Say "Event Sourcing gives us audit trail and temporal queries, but requires architectural changes and adds storage overhead."

✅ **Use real examples**: "In an e-commerce system, we'd use Outbox Pattern for critical operations like payments, and CDC for legacy inventory system."

✅ **Discuss operational concerns**: How do you monitor? Handle dead-letter queues? Reconcile inconsistencies? Show you've thought about production.

✅ **Know when to KISS (Keep It Simple)**: "If your system doesn't span multiple services yet, don't add event-driven complexity. Start simple, evolve when needed."
