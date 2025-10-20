Here's the full notepad-style answer, with all links and citations removed so you can paste it directly into a `.md` file!

***

# SQL vs NoSQL – Comprehensive System Design Interview Notes

## 1. Structure

### SQL (Relational Databases) – Structure

Data Model:
- Based on relational model with structured tables (relations)
- Data stored in rows and columns format
- Each row = record/entity, each column = attribute
- Tables connected via relationships with primary/foreign keys

Schema:
- Rigid, predefined schema enforced at write time
- Schema must be set before data insertion
- All rows in a table must follow same structure
- Schema changes require migrations and can be complex
- Data types, constraints, relationships declared upfront
- Follows normalization principles (reduces redundancy)

Structure Characteristics:
- Fixed schema with strong type enforcement
- Structured data with clear relationships
- Tables related via foreign keys
- Supports complex joins across tables
- Referential integrity maintained automatically
- ACID compliance ensures data correctness

Examples: MySQL, PostgreSQL, Oracle, SQL Server, MariaDB

Real Structure Example – E-commerce:
- Users: user_id (PK), name, email, created_at
- Orders: order_id (PK), user_id (FK), total_amount, order_date
- Products: product_id (PK), name, price, category
- Order_Items: order_item_id (PK), order_id (FK), product_id (FK), quantity

***

### NoSQL (Non-Relational Databases) – Structure Types

#### Type 1: Key-Value Stores

Data Model:
- Simplest NoSQL model, like a hash table/dictionary
- Data is unique key → value
- Key is unique (string/number/UUID)
- Value can be any data (string, JSON, binary blob, object)
- No schema enforced on values

Structure Characteristics:
- Schemaless – any value structure
- Very fast read/write
- No relationships, only get/set/delete by key
- Cannot query by value content

How Data is Accessed:
- `GET key` returns value
- `SET key value` stores/updates
- `DELETE key` removes
- Fast O(1) operations, but no value-based queries

Examples: DynamoDB, Redis, Riak, Memcached

Use Cases:
- Session management
- Cache layer
- User settings/preferences
- Shopping carts
- Rate limiting, leaderboards

Access Example – Redis/DynamoDB:
```
SET user:1001 '{"name":"John","email":"john@example.com"}'
GET user:1001
DEL user:1001
INCR page_views:homepage
EXPIRE session:abc123 3600
```
- DynamoDB: Must use partition key for efficient access. Query/Scan with key; no SQL joins, no multi-key queries unless using index.

***

#### Type 2: Document Stores

Data Model:
- Data as self-contained, semi-structured documents (JSON/BSON/XML)
- Documents grouped into collections (like tables)
- Each doc can have different structure/types/nested objects

Structure Characteristics:
- Flexible schema, no upfront definition
- Nested objects, arrays
- Hierarchical data stored naturally
- Denormalization encouraged
- No enforced foreign keys

How Data is Accessed:
- Query any field (with proper indexes)
- Rich query language: filter, project, aggregate
- Can query nested fields with dot notation
- Aggregation pipelines for analytics

Examples: MongoDB, CouchDB, DocumentDB, Couchbase

Use Cases:
- Content management, blogs, CMS
- Product catalogs (variable product attributes)
- User profiles, event logging
- Mobile/app backends, rapidly changing data schema

MongoDB Example – Usage:
```javascript
db.products.insertOne({
  name: "Laptop",
  category: "Electronics",
  price: 999,
  specs: { cpu: "Intel i7", ram: "16GB" },
  reviews: [ { user: "john", rating: 5 }, { user: "jane", rating: 4 } ]
})

// Find products under $1000
db.products.find({ "price": { $lt: 1000 } })

// Find by nested field
db.products.find({ "specs.ram": "16GB" })

// Aggregation example
db.products.aggregate([
  { $match: { category: "Electronics" } },
  { $group: { _id: "$category", avgPrice: { $avg: "$price" } } }
])
```
MongoDB Patterns: Embedding (one-to-few), referencing (one-to-many), bucketing (grouping by time/range), attribute pattern (for flexible fields).

***

#### Type 3: Column-Family (Wide-Column) Stores

Data Model:
- Stores data by row/column, but grouped into column families
- Each row identified by unique row key (partition key)
- Each row can have different columns

