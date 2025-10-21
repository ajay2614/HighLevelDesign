# Active-Passive vs Active-Active Architecture: Interview Notes

## How This Topic Can Be Asked in Interviews

This architectural pattern is frequently tested in system design interviews through various question formulations:

- **"Design a highly available architecture for [application/system]"**
- **"Design a data resiliency architecture that can withstand datacenter failures"**
- **"How would you design a system to achieve 99.9% or 99.99% availability?"**
- **"Design an architecture to avoid single point of failure (SPOF)"**
- **"How would you implement disaster recovery for a critical application?"**
- **"Design a system that can handle regional outages with minimal downtime"**
- **"What strategies would you use to eliminate SPOFs in a database tier?"**

When you hear these questions, the interviewer is evaluating your understanding of high availability patterns, failover mechanisms, redundancy strategies, and your ability to make trade-offs between consistency, availability, and operational complexity.

---

## Understanding the Fundamentals

### What is High Availability?

High availability (HA) refers to systems designed to be operational and accessible for a maximum amount of time, minimizing downtime and ensuring business continuity. The goal is to eliminate single points of failure through redundancy and automated failover mechanisms.

**Key Metrics:**

- **Uptime Percentage**: 99.9% allows ~8.76 hours downtime/year; 99.99% allows ~52.56 minutes/year
- **RTO (Recovery Time Objective)**: Maximum tolerable duration of outage
- **RPO (Recovery Point Objective)**: Maximum tolerable data loss measured in time

### Single Point of Failure (SPOF)

A SPOF is any component whose failure causes the entire system to fail. Eliminating SPOFs is fundamental to high availability:

- **Database SPOF**: Single database instance without replicas
- **Application Server SPOF**: Single server handling all requests
- **Network SPOF**: Single network path or load balancer
- **Datacenter SPOF**: All resources in one geographic location

---

## Active-Passive Architecture

### Definition

Active-Passive (also called standby or failover architecture) involves a primary active system handling all traffic, while one or more secondary passive systems remain on standby, ready to take over if the primary fails.

### Key Components

**1. Primary (Active) Server**
- Handles all production traffic under normal conditions
- Processes all read and write operations
- Maintains authoritative state of the system

**2. Standby (Passive) Server(s)**
- Remains idle or processes only read operations
- Continuously synchronized with primary through replication
- Activated only during failover events

**3. Heartbeat Mechanism**
- Monitors health of active server through regular pings
- Detects failures like unresponsiveness, crashes, or network issues
- Triggers failover when failure threshold exceeded

**4. Data Replication**
- Keeps standby synchronized with active system
- Can be synchronous or asynchronous
- Ensures minimal data loss during failover

### Failover Process

1. **Failure Detection**: Heartbeat mechanism detects primary failure (typically 3-10 seconds)
2. **Automatic Failover Trigger**: System initiates failover procedure
3. **Traffic Redirection**: Requests redirected from failed primary to standby
4. **Standby Activation**: Passive server promoted to active role
5. **Recovery**: Failed server repaired and reintegrated as new standby

### Use Case Examples

**SQL Database (MySQL/PostgreSQL) - Master-Slave Replication**

```
Primary (Master) DB
    ↓ (replication)
Secondary (Slave) DB (passive, standby)

During normal operation:
- Master handles all writes
- Slave can handle read-only queries (read replica)
- Async/semi-sync replication keeps slave updated

On failure:
- Slave promoted to master (failover ~30-60 seconds)
- Applications redirected to new master
- Brief downtime during promotion
```

**Configuration characteristics:**
- Replication lag: 0-5 seconds (async) or 0 seconds (sync)
- Failover time: 30-60 seconds manual; 10-30 seconds automated
- Data loss risk: Low with semi-synchronous; Near-zero with synchronous

**Real-world example:**
- Azure SQL Database with failover groups uses active-passive with automatic failover
- Amazon RDS Multi-AZ deployments maintain synchronous standby in different availability zone

### Advantages

