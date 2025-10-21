# System Design: Rate Limiter

## Problem Statement

A **rate limiter** is a crucial system component that controls the number of requests a client can make within a specific time window. It serves as a defensive mechanism to prevent system overload, protect against denial-of-service attacks, ensure fair resource usage, and maintain service quality across distributed systems.

The core challenge is designing a system that can handle millions of requests per second while accurately enforcing rate limits across multiple servers, maintaining low latency, and ensuring high availability.

## Requirements

### Functional Requirements

- **Multiple Rate-Limiting Rules**: Support different limits per user, IP address, API endpoint, or custom identifiers
- **Configurable Thresholds**: Allow dynamic adjustment of rate limits without code changes
- **Custom Response Handling**: Return appropriate HTTP status codes (429 Too Many Requests) with clear error messages
- **Multi-Window Support**: Enforce limits across different time granularities (per second, minute, hour)
- **Client Identification**: Support various identification methods including IP addresses, API keys, and user IDs

### Non-Functional Requirements

- **High Availability**: Continue operating even during peak traffic or partial system failures
- **Low Latency**: Add minimal overhead (under 10ms) to request processing
- **Scalability**: Handle massive request volumes and scale horizontally
- **Strong Consistency**: Maintain consistent request counts across distributed nodes
- **Fault Tolerance**: No single point of failure in the system
- **Security**: Protect against malicious attacks and bypass attempts

## Rate Limiting Algorithms

### 1. Token Bucket Algorithm

**How it works**: Tokens are added to a bucket at a fixed rate up to a maximum capacity. Each request consumes one token. If no tokens are available, requests are rejected or delayed.

**Key Components**:
- **Bucket Capacity**: Maximum number of tokens that can be stored
- **Refill Rate**: Rate at which tokens are added to the bucket
- **Token Consumption**: One token per request (typically)

**Pros**:
- Handles bursty traffic effectively by accumulating tokens during quiet periods
- Flexible and allows temporary spikes above the average rate
- Memory efficient with simple counter implementation
- Granular control over request processing rate

**Cons**:
- Token management overhead
- Can be exploited by greedy clients who consume all available tokens
- Requires careful tuning of refill rate and capacity

**When to use**:
- Applications with bursty traffic patterns (video streaming, file uploads)
- APIs requiring flexibility for legitimate traffic spikes
- Quality of Service (QoS) implementations

---

### 2. Leaky Bucket Algorithm

**How it works**: Requests are added to a queue (bucket) and processed at a constant rate. If the bucket overflows, excess requests are dropped.

**Key Components**:
- **Bucket Size**: Maximum queue capacity
- **Leak Rate**: Constant rate at which requests are processed
- **Overflow Handling**: Drop or delay excess requests

**Pros**:
- Ensures smooth, constant output rate regardless of input burstiness
- Predictable behavior makes system maintenance easier
- Prevents sudden traffic spikes from overwhelming downstream systems

**Cons**:
- Limited flexibility for handling legitimate bursts
- Can result in packet loss during high traffic periods
- Less adaptive to varying traffic patterns

**When to use**:
- Systems requiring constant, predictable traffic flow
- Network congestion management
- Applications where smooth traffic delivery is critical over burst handling

---

### 3. Fixed Window Counter

**How it works**: Time is divided into fixed windows, with each window having a request counter. Requests increment the counter, and excess requests are rejected until the next window begins.

**Pros**:
- Simple to implement and understand
- Memory efficient
- Good for applications with predictable traffic patterns

**Cons**:
- **Boundary Problem**: Can allow twice the rate limit at window boundaries
- Poor handling of bursty traffic
- Not suitable for strict rate enforcement

**When to use**:
- Simple applications with loose rate limiting requirements
- Systems where slight rate limit violations are acceptable
- Scenarios requiring minimal computational overhead

---

### 4. Sliding Window Log

**How it works**: Maintains a log of timestamps for each request. When a new request arrives, outdated timestamps are removed, and the remaining count is checked against the limit.

**Pros**:
- Very accurate with no boundary issues
- Precise enforcement of rate limits
- Works well for low-volume APIs