Structure Characteristics:
- Schema-flexible (can add new columns anytime)
- Group related columns into families (physical storage)
- Sparse data handled efficiently
- Supports high-volume, write-heavy workloads
- Denormalized model; multiple tables for different patterns

How Data is Accessed:
- Primary access by partition (row) key
- Range scan on clustering columns
- No joins, limited aggregation
- Queries must include partition key for efficiency

Examples: Cassandra, HBase, ScyllaDB, Bigtable

Use Cases:
- Time-series data (metrics, logs, sensors)
- Event logging, analytics
- Messaging (chat, notifications)
- Product recommendations, financial transactions

Cassandra CQL Example:
```sql
CREATE TABLE user_activity (
  user_id UUID,
  activity_date DATE,
  activity_time TIMESTAMP,
  action TEXT,
  details MAP<TEXT, TEXT>,
  PRIMARY KEY (user_id, activity_date, activity_time)
);

INSERT INTO user_activity (user_id, activity_date, activity_time, action, details)
VALUES (uuid(), '2025-10-20', toTimestamp(now()), 'purchase', {'product_id': '123', 'amount': '99.99'});

SELECT * FROM user_activity WHERE user_id = ? AND activity_date = '2025-10-20';
```
(No efficient arbitrary value queries, no joins.)

***

#### Type 4: Graph Databases

Data Model:
- Data as nodes (entities) and edges (relationships)
- Nodes/edges can have properties (key-value pairs)
- A key can also act as a value to another key same time
- Relationships and traversals are first-class

Structure Characteristics:
- Schema-flexible; nodes/edges can have any property
- Optimized for complex relationships
- Fast traversals, deep relationships
- Pattern matching for queries

How Data is Accessed:
- Graph query languages (Cypher in Neo4j)
- Pattern matching
- Efficient multi-hop relationships

Examples: Neo4j, Amazon Neptune, JanusGraph, ArangoDB

Use Cases:
- Social networks (followers, friends)
- Recommendation engines
- Fraud detection (suspicious relationships)
- Knowledge graphs, network topology
- Access control/permissions

Neo4j Cypher Example:
```cypher
CREATE (john:Person {name: "John"})
CREATE (acme:Company {name: "Acme Corp"})
CREATE (john)-[:WORKS_FOR {since:2020}]->(acme)

MATCH (john:Person {name: "John"})-[:WORKS_FOR]->(company)
RETURN company.name
```
- Multi-hop traversals and pattern queries efficient, compared to joins in SQL/NoSQL which are expensive/deeply nested.

***

## 2. Nature (Architecture & Distribution)

### SQL: Centralized

- Designed traditionally for single-node or master-slave replication
- All writes go to master; reads can be served by replicas (eventually consistent on replicas)
- Consistency enforced via single transaction manager
- Scaling means upgrading hardware (vertical scaling)
- Sharding is possible, but must be managed by application or middleware (adds complexity)
- Complex distributed transactions and joins hard to scale

### NoSQL: Distributed (Sharded)

- Built for distributed, peer-to-peer or masterless architectures
- Automatic sharding/partitioning – key-based or range-based distribution
- Replication for availability/fault-tolerance
- Can operate in AP (Availability/Partition Tolerance) mode (Cassandra), or CP (Consistency/Partition Tolerance) (HBase)
- Node failures handled transparently
- Geo-replicated/global clusters possible

- **Masterless (Cassandra, Riak):** Any node handles reads/writes, data replicated; no single point of failure
- **Master-Slave (MongoDB, Redis):** Primary node for writes; auto-failover to secondary
- **Coordinator-based (DynamoDB):** Coordinator handles routing, replicas managed automatically

***

## 3. Scalability

### SQL: Vertical Scaling

- Add more CPU, RAM, disk to single server
- Other than read replicas, scaling out (horizontally) is complex
- Hardware, licensing costs scale up rapidly
- Physical limits (CPU, RAM, SSD) eventually hit
- Single point of failure risk (server dies, system down)
- Suitable for moderate scale, complex transactional workloads
- Read replicas can offload reads, but writes still scale vertically

### NoSQL: Horizontal Scaling

- Add more (commodity) servers to cluster
- Data auto-distributed (sharding/replication)
- Can scale to massive volumes (hundreds of TBs or PBs, millions of ops/sec)
- Fault-tolerant: node failures handled, data replicated
- Elastic: nodes can be added/removed dynamically without downtime
- Write and read throughput both scale with nodes
- Architecture (AP, CP, geo-replication) tuned per need; always assess CAP trade-offs

