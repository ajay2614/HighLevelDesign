# DNS System Design - Interview Notes

## 1. DNS Overview & Problem Statement

**What is DNS?**
- DNS stands for Domain Name System
- A hierarchical, distributed database that translates human-readable domain names (e.g., `google.com`) into IP addresses (e.g., `142.250.185.46`)
- Billions of DNS queries are performed daily, requiring high scalability and low latency
- Designed to be fault-tolerant and resilient

**Why DNS is Critical:**
- Enables users to access websites by memorable names instead of remembering IP addresses
- Forms the backbone of internet infrastructure
- Performance directly impacts user experience
- Security vulnerabilities in DNS can compromise entire applications

---

## 2. DNS Hierarchy & Architecture

**Hierarchical Structure:**
The DNS follows a tree-like hierarchy from top to bottom:

```
                        Root (.)
                          |
                    TLD Servers (.com, .org, etc.)
                          |
                Authoritative Nameservers
                   (google.com, example.com)
```

**Key Components:**

### 2.1 Root Nameservers
- There are 13 types of root nameservers (managed by ICANN)
- Over 600 physical instances worldwide using Anycast routing
- Responsible for directing queries to appropriate TLD servers
- Know the location of all TLD servers

### 2.2 TLD (Top-Level Domain) Nameservers
- Manage top-level domains like .com, .org, .net, .edu, .gov, country codes (.uk, .in, etc.)
- Maintain information about which authoritative nameservers host specific domains
- Example: `.com` TLD server knows that Google's authoritative nameservers are ns1.google.com, ns2.google.com, etc.

### 2.3 Authoritative Nameservers
- Hold actual DNS records (A, AAAA, MX, CNAME, etc.) for specific domains
- Return definitive answers to DNS queries for their domains
- Each domain can have multiple authoritative nameservers for redundancy
- Controlled by the domain owner or their DNS provider

### 2.4 Recursive Resolvers (DNS Resolvers)
- Act as intermediaries between clients and authoritative nameservers
- Typically provided by ISPs or public services (8.8.8.8, 1.1.1.1, etc.)
- Make iterative queries on behalf of clients
- Cache results to reduce future lookups
- Return final answers to clients

---

## 3. DNS Query Resolution Process

**Step-by-Step Resolution (google.com lookup):**

1. **Client Query:** User types `www.google.com` in browser
2. **Stub Resolver Check:** Operating system checks its local cache
3. **Recursive Resolver:** Query reaches ISP's recursive resolver
4. **Root Server Query:** Resolver queries root nameserver with FQDN `www.google.com`
5. **Root Response:** Root returns list of TLD servers for `.com`
6. **TLD Query:** Resolver queries `.com` TLD server
7. **TLD Response:** TLD returns authoritative nameservers for `google.com`
8. **Authoritative Query:** Resolver queries `google.com` authoritative nameserver
9. **Authoritative Response:** Authoritative server returns IP address for `www.google.com`
10. **Cache & Return:** Resolver caches result and returns IP to client
11. **Connection:** Browser connects to IP address

**Key Concept: Recursive vs. Iterative Queries**
- **Recursive Query:** Client asks resolver to fetch complete answer (resolver must follow all referrals)
- **Iterative Query:** Each server provides referral to next server; querier must follow

---

## 4. DNS Records

**Common Record Types:**

| Record Type | Purpose | Example |
|---|---|---|
| **A** | Maps domain to IPv4 address | google.com → 142.250.185.46 |
| **AAAA** | Maps domain to IPv6 address | google.com → 2607:f8b0:400e:c07::8e |
| **MX** | Mail exchange; directs email | google.com → mail.google.com (priority 10) |
| **CNAME** | Canonical name; alias for another domain | www.example.com → example.com |
| **NS** | Nameserver; delegates zone to another server | example.com → ns1.example.com |
| **SOA** | Start of Authority; zone control & metadata | Contains serial, refresh, retry, expire, TTL |
| **TXT** | Arbitrary text; used for SPF, DKIM, DMARC | v=spf1 include:_spf.google.com ~all |
| **PTR** | Reverse DNS; IP to hostname mapping | 142.250.185.46 → google.com |
| **SRV** | Service records; locates specific services | _sip._tcp.example.com → sipserver.com:5060 |

