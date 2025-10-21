# Load Balancer System Design - Detailed Interview Notes

## Overview

A **load balancer** is a critical component in system design that distributes incoming network traffic across multiple servers to ensure high availability, reliability, and optimal performance. It sits between clients and backend servers, preventing any single server from becoming a bottleneck or single point of failure.

## Types of Load Balancers

Load balancers operate at different layers of the OSI (Open Systems Interconnection) model, primarily at Layer 4 (Transport Layer) and Layer 7 (Application Layer).

### Network Load Balancer (Layer 4)

Network Load Balancers operate at the **transport layer** (Layer 4) of the OSI model. They make routing decisions based on network-level information without inspecting the actual content of packets.

**Key Characteristics:**

- **Protocols:** Handles TCP, UDP, and TLS protocols
- **Routing Basis:** Makes decisions using IP addresses, port numbers, and TCP/UDP protocol information
- **Connection Handling:** Routes individual TCP connections to a single target for the life of the connection. For UDP traffic, uses flow hash algorithm based on 5-tuple (source IP, source port, destination IP, destination port, protocol)
- **Performance:** Extremely fast with low latency since no packet content inspection is required
- **Preserves Client IP:** Maintains source IP addresses throughout the connection

**Advantages:**

- Simpler to run and maintain
- Better performance due to no data inspection overhead
- More secure as no TLS decryption required at this layer
- Only one TCP connection needed
- Handles millions of requests per second with ultra-low latencies

**Limitations:**

- No smart, content-based routing
- Cannot route to different service types based on request content
- No caching capabilities
- Limited awareness of application-level health

**Use Cases:**

- High-performance, low-latency applications
- Gaming systems and media streaming services
- Database query distribution
- IoT systems requiring network-level balancing

### Application Load Balancer (Layer 7)

Application Load Balancers operate at the **application layer** (Layer 7) of the OSI model. They examine application-specific data within packets to make intelligent routing decisions.

**Key Characteristics:**

- **Protocols:** Supports HTTP, HTTPS, gRPC, WebSocket, and SMTP
- **Routing Basis:** Makes decisions based on HTTP headers, URLs, cookies, query strings, request methods, and message content
- **Connection Handling:** Maintains two separate TCP connections - one from client to load balancer, another from load balancer to server
- **Intelligence:** Can decrypt and inspect traffic to perform advanced routing

**Advanced Routing Features:**

- **Path-based routing:** Routes requests to different target groups based on URL paths (e.g., `/blog` → blog service, `/api` → API service)
- **Host-based routing:** Routes based on the Host header in HTTP requests
- **Header-based routing:** Routes based on standard or custom HTTP headers
- **Query string routing:** Routes based on query parameters
- **Source IP routing:** Routes based on client IP addresses
- **HTTP method routing:** Routes based on GET, POST, PUT, DELETE methods

**Advantages:**

- Smart, content-aware load balancing
- Can cache frequently accessed content
- SSL/TLS termination capability
- Advanced security filtering
- Application health checks
- Session persistence (sticky sessions)
- Can act as reverse proxy

**Limitations:**

- More expensive to run and maintain
- Slightly slower due to content inspection overhead
- Requires decrypting TLS traffic for inspection
- Maintains two separate TCP connections

**Use Cases:**

- Microservices architectures
- Web applications requiring content-based routing
- E-commerce platforms needing intelligent request distribution
- Containerized applications
- Applications requiring session persistence

## Load Balancing Algorithms

Load balancing algorithms determine how incoming requests are distributed across backend servers. They fall into two categories: **static** (predefined rules) and **dynamic** (adapt to current server conditions).

### Static Load Balancing Algorithms

Static algorithms distribute traffic based on predefined rules without considering current server load or performance.

#### 1. Round Robin

Distributes requests sequentially across servers in a circular, rotational manner.

**How it works:** First request → Server 1, second request → Server 2, third request → Server 3, then back to Server 1.

**Advantages:**

- Simple to implement and understand
- Ensures even distribution when requests have similar processing times
- Low computational overhead

**Limitations:**

- Doesn't consider server capacity differences
- Inefficient when servers have different specifications
- Doesn't account for current server load
- Can overload weaker servers

**Use Cases:**

- Servers with identical specifications
- Applications with uniform request processing times
- Basic web traffic distribution

#### 2. Weighted Round Robin

Extends round robin by assigning numerical weights to servers based on their capacity.

**How it works:** Servers with higher weights receive proportionally more requests. A server with weight 100 receives twice as many requests as a server with weight 50.

**Example Configuration:**
- Server A (weight 5) → receives 5 requests
- Server B (weight 2) → receives 2 requests  
- Server C (weight 1) → receives 1 request
Then the cycle repeats.

**Advantages:**

- Better utilization of heterogeneous server pools
- Administrators can allocate resources based on server capacity
- Prevents weaker servers from being overloaded
- Deterministic and easy to troubleshoot

**Limitations:**

- Still doesn't consider real-time server load
- Not suitable for requests with varying processing times
- Requires manual weight configuration

**Use Cases:**

- Mixed server environments with different capacities
- Gradual deployment scenarios (blue-green deployments)
- When server capabilities are known and stable

#### 3. IP Hash

Uses a hash function on the client's IP address to determine which server handles the request.

**How it works:** The load balancer applies a hash algorithm to the client's source IP address and destination IP address, generating a hash value that maps consistently to a specific server.

**Formula concept:** `hash(client_IP) % number_of_servers = target_server`

**Advantages:**

- **Session persistence:** Same client always connects to same server
- Reduced server overhead by eliminating session replication needs
- Enhanced security by consistently associating IP with server
- No cookie management required

**Limitations:**

