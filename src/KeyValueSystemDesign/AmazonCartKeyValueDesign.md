AMAZON CART – KEY VALUE STORE (SCALABILITY, DECENTRALIZATION, EVENTUAL CONSISTENCY)
------------------------------------------------------------------------------------

GOALS
Scalability : System must handle millions of users and shopping carts efficiently
Decentralization : No single master, all nodes are equal in responsibility
Eventual Consistency : Updates propagate asynchronously and converge over time

DATA MODEL
Key Value based store
Key = cart:<user_id>
Value = JSON or object holding cart items, quantity, timestamps
Inspired by Amazon Dynamo architecture

------------------------------------------------------------------------------------
1. PARTITIONING (CONSISTENT HASHING)

Used to distribute keys evenly across cluster nodes.
Each node is assigned a position on a hash ring.
Hash of the key decides which node is responsible for that key (called the coordinator).
The key and its data are stored on N replicas.
Coordinator is the first node responsible for the key.
Next N-1 nodes clockwise on the ring are also chosen to store replicas.
When a node joins or leaves, only nearby data ranges are moved, not all.

Important Notes:
All nodes in the cluster are not replicas for every key.
Only N replicas are selected for each key.
Example
Total cluster nodes = 10
Replication factor N = 3
Each key stored only on 3 nodes (approximately 30 percent of cluster)
Each node stores only a subset of the total keyspace. This ensures scalability.

------------------------------------------------------------------------------------
2. REPLICATION

Each key has N total replicas for high availability.
Coordinator node plus N-1 other nodes makes total N replicas.
If one node fails, other replicas keep data available.
This provides durability and fault tolerance.

Example
N = 3
Coordinator : Node A
Replica1 : Node B
Replica2 : Node C
Total replicas = 3

------------------------------------------------------------------------------------
3. WRITE OPERATION (PUT)

Step 1  Client sends write request to coordinator node.
Step 2  Coordinator writes data locally and forwards to other N-1 replicas.
Step 3  Coordinator waits for W acknowledgments from replicas.
Step 4  When W acknowledgments are received, the write is considered successful.
Step 5  Remaining replicas are updated asynchronously if needed.

Example
N = 3, W = 2
Coordinator (A) writes to itself, sends to B and C.
After receiving 2 acks out of 3, write is successful.

------------------------------------------------------------------------------------
4. READ OPERATION (GET)

Step 1  Client sends read request to coordinator.
Step 2  Coordinator requests data from all N replicas.
Step 3  Coordinator waits for R responses.
Step 4  Coordinator merges versions based on timestamps or vector clocks.
Step 5  Returns merged value to client.
Step 6  If any stale copies found, triggers read repair in background.

Example
N = 3, R = 2
Coordinator queries A, B, and C.
Waits for any 2 replies.
Compares versions, merges if needed, and returns latest or merged value.

------------------------------------------------------------------------------------
5. RELATIONSHIP BETWEEN N, R, AND W

N represents total number of replicas per key (including coordinator)
W represents number of replicas that must acknowledge a write
R represents number of replicas that must respond to a read

Consistency conditions
If R + W > N  →  Stronger consistency (read and write overlap)
If R + W ≤ N  →  Eventual consistency (possible stale reads)

Example scenarios
N = 3, W = 2, R = 2  →  Strong consistency (one node overlap)
N = 3, W = 1, R = 1  →  High availability, eventual consistency
N = 3, W = 3, R = 1  →  Safer writes, slower
N = 3, W = 1, R = 3  →  Safer reads, slower

------------------------------------------------------------------------------------
6. DATA VERSIONING

Purpose is to manage concurrent updates and conflicts.

Vector Clocks
Each replica maintains a version vector (replica_id to counter mapping).
If one version dominates another, it is newer.
If two versions are concurrent, merge is required.

Timestamp Based (Last Write Wins)
Each update tagged with timestamp.
Latest timestamp wins.
Simple but may lose concurrent updates.

CRDTs (Conflict Free Replicated Data Types)
Used for data like shopping carts where merges are frequent.
All operations are commutative and can merge safely.
Example  OR-Set or PN-Counter for cart items.
Ensures no items lost when concurrent adds happen.

------------------------------------------------------------------------------------
7. GOSSIP PROTOCOL

Each node periodically communicates with random peers.
Information exchanged includes node liveness, membership changes, and hinted handoff data.
Gradually, the entire cluster learns about the latest state.

