# Service Discovery & Eureka Server - System Design Interview Notes

## Table of Contents
1. [Service Discovery Overview](#service-discovery-overview)
2. [Discovery Patterns](#discovery-patterns)
3. [Eureka Architecture](#eureka-architecture)
4. [Eureka Client-Server Communication](#eureka-client-server-communication)
5. [Key Mechanisms & Configurations](#key-mechanisms--configurations)
6. [Self-Preservation Mode](#self-preservation-mode)
7. [Fault Tolerance & Resilience](#fault-tolerance--resilience)
8. [Comparison with Alternatives](#comparison-with-alternatives)
9. [Interview Scenarios](#interview-scenarios)

---

## Service Discovery Overview

**What is Service Discovery?**
Service discovery is the process of automatically detecting available service instances in a distributed system. In a microservices architecture, services come and go dynamically, so clients need a way to find them without hardcoding addresses.

**Why is it needed?**
- Services scale up/down dynamically
- Instances fail and get replaced
- Services may migrate across machines
- Prevents tight coupling between service consumers and producers
- Enables intelligent load balancing and failover

**Core Problem:**
In microservices, we can't hardcode service URLs because instances are dynamic. We need a centralized registry where services register themselves and clients query for available instances.

---

## Discovery Patterns

### 1. Client-Side Discovery Pattern

**How it works:**
- Service registers itself with a **service registry**
- Client queries the **service registry** to get all available instances
- Client selects an instance (using load balancing logic)
- Client makes a direct request to the selected instance

**Flow:**
```
Service A boots → Registers with Registry
Service B (client) → Queries Registry for Service A
Registry → Returns list of Service A instances
Service B → Uses load balancer to pick one instance
Service B → Calls Service A directly
```

**Advantages:**
- Client makes intelligent decisions (consistent hashing, weighted LB)
- No extra network hop (no load balancer in between)
- Simple architecture
- Low latency (direct calls to service)

**Disadvantages:**
- Client must implement discovery logic for each language/framework
- Client is tightly coupled to service registry
- Clients must handle failover themselves
- Duplicate logic across multiple clients

**Example:** Netflix Eureka with Ribbon (client-side LB)

---

### 2. Server-Side Discovery Pattern

**How it works:**
- Service registers with registry
- Client requests go to a **load balancer** (API Gateway)
- Load balancer queries registry to find healthy instances
- Load balancer forwards request to selected instance

**Flow:**
```
Service A → Registers with Registry
Client → Requests go to Load Balancer (well-known address)
Load Balancer → Queries Registry
Load Balancer → Selects healthy instance
Load Balancer → Forwards request to Service A
```

**Advantages:**
- Loose coupling between client and discovery
- Client doesn't need discovery logic
- Centralized load balancing decisions
- Easier to change LB strategy

**Disadvantages:**
- Extra network hop (added latency)
- Load balancer becomes single point of failure (needs HA)
- More complex infrastructure
- Increased system complexity

---

### 3. DNS-Based Discovery

**How it works:**
- Services register hostnames in DNS
- Clients perform DNS lookups to discover services
- Some clients periodically update DNS entries

**Advantages:**
- Language-agnostic
- No code changes needed
- Simple

**Disadvantages:**
- No real-time visibility
- Different DNS caching semantics across clients
- Operational overhead
- Limited health checking

---

## Eureka Architecture

### Components

| Component | Role |
|-----------|------|
| **Eureka Server** | Central service registry; maintains list of all registered service instances |
| **Eureka Client** | Library embedded in services; handles registration, heartbeats, discovery |
| **Service Registry** | In-memory database storing instance information |
| **Response Cache** | Two-level caching (read-write + read-only) for performance |

### Eureka 2.0 Architecture (Write + Read Cluster)

**Key Innovation:** Separation of concerns between writes and reads

**Write Cluster:**
- Handles client registrations
- Maintains authoritative registry
- Processes heartbeats and updates
- Manages evictions
- Replicates state to other write nodes (eventually consistent)

**Read Cluster:**
- Serves as cache layer
- Handles discovery queries
- Scales independently
- Can be rapidly scaled up/down
- Always reads from write cluster

**Why two clusters?**
- Read operations are much more frequent than writes
- Read cluster can scale linearly without load on write cluster
- Write cluster can be pre-scaled based on peak capacity
- Decoupled scaling patterns

---

## Eureka Client-Server Communication

### Registration Flow

**Step 1: Initial Registration**
```
Client starts → 
Sends POST /eureka/apps/{appID} with instance metadata →
Server stores instance in registry (status: STARTING) →
Server replicates to peer nodes
```

**Instance Metadata includes:**
- hostname, IP address, port
- app name, instance ID
- health check URL, status page URL, home page URL
- metadata (custom key-value pairs)
- lease duration (how long instance can live without heartbeat)

**Step 2: Status Transition**
```
STARTING (initial) → 
Application performs health checks →
Eureka client calls HealthCheckCallback →
Returns UP status →
Server updates instance to UP
```

**Timeline:**
- `t=0`: Client boots, registers with status STARTING
- `t=30s`: First heartbeat sent (lease renewal)
- `t=~30-90s`: Instance status becomes UP (after app is ready)
- A service may not be visible to clients until it reaches UP status

---

### Heartbeat & Lease Renewal

**Heartbeat Mechanism:**
- Client sends heartbeat every **30 seconds** (default, not easily changeable)
- Heartbeat = `PUT /eureka/apps/{appID}/{instanceID}`
- Server responds with 200 OK or 404 (not found)
- Server resets the "time since last heartbeat" counter

**Lease Configuration:**

| Property | Default | Purpose |
|----------|---------|---------|
| `leaseRenewalIntervalInSeconds` | 30 | How often client sends heartbeat |
| `leaseExpirationDurationInSeconds` | 90 | Time after which server marks instance as DOWN if no heartbeat received |

**Important:** You cannot change `leaseRenewalIntervalInSeconds` from 30 seconds in production because:
- Self-preservation mode hardcodes the assumption of 30-second intervals
- Changing it breaks the heartbeat threshold calculation
- Netflix code internally assumes this value

**Lease Lifecycle:**
```
t=0: Heartbeat sent, lease timestamp updated
t=30s: Next heartbeat sent
t=60s: Last heartbeat still valid (90s expiration)
t=91s: No heartbeat received → Instance marked DOWN
t=120s: Eviction timer evicts instance (if not in self-preservation)
```

---

### Deregistration Flow

**Graceful Shutdown:**
```
Application receives shutdown signal →
Client sends DELETE /eureka/apps/{appID}/{instanceID} →
Server immediately removes instance from registry →
Server replicates deletion to peer nodes
```

**Ungraceful Shutdown (process crash):**
```
No deregistration sent →
Server doesn't receive heartbeat for 90+ seconds →
Eviction task marks instance DOWN →
Instance evicted from registry
```

**Best Practice:**
Always enable graceful shutdown so services deregister immediately, rather than waiting 90+ seconds.

---

### Discovery Query Flow

**Client Queries Registry:**
```
Client calls discoveryClient.getInstances("SERVICE-NAME") →
Eureka client queries server: GET /eureka/apps/SERVICE-NAME →
Server returns list from response cache →
Client caches locally (updates every 30s by default)
```

**Cache Strategy (Two-Level):**

1. **Read-Write Cache:**
   - Writable cache on server
   - Updates immediately on changes
   - Invalidated when instances change

2. **Read-Only Cache:**
   - Copy of read-write cache
   - Updated at intervals (default 0ms, can be configured)
   - Served to clients for better performance

**Why Two Caches?**
- Write-only users get immediate updates
- Read-only users get eventually consistent data
- Reduces lock contention on write cache
- Improves performance for high-read workloads

---

## Key Mechanisms & Configurations

### Response Cache Configuration

| Property | Default | Purpose |
|----------|---------|---------|
| `responseCacheUpdateIntervalMs` | 0 | Sync read-only cache with read-write cache |
| `responseCacheAutoExpirationInSeconds` | 180 | Cache expiry time |
| `useReadOnlyResponseCache` | true | Enable two-level cache strategy |
| `shouldDisableDelta` | false | If false, send only delta changes (not full registry) |

**Delta Updates:**
- Instead of sending full registry every time, server sends only changes
- Reduces bandwidth significantly
- Client merges deltas with local cache
- More efficient for frequent polling

---

### Instance Status Lifecycle

Possible instance statuses:

| Status | Meaning | Receives Traffic? |
|--------|---------|-------------------|
| **UP** | Running and healthy | Yes |
| **DOWN** | Explicitly marked down | No |
| **OUT_OF_SERVICE** | Maintenance/overrides | No |
| **STARTING** | Still initializing | No |
| **UNKNOWN** | Can't determine health | No |

**Status Transitions:**
```
STARTING (on registration)
    ↓
UP (when health check passes) ← Normal state
    ↓
Can transition to DOWN or OUT_OF_SERVICE
    ↓
Back to UP when issue resolved
```

---

### Health Checking

**Two Approaches:**

**1. Heartbeat-Based (Default):**
- Server assumes service is healthy if heartbeat received
- Simple, low-overhead
- Doesn't check actual service health
- Problem: Dead service can still send heartbeats

**2. Health Check Callback (Recommended for Production):**
```properties
eureka.client.healthcheck.enabled=true
```

- Client implements `HealthCheckCallback`
- Client calls `/health` endpoint before sending heartbeat
- Only sends heartbeat if health check passes
- Server uses health status for decisions

**Best Practice:**
Use health check callbacks in production. Services should not be visible as UP unless they're truly healthy.

---

## Self-Preservation Mode

### What is Self-Preservation?

A **fail-safe mechanism** that prevents mass evictions during network partitions.

**Problem it Solves:**
```
Scenario: Network partition between Eureka server and clients
- Server stops receiving heartbeats
- Thinks instances are down
- Mass-evicts all instances
- Downstream clients think everything is gone
- Cascading failure

What we want: Keep stale data rather than delete everything
```

**Solution:** Enter self-preservation mode and stop evicting instances

---

### How Self-Preservation Works

**Threshold Calculation:**

The server calculates:
```
Expected heartbeats per minute = 2 × N × 0.85

Where:
N = number of registered instances
2 = assumes 30-second interval (2 heartbeats per minute)
0.85 = renewal percent threshold (85% default)
```

**Trigger Condition:**
```
If actual_heartbeats_in_last_minute < expected_heartbeats_per_minute:
    Enter self-preservation mode
    Stop evicting instances
    Display warning: "INSTANCES ARE NOT BEING EXPIRED JUST TO BE SAFE"
```

**Example:**
```
Registered instances: 100
Expected heartbeats/min = 2 × 100 × 0.85 = 170

Scenario A (Normal):
- Actual heartbeats = 198/min → Above threshold → No self-preservation

Scenario B (Network issue):
- Actual heartbeats = 150/min → Below 170 → Enter self-preservation
- Stop evicting, keep instances in registry
- Wait for network to stabilize

Scenario C (Cascading failure):
- Actual heartbeats = 20/min → Well below 170 → Self-preservation
- Even though it looks bad, keeping stale data is better than nothing
```

---

### When to Disable Self-Preservation

**Development/Testing:**
```properties
eureka.server.enable-self-preservation=false
```

**Use cases:**
- Development environments with few instances
- Testing scenarios where you want immediate evictions
- Controlled environments without network partition risk

**Never disable in production** unless you understand the risks and have alternative fault tolerance mechanisms.

---

### Configuration Parameters

| Property | Default | Purpose |
|----------|---------|---------|
| `enableSelfPreservation` | true | Enable/disable the feature |
| `renewalPercentThreshold` | 0.85 | Percentage of expected heartbeats |
| `renewalThresholdUpdateIntervalMs` | 15 min | How often to recalculate threshold |
| `evictionIntervalTimerInMs` | 60 sec | How often to run eviction task |
| `leaseExpirationDurationInSeconds` | 90 | Grace period before marking instance DOWN |

---

## Fault Tolerance & Resilience

### Peer-to-Peer Replication

**Architecture:**
```
Eureka Server 1 ←→ Eureka Server 2 ←→ Eureka Server 3
     ↑                    ↑                    ↑
   writes from          writes from          writes from
   clients               clients               clients
```

**How Replication Works:**
1. Client registers with first server in list
2. That server replicates to all peer nodes
3. If replication fails, info is reconciled on next heartbeat
4. All servers eventually have consistent data

**CAP Theorem:**
Eureka is an **AP system** (Available + Partition tolerant):
- **Availability:** System always responds, even during partitions
- **Partition Tolerant:** System continues operating despite network splits
- **Consistency:** Traded off for availability (eventually consistent)

**Implication:**
You might temporarily see stale data, but the system never becomes unavailable.

---

### Client-Side Resilience

**Eureka Client Has:**

1. **Local Registry Cache:**
   - Clients cache service registry locally
   - Updates every 30 seconds
   - If server unreachable, client uses cached data
   - Survives server outages for ~30 seconds

2. **Retry Logic:**
   - Retries failed registrations/heartbeats
   - Configured retry count and delays
   - Implements exponential backoff

3. **Failover to Other Zones:**
   - Prefers local zone servers
   - Falls back to other zones if primary unavailable
   - Distributes load across availability zones

**Configuration:**
```properties
eureka.client.registryFetchIntervalSeconds=30  # Poll registry
eureka.client.availabilityZones.us-east-1=us-east-1a,us-east-1b
eureka.client.region=us-east-1
eureka.client.preferSameZone=true
```

---

### Cascading Failure Prevention

**Issue:** One failing service doesn't cascade

**Mechanisms:**

1. **Local Cache:**
   - Clients continue using cached data if registry unavailable
   - 30-second stale data is acceptable

2. **Circuit Breaker (Ribbon):**
   - Netflix Ribbon wraps Eureka with circuit breaker logic
   - If target service fails, stop sending requests
   - Automatically retry after timeout

3. **Timeout Handling:**
   - Clients must implement timeouts on every service call
   - Don't wait indefinitely for dead services

4. **Fallback Responses:**
   - Implement fallback logic for degraded scenarios
   - Serve cached/stale data rather than fail

---

## Comparison with Alternatives

### Eureka vs Consul vs ZooKeeper

| Feature | Eureka | Consul | ZooKeeper |
|---------|--------|--------|-----------|
| **Consistency** | Eventually consistent (AP) | Strongly consistent (CP) | Strongly consistent (CP) |
| **Discovery Strategy** | Client-side | Client-side | Server-side (via Curator) |
| **Health Checking** | Built-in | Built-in | Manual (app-specific) |
| **Multi-DC Support** | Limited | Native | Built-in |
| **Key-Value Store** | No | Yes | Yes (for distributed config) |
| **Setup Complexity** | Simple | Moderate | Moderate |
| **Netflix Compatible** | Yes | No | No |
| **Spring Cloud Integration** | Native | Supported | Supported |

---

### When to Use Each

**Use Eureka when:**
- Netflix/Spring Cloud ecosystem
- Want simple, lightweight setup
- Okay with eventual consistency
- Single data center or simple multi-DC
- Client-side discovery is acceptable
- Need Java/Spring compatibility

**Use Consul when:**
- Need strong consistency
- Want centralized configuration management
- Need native multi-DC support
- Want key-value store for configs
- Language-agnostic solution needed
- DNS and HTTP both needed for discovery

**Use ZooKeeper when:**
- Already using Kafka or Hadoop
- Need distributed coordination (locks, leader election)
- Want configuration management (Curator)
- Okay with operational complexity
- Need to coordinate across systems beyond discovery

---

## Interview Scenarios

### Scenario 1: Service Registration Delay

**Question:** "A new service instance starts and registers with Eureka. Why does it take ~3 minutes for clients to start seeing it?"

**Answer:**
```
Multiple delays stack up:

1. Initial registration: ~0s
   - Service sends POST /eureka/apps/...
   - Registered in server with status STARTING

2. Health check interval: ~10-30s
   - Service implements health check callback
   - Reports STARTING until ready

3. Server cache update: ~0-180s
   - Server's response cache is updated (if using caching)
   - Read-only cache syncs from read-write cache

4. Client registry fetch: ~30s
   - Clients pull registry every 30 seconds
   - If they just fetched, wait until next cycle

5. Client local cache: ~0s (usually minimal)

Total: ~30-180s depending on timings

Solution: Stagger these intervals:
- Reduce response cache update interval
- Increase health check frequency
- Or accept the delay in dev but tune for production
```

---

### Scenario 2: Service Keeps Disappearing (Self-Preservation Question)

**Question:** "Eureka shows 'INSTANCES ARE NOT BEING EXPIRED' warning. Services keep appearing and disappearing. How do we fix this?"

**Answer:**
```
This is self-preservation mode engaging. Likely causes:

1. Network instability
   - Network partition between server and clients
   - Clients can't reach server for heartbeats
   - Server enters self-preservation

2. Too many instances crashing at once
   - More than 15% didn't send heartbeat
   - Triggers self-preservation threshold

Solutions:

Option 1: Fix the network issue
   - Check connectivity between services and Eureka
   - Fix DNS resolution issues
   - Increase network reliability

Option 2: Tune self-preservation (for dev/test)
   - Lower renewal-percent-threshold (e.g., 0.5)
   - Shorter lease-expiration-duration
   - Only in non-production!

Option 3: Disable self-preservation (dev only)
   eureka.server.enable-self-preservation=false

Option 4: Fix root cause
   - Ensure services shut down gracefully
   - Implement proper health checks
   - Monitor why services are down
```

---

### Scenario 3: Dead Service Still Receives Traffic

**Question:** "A service is down, but clients still send requests to it. Why?"

**Answer:**
```
Multiple possible causes:

1. Client-side cache lag
   - Client cached the instance
   - Registry updates every 30 seconds
   - Client might take 30+ seconds to notice

2. No health checks enabled
   eureka.client.healthcheck.enabled=false (default!)
   - Server never checks service health
   - Just monitors heartbeats (which always succeed even if app is broken)

3. Service sends heartbeat but is broken
   - JVM alive but application frozen
   - Heartbeat succeeds but app doesn't work

Solutions:

1. Enable health checks:
   eureka.client.healthcheck.enabled=true
   
2. Implement circuit breaker on client side:
   - Netflix Ribbon (older)
   - Resilience4j (modern)
   - Stop sending to failed services

3. Implement timeout on every call:
   - Don't wait indefinitely
   - Fail fast
   
4. Reduce registry fetch interval (trade-off):
   eureka.client.registryFetchIntervalSeconds=10
   - More frequent updates = faster detection
   - More load on Eureka server
```

---

### Scenario 4: Eureka Server Failure Impact

**Question:** "What happens when the Eureka server goes down? How long can services survive?"

**Answer:**
```
Impact timeline:

Immediate (0-30s):
- Services keep sending heartbeats (to dead server)
- Existing clients use cached registry
- New clients can't discover services

Short-term (30s-5min):
- Services realize server is down (connection timeouts)
- Clients still use local cache
- Systems continue operating on stale data
- Cache is ~30 seconds old (update interval)

Medium-term (5-30min):
- Services may get confused about their own status
- Client cache becomes very stale (5+ updates missed)
- If services try to discover other services, they might fail

Long-term (>30min):
- Services lose confidence in cached data
- New instances can't register anywhere
- System becomes degraded

Prevention:

1. Run Eureka in HA cluster:
   - At least 3 server instances
   - Peer-to-peer replication
   - If one fails, others continue

2. Configure zone awareness:
   eureka.client.availabilityZones.zone=az1,az2,az3
   - Clients try multiple zones
   - Survives zone failure

3. Implement circuit breaker:
   - Clients should have fallback logic
   - Use cached data gracefully

Duration depends on:
- How fresh is client cache?
- How long until service restarts?
- How frequently services discover new peers?
```

---

### Scenario 5: Designing Multi-Region Service Discovery

**Question:** "How would you design Eureka for a global application with US and EU regions?"

**Answer:**
```
Architecture:

Region: US-East
├── Eureka Server (3 instances in AZ-1a)
├── Eureka Server (3 instances in AZ-1b)
└── Eureka Server (3 instances in AZ-1c)

Region: EU-West
├── Eureka Server (3 instances in AZ-2a)
├── Eureka Server (3 instances in AZ-2b)
└── Eureka Server (3 instances in AZ-2c)

Service Configuration (US service):
eureka:
  client:
    region: us-east-1
    availabilityZones:
      us-east-1: us-east-1a,us-east-1b,us-east-1c
    serviceUrl:
      defaultZone: http://eureka-us-1:8761/eureka/,http://eureka-us-2:8761/eureka/,...
    preferSameZone: true

Service Configuration (EU service):
eureka:
  client:
    region: eu-west-1
    availabilityZones:
      eu-west-1: eu-west-1a,eu-west-1b,eu-west-1c
    serviceUrl:
      defaultZone: http://eureka-eu-1:8761/eureka/,http://eureka-eu-2:8761/eureka/,...

Cross-Region Discovery:
- US services primarily discover other US services (low latency)
- Fallback to EU if US registry unavailable
- But then cross-region latency is high

Alternative: Cross-Region Replication
- Replicate registry between regions
- Eventual consistency
- Trade-off: Stale data across regions

Considerations:
1. Latency: Prefer local region
2. Consistency: Different regions have different data
3. Failover: Need fallback to other region
4. Monitoring: Track cross-region calls

Best Practice:
- Keep services within region
- Cross-region calls only for failover
- Use separate Eureka per region (not shared)
- Replicate config separately
```

---

### Scenario 6: Performance Tuning During High Scale

**Question:** "You have 10,000 services registering with Eureka. Clients are slow discovering services. How do you optimize?"

**Answer:**
```
Bottlenecks and solutions:

Problem 1: Server response cache contention
Solution:
  eureka.server.useReadOnlyResponseCache=true  # Ensure enabled
  eureka.server.responseCacheUpdateIntervalMs=5000  # Sync every 5s
  eureka.server.responseCacheAutoExpirationInSeconds=180

Problem 2: Clients fetching full registry
Solution:
  eureka.server.disableDelta=false  # Enable delta updates
  - Server sends only changes, not full registry
  - Dramatically reduces payload size

Problem 3: Too many heartbeats
Solution:
  eureka.instance.leaseRenewalIntervalInSeconds=30  # Can't change safely
  - Implement server-side batching
  - Process heartbeats in batches

Problem 4: Client-side cache not used
Solution:
  eureka.client.registryFetchIntervalSeconds=30  # Already default
  - Ensure clients cache locally
  - Reduce calls to server

Problem 5: Query latency
Solution:
  - Use read-only cache (enabled by default)
  - Shard Eureka servers by region
  - Implement local caching in clients

Architecture for 10,000 services:

Eureka Write Cluster:
- 3-5 instances
- Handles registrations and updates
- Peer-to-peer replication

Eureka Read Cluster:
- 10-20 instances
- Scales independently
- Serves discovery queries
- Read-only cache

Load Balancer:
- Directs reads to read cluster
- Directs writes to write cluster

Client-side:
- Cache registry locally
- Use delta updates
- Implement connection pooling
```

---

### Scenario 7: Handling Graceful vs Ungraceful Shutdown

**Question:** "How do you ensure services cleanly deregister from Eureka?"

**Answer:**
```
Graceful Shutdown:

Steps:
1. Service receives shutdown signal (SIGTERM)
2. Spring Boot calls @PreDestroy methods
3. Eureka client sends DELETE /eureka/apps/.../{instanceID}
4. Server removes immediately
5. Peers replicate deletion
6. Clients see removal on next refresh

Implementation:
@Component
public class EurekaDeregistration {
    @Autowired
    private EurekaClient eurekaClient;
    
    @PreDestroy
    public void deregister() {
        eurekaClient.shutdown();  // Explicit deregistration
    }
}

Configuration:
- Enable shutdown endpoint: management.endpoint.shutdown.enabled=true
- Implement GracefulShutdownHook
- Set termination grace period (K8s: terminationGracePeriodSeconds: 30)

Ungraceful Shutdown:

Scenario: Process killed (SIGKILL) or crashes
- No deregistration sent
- Server doesn't receive heartbeat
- After leaseExpirationDurationInSeconds (90s default):
  - Instance marked DOWN
- After eviction interval:
  - Instance removed from registry

Timeline:
t=0: Service crashes
t=90: Eureka server marks instance as DOWN
t=150: Instance evicted from registry
t=180: Clients see removal in refreshed registry (if updated)

Worst case: 180+ seconds before clients know

Mitigations:
1. Reduce leaseExpirationDurationInSeconds (risky)
   - Makes system sensitive to network glitches
   
2. Use health checks:
   - Detect actual service failures faster
   - Not fooled by dead processes
   
3. Implement client-side health checks:
   - Circuit breaker detects dead instances
   - Clients stop using them faster

4. Kubernetes liveness probes:
   - Kill container if health check fails
   - Trigger faster than Eureka eviction
```

---

## Summary: Key Takeaways

1. **Client-side discovery (Eureka):** Clients query registry and load-balance themselves
   - Simple, no extra hops, intelligent LB decisions
   - Trade-off: Client complexity, tight coupling

2. **Eureka architecture:** Service registry with peer-to-peer replication and two-level caching
   - Eventually consistent (AP system)
   - Survives network partitions gracefully

3. **Heartbeat mechanism:** 30-second intervals, non-negotiable
   - 90-second lease duration before marking DOWN
   - Hardcoded in self-preservation calculation

4. **Self-preservation mode:** Fail-safe for network partitions
   - Stops eviction if heartbeat threshold drops >15%
   - Keeps stale data rather than delete everything
   - Default: enabled (good for production)

5. **Registration delay:** ~30-180 seconds from startup to UP status
   - Stacking of health checks, caching, polling intervals

6. **Fault tolerance:** Local caching, peer replication, zone awareness
   - Services survive Eureka server outage for 30+ seconds
   - Cluster provides redundancy

7. **Configuration trade-offs:** Simpler vs faster vs resilient
   - Shorter intervals = faster discovery (more load)
   - Self-preservation = resilience (might use stale data)
   - Health checks = accuracy (more overhead)

---

## Common Configurations for Production

```properties
# Client (Service)
eureka.client.serviceUrl.defaultZone=http://eureka-1:8761/eureka/,http://eureka-2:8761/eureka/
eureka.client.healthcheck.enabled=true
eureka.client.registryFetchIntervalSeconds=30
eureka.instance.leaseRenewalIntervalInSeconds=30
eureka.instance.leaseExpirationDurationInSeconds=90
eureka.instance.instance-enabled-on-init=false

# Server
eureka.server.enable-self-preservation=true
eureka.server.renewal-percent-threshold=0.85
eureka.server.eviction-interval-timer-in-ms=60000
eureka.server.response-cache-update-interval-ms=5000
eureka.server.use-read-only-response-cache=true
eureka.client.fetch-registry=false
eureka.client.register-with-eureka=false
```

---

**Last Updated:** October 2025