**Cons**:
- Memory intensive for high-volume applications
- Requires storing and processing timestamps for each request
- Higher computational overhead

**When to use**:
- Applications requiring precise rate limiting
- Low to moderate traffic volumes
- Security-critical systems where accuracy is paramount

---

### 5. Sliding Window Counter

**How it works**: Combines fixed window and sliding log approaches. Uses weighted counts from current and previous windows to estimate requests in the sliding window.

**Formula**: `Requests in sliding window = current_window_requests + (previous_window_requests × overlap_percentage)`

**Pros**:
- More accurate than fixed window
- Memory efficient compared to sliding window log
- Smooths out traffic spikes

**Cons**:
- Assumes uniform request distribution within windows
- More complex than fixed window implementation
- Approximation may not be 100% accurate

**When to use**:
- High-traffic applications requiring good accuracy
- Systems needing balance between precision and efficiency
- Applications with moderate to high request volumes

---

## Algorithm Comparison Table

| Algorithm | Accuracy | Memory Efficiency | Complexity | Burst Handling | Best For |
|-----------|----------|-------------------|------------|----------------|----------|
| **Token Bucket** | Good | High | Medium | Excellent | Bursty traffic, flexible APIs |
| **Leaky Bucket** | Good | High | Medium | Poor | Constant rate, network control |
| **Fixed Window** | Low | Very High | Low | Poor | Simple apps, relaxed limits |
| **Sliding Window Log** | Excellent | Low | High | Good | Precision-critical, low volume |
| **Sliding Window Counter** | Very Good | Medium | Medium | Good | High volume, balanced approach |

---

## Algorithm Selection Criteria

### Choose Token Bucket When:
- **Traffic Pattern**: Bursty, variable traffic with legitimate spikes
- **Flexibility Need**: Require adaptability to changing traffic patterns
- **Burst Handling**: Applications like video streaming, file transfers
- **Resource Control**: Need granular control over request processing rates

### Choose Leaky Bucket When:
- **Traffic Pattern**: Need constant, smooth output rate
- **Simplicity Priority**: Want straightforward implementation
- **Network Protection**: Preventing downstream system overload
- **Predictable Load**: Applications requiring steady resource consumption

### Choose Fixed Window When:
- **Simplicity Required**: Need minimal implementation complexity
- **Relaxed Requirements**: Slight rate limit violations are acceptable
- **Resource Constraints**: Limited computational resources
- **Stable Traffic**: Predictable, non-bursty traffic patterns

### Choose Sliding Window Log When:
- **Precision Critical**: Absolute accuracy in rate limiting required
- **Low Volume**: Manageable request volumes
- **Security Focus**: Preventing abuse with strict enforcement
- **Compliance**: Regulatory requirements for precise limits

### Choose Sliding Window Counter When:
- **High Volume**: Need to handle significant traffic efficiently
- **Balanced Approach**: Want accuracy without excessive resource usage
- **Moderate Precision**: Good accuracy acceptable over perfect precision
- **Production Systems**: Real-world applications with varied traffic

---

## System Architecture

### High-Level Components

1. **Rate Limiter Service**: Core component that evaluates requests against configured rules

2. **Rules Engine**: Manages and stores rate limiting policies and configurations

3. **Distributed Cache**: Stores request counters and state (typically Redis)

4. **Client Identifier**: Extracts user/IP/API key information from requests

5. **Response Handler**: Returns appropriate responses for allowed/blocked requests

### Distributed Considerations

**Consistency vs Availability**: Most systems prioritize availability over strict consistency, allowing slight over-limiting during node failures.

**Synchronization**: Use centralized datastores like Redis with atomic operations to prevent race conditions.

**Clock Synchronization**: Critical for time-based windows across distributed nodes.

**Network Overhead**: Each rate limit check may require remote datastore access, impacting latency.

---

## Conclusion

A well-designed rate limiter is essential for maintaining system stability, preventing abuse, and ensuring fair resource allocation in modern distributed applications. The choice of algorithm depends heavily on your specific traffic patterns, accuracy requirements, and system constraints.

For most production systems, the **Sliding Window Counter** algorithm offers the best balance between accuracy, efficiency, and complexity, making it a popular choice for large-scale distributed applications.