- Uneven distribution if client IPs are not well distributed
- Issues with clients behind NAT or proxies (same IP for multiple users)
- Doesn't adapt to server load
- Problems when servers are added/removed (hash remapping)

**Use Cases:**

- Applications requiring session persistence
- Online banking and e-commerce platforms
- Scenarios where sticky sessions are critical
- When cookie-based persistence isn't feasible

### Dynamic Load Balancing Algorithms

Dynamic algorithms adapt to real-time server conditions, monitoring current load and performance metrics.

#### 1. Least Connection

Routes new requests to the server with the **fewest active connections**.

**How it works:** Load balancer tracks active connections on each server and assigns new requests to the server currently handling the least number.

**Advantages:**

- Prevents server overload from long-lived connections
- Adapts to changing server workloads dynamically
- Ideal for varying connection durations
- Better distribution than round robin for persistent connections

**Limitations:**

- Ignores server capacity differences (unless weighted)
- More complex to implement than round robin
- Requires continuous connection tracking
- May not suit applications requiring session persistence

**Use Cases:**

- Video streaming services
- Large file uploads/downloads
- Applications with varying request processing times
- Chat applications with persistent connections
- Database connection pools

#### 2. Weighted Least Connection

Combines least connection with server weighting, considering both active connections and server capacity.

**How it works:** If two servers have equal connections, the one with higher weight receives the new request. If connection counts differ, both weight and connection count factor into the decision.

**Advantages:**

- Works well with heterogeneous server pools
- Combines benefits of both weighted round robin and least connection
- Intelligent traffic distribution for varied workloads

**Limitations:**

- More complex to configure and manage
- Requires both weight assignment and connection tracking

**Use Cases:**

- Mixed environments with servers of different capacities
- High-traffic applications with varying connection durations
- Enterprise environments with diverse hardware

#### 3. Least Response Time

Routes requests to the server with the **lowest response time and fewest active connections**.

**How it works:** Load balancer monitors both the number of active connections and the average response time (Time to First Byte - TTFB) for each server. It calculates a score (active connections × response time) and selects the server with the lowest score.

**Example:**
- Server A: 3 connections, 2s response time → score = 6
- Server B: 7 connections, 1s response time → score = 7
- Server C: 0 connections, 2s response time → score = 0
New request goes to Server C.

**Advantages:**

- Optimizes for fastest user experience
- Increases server availability
- Prevents overloading by distributing evenly
- Adapts to server performance variations

**Limitations:**

- Non-deterministic, making troubleshooting difficult
- Complex algorithm requiring more processing
- Performance depends on accuracy of response time estimates
- Higher computational overhead

**Use Cases:**

- Environments with varying server performance
- Applications where response time is critical
- Mixed workload scenarios
- Real-time applications requiring low latency

## Additional Load Balancer Concepts

### Health Checks

Load balancers continuously monitor backend server health to ensure traffic only routes to healthy targets.

**Types:**

- **TCP-level checks:** Attempts TCP three-way handshake with backend servers
- **HTTP-level checks:** Sends HTTP requests to specific URIs and validates response codes
- **HTTPS checks:** Uses SSL/TLS for health check requests

**Mechanisms:**

- Periodic requests sent at configured intervals (e.g., every 30 seconds)
- Unhealthy threshold: Number of consecutive failures before marking server down
- Healthy threshold: Number of consecutive successes before marking server healthy
- Failed servers removed from rotation until recovered

### Session Persistence (Sticky Sessions)

Ensures all requests from the same client route to the same server throughout the session.

**Implementation Methods:**

- **Cookie-based:** Load balancer issues a cookie (e.g., AWSELB) to track client-server mapping
  - Duration-based: Cookie valid for specific time period
  - Application-controlled: Application determines cookie lifetime
- **IP-based:** Uses client IP address for routing consistency
- **Consistent hashing:** Computes server assignment using client data algorithmically

**Benefits:**

- Maintains session state on specific server
- Keeps users authenticated throughout session
- Preserves shopping carts and user preferences

**Drawbacks:**

- Can cause uneven load distribution
- Less fault tolerance (session lost if server fails)
- Complexity in distributed systems

### SSL/TLS Termination

The process of decrypting SSL/TLS-encrypted traffic at the load balancer.

**Approaches:**

- **SSL Termination (Offloading):** Load balancer decrypts traffic, sends plaintext to backends, encrypts responses
- **SSL Pass-Through:** Encrypted traffic forwarded directly to servers for decryption
- **SSL Bridging:** Load balancer decrypts for inspection, then re-encrypts before sending to servers

**Advantages:**

- Reduces CPU burden on backend servers
- Centralized certificate management
- Enables content inspection and routing
- Simplifies server configuration

**Considerations:**

- Traffic between load balancer and servers unencrypted (in termination mode)
- Security implications for multi-tenant environments

## Load Balancer Placement

Load balancers can be deployed at multiple layers of system architecture:

- **Between users and web servers:** First entry point for external traffic
- **Between web servers and application layer:** Distributes requests to application servers or cache servers
- **Between application layer and database:** Balances database queries across read replicas

## Summary

Load balancers are essential for building scalable, highly available systems. **Network Load Balancers** excel at high-performance, low-latency scenarios operating at Layer 4, while **Application Load Balancers** provide intelligent, content-aware routing at Layer 7. 

Static algorithms like **round robin, weighted round robin, and IP hash** use predefined rules, suitable when server conditions are stable. Dynamic algorithms like **least connection, weighted least connection, and least response time** adapt to real-time conditions, ideal for varying workloads.

Understanding these concepts, their tradeoffs, and appropriate use cases is crucial for system design interviews where you'll need to explain how load balancers enable horizontal scaling, fault tolerance, and optimal resource utilization.