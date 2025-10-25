# System Design Interview Notes: Retry Pattern & Fault Tolerance in Distributed Microservices

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Retry Pattern Deep Dive](#retry-pattern-deep-dive)
3. [Backoff Strategies](#backoff-strategies)
4. [Circuit Breaker Pattern](#circuit-breaker-pattern)
5. [Complementary Patterns](#complementary-patterns)
6. [Idempotency Fundamentals](#idempotency-fundamentals)
7. [Common Anti-Patterns & Pitfalls](#common-antipatterns--pitfalls)
8. [Monitoring & Observability](#monitoring--observability)
9. [Interview Tips & Tricky Questions](#interview-tips--tricky-questions)
10. [Real-World Examples](#real-world-examples)

---

## Core Concepts

### What is Fault Tolerance?
**Fault tolerance** is a system's capacity to maintain functionality in the presence of hardware or software failures. It involves:
- Implementing redundancy and error detection
- Building error recovery mechanisms
- Ensuring continuous operation or graceful degradation
- Minimizing impact of faults on user experience

### Why Retry Patterns Matter
In distributed microservices:
- **Network failures are inevitable** – packet loss, latency spikes, transient timeouts
- **Services experience temporary unavailability** – deployments, brief overload, garbage collection pauses
- **Not all failures are permanent** – many transient errors resolve themselves within milliseconds

A retry mechanism allows systems to automatically recover from **transient failures** without manual intervention or user-facing errors.

### Transient vs. Permanent Failures
| Failure Type | Example | Should Retry? | Handling |
|---|---|---|---|
| **Transient** | Network timeout, 503 Service Unavailable, connection refused | Yes | Retry with backoff |
| **Permanent** | 404 Not Found, 401 Unauthorized, invalid data format | No | Fail fast, log error |
| **Throttling** | 429 Too Many Requests | Yes | Retry with longer backoff |

---

## Retry Pattern Deep Dive

### How Retry Pattern Works

**Flow:**
```
1. Client sends request to Service B
       ↓
2. Service B fails (transient error detected)
       ↓
3. Wait for delay (backoff)
       ↓
4. Retry the request
       ↓
5. Success? → Return result
   Max retries reached? → Fail with error
   Still failing? → Consider circuit breaker
```

### Basic Implementation Characteristics

**Key decision points:**
- Which errors are retryable? (status codes, exception types)
- How many retry attempts? (typically 3-5)
- What delay between retries? (constant, exponential, linear)
- When to add jitter? (always, for distributed systems)
- When to give up? (max attempts, timeout window)

### Identifying Retryable Errors

**HTTP Status Codes:**
- **Retryable:** 408 (Request Timeout), 429 (Too Many Requests), 500 (Internal Server Error), 502 (Bad Gateway), 503 (Service Unavailable), 504 (Gateway Timeout)
- **Not Retryable:** 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found)

**Network Errors:**
- `ECONNREFUSED` – Connection refused (service down) → Usually retryable
- `ETIMEDOUT` – Timeout → Retryable
- `ECONNRESET` – Connection reset → Retryable
- `EPIPE` – Broken pipe → Retryable (write operation)

**Custom Logic:**
```
isRetryable(error):
  if error is network-related → return true
  if error is timeout → return true
  if error.statusCode in [429, 500, 502, 503, 504] → return true
  if error is authentication-related → return false
  if error is validation-related → return false
  return false (default: fail fast)
```

### Configuration Parameters

**Standard retry configuration:**
```
max_retries: 3-5               # Limit retry attempts
initial_delay: 100-1000ms      # First backoff delay
max_delay: 30-60s              # Cap on backoff
backoff_factor: 2              # Exponential multiplier
jitter: true                   # Add randomization
timeout_per_attempt: 5-10s     # Timeout for each retry
```

---

## Backoff Strategies

### 1. No Backoff (Immediate Retry)
**❌ Problem:** Causes thundering herd and amplifies load on failing services.
```
Retry immediately on failure → Multiple synchronized requests → Service overwhelmed → More failures
```

### 2. Constant Backoff
**Strategy:** Wait fixed duration between retries (e.g., 1s, 1s, 1s)

**Pros:**
- Simple to understand and implement
- Predictable timing

**Cons:**
- All clients retry at same time (synchronized)
- Doesn't give service time to recover incrementally
- Can still cause retry storms

### 3. Linear Backoff
**Strategy:** Increase delay linearly (e.g., 1s, 2s, 3s, 4s)

**Pros:**
- Better than constant backoff
- Gradually increases pressure relief

**Cons:**
- Slower convergence than exponential
- Still potential for synchronization

### 4. Exponential Backoff (Industry Standard)
**Strategy:** Delay increases exponentially (e.g., 1s, 2s, 4s, 8s)

**Formula:**
```
delay = initial_delay × (backoff_factor ^ retry_number)
capped_delay = min(delay, max_delay)
```

**Example Timeline (1s initial, 2x factor, 32s cap):**
```
Attempt 1: fails
  → Wait 1s

Attempt 2: fails
  → Wait 2s

Attempt 3: fails
  → Wait 4s

Attempt 4: fails
  → Wait 8s

Attempt 5: fails
  → Wait 16s

Attempt 6: fails
  → Wait 32s (capped)

Attempt 7: fails
  → FAIL (max retries reached)
```

**Why it works:**
- Starts aggressive (quick retries for brief issues)
- Gradually backs off (gives service time to recover)
- Avoids overwhelming recovering service
- Gives load time to distribute

### 5. Exponential Backoff with Jitter (Best Practice)
**Strategy:** Add randomization to prevent synchronized retries

**Formula:**
```
delay = min(initial_delay × (backoff_factor ^ attempt), max_delay)
jittered_delay = random_value_between(0, delay)
```

**Why Jitter Matters:**
```
WITHOUT JITTER:
  Client A: Wait 4s → Retry at T=4s → Fails
  Client B: Wait 4s → Retry at T=4s → Fails
  Client C: Wait 4s → Retry at T=4s → Fails
  
  Result: All clients retry in synchronized wave → Thundering herd

WITH JITTER:
  Client A: Wait 2.3s → Retry at T=2.3s → Success
  Client B: Wait 3.8s → Retry at T=3.8s → Success
  Client C: Wait 1.1s → Retry at T=1.1s → Success
  
  Result: Retries spread out → Controlled recovery
```

**Typical Implementation:**
```
delay = exponential_backoff(attempt)
jitter = random() * delay
final_delay = delay + jitter  # OR
final_delay = random() * delay  # Alternate approach
```

---

## Circuit Breaker Pattern

### Three States

#### 1. **CLOSED State** (Normal Operation)
- Requests pass through normally
- Tracks failure count over rolling time window
- When failures exceed threshold → Transition to **OPEN**

**Example:** 100+ errors in 1 second window triggers transition

#### 2. **OPEN State** (Failing Fast)
- No requests allowed through
- Immediately returns error to caller
- Prevents cascading failures
- Timer starts counting down to recovery
- After timeout expires → Transition to **HALF-OPEN**

**Why open state helps:**
```
WITHOUT circuit breaker:
  Service B down → All clients retry aggressively
  → Retry storm → Connections exhausted → System degraded

WITH circuit breaker:
  Service B down → Detects failure threshold
  → Opens circuit (blocks requests)
  → Clients get fast errors → Can serve from cache/fallback
  → Service B gets breathing room → Recovers → Circuit closes
```

#### 3. **HALF-OPEN State** (Testing Recovery)
- Allows limited test requests through
- If test succeeds → Transition to **CLOSED**
- If test fails → Transition back to **OPEN**
- Success counter resets if any request fails

**State Diagram:**
```
        Failure threshold reached
                    ↓
CLOSED ────────────→ OPEN
  ↑                   ↓
  │         Timeout expires
  │                   ↓
  └─────── HALF-OPEN
              ↓
        Test succeeds? → CLOSED
        Test fails? → OPEN
```

### Key Configuration

**Circuit Breaker Parameters:**
```
failure_threshold: 100        # Errors triggering transition
time_window: 1 second         # Rolling window for counting
timeout: 30 seconds           # Before attempting half-open
success_threshold: 3          # Successful requests to close
failure_count_reset: -        # Reset on success in closed state
```

### Integration with Retry Pattern

**Pattern Combination:**
```
Make request
  ↓
Catch error
  ├─ Check circuit breaker
  │  ├─ If OPEN → Return error (fast fail)
  │  ├─ If HALF-OPEN → Allow test request
  │  └─ If CLOSED → Proceed with retry
  │
  ├─ Retry logic (with exponential backoff)
  │  ├─ If succeeds → Return result
  │  └─ If fails & retries left → Backoff & retry
  │
  └─ All retries exhausted → Permanent error
```

---

## Complementary Patterns

### Timeout Pattern

**Purpose:** Prevent indefinite waiting for responses

**Types:**
- **Connection Timeout:** Time to establish TCP connection
- **Read Timeout:** Time waiting for response after connected
- **Write Timeout:** Time to send request
- **Global Timeout:** Total time for entire operation

**Best Practice Settings:**
```
Connection timeout: 1-5 seconds
Read timeout: 5-15 seconds (depends on service SLA)
Global timeout: sum of all service timeouts + buffer

Example:
  Service A → Service B (5s) → Service C (5s) = 10s total
  Global timeout = 10s + 2s buffer = 12s
```

**Anti-Pattern Alert:** Don't set timeouts too high thinking it "gives service time." This just delays failure and doesn't help recovery.

### Bulkhead Pattern (Isolation)

**Purpose:** Contain failures to isolated compartments

**Implementation Approaches:**
1. **Thread Pool Isolation:** Separate thread pools for different operations
   - User requests use Thread Pool A (100 threads)
   - Background jobs use Thread Pool B (50 threads)
   - Failure in B doesn't block A

2. **Resource Quotas:** CPU, memory, network limits per service
   - Service A allocated 2GB memory
   - Service B allocated 3GB memory
   - Service A crash doesn't affect B's resources

3. **Network Segmentation:** Separate subnets/security groups
   - Payment service in isolated subnet
   - Recommendation service in different subnet

**Benefits:**
- **Blast radius limitation:** Failure contained
- **Independent scaling:** Services scale separately
- **Resource guarantees:** Each service gets minimum resources

### Fallback Pattern (Graceful Degradation)

**Purpose:** Maintain partial functionality when dependencies fail

**Strategies:**

**1. Cached Data Fallback**
```
Try:
  Fetch latest user recommendations from Recommendation Service
Catch:
  If fails, return cached recommendations from 1 hour ago
  Better stale data than no data
```

**2. Read-Only Mode**
```
Primary database down:
  Switch to read-only replica
  Users can view content but can't make purchases
```

**3. Default/Minimal Response**
```
Personalization service down:
  Show generic top-10 products instead of personalized list
  Still provides value, just less targeted
```

**4. Feature Toggle (Circuit at Feature Level)**
```
Recommendation engine degrading:
  Disable recommendation widget entirely
  Show other page sections normally
```

**Hierarchy of Degradation:**
```
Full functionality
        ↓
Degraded functionality (with fallback)
        ↓
Read-only mode
        ↓
Basic/Minimal functionality
        ↓
Error/Unavailable
```

### Dead Letter Queue (DLQ) Pattern

**Purpose:** Handle messages that can't be processed

**When Messages Go to DLQ:**
- Message structure is invalid (missing fields)
- Message exceeds size limits
- Receiver doesn't exist
- TTL (time-to-live) expired
- Max retries exceeded for specific message

**Lifecycle:**
```
Source Queue
     ↓
Attempt processing (retry_count < max_retries)
     ↓
Success? → Complete
  Fail? → Increment retry_count → Retry
    Max retries exceeded? → Move to DLQ
```

**Operational Strategy:**
1. Monitor DLQ depth (alert if growing)
2. Analyze failed messages (why did they fail?)
3. Fix underlying issue
4. Replay messages from DLQ
5. Verify success, then remove from DLQ

**Key Insight:** DLQ retention should be longer than source queue retention to allow investigation window.

### Timeout-Retry Interaction

**Critical Consideration:** Timeout is outer boundary for retries

```
Scenario: Operation has 10 second global timeout, retry with exponential backoff

Attempt 1 (T=0s): 5s timeout → Fails at T=5s
  Backoff: 1s

Attempt 2 (T=6s): 5s timeout → Fails at T=11s
  BUT: Global timeout already exceeded! → FAIL

This is why total retry time must fit within global timeout.

Better approach:
  Global timeout: 10s
  Per-attempt timeout: 2s
  Backoff: exponential starting from 500ms
  
  Attempt 1 (T=0s): 2s timeout → Fails at T=2s
    Backoff: 500ms
  Attempt 2 (T=2.5s): 2s timeout → Fails at T=4.5s
    Backoff: 1s
  Attempt 3 (T=5.5s): 2s timeout → Fails at T=7.5s
    Backoff: 2s
  Attempt 4 (T=9.5s): 2s timeout → Fails at T=11.5s
    BUT: Global timeout (10s) already exceeded! → FAIL
  
  Result: At least 3 good retry attempts before global timeout
```

---

## Idempotency Fundamentals

### Core Definition
**Idempotent operation:** Executing it multiple times produces the same result as executing it once.

### Why Idempotency Matters for Retries

**Without Idempotency:**
```
POST /orders with {product: "pizza", quantity: 2}
  ↓
First attempt succeeds, creates order #123 (2 pizzas)
Network latency hides success response
  ↓
Client retries same request
  ↓
Second attempt creates order #124 (another 2 pizzas) ← BUG!
  ↓
Result: User charged twice, two separate orders
```

**With Idempotency:**
```
POST /orders with {product: "pizza", quantity: 2, idempotency_key: "uuid-12345"}
  ↓
First attempt succeeds, creates order #123, stores mapping:
  idempotency_key="uuid-12345" → order_id=123
  ↓
Client retries same request with same idempotency_key
  ↓
Second attempt checks mapping, finds existing order #123
  ↓
Returns 200 OK with order #123 (same as first time)
  ↓
Result: Single order, exactly-once semantics
```

### Methods to Achieve Idempotency

#### 1. **Unique Request IDs (Idempotency Keys)**
- Generate UUID for each request
- Store mapping: idempotency_key → result
- If retry arrives with same key, return cached result

```
Implementation:
- Client generates idempotency_key (UUID)
- Send in request header or body
- Server checks cache with this key
- If hit: return cached response
- If miss: process request, cache result

Cache storage: In-memory cache, distributed cache (Redis), or database
Cache TTL: 24 hours (or business requirement)
```

#### 2. **Idempotent Operations by Design**
Make operations naturally idempotent through state checks:

```
Example: SET operation is idempotent
  user.name = "John" (executed 10 times → name is still "John")

Example: UPDATE with condition is idempotent
  UPDATE users SET status='active' WHERE id=123
  (executed 10 times → status is 'active' after first update)

Counter-Example: INCREMENT is NOT idempotent
  counter++  (executed 10 times → counter increases by 10)
```

#### 3. **Message Queue Deduplication**
Message queues handle duplicate detection:
```
When message consumed:
- Message ID tracked
- If same message consumed again → discarded automatically
- Exactly-once semantics enforced

Benefit: Automatic deduplication without application logic
```

#### 4. **Database Constraints**
Use unique constraints to enforce idempotency:
```
Database schema:
  transactions table
  unique constraint on (user_id, transaction_id, timestamp)

If duplicate arrives:
  Database rejects with unique constraint violation
  Application handles gracefully (recognizes as duplicate)
```

### Idempotency in Different HTTP Methods

| Method | Idempotent? | Typical Behavior |
|--------|-------------|------------------|
| GET | Yes | Retrieves data, doesn't modify |
| POST | No (by default) | Creates resource, multiple POSTs create multiple resources |
| PUT | Yes | Replaces resource, same result regardless of repetition |
| DELETE | Yes | Removes resource, already gone on retry |
| PATCH | No (by default) | Partial update, depends on implementation |

**Key Insight:** POST is problematic for retries unless implemented as idempotent (via keys or design).

---

## Common Anti-Patterns & Pitfalls

### 1. Retry Storm Anti-Pattern

**Definition:** Retrying indefinitely or too aggressively, amplifying failures

**Example:**
```csharp
// ❌ WRONG: Infinite retry loop
while(true) {
  try {
    return await service.CallAsync();
  } catch {
    // No backoff, no max attempts, just keep retrying
  }
}
```

**Impact:**
- All clients retry simultaneously → Thundering herd
- Overwhelms recovering service → Prevents recovery
- Wastes bandwidth and resources
- Creates cascading failures

**How It Happens:**
```
Scenario: Service B experiences brief overload
  1. Clients experience failures
  2. Clients immediately retry (no backoff)
  3. Retry load amplifies original overload
  4. Service B gets worse, not better
  5. More clients retry
  6. System enters death spiral
```

**Prevention:**
- ✅ Use exponential backoff with jitter
- ✅ Set maximum retry limits
- ✅ Add circuit breaker (stop retries if failure rate too high)
- ✅ Monitor retry rates and alert if abnormal

### 2. Retry Blindness

**Definition:** Retrying operations that shouldn't be retried

**Example:**
```java
// ❌ WRONG: Retrying non-idempotent operation
try {
  charge_credit_card(amount);  // Side effect!
} catch (Exception e) {
  // Retry might charge twice
  charge_credit_card(amount);
}
```

**How to Avoid:**
- Classify errors before retrying
- Only retry transient errors (timeouts, network, 503)
- Never retry permanent errors (4xx status codes)
- Ensure operations are idempotent

### 3. Cascading Retries

**Definition:** Retry storms propagating across multiple service layers

**Example:**
```
Service A calls Service B, which calls Service C

Service C is down:
  B retries C → C still down
  B's responses timeout → A times out
  A retries B → B busy retrying C
  Retries cascade upward → Entire system overwhelmed
```

**Solution:**
- Use circuit breaker to fail fast (don't retry indefinitely)
- Implement timeout limits
- Add bulkhead isolation
- Fallback patterns for quick failure

### 4. Unbounded Retry Duration

**Definition:** Retry timeout window too long or unbounded

**Example:**
```python
# ❌ WRONG: Retry for entire request lifetime
global_timeout = 300  # 5 minutes
while True:
  try:
    return do_operation()
  except:
    if time_elapsed < global_timeout:
      wait_and_retry()
```

**Problem:**
- User waits forever for failure to be detected
- Resources held for too long
- Cascading impact on other requests

**Solution:**
- Set reasonable retry windows (30-60 seconds typically)
- Timeout should be significantly less than request timeout
- Fail fast for permanent errors

### 5. Improper Timeout Configuration

**Definition:** Timeout values misaligned with retry logic

**Example:**
```
❌ Timeout too short:
  Per-attempt timeout: 100ms (too aggressive)
  Normal response time: 500ms
  Result: Most legitimate requests timeout and retry
  
❌ Timeout too long:
  Per-attempt timeout: 60s (too lenient)
  Stale request in system for too long
  Resource exhaustion (connections, threads held)

✅ Correct approach:
  Measure p95/p99 response time in production
  Set timeout to 2-3x of p99
  Per-attempt timeout: 5-15s (depends on service)
  Global timeout: 30-60s
```

### 6. No Jitter Leading to Thundering Herd

**Definition:** Synchronized retries from multiple clients

**Example:**
```
❌ WITHOUT JITTER:
  100 clients experience failure at T=0
  All retry at exactly T=2s (exponential backoff with 2s)
  → 100 requests hit service simultaneously
  → Service overloaded again
  
✅ WITH JITTER:
  100 clients experience failure at T=0
  Retries spread: T=0.5s, T=1.2s, T=1.8s, T=2.1s...
  → Requests distributed
  → Service handles gracefully
```

### 7. Ignoring Idempotency Requirements

**Definition:** Implementing retries on non-idempotent operations

**Example:**
```
❌ WRONG: Payment processing without idempotency key
  POST /pay {amount: 100}
  → Retries create duplicate payments

✅ CORRECT: With idempotency key
  POST /pay {amount: 100, idempotency_key: uuid}
  → Retries return same result
  → Exact-once payment guarantee
```

### 8. Using Retry as Scheduled Jobs

**Definition:** Configuring retry to run like periodic scheduler

**Example:**
```csharp
// ❌ WRONG: This is not what retry is for
var retryForeverDaily = Policy
  .HandleResult<bool>(_ => true)
  .WaitAndRetryForeverAsync(_ => TimeSpan.FromHours(24));
```

**Problems:**
- Retries consume memory indefinitely
- Not designed for long-duration periodic execution
- Should use dedicated scheduler (Quartz.NET, Hangfire)

### 9. Mixing Incompatible Backoff Strategies

**Definition:** Combining different backoff strategies inconsistently

**Example:**
```csharp
// ❌ WRONG: Mixing exponential and constant backoff
.WaitAndRetry(10, retryAttempt => retryAttempt switch {
  <= 5 => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),  // Exponential
  _ => TimeSpan.FromMinutes(3)  // Constant
})
```

**Problem:**
- Inconsistent behavior is hard to reason about
- Breaks out of exponential backoff pattern
- Difficult to reuse individual strategies

### 10. Missing Comprehensive Error Classification

**Definition:** No clear criteria for which errors to retry

**Example:**
```python
# ❌ WRONG: Catches everything
try:
  do_something()
except Exception:  # All exceptions?
  retry()

# ✅ CORRECT: Specific classification
try:
  do_something()
except TimeoutError:
  retry()
except ConnectionError:
  retry()
except IOError:
  retry()
except ValidationError:
  fail_immediately()  # Don't retry
except Exception:
  log_and_alert()
```

---

## Monitoring & Observability

### Critical Metrics to Track

#### 1. **Retry Rate**
- Percentage of requests that retry
- Alert if rate spikes (indicates new issue)
- Trend over time shows system health

```
Formula: (Retry attempts / Total requests) × 100
Normal range: 1-5% in healthy system
Alert threshold: > 10% indicates problem
```

#### 2. **Retry Success Rate**
- Percentage of retries that succeed
- High success rate (80-90%+) = pattern working well
- Low success rate = need more aggressive backoff or circuit breaker

```
Formula: (Successful retries / Total retry attempts) × 100
```

#### 3. **Max Retries Hit Count**
- How many operations exhausted all retry attempts
- Increasing trend = systemic degradation
- Should be < 1% of operations

### Metrics by Pattern

| Pattern | Key Metrics | Alert Threshold |
|---------|------------|-----------------|
| **Retry** | Retry rate, retry success rate, max retries exceeded | Rate > 10%, success < 70%, max hit > 1% |
| **Circuit Breaker** | State transitions, rejected calls, time in open state | Frequent state changes, high rejections |
| **Timeout** | Timeout rate, p99 latency, timeout vs actual failures | Rate > 5%, p99 approaching timeout |
| **Bulkhead** | Thread pool utilization, queue depth, rejection rate | High utilization, growing queue |
| **DLQ** | Queue depth, message age, processing rate | Depth growing, messages aging > 24h |

### Observability Implementation

**Golden Signals for Distributed Systems (Google SRE):**
1. **Latency:** Response time (focus on p99)
2. **Traffic:** Request rate
3. **Errors:** Error rate
4. **Saturation:** Resource utilization

**For Retry Patterns Specifically:**
```
Instrument:
  ✓ Mark retry attempt (with attempt number)
  ✓ Record delay between retries
  ✓ Track outcome (success/failure)
  ✓ Record final result
  ✓ Measure end-to-end latency

Example metrics:
  retry.attempts (histogram) – Distribution of retry counts
  retry.success_rate (gauge) – % of requests succeeding
  retry.delay_ms (histogram) – Backoff delays
  retry.exhausted (counter) – Requests hitting max retries
  request.total_latency_ms (histogram) – Including retry time
```

### Logging Strategy

**Structured Logging for Retries:**
```json
{
  "timestamp": "2024-10-25T10:30:45Z",
  "service": "order-service",
  "operation": "create_order",
  "request_id": "uuid-12345",
  "attempt": 2,
  "result": "failure",
  "error_type": "TimeoutError",
  "delay_ms": 1000,
  "total_attempts": 3,
  "final_result": "success",
  "end_to_end_latency_ms": 2500
}
```

**Correlation IDs:**
- Attach unique ID to request when it arrives
- Propagate through all retries and service calls
- Enables tracing request journey across system

### Alerting Strategy

**Alert on:**
1. **High retry rate** – Something is degrading
2. **High max-retry exhaustion** – Systematic failure
3. **Circuit breaker stuck open** – Service unable to recover
4. **DLQ depth growing** – Messages not being processed
5. **Timeout rate spike** – Response times deteriorating

**Alert hierarchy:**
```
CRITICAL: Circuit breaker open for 5+ minutes
HIGH: Retry rate > 25% for sustained period
MEDIUM: DLQ depth > threshold
LOW: Retry success rate < 70% (informational)
```

---

## Interview Tips & Tricky Questions

### Common Interview Questions

#### Q1: "Design a retry mechanism with exponential backoff"

**Good Answer Structure:**
```
1. Start simple:
   "I would identify retryable errors first – timeouts, 503s, 429s"
   
2. Add backoff strategy:
   "Use exponential backoff: delay = 1s × 2^attempt, capped at 32s"
   
3. Add jitter:
   "Add randomization to prevent synchronized retries"
   
4. Circuit breaker:
   "Combine with circuit breaker to stop retrying when failure rate is high"
   
5. Monitoring:
   "Instrument to track retry rates, success rates, max attempts"

Code sample:
  max_retries = 5
  initial_delay = 1000 ms
  max_delay = 32000 ms
  
  delay = initial_delay × (2 ^ attempt)
  delay = min(delay, max_delay)
  jittered_delay = delay + random(0, delay)
  
  Wait jittered_delay
  Retry
```

#### Q2: "How do you prevent retry storms?"

**Key Points:**
1. **Exponential backoff with jitter** – Spreads retries
2. **Max retry limits** – Prevent infinite retries
3. **Circuit breaker** – Stop retrying when pattern detected
4. **Appropriate timeout settings** – Fail fast for permanent failures
5. **Monitoring** – Detect and alert on retry storms

#### Q3: "Why is idempotency important for retries?"

**Answer:**
```
Retries only make sense if they're safe to repeat.
Without idempotency:
  - Charge credit card twice
  - Create duplicate orders
  - Double-count inventory
  
With idempotency (via idempotency key):
  - Same request ID → Same result
  - Retries are safe
  - Exactly-once semantics
```

#### Q4: "What's the difference between retry and circuit breaker?"

**Comparison:**
| Aspect | Retry | Circuit Breaker |
|--------|-------|-----------------|
| **Purpose** | Handle individual failures | Handle repeated failures |
| **Scope** | Single request | Multiple requests |
| **When used** | Transient error on one attempt | Pattern of many failures |
| **Response** | Retry request | Block/fast-fail request |
| **Recovery** | Happens gradually | Waits for timeout |

**Relationship:**
```
Circuit breaker USES retry internally (in half-open state),
but prevents retry from being harmful (stops cascade when systemic).
```

#### Q5: "How do you decide timeout values?"

**Approach:**
```
1. Measure actual response times in production
   p95 response time: 150ms
   p99 response time: 500ms
   
2. Set timeout to 2-3x of p99
   timeout = 500ms × 2.5 = 1250ms
   
3. Test and iterate
   Monitor timeout rate (should be < 1%)
   If too many timeouts, increase
   If never triggered, might be too high
   
4. Consider service tier
   Internal services: shorter timeout (500ms-2s)
   External services: longer timeout (5-30s)
```

#### Q6: "Design a retry strategy for an external payment API"

**Comprehensive Answer:**
```
Requirements:
  - Payment operations MUST be idempotent (financial transaction)
  - Failures can be network or service-related
  - Don't retry authentication errors (4xx)
  - Handle rate limiting (429)
  - Account for API response times (typically 2-5s)

Design:
  1. Idempotency:
     - Generate idempotency_key = UUID for each payment
     - API returns same result for same idempotency_key
     
  2. Error classification:
     - Retry: 429, 500, 502, 503, 504, timeouts
     - Don't retry: 401, 403, 400, 404
     
  3. Retry policy:
     - Max retries: 3
     - Initial delay: 1s
     - Factor: 2
     - Max delay: 16s
     - Add jitter
     
  4. Timeouts:
     - Connection: 5s
     - Read: 10s
     - Global: 30s (enough for 3-4 retry attempts)
     
  5. Circuit breaker:
     - If error rate > 50% in 1s window
     - Open for 30s before retry
     - Half-open allows test request
     
  6. Fallback:
     - If circuit open or all retries exhausted
     - Return error to user
     - Store for async retry/monitoring
     
  7. Monitoring:
     - Track retry rate (should be < 5%)
     - Success rate of retries (should be 80%+)
     - Circuit breaker state changes
     - Max-retries exhausted events
```

#### Q7: "Explain cascading failures in distributed systems"

**Answer:**
```
Definition: One service's failure propagating to others

Scenario:
  Service A → Service B (calls) → Service C (calls)
  
  Service C fails:
    B retries C aggressively
    B's responses slow/timeout
    A times out waiting for B
    A retries B
    B now handling: original requests + C failures + A retries
    B becomes overloaded and fails
    A sees both B and C failing
    A's clients see entire system broken

Prevention:
  1. Circuit breaker on each service boundary
     - A detects B failing → Opens circuit → Fails fast
     - Prevents A from hammering B
     
  2. Bulkhead isolation
     - B allocates resources for C calls separately
     - C failure doesn't affect B's other work
     
  3. Timeout hierarchy
     - Global timeout > sum of service timeouts
     - Prevents unbounded waiting
     
  4. Fallback strategies
     - Use cache if B unavailable
     - Degrade gracefully, don't fail completely
```

#### Q8: "How would you handle timeouts differently for different services?"

**Answer:**
```
Service tier | Typical response | Timeout | Retry count | Example |
---|---|---|---|---|
Internal RPC | 10-50ms | 500ms-1s | 3-5 | User service |
Cache/store | 50-200ms | 1-2s | 3-5 | Redis, Memcached |
Database | 100-500ms | 2-5s | 2-3 | PostgreSQL |
External API | 500ms-5s | 5-30s | 2-3 | Payment processor |
Batch job | 5s-1m | 1-5min | 1-2 | ML model inference |

Considerations:
  - Network latency to service
  - Service SLA guarantees
  - Impact of timeout vs. slow response
  - Resource costs of waiting
```

### Tricky Questions & Red Flags

#### ❓ "Should I retry all HTTP 5xx errors?"

**Better Answer:**
```
❌ No, not exactly. Context matters:
  - 500 (Internal Server Error): Probably retryable
  - 501 (Not Implemented): Don't retry
  - 503 (Service Unavailable): Retryable
  - 504 (Gateway Timeout): Retryable
  - 507 (Insufficient Storage): Might not be transient

Use case-specific classification:
  - Know your downstream APIs
  - Classify based on service behavior
  - Update classification as needed
```

#### ❓ "What if the service is down for extended time?"

**Better Answer:**
```
❌ Don't just keep retrying
✅ Use circuit breaker:
  - Detects extended failure
  - Opens circuit (blocks requests)
  - Returns fast failure to caller
  - Gives service time to recover
  - Re-enables when service recovers
  
Also:
  - Fallback: serve cached data, reduced functionality
  - Bulkhead: isolate failures to prevent cascade
  - Alert: notify ops to investigate root cause
```

#### ❓ "Should retry delay ever exceed request timeout?"

**Better Answer:**
```
❌ No, this is a logic error:
  timeout = 5s
  retry_delay = 10s
  
  This means first attempt has 5s.
  After failure, you wait 10s.
  Total time = 15s before first retry even starts.
  
✅ Ensure: Total retry time < Global timeout
  Global timeout = 10s
  Attempt 1: 0-2s
  Backoff: 500ms
  Attempt 2: 2.5-4.5s
  Backoff: 1s
  Attempt 3: 5.5-7.5s
  All attempts done by 7.5s < 10s ✓
```

---

## Real-World Examples

### Example 1: E-Commerce Order Processing

**Scenario:**
```
User submits order
  → Order Service calls Inventory Service (check stock)
  → Inventory Service calls Database
  → Database temporarily overloaded (2 second latency spike)
```

**Retry Strategy:**
```
1. Error classification:
   Database timeout → Retryable
   404 (product not found) → Not retryable
   
2. Retry config:
   Max retries: 3
   Initial delay: 100ms
   Backoff factor: 2
   Max delay: 2s
   Jitter: Yes
   
3. Implementation:
   Attempt 1: 0ms → Timeout 2s → Fails
   Wait: 100ms + jitter
   
   Attempt 2: 150ms → Timeout 2s → Succeeds ✓
   
4. Result:
   User sees successful order in 2.5 seconds
   Without retry, would timeout and fail
```

### Example 2: Microservices Chain

**Scenario:**
```
API Gateway → Product Service → Recommendation Service → Analytics Service

Product Service calls Recommendation, Recommendation calls Analytics
```

**Failure Case:**
```
Analytics Service down
  → Recommendation retries Analytics
  → Recommendation's responses slow
  → Product Service times out waiting for Recommendation
  → Product Service retries Recommendation
  → Load on Recommendation increases
  → All products become unavailable

Solution: Circuit Breaker at Each Hop
  Recommendation:
    - Detects Analytics failures
    - Opens circuit after 50 failures/s
    - Returns fallback (generic recommendations)
    - Product Service gets response in 100ms instead of timeout
    
  Product Service:
    - Detects Recommendation timeout
    - Doesn't retry indefinitely
    - Returns basic product page without recommendations
    
  Result: Graceful degradation, system stays up
```

### Example 3: Handling Thundering Herd

**Scenario:**
```
Load balancer sends 10,000 requests to API server
Server crashes and restarts
All 10,000 clients get connection refused errors
```

**Without Jitter:**
```
All 10,000 clients retry after exactly 1 second
  → 10,000 requests arrive simultaneously
  → Server overloaded again
  → Crash again
  → Infinite loop
```

**With Jitter + Backoff:**
```
Clients retry with delays:
  100 clients at 0.5s
  100 clients at 1.2s
  100 clients at 1.8s
  100 clients at 2.1s
  ... (spread over 10s)
  
Server handles ~1,000 requests/s
Recovery completes successfully
```

### Example 4: Payment Processing with Idempotency

**Scenario:**
```
User clicks "Pay Now"
Payment API called with $100 charge
Response packet lost on the way back
```

**Without Idempotency:**
```
Client: Request timeout, retries
Server: Receives retry, charges $100 again
Result: $200 charged, user furious ✗
```

**With Idempotency:**
```
Client generates idempotency_key = UUID1
  → Sends POST /charge {amount: 100, idempotency_key: UUID1}
  → Server charges $100, stores UUID1 → $100
  
Response lost, client retries
  → Sends POST /charge {amount: 100, idempotency_key: UUID1}
  → Server checks: UUID1 already processed → $100
  → Returns 200 OK {charged: 100} (same as first time)
  
Result: User charged exactly $100 ✓
```

---

## Quick Reference Checklist

### When Designing Retry Logic
- ✅ Classify errors (retryable vs. permanent)
- ✅ Use exponential backoff with jitter
- ✅ Set reasonable max retry limits (3-5)
- ✅ Ensure idempotency for state-changing operations
- ✅ Set appropriate timeouts (2-3x of p99 latency)
- ✅ Implement circuit breaker for systematic failures
- ✅ Add monitoring and alerting
- ✅ Test failure scenarios (chaos engineering)

### When Implementing Fault Tolerance
- ✅ Use timeouts to fail fast
- ✅ Implement bulkheads for isolation
- ✅ Have fallback strategies
- ✅ Design for graceful degradation
- ✅ Use DLQs for message failures
- ✅ Correlate logs for tracing
- ✅ Monitor golden signals (latency, traffic, errors, saturation)

### Common Mistakes to Avoid
- ❌ Retrying everything indefinitely
- ❌ No jitter with exponential backoff
- ❌ Retrying non-idempotent operations
- ❌ Timeout values misaligned with retry strategy
- ❌ No circuit breaker to stop cascading
- ❌ Missing error classification
- ❌ Ignoring monitoring and observability
- ❌ Using retry for scheduled jobs

---

## Summary

Retry patterns and fault tolerance in microservices form the foundation of resilient systems. The key principles are:

1. **Identify transient failures** – Not all failures should be retried
2. **Use exponential backoff with jitter** – Prevents thundering herd
3. **Combine with circuit breaker** – Stops cascading failures
4. **Ensure idempotency** – Makes retries safe
5. **Monitor everything** – Track retry rates, success rates, degradation
6. **Design for graceful degradation** – Maintain partial functionality

The combination of these patterns creates robust systems that maintain availability even during failures, providing excellent user experience and operational reliability.