**SOA Record Details:**
```
example.com. SOA ns1.example.com. admin@example.com. (
    2024101501  ; Serial number (version control)
    7200        ; Refresh (secondary checks primary every 2 hours)
    3600        ; Retry (if primary unreachable, retry after 1 hour)
    1209600     ; Expire (stop answering after 14 days if unreachable)
    3600        ; NXDOMAIN TTL (negative cache TTL)
)
```

---

## 5. DNS Zones & Delegation

**What is a Zone?**
- A portion of the DNS namespace managed by a specific authoritative nameserver
- Not the same as a domain (a domain can span multiple zones)

**Zone Delegation:**
- Process of assigning responsibility for a subdomain to different nameservers
- Parent zone contains NS records pointing to child zone's authoritative servers

**Example:**
```
Parent Zone (.com) contains:
  google.com. NS ns1.google.com.
  google.com. NS ns2.google.com.

Child Zone (google.com) contains:
  www.google.com. A 142.250.185.46
  mail.google.com. A 209.85.200.200
```

**Glue Records:**
- Additional A/AAAA records in parent zone that resolve nameserver names
- Prevents circular dependencies (need IP of nameserver to query nameserver)
- Example: `.com` zone includes `ns1.google.com. A 216.58.217.14`

---

## 6. Caching Strategy

**Why Caching is Critical:**
- Reduces load on authoritative servers
- Improves query response time
- Decreases network traffic

**TTL (Time-To-Live):**
- Measured in seconds
- Determines how long DNS records can be cached
- When TTL expires, resolver must fetch fresh data from authoritative server

**Caching Layers:**
1. **Browser Cache:** Fastest; typically short TTL
2. **OS Cache:** Operating system-level cache
3. **ISP Resolver Cache:** Shared among many clients
4. **Authoritative Server Cache:** Internal caching (if resolver)

**TTL Trade-offs:**

| TTL Strategy | Pros | Cons |
|---|---|---|
| **Short TTL (300-900 sec)** | Quick propagation of updates; low stale data risk | Higher DNS query volume; more server load; higher costs |
| **Medium TTL (3600-7200 sec)** | Balanced approach | Still relatively high query volume |
| **Long TTL (86400+ sec)** | Minimal query load; cost-effective | Slow propagation; users may see stale data longer |

**Best Practices:**
- Use longer TTLs for stable records (e.g., MX records)
- Reduce TTL before planned changes
- Typical default: 3600 seconds (1 hour)
- Domain registrar-specific: 172800 seconds (2 days)

**Negative Caching (NXDOMAIN):**
- Caches non-existent domain responses
- Uses SOA minimum field (or explicit TTL in RFC 2308)
- Typical: 3600 seconds
- Reduces repeated queries for domains that don't exist
- Maximum allowed: 86400 seconds (24 hours) per RFC 2308

---

## 7. DNS Load Balancing