***

## 4. Properties – ACID vs BASE

### SQL: ACID

Atomicity: Transactions all-or-nothing; partial updates never committed  
Consistency: Data always valid, constraints enforced  
Isolation: Concurrent transactions don’t see each other’s uncommitted changes  
Durability: Once committed, changes survive failures/crashes

- Ensures correctness, data integrity
- Mandatory for banking, payments, strict application requirements

Sample Transaction:
```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE user_id = 2;
COMMIT;
```

***

### NoSQL: BASE

Basically Available: System always responds (may return stale data)
Soft State: State may change over time (without new writes)  
Eventual Consistency: If no new writes, all replicas converge to same state eventually

- Prefers availability/partition tolerance to consistency
- Good for high-availability, distributed systems
- Acceptable staleness, eventual consistency (but can be tuned in some DBs)
- Suitable for less critical transactional needs (social posts, analytics, logs)

- Conflict resolution (last-write-wins, vector clocks, MVCC) handled at app or DB level

***

## 5. When to Use SQL vs NoSQL

**Use SQL when:**
- Data is highly structured, with clear and stable relationships
- Data integrity and consistency is critical (banking, accounting)
- Need complex queries, joins, reports, aggregation
- Schema stable and rarely changes
- Mature operational tooling and team skills in SQL

**Examples:**
- Banking transactional systems
- Inventory, ERP, accounting
- Booking, ticketing, logistics
- Analytics and business reporting

**Use NoSQL when:**

**Key-Value:**
- Very high throughput/low latency, key-based access, no query on values needed
- Session store, simple cache, rate limiters, leaderboards, microservices API data

**Document:**
- Semi/unstructured data, schema flexibility
- CMS, product catalogs, user profiles, fast-growing app backends

**Column-Family:**
- Write-heavy, time-series/event logging, analytics, scalable messaging
- Metrics, logs, feeds, IoT event data

**Graph:**
- Data is highly interconnected, multi-hop relationship queries, real-time recommendations
- Social networks, fraud detection, access control, supply chain tracking

**Examples:**
- Netflix: Cassandra for user activity/history
- Facebook: TAO (graph) for social data, MySQL for transactional
- Amazon: DynamoDB for cart/session, Oracle for ERP

***

## 6. Practical Access Patterns

**Key-Value:**
- Example: Shopping cart item for user
```
GET cart:1001
SET cart:1001 '{"user_id":1001,"items":[{"sku":"ABC","qty":2}]}'
```
- Only direct key lookups – cannot query on item details.

**Document:**
- Example: Product catalog for e-commerce
```
db.products.insertOne(
  { name: "Shoes", category: "Footwear", price: 79, stock: 200 }
)
db.products.find({ category: "Footwear", price: { $lt: 100 } })
```
- Can query across document fields, nested objects, arrays.

**Column-Family:**
- Example: User activity log
```
INSERT INTO user_activity (user_id, ts, event)
VALUES (1234, '2025-10-20T12:00', 'login');
SELECT * FROM user_activity WHERE user_id = 1234 AND ts >= '2025-10-01';
```
- Must use partition key for efficient access, range scan over clustering columns.

**Graph:**
- Example: Social network mutual friends
```
MATCH (a:User {id:1})-[:FRIEND]->()-[:FRIEND]->(c:User)
WHERE NOT (a)-[:FRIEND]-(c)
RETURN c.name
```
- Pattern-matching and traversals over relationships are efficient.

***

## 7. Summary – What to Remember

- SQL: Structured, enforced schema, ACID, vertical scaling, strictly consistent, best for integrity-critical and highly structured data.
- NoSQL: Flexible/denormalized, scalable out, BASE/ACID tradeoffs, event-driven, best for rapidly changing, massive, or semi-structured data, highly available/distributed.
- Key-Value: Simple and very fast for key-access, no queries.
- Document: Flexible, supports more queries, good for semi-structured/nested data.
- Column-Family: Write-heavy, time/range-based reads, scales for bulk data.
- Graph: Connected data, deep traversals are fast.

**Pick SQL if in doubt and requirements are traditional/business-heavy.  
Pick NoSQL when scale, agility, or non-relational data come first.**