1. **Simplicity**: Easier to configure and manage than active-active
2. **Data Consistency**: Single write source prevents conflicts
3. **Predictable Failover**: Well-defined transition from active to passive
4. **Cost-Effective**: Passive resources don't require same capacity
5. **Clear Authority**: Always clear which node has latest data
6. **Suitable for Legacy Systems**: Works with systems not designed for distributed operation

### Disadvantages

1. **Resource Underutilization**: Passive servers sit idle most of time
2. **Downtime During Failover**: Brief service interruption (seconds to minutes)
3. **Limited Scalability**: Single active node limits capacity
4. **Manual Intervention**: May require human decision-making for failback
5. **Recovery Time**: RTO typically measured in seconds to minutes

### When to Use Active-Passive

- **Strong consistency requirements**: Financial transactions, banking systems
- **Regulatory compliance**: Systems requiring clear audit trail
- **Legacy applications**: Software not designed for distributed operation
- **Lower traffic volumes**: Single server can handle normal load
- **Budget constraints**: Lower infrastructure costs acceptable
- **Acceptable brief downtime**: RTO of 30-60 seconds is tolerable

---

## Active-Active Architecture

### Definition

Active-Active architecture involves multiple identical instances running simultaneously, all actively serving production traffic. All nodes share the workload, and if one fails, others continue processing without interruption.

### Key Components

**1. Multiple Active Nodes**
- All instances actively serve production traffic
- Each handles portion of overall workload
- No designated "primary" - all nodes equal

**2. Load Balancer**
- Distributes traffic across all active nodes
- Performs health checks on each node
- Removes failed nodes from rotation automatically
- Can be hardware or software (HAProxy, NGINX, AWS ELB)

**3. Data Synchronization**
- Multi-master replication or distributed consensus
- Conflict resolution mechanisms (CRDTs, Last-Write-Wins)
- Eventually consistent or strongly consistent depending on design

**4. Geographic Distribution**
- Nodes in multiple datacenters/regions
- Users routed to nearest node for low latency
- Survives entire datacenter failures

### Load Balancing and Traffic Distribution

**Health Check Mechanisms:**
- HTTP/HTTPS probes every 5-30 seconds
- TCP connection tests
- Application-level health endpoints
- Consecutive failure threshold (typically 2-3 failures)

**Distribution Strategies:**
- Round-robin: Equal distribution
- Least connections: Send to least loaded server
- Geographic routing: Route to nearest datacenter
- Weighted distribution: Based on capacity

### Use Case Examples

**Cassandra Multi-Datacenter Active-Active**

```
Datacenter 1 (US-East)          Datacenter 2 (EU-West)
   Node A ←------------------→ Node D
   Node B   ← async repl →     Node E
   Node C ←------------------→ Node F

Configuration:
- NetworkTopologyStrategy
- Replication Factor: {US-East: 3, EU-West: 3}
- Consistency Level: LOCAL_QUORUM

Write Operation:
- Write to US-East with LOCAL_QUORUM (2 of 3 nodes)
- Async replication to EU-West in background
- EU-West reads from local nodes (low latency)

On datacenter failure:
- US-East goes down: EU-West continues serving traffic
- No downtime, just reduced capacity
- When US-East returns, catches up automatically
```

**Key Cassandra concepts for active-active:**

- **NetworkTopologyStrategy**: Datacenter-aware replication
- **LOCAL_QUORUM**: Consistency within datacenter (RF/2 + 1)
- **EACH_QUORUM**: Strong consistency across all datacenters (higher latency)
- **Multi-datacenter replication**: Asynchronous across datacenters for low latency

**SQL Database Active-Active (Complex)**

Active-active for SQL databases is challenging due to consistency requirements:

```
Primary DB (Region 1) ←→ Primary DB (Region 2)
         ↓                        ↓
    Read Replicas           Read Replicas

Challenges:
- Write conflicts: Both regions accept writes
- Conflict resolution: Application-level logic required
- Eventual consistency: May see temporary inconsistencies
- Complex failover: No single source of truth

Solutions:
- Application-level routing: Partition data by region
- Conflict-free data types: Use CRDTs
- MySQL Group Replication: Multi-primary mode
- PostgreSQL BDR: Bidirectional replication
```

