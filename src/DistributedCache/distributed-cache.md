# Distributed Cache - System Design Interview Notes

## What is Distributed Cache?

A distributed cache is a caching system where data is stored across multiple servers (nodes) in memory to provide fast data access. Unlike a single-node cache that runs on one machine, a distributed cache spreads data across many machines working together as a cluster. This allows the system to handle massive amounts of data and serve millions of requests per second while maintaining low latency (usually under 10ms for read and write operations).

Think of it like having multiple small libraries across a city instead of one huge library. When people need books, they can go to the nearest library, making access faster and preventing overcrowding at a single location.

**Key benefits:**
- **Speed** - Data is stored in memory (RAM), which is much faster than reading from disk or database
- **Scalability** - Can add more nodes to handle more data and traffic
- **Reduced database load** - Fewer requests hit your main database, preventing it from getting overwhelmed
- **High availability** - If one node fails, others continue working

## Core Components of Distributed Cache

**Cache Clients** - Your application servers that request and store data in the cache

**Cache Cluster** - Multiple cache nodes (servers) working together to store data. Usually organized in a master-slave setup where writes go to master nodes and reads can come from either master or slave nodes

**Cache Nodes** - Individual servers that store portions of cached data in memory

**Data Source** - Your primary database or backend system where original data lives. The cache fetches from here on cache misses

**Load Balancer** - Distributes incoming requests evenly across cache nodes to prevent any single node from being overwhelmed

**Monitoring Service** - Tracks cache performance metrics like cache hits, cache misses, and node health

## Caching Patterns and Strategies

### Cache-Aside (Lazy Loading)

This is the most common caching pattern. Your application code is responsible for managing the cache.

**How it works:**

**Read Operation:**
1. Application checks if data exists in cache
2. **Cache Hit** - If data is found, return it immediately
3. **Cache Miss** - If data is not found, fetch from database
4. Store the fetched data in cache for future requests
5. Return data to user

**Write Operation:**
1. Application updates data in the database
2. Application invalidates (removes) the corresponding cache entry
3. Next read will fetch fresh data from database and cache it

**When to use:** Best when you have unpredictable read patterns or when you want full control over what gets cached

**Advantages:**
- Only frequently accessed data is cached, saving memory
- Application has full control over cache logic
- Cache failure doesn't break the application

**Disadvantages:**
- First request for data is always slow (cache miss)
- Application code becomes more complex
- Potential for stale data if cache invalidation isn't handled properly

### Write-Through

Data is written to both cache and database at the same time. The write operation only completes after both are updated.

**How it works:**
1. Application writes data
2. Data goes to cache first
3. Cache immediately writes to database
4. Only after both succeed, write is confirmed complete

**When to use:** Best for applications that read data frequently after writing it, like user profile updates

**Advantages:**
- Cache always has up-to-date data
- Better read performance since data is already cached
- Data consistency between cache and database

**Disadvantages:**
- Write operations are slower (must write twice)
- Unused data might get cached, wasting memory
- Database still handles all write traffic

### Write-Back (Write-Behind)

Data is first written to cache only, then written to database later in the background (asynchronously).

**How it works:**
1. Application writes data to cache
2. Write operation completes immediately
3. Cache writes to database in background after a delay
4. User doesn't wait for database write

**When to use:** Best for write-heavy applications where you can tolerate some risk of data loss, like logging systems or analytics

**Advantages:**
- Extremely fast write performance
- Reduces database write load significantly
- Can batch multiple writes together for efficiency

**Disadvantages:**
- Risk of data loss if cache crashes before writing to database
- More complex to implement
- Eventual consistency - database might be slightly behind cache

### Write-Around

Data is written directly to the database, bypassing the cache entirely. Cache is only populated on reads.

**How it works:**
1. Application writes directly to database
2. Cache is not updated during write
3. When data is later read, it's fetched from database and cached

**When to use:** Best when data is written but rarely read afterwards, preventing cache pollution

**Advantages:**
- Prevents filling cache with rarely-read data
- Simple to implement

**Disadvantages:**
- Reading recently written data will be slow (cache miss)
- Doesn't help with write performance

### Read-Through

The cache itself is responsible for loading data from the database, not the application. The cache acts as a middleman.

**How it works:**
1. Application always requests from cache
2. **Cache Hit** - Cache returns data
3. **Cache Miss** - Cache fetches from database, stores it, then returns it
4. Application never talks to database directly

**When to use:** When you want to simplify application code and let the cache handle data fetching

**Advantages:**
- Simpler application code
- Cache automatically handles loading data

**Disadvantages:**
- First read is slow
- Requires cache to understand database structure

## Cache Eviction Policies

Since cache memory is limited, you need rules for what to remove when cache gets full. Here are the most important ones:

### Least Recently Used (LRU)

Removes the item that hasn't been accessed for the longest time.

Think of it like a stack of papers on your desk - the ones you haven't touched in a while get moved to the bottom and eventually thrown away.

**When to use:** When recent access patterns predict future access (most common scenario)

**Example:** If cache holds items A, B, C and you access B, C, D in order, when cache is full and you need E, item A gets evicted because it's least recently used

### Least Frequently Used (LFU)

Removes the item that has been accessed the fewest times overall.

**When to use:** When you want to keep consistently popular items, even if not accessed recently

**Example:** A homepage that gets accessed thousands of times daily would stay cached even if not accessed in the last few minutes

### First In First Out (FIFO)

Removes the oldest item in the cache, regardless of how often it's accessed.

