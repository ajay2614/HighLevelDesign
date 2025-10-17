DISTRIBUTED SYSTEMS AND CAP THEOREM

1. What is a Distributed System

A distributed system is a collection of independent computers (called nodes) that work together as a single system.
They communicate over a network to share data, computation, or services.
It appears as one unified system to the user.

Note: Only systems with multiple nodes over a network are considered distributed.

2. Characteristics of Distributed Systems

Multiple nodes (machines) working together.
Coordination between nodes to achieve a common goal.
Transparency – hides the fact that components are spread across machines.
Fault tolerance – continues working even if one or more nodes fail.
Scalability – easy to add more nodes to handle more load.

3. Examples of Distributed Systems

Databases: Cassandra, MongoDB, DynamoDB (multi-node clusters).
File systems: Google File System (GFS), Hadoop Distributed File System (HDFS).
Computation systems: Apache Spark, Hadoop MapReduce.
Coordination systems: ZooKeeper, etcd.
Microservices or Web Apps: Netflix, Amazon, etc.
Note: Single-node databases like standalone MySQL or PostgreSQL are not distributed systems.

4. CAP Theorem – Definition

Proposed by Eric Brewer in 2000.
It states that in a distributed system, it is impossible to guarantee all three of the following properties simultaneously:
C – Consistency
A – Availability
P – Partition Tolerance

5. Meaning of Each Property

Consistency – All nodes see the same data at the same time.
Availability – Every request gets a valid response (no errors or timeouts).
Partition Tolerance – The system continues to operate even if some nodes cannot communicate due to network failure.

6. The Trade-Off

A distributed system can only guarantee two of the three properties at once:
CA (Consistency + Availability)
CP (Consistency + Partition Tolerance)
AP (Availability + Partition Tolerance)
Important: Partition Tolerance (P) is unavoidable in distributed systems.
So systems must choose between Consistency (C) or Availability (A) during network partitions.

7. CA Systems – Consistency + Availability

Every request returns the latest consistent data.
System always available as long as network is reliable.
Cannot handle network partitions.
Examples: Single-node MySQL, PostgreSQL, Oracle.
Note: CA is possible only in non-distributed systems where partitions cannot occur.

8. CP Systems – Consistency + Partition Tolerance

Ensures data consistency even during network issues.
Some requests may be blocked (temporarily unavailable) to preserve consistency.
Focus on correctness rather than uptime.
Examples: MongoDB (cluster), HBase, Zookeeper, Redis Sentinel.

9. AP Systems – Availability + Partition Tolerance

System always responds to requests, even when some nodes are down.
May return outdated (stale) data temporarily.
Data eventually becomes consistent across all nodes (eventual consistency).
Examples: Cassandra, CouchDB, Amazon DynamoDB.

10. Real-World Analogy

Two friends maintain copies of the same notebook:
They stop writing until they meet to sync → CP (Consistency preferred).
They keep writing separately and sync later → AP (Availability preferred).
They are always connected and instantly share updates → CA (ideal, rarely possible).

11. Important Concepts

Eventual Consistency – Data becomes consistent across all nodes after some time.
Strong Consistency – Every read returns the latest write instantly.
Network Partition – Some parts of the network cannot communicate, but both sides keep running.

12. CAP in Practice

CAP theorem only applies to distributed systems.
Partition Tolerance (P) is mandatory in distributed systems.
Single-node databases are CA systems (CAP theorem does not apply).
Distributed databases must choose C or A during partitions → CP or AP.

13. Examples

Single-node MySQL/PostgreSQL → CA (not distributed).
Multi-node MySQL cluster → CP (blocks writes on partition to maintain consistency).
MongoDB cluster → CP.
Cassandra cluster → AP (eventually consistent, always available).

DynamoDB → AP by default (tunable consistency available).

14. Key Interview Takeaways

CAP theorem applies only to distributed systems.
MySQL/Postgres/Oracle are not distributed, they run on single node.
Partition Tolerance (P) is unavoidable in distributed systems.
Single-node databases are CA, not subject to CAP trade-offs.
Distributed systems must choose between Consistency (C) or Availability (A) during partitions → CP or AP.
AP systems → eventual consistency.
CP systems → strong consistency.

15. Quick Summary

CA → Consistency + Availability (non-distributed systems).
CP → Consistency + Partition Tolerance (distributed systems, may block requests).
AP → Availability + Partition Tolerance (distributed systems, may return stale data).