**Real-world example:**
- Netflix Active-Active: Multiple AWS regions with Cassandra multi-master replication
- Global users routed to nearest region
- Survives entire region failures
- Achieves 99.99%+ availability

### Advantages

1. **Near-Zero Downtime**: Automatic failover with no service interruption
2. **High Scalability**: Horizontal scaling by adding more nodes
3. **Load Distribution**: All resources actively used, better performance
4. **Geographic Distribution**: Low latency for global users
5. **Fault Tolerance**: Tolerates multiple node failures
6. **Better Resource Utilization**: No idle standby resources
7. **Disaster Recovery**: Survives datacenter-wide failures

### Disadvantages

1. **Complexity**: Difficult to configure and maintain
2. **Data Conflicts**: Write conflicts require resolution mechanisms
3. **Eventual Consistency**: May see temporary data inconsistencies
4. **Higher Cost**: More infrastructure resources required
5. **Synchronization Overhead**: Network traffic for replication
6. **Testing Challenges**: Complex failure scenarios to validate
7. **Operational Overhead**: Requires skilled teams to manage

### When to Use Active-Active

- **Zero downtime requirement**: Critical systems (e-commerce, financial trading)
- **Global user base**: Users distributed across continents
- **High traffic volumes**: Single server insufficient
- **Scalability needs**: Traffic grows unpredictably
- **Geographic redundancy**: Disaster recovery across regions
- **Performance critical**: Low latency for all users worldwide

---

## Detailed Comparison: Active-Passive vs Active-Active

| Aspect | Active-Passive | Active-Active |
|--------|---------------|---------------|
| **System Configuration** | Primary active + secondary passive standby | Multiple identical active instances |
| **Traffic Handling** | Only primary handles traffic | All nodes handle traffic simultaneously |
| **Failover Time** | 30-60 seconds (brief downtime) | Near-instant (0-5 seconds) |
| **Resource Utilization** | Passive resources idle (50% utilization) | All resources active (near 100%) |
| **Data Consistency** | Strong consistency (single writer) | Eventually consistent (multi-writer) |
| **Complexity** | Low - simpler to configure | High - complex synchronization |
| **Conflict Resolution** | Not required | Required (CRDTs, LWW, etc.) |
| **Scalability** | Limited by single active node | Highly scalable horizontally |
| **Cost** | Lower (fewer active resources) | Higher (more infrastructure) |
| **Fault Tolerance** | Tolerates single failure | Tolerates multiple failures |
| **Geographic Distribution** | Typically single region | Multi-region capable |
| **Use Case** | Mission-critical with brief downtime OK | Zero-downtime requirements |
| **RTO** | Seconds to minutes | Milliseconds to seconds |
| **RPO** | Seconds (async) to zero (sync) | Near-zero with proper design |

---

## Replication Strategies

### Synchronous vs Asynchronous Replication

**Synchronous Replication:**
- Write confirmed only after replicas acknowledge
- Zero data loss (RPO = 0)
- Higher latency due to wait time
- Used in active-passive for strong consistency
- Example: SQL Server AlwaysOn Synchronous mode

**Asynchronous Replication:**
- Write confirmed immediately, replication happens later
- Potential data loss during failure (RPO > 0)
- Lower latency, better performance
- Used in active-active for multi-datacenter
- Example: MySQL async replication, Cassandra cross-DC

**Semi-Synchronous Replication:**
- Hybrid approach - at least one replica acknowledges
- Balances performance and consistency
- RPO near-zero with acceptable latency
- Example: MySQL semi-sync replication

---

## Conflict Resolution in Active-Active

### Common Strategies

**1. Last-Write-Wins (LWW)**
- Simplest approach: most recent timestamp wins
- Problem: Can lose concurrent updates
- Used when some data loss acceptable

**2. Conflict-Free Replicated Data Types (CRDTs)**
- Data structures designed to merge conflicts automatically
- Different types: Counters, Sets, Registers
- Example: Redis Enterprise CRDT-based Active-Active
- Advantage: Mathematically guaranteed convergence

**3. Application-Level Resolution**
- Business logic determines conflict resolution
- Preserves both versions for manual merge
- Used in complex domain models