**Round-Robin DNS:**
- Multiple A records for single domain
- Resolver returns different IP addresses in rotation
- Simple but not true load balancing (doesn't consider server health)

```
www.example.com. A 192.0.2.1
www.example.com. A 192.0.2.2
www.example.com. A 192.0.2.3
```

**Anycast Routing:**
- Multiple servers share same IP address
- Network routing directs traffic to nearest/fastest server
- Uses BGP (Border Gateway Protocol) for announcement
- Inherently includes failover (if nearest server down, routes to next)
- No DNS TTL issues; failover immediate

**Weighted Load Balancing:**
- Cloud providers (Route 53, Azure DNS) support weighted routing
- Example: 70% traffic to server A, 30% to server B
- Set via policies in authoritative nameserver

**Geo-Based Routing:**
- Route queries based on client location
- Returns different IPs for users in different regions
- Enables content localization and compliance

---

## 8. DNS Failover & High Availability

**Primary Considerations:**
- DNS must be highly available; any downtime impacts all services
- Typically requires at least 2 authoritative nameservers (RFC requirement)

**Active Failover:**
- Health checks monitor primary server
- If primary fails, traffic automatically reroutes to secondary
- Requires active monitoring from multiple geographic locations

**Challenges:**
- TTL delays: If client has cached old IP, failover may not work immediately
- Client-side caching: Operating system or browser may cache stale results
- Propagation time: DNS changes take time to spread across internet

**Mitigation Strategies:**
- Use Anycast for immediate, automatic failover
- Implement health checks with rapid detection (5-30 second intervals)
- Reduce TTL before expected failures (if possible)
- Use multiple nameservers in different locations
- Implement DNS monitoring and alerting

---

## 9. DNS Security Concerns

**DNS Vulnerabilities:**

### 9.1 DNS Spoofing / Cache Poisoning
- Attacker injects fake DNS records into resolver cache
- Victim redirected to malicious server
- Works by sending forged response faster than authoritative server
- Amplified by lack of authentication in original DNS protocol

### 9.2 DNS Hijacking
- Attacker gains control of domain registrar account
- Changes nameserver pointers to attacker's servers
- Full compromise of domain traffic

### 9.3 DNS DDoS Attacks
- Large volume of DNS queries overwhelm servers
- Amplification attacks: use DNS servers as reflectors
- Can render domain inaccessible

### 9.4 Man-in-the-Middle (MITM)
- Attacker intercepts DNS queries on unencrypted connection
- Modifies responses to redirect traffic
- Particularly effective on open networks

---

## 10. DNSSEC (DNS Security Extensions)

**What is DNSSEC?**
- Cryptographic protocol to authenticate DNS responses
- Adds digital signatures to DNS records
- Creates chain of trust from root down to individual records

**How DNSSEC Works:**
- Each zone operator signs their records with private key
- Resolvers verify signatures using public key
- Chain of trust validated at each level

**Chain of Trust:**
```
Root Zone (signs .com key)
    ↓
.COM Zone (signs example.com key)
    ↓
example.com Zone (signs resource records)
```

**DNSSEC Records:**
- **DNSKEY:** Contains public key for zone
- **RRSIG:** Resource Record Signature; cryptographic signature
- **DS:** Delegation Signer; hash of parent zone's DNSKEY

**Benefits:**
- Prevents cache poisoning attacks
- Verifies responses come from authoritative source
- Ensures data integrity (not tampered in transit)

**Challenges:**
- Adds computational overhead
- Increased response packet sizes
- Zone transfer complexity
- Known vulnerabilities (e.g., KeyTrap DoS attack)
- Not backward compatible without careful implementation

---

## 11. DNS Performance Optimization

**Techniques:**

### 11.1 Smart Caching
- Cache sizing: Balance memory usage vs. hit rate
- Cache coherency: Invalidate stale entries efficiently
- Shared caches: Pool results across multiple clients

### 11.2 Query Optimization
- Query pipelining: Send multiple queries without waiting for responses
- Batch requests: Group related queries
- Parallel resolution: Query multiple nameservers concurrently

### 11.3 Resolver Optimization
- Implement root hints: Local copies of root server addresses
- Forwarding: Recursive resolvers forward to upstream resolvers
- Rate limiting: Prevent abuse without blocking legitimate traffic

### 11.4 Connection Reuse
- Keep-alive TCP connections to authoritative servers
- Reduces connection establishment overhead
- Improves query throughput

### 11.5 Monitoring & Metrics
- Query latency distribution
- Cache hit rate
- Failure rate
- Nameserver response times
- GeoIP distribution of queries

---

## 12. DNS Scalability Challenges

**Known Bottlenecks:**

### 12.1 Root Server Load
- Every uncached query requires root server contact
- Millions of queries per second globally
- Mitigated by caching and Anycast distribution

### 12.2 TLD Server Capacity
- TLD servers must handle queries for millions of domains
- No caching possible (must answer for unknown subdomains)
- Requires significant infrastructure

### 12.3 Authoritative Server Load
- Must respond to all queries for their zones
- Load depends on query patterns and caching effectiveness
- Requires careful capacity planning

### 12.4 Zone Transfer Complexity
- Secondary nameservers must fetch zone data from primary
- Large zones can take time to transfer
- Incremental zone transfers (IXFR) help but add complexity

### 12.5 Consistency Issues
- Multiple authoritative servers may be out-of-sync
- Delegation inconsistency between parent and child zones
- Leads to unpredictable resolution paths

---

## 13. DNS System Design Patterns

**Pattern: Distributed Resolver with Local Caching**
```
Client
  ↓
Local Resolver (with cache)
  ↓
ISP Resolver (with larger cache)
  ↓
Root/TLD/Authoritative (no cache for new queries)
```

**Pattern: Anycast for HA**
```
User A → Nearest Anycast Node 1 (142.251.41.1)
User B → Nearest Anycast Node 2 (142.251.41.1)
User C → Nearest Anycast Node 3 (142.251.41.1)
All share same IP but different physical locations
```

**Pattern: Split-Brain DNS**
- Same domain with different records inside/outside corporate network
- Internal nameserver serves internal clients
- External nameserver serves internet clients
- Enables security, compliance, and performance optimization

---

## 14. Interview Questions & Scenarios

**Design Questions:**

**Q1: Design a global CDN's DNS infrastructure**
- Answer hints:
  - Use Anycast for geographically distributed nameservers
  - Implement geo-routing to direct traffic to nearest edge locations
  - Cache popular content domains with appropriate TTLs
  - Use health checks for failover
  - Monitor all nameservers for performance/availability
  - Separate internal and external DNS views if needed

**Q2: What happens when a resolver gets NXDOMAIN?**
- Answer hints:
  - NXDOMAIN = Non-existent domain response
  - Resolver caches this negative response for TTL duration
  - Future queries for same domain within TTL return cached NXDOMAIN
  - TTL comes from SOA record's minimum field
  - Prevents repeated queries to authoritative server

**Q3: How would you handle DNS failover with minimal downtime?**
- Answer hints:
  - Use Anycast (instant failover)
  - Implement multi-region nameservers
  - Health checks every 5-30 seconds
  - Keep TTLs reasonable for your use case
  - Pre-reduce TTL before planned maintenance
  - Have documented runbooks for incidents

**Q4: Compare TTL of 300s vs 86400s**
- Answer hints:
  - 300s: Quick updates, higher query volume, more server load, higher cost
  - 86400s: Stable, fewer queries, slower propagation, may see stale data
  - 300s good for: Frequently changing records, dynamic IPs
  - 86400s good for: Stable records, cost optimization

**Q5: What's the impact of DNSSEC on performance?**
- Answer hints:
  - Larger response packets (signatures add size)
  - Higher CPU usage (cryptographic validation)
  - May trigger UDP fallback to TCP
  - Can increase latency
  - Requires additional server resources
  - Trade-off: Security vs. Performance

---

## 15. Common DNS Issues & Debugging

**Issue: Slow DNS Resolution**
- Root cause check: TTL too low? → increase
- Check: Nameserver overloaded? → add replicas or upgrade
- Check: Network latency to nameserver? → use Anycast
- Check: Cache hit rate low? → increase cache size

**Issue: DNS Propagation Delay**
- Root cause: TTL still valid on caches → wait for expiry
- Solution: Pre-reduce TTL before changes
- Solution: Flush caches (ISP resolver, browser)
- Note: Instant propagation impossible due to TTL semantics

**Issue: Intermittent Resolution Failures**
- Root cause: One nameserver down? → check monitoring
- Check: Nameserver responding slowly? → add health monitoring
- Check: Network issues? → check connectivity
- Solution: Ensure all nameservers are healthy and reachable

**Issue: Wrong IP Returned**
- Root cause: Stale cache → wait for TTL expiry
- Check: Zone misconfiguration → verify SOA and NS records
- Check: Split-brain DNS issue → verify internal vs. external views
- Solution: Reduce TTL, add debugging to understand which nameserver responded

---

## 16. Key Takeaways for Interviews

1. **DNS is hierarchical and distributed** - enables global scale
2. **Caching is critical** - TTL determines effectiveness
3. **Multiple nameservers required** - for redundancy and HA
4. **TTL is always a tradeoff** - between consistency and load
5. **Anycast enables failover** - but clients see IP-level routing
6. **DNSSEC adds security but complexity** - evaluate carefully
7. **DNS is often overlooked** - but failures are catastrophic
8. **Monitoring and alerting essential** - DNS issues hard to debug
9. **Understand propagation delays** - TTL and caching cause them
10. **Performance + Availability + Security** - choose your priorities

---

## 17. Additional Resources for Study

**Topics to Explore Deeper:**
- RFC 1035 (DNS specification)
- RFC 2308 (Negative caching)
- RFC 4034 (DNSSEC)
- BGP and Anycast routing fundamentals
- Packet analysis with Wireshark for DNS
- nslookup, dig, host command-line tools
- DNS monitoring tools (Catchpoint, ThousandEyes)
- Common DNS server software (BIND, PowerDNS, Unbound)

---

*Last Updated: October 2025*