**When to use:** Simple scenarios where age is the only consideration

### Time-To-Live (TTL)

Items automatically expire after a set time period.

**Common TTL values:**
- Short TTL (30 seconds - 5 minutes): Dynamic content like stock prices, live scores
- Medium TTL (1 hour): Semi-dynamic content like news articles
- Long TTL (24 hours): Static content like images, CSS files

**When to use:** To ensure data freshness and automatic cleanup of stale data

## Data Distribution Strategies (Sharding)

To spread data across multiple nodes, distributed caches use sharding strategies:

### Consistent Hashing

Each cache key is hashed to determine which node stores it. When nodes are added or removed, only a small portion of keys need to be redistributed.

**Advantages:**
- Minimal data movement when nodes change
- Evenly distributes data across nodes

### Modulus Sharding

Node = hash(key) % number_of_nodes

**Advantages:**
- Simple to implement
- Uniform distribution

**Disadvantages:**
- Adding or removing nodes causes massive data redistribution

### Range-Based Sharding

Keys are divided into ranges, each range assigned to a specific node.

**Advantages:**
- Efficient for range queries
- Related data stored together

**Disadvantages:**
- Can create uneven load if data isn't evenly distributed

## Cache Replication

To ensure high availability, data is replicated across multiple nodes.

**Master-Slave Replication:** 
- All writes go to master node
- Master replicates data to slave nodes
- Reads can come from master or slaves
- If master fails, a slave is promoted to master

**Benefits:**
- Fault tolerance - system continues working if nodes fail
- Load distribution - read requests spread across multiple nodes

## Common Problems and Solutions

### Cache Stampede (Thundering Herd)

**Problem:** A popular cache key expires, causing thousands of simultaneous requests to hit the database all at once, potentially crashing it.

**Example:** During a flash sale, product information cache expires at exactly 12:00:00. All 10,000 shoppers trying to check out at that moment experience a cache miss and query the database simultaneously.

**Solutions:**

**1. Mutex/Lock approach** - Only allow one request to update cache while others wait
```
When cache miss occurs:
1. First request acquires lock
2. Other requests wait
3. First request fetches from database and updates cache
4. Lock released
5. Waiting requests now get cached data
```

**2. Probabilistic early expiration** - Refresh cache before it actually expires

**3. Cache warming** - Pre-populate cache before traffic arrives

### Hot Key Problem

**Problem:** One specific key gets extremely high traffic, overwhelming the single cache node storing it.

**Solutions:**
- Replicate hot keys across multiple nodes
- Use local caching on application servers for extremely hot data
- Split hot key into multiple keys

### Cache Avalanche

**Problem:** Many cache keys expire at the same time or entire cache node goes down, causing massive database load.

**Solutions:**
- Add random jitter to TTL values so keys don't all expire simultaneously
- Use consistent hashing with replication for fault tolerance

### Cache Penetration

**Problem:** Requests for non-existent data keep hitting the database because there's nothing to cache.

**Solutions:**
- Cache negative results (store "not found" in cache)
- Use bloom filters to quickly check if data exists before querying database

## Cache Warming Strategies

Cache warming means pre-loading cache with data before users request it.

**Pre-warming on Deployment:** Load critical data into cache immediately after deploying new servers

**Scheduled Warming:** Run batch jobs regularly to refresh cache with commonly accessed data

**Event-Driven Warming:** Trigger cache loading before predictable events like flash sales

**Just-in-Time Warming:** When one item is requested, also cache related items likely to be requested next

**Benefits:**
- Eliminates slow "first request" for users
- Prevents database spikes during traffic surges
- Improves overall system stability

## Consistency Models

### Strong Consistency

All nodes see the same data at the same time. Every read returns the most recent write.

**Use when:** Banking transactions, inventory management where accuracy is critical

**Trade-off:** Slower performance due to coordination overhead

### Eventual Consistency

Nodes may temporarily have different data, but will eventually sync up.

**Use when:** Social media feeds, content delivery where temporary differences are acceptable

**Trade-off:** Better performance but risk of serving stale data temporarily

## Popular Distributed Cache Technologies

### Redis
- Supports complex data structures (strings, lists, sets, hashes)
- Optional persistence to disk
- Built-in replication and clustering
- Single-threaded but extremely fast
- Rich feature set

### Memcached
- Simple key-value storage (strings only)
- Multi-threaded
- No persistence
- Simpler and lightweight
- Good for basic caching needs

**When to choose Redis:** Need complex data types, persistence, or advanced features

**When to choose Memcached:** Need simple string caching with multi-threading

## Key Performance Metrics

**Cache Hit Rate:** Percentage of requests served from cache vs total requests. Target: >80%

**Latency:** Response time for cache operations. Target: <10ms

**Throughput:** Number of requests handled per second

**Eviction Rate:** How often items are removed from cache

**Memory Usage:** How much of available cache memory is being used

## Best Practices Summary

1. **Choose the right caching pattern** based on your read/write workload patterns
2. **Set appropriate TTL values** - balance between freshness and performance
3. **Implement cache warming** for predictable traffic patterns
4. **Use proper eviction policies** (LRU for most cases)
5. **Monitor cache hit rates** - low hit rates indicate caching strategy issues
6. **Protect against cache stampede** using locks or probabilistic expiration
7. **Replicate data** across nodes for high availability
8. **Use consistent hashing** for easier scaling
9. **Add jitter to TTLs** to prevent synchronized expirations
10. **Cache negative results** to prevent repeated queries for non-existent data