**4. Partition by Geography/Tenant**
- Avoid conflicts by routing data to specific region
- Each region owns subset of data
- Cross-region access rare or read-only

---

## Practical Implementation Examples

### Example 1: E-commerce Application (Active-Passive)

**Requirements:**
- 99.9% availability (8 hours downtime/year acceptable)
- Strong consistency for inventory and orders
- Regional customer base

**Architecture:**
```
                    Load Balancer
                         ↓
              ┌──────────┴──────────┐
              ↓                     ↓
    Application Tier        Application Tier
    (Active - AZ-1)        (Passive - AZ-2)
              ↓                     ↓
         MySQL Master ──────→ MySQL Slave
         (Active)       repl   (Standby)
              ↓                     ↓
       Shared Storage          Shared Storage
```

**Configuration:**
- MySQL Master-Slave with semi-synchronous replication
- Heartbeat monitoring every 10 seconds
- Automatic failover in 30-45 seconds
- RTO: 60 seconds, RPO: 0-5 seconds

### Example 2: Global Social Media Platform (Active-Active)

**Requirements:**
- 99.99% availability (52 minutes downtime/year)
- Global user base across continents
- High read/write throughput
- Low latency worldwide

**Architecture:**
```
        Users (US) ←→ Load Balancer → Cassandra Cluster
                            ↓           (US-East DC)
                       App Servers           ↓
                            ↓           (async repl)
                      (stateless)           ↓
                                      Cassandra Cluster
        Users (EU) ←→ Load Balancer → (EU-West DC)
                            ↓               ↓
                       App Servers     (async repl)
                            ↓               ↓
                      (stateless)     Cassandra Cluster
                                       (APAC-SG DC)
                                            ↑
        Users (APAC) ←→ Load Balancer ─────┘
                             ↓
                        App Servers
```

**Configuration:**
- Cassandra NetworkTopologyStrategy
- RF = 3 per datacenter (9 total nodes)
- LOCAL_QUORUM for reads/writes (low latency)
- Async cross-DC replication
- RTO: < 1 second, RPO: seconds

### Example 3: Financial Transaction System (Active-Passive)

**Requirements:**
- Zero data loss (RPO = 0)
- Strong consistency for transactions
- Fast failover (RTO < 30 seconds)

**Architecture:**
```
    Primary Database (PostgreSQL)
            ↓ (sync streaming replication)
    Standby Database (Hot Standby)
            ↓
    Load Balancer (health check)
            ↓
    Application Servers (connection pooling)
```

**Configuration:**
- PostgreSQL synchronous streaming replication
- Standby in different availability zone
- Automatic failover with Patroni/pg_auto_failover
- Connection pooling with pgBouncer
- RTO: 10-20 seconds, RPO: 0

---

## Interview Talking Points

### When Asked About Eliminating SPOFs

1. **Identify all SPOFs systematically:**
   - Single database instance → Add replicas
   - Single load balancer → Add redundant LB
   - Single network path → Multi-path networking
   - Single datacenter → Multi-region deployment

2. **Apply redundancy at every layer:**
   - Application tier: Multiple stateless instances
   - Database tier: Replication and clustering
   - Network: Redundant switches and routers
   - Power: Backup generators, UPS

3. **Implement automated failover:**
   - Health checks and monitoring
   - Automatic traffic rerouting
   - Self-healing capabilities

### When Asked About 99.9% vs 99.99% Availability

**99.9% (three nines) = 8.76 hours downtime/year**
- Acceptable for many business applications
- Can use active-passive with manual intervention
- Lower cost, simpler operations

**99.99% (four nines) = 52.56 minutes downtime/year**
- Required for critical systems
- Needs active-active or fast automated failover
- Higher cost, complex operations
- Multi-region deployment typically required

**Achieving 99.99%:**
- Eliminate all SPOFs with redundancy
- Automated monitoring and failover
- Geographic distribution
- Chaos engineering and failure testing
- 24/7 operations team

### When Asked About Designing Highly Available Database