Used for
Node join and leave detection
Propagation of partition map
Failure detection and cluster state synchronization

This keeps the system decentralized with no central controller.

------------------------------------------------------------------------------------
8. MERKLE TREE (ANTI ENTROPY)

Used to detect and repair inconsistencies between replicas.
Each node maintains a Merkle tree for the range of keys it owns.
Each leaf represents a hash of a key range.
Two replicas compare root hashes.
If roots match, data identical.
If roots differ, compare child hashes until mismatched keys found.
Only the differing ranges are synchronized.

This saves bandwidth and avoids sending full data during repair.

------------------------------------------------------------------------------------
9. HINTED HANDOFF

When a replica node is temporarily down, coordinator keeps the write locally as a hint.
Once the failed node recovers, the hinted data is sent to it.
This ensures high availability even during node outages.

------------------------------------------------------------------------------------
10. READ REPAIR

If coordinator detects that some replicas have stale data during a read,
it merges the latest version and writes it back to those replicas.
This helps the cluster converge faster to the latest state.

------------------------------------------------------------------------------------
11. EVENTUAL CONSISTENCY EXAMPLE

User adds an item to the cart from phone.
Replica A and B update successfully but C misses update due to network issue.
Later, user reads cart from laptop.
Coordinator fetches data from A, B, and C.
Finds C has older version.
Merges A and B version.
Returns merged cart to user.
Also updates C in background (read repair).
Eventually, all three replicas have the same data.

------------------------------------------------------------------------------------
12. CART MERGE STRATEGY

For multi device or offline updates, additive merge strategy is used.
Union of items across all versions is taken.
CRDT based item structure ensures deterministic merges.
Checkout operation handled separately with stronger consistency.

------------------------------------------------------------------------------------
13. TRADE OFFS

Strong quorum (R + W > N)
Strong consistency
Lower availability
Higher latency
Used for checkout and payment operations

Eventual consistency (R + W ≤ N)
Higher availability
Lower latency
May show stale data
Used for shopping cart updates and browsing

------------------------------------------------------------------------------------
14. FAILURE HANDLING

If node fails, writes still accepted on remaining replicas and stored as hinted handoff.
If network partition occurs, both sides accept writes and reconcile later using CRDT or Merkle tree.
If stale data read, read repair triggered automatically.
When node recovers, it receives hinted handoff data and syncs using anti entropy.

------------------------------------------------------------------------------------
15. CHECKOUT HANDLING

Checkout process is separate from cart updates.
Checkout requires strong consistency to avoid duplicate or missing orders.
Typically uses W = R = N for complete confirmation.
Cart operations remain eventually consistent.

------------------------------------------------------------------------------------
16. SUMMARY SNAPSHOT

N = Total replicas per key (includes coordinator)
Coordinator = Node handling client request for key
R and W = Control consistency level
Consistent Hashing = Handles partitioning and rebalancing
Replication = Ensures fault tolerance
Gossip = Propagates cluster state
Merkle Tree = Efficient replica repair
Hinted Handoff = Keeps availability during failures
Read Repair = Fixes stale copies during reads
CRDTs = Automatic merge and conflict free updates
Eventual Consistency = All replicas converge over time

------------------------------------------------------------------------------------
17. TYPICAL OPERATION FLOW

Write (PUT)
Client sends request to coordinator
Coordinator writes locally and forwards to N−1 replicas
Waits for W acknowledgments
Confirms success to client

Read (GET)
Client sends request to coordinator
Coordinator requests from all N replicas
Waits for R responses
Compares and merges versions
Returns merged result

------------------------------------------------------------------------------------
18. KEY FORMULAS AND RULES

Total replicas = N = coordinator + (N−1) others
Strong consistency condition = R + W > N
Eventual consistency condition = R + W ≤ N
Higher N → better durability
Lower R/W → higher availability

------------------------------------------------------------------------------------
19. WHY THIS DESIGN WORKS FOR AMAZON CART

Users add or remove items from multiple devices.
Offline changes or delayed writes can still merge safely.
Cart data is not mission critical, so eventual consistency is acceptable.
Availability is more important than instant accuracy.
Checkout handled separately with strong consistency.

Final Understanding
Each cart stored as key value on N replicas (coordinator plus N−1 others).
System uses consistent hashing, replication, gossip and CRDTs to stay scalable and decentralized.
Provides high availability and eventual consistency for user carts.