**Start with questions:**
1. What are consistency requirements?
   - Strong consistency → Active-Passive
   - Eventual consistency OK → Active-Active
   
2. What is acceptable RTO/RPO?
   - RTO < 10 seconds → Active-Active
   - RTO < 60 seconds → Active-Passive with automation
   
3. Is the data model suitable for distribution?
   - Relational with transactions → Active-Passive
   - NoSQL/eventual consistency → Active-Active
   
4. What is the geographic distribution of users?
   - Single region → Active-Passive sufficient
   - Multi-region → Consider active-active

**Then propose architecture:**
- SQL databases: Master-slave replication (active-passive)
- Cassandra: Multi-DC replication (active-active)
- Hybrid: Active-passive for writes, read replicas (active for reads)

---

## Key Takeaways

1. **Active-Passive** = Simple, consistent, brief downtime acceptable
   - Best for: Financial systems, strong consistency needs, single region
   - Tradeoff: Lower resource utilization, brief interruption

2. **Active-Active** = Complex, scalable, zero downtime
   - Best for: Global applications, high availability needs, read-heavy workloads
   - Tradeoff: Higher complexity, eventual consistency challenges

3. **No silver bullet** - Choose based on:
   - Consistency requirements
   - RTO/RPO objectives  
   - Geographic distribution
   - Budget and operational capability
   - Application architecture

4. **Real-world often hybrid**:
   - Active-passive for writes (strong consistency)
   - Active-active for reads (low latency, high availability)
   - Geographic partitioning to avoid conflicts

5. **Testing is critical**:
   - Chaos engineering
   - Regular failover drills
   - Load testing at scale
   - Disaster recovery exercises

---

## Common Interview Questions & Answers

**Q: Which is better - active-passive or active-active?**

A: Neither is universally better. Active-passive offers simplicity and strong consistency with brief downtime. Active-active provides zero downtime and scalability but adds complexity. Choose based on requirements: financial systems often use active-passive for consistency; global web applications use active-active for availability and performance.

**Q: How would you handle database conflicts in active-active?**

A: Multiple strategies: (1) Use CRDTs for automatic conflict resolution with guaranteed convergence (2) Partition data by geography/tenant to avoid conflicts (3) Implement application-level resolution logic (4) Use Last-Write-Wins for non-critical data (5) For SQL databases, consider limiting to active-passive for writes with active-active read replicas.

**Q: What causes downtime in active-passive failover?**

A: Downtime occurs during: (1) Failure detection by heartbeat (5-15 seconds) (2) Failover decision and promotion of standby (10-20 seconds) (3) DNS/connection updates to route traffic (5-30 seconds) (4) Application connection pool refresh. Total typically 30-60 seconds, but can be reduced to 10-20 seconds with proper automation.

**Q: How does Cassandra achieve active-active?**

A: Cassandra uses: (1) NetworkTopologyStrategy for datacenter-aware replication (2) Configurable replication factor per datacenter (3) LOCAL_QUORUM consistency for low-latency local operations (4) Asynchronous cross-datacenter replication (5) Tunable consistency levels balancing availability vs consistency (6) Eventually consistent model allowing independent datacenter operation.

**Q: Can you do active-active with SQL databases?**

A: Possible but challenging. Options: (1) MySQL Group Replication in multi-primary mode - supports limited active-active (2) PostgreSQL with BDR - bidirectional replication (3) Application-level sharding - partition data across regions (4) CRDT-like approaches at application layer. However, most production systems use active-passive for writes with read replicas for scaling reads, as this preserves ACID guarantees.

---

## Summary

Active-Passive and Active-Active architectures represent fundamental patterns for achieving high availability. Active-Passive provides simplicity, strong consistency, and predictable failover with brief downtime. Active-Active offers near-zero downtime, global scalability, and better resource utilization at the cost of complexity. 

The choice between them depends on your specific requirements for consistency, availability, latency, geographic distribution, and operational complexity. Many production systems employ hybrid approaches, using active-passive for write paths requiring strong consistency and active-active patterns for read scaling and geographic distribution. Understanding these patterns, their trade-offs, and when to apply each is essential for designing resilient, highly available systems.