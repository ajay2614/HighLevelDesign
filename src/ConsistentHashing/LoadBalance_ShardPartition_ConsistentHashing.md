DISTRIBUTED SYSTEMS CONCEPTS

LOAD BALANCING
Distributes client requests across multiple servers to avoid overload.
Example:
Server1, Server2, Server3
Requests go round robin:
R1 -> Server1
R2 -> Server2
R3 -> Server3
R4 -> Server1 (and repeat)
Goal: equal distribution of incoming load.

SHARDING / PARTITIONING
Divides a large dataset into smaller subsets (shards).
Example:
We have 4 shards (Shard0, Shard1, Shard2, Shard3)

If we use hash partitioning:
shard_id = hash(user_id) % 4

Then:
user_id = 10 -> hash(10) % 4 = 2 -> Shard2
user_id = 25 -> hash(25) % 4 = 1 -> Shard1

Problem:
If a new shard is added (now %5), all hashes change and data reshuffles.

CONSISTENT HASHING
Solves reshuffling problem.
Concept: Servers are placed on a circular hash ring (0 to M-1).
Keys are also hashed to same space.
Each key belongs to first server clockwise on ring.

Example:
M = 360 (like degrees)
hash(Server1) = 30°
hash(Server2) = 150°
hash(Server3) = 270°

hash(KeyA) = 100° -> goes to Server2
hash(KeyB) = 200° -> goes to Server3
hash(KeyC) = 20° -> goes to Server1

If Server2 is added:
hash(Server2) = 90°
Now only keys between 30°–90° move to Server2, rest remain same.
=> minimal data movement.

LOAD DISTRIBUTION
Suppose N = number of physical nodes. Each node gets roughly 1/N of the total hash space.
Load = number of keys in its range ≈ total keys / N

Example:
3 servers (Server1, Server2, Server3)
Total keys = 3000
Approximate load per node = 3000 / 3 = 1000 keys

HASH FUNCTION USED

Hash function (like h1) used to calculate token position on ring.

Short Example:
token = h1(ServerName) % M
key_token = h1(KeyName) % M

In Depth Example:
SERVER TOKEN
token = h1(ServerName) % M
ServerName = identifier of the server (e.g., "Server1")
h1() = hash function (Murmur3, MD5, SHA-1)
% M ensures value is within ring.
This token is position of server on the ring.

M = 360
ServerName = "Server1"
h1("Server1") = 1234
token = 1234 % 360 = 154
→ Server1 placed at 154° on ring

KEY TOKEN
key_token = h1(KeyName) % M
KeyName = identifier of data key (e.g., "user_101")
Same hash function used as for servers % M maps key to ring space
This token is position of key on ring

Example:
KeyName = "user_101"
h1("user_101") = 567
key_token = 567 % 360 = 207

STORE LOGIC
Place key on first server clockwise from key_token

Example:
Server tokens:
Server1 -> 154
Server2 -> 250
Server3 -> 330
Key token = 207
→ First server clockwise = Server2
→ Store key "user_101" on Server2

Both keys and servers are placed using same hash function.
Common hash functions: Murmur3, MD5, SHA-1, FNV

PROBLEM WITH CONSISTENT HASHING
Nodes can still get uneven load if their hash positions are uneven.
One node may own a large section of the ring.
If that node fails, its whole range moves to the next node, causing overload.
Nodes may get more than 1/N hash space, others less

SOLUTION: VIRTUAL NODES (Vnodes)
Each physical server is assigned multiple smaller ranges using the same hash function.

Example:
M = 360
Server1 -> h1("S1#1") % M = 20°, h1("S1#2") % M = 180°, h1("S1#3") % M = 300°
Server2 -> h1("S2#1") % M = 60°, h1("S2#2") % M = 240°
Server3 -> h1("S3#1") % M = 120°, h1("S3#2") % M = 330°

Now each physical node owns multiple small ranges.
When a key is hashed:
key_token = h1(Key) % M
Key is stored in first vnode clockwise from that token.

Example:
hash(KeyX) = 110° -> stored in Server3 (S3#1)
hash(KeyY) = 250° -> stored in Server2 (S2#2)
hash(KeyZ) = 310° -> stored in Server1 (S1#3)

WHEN A NODE FAILS
Without vnodes: 
Server2 (150°) fails → all keys in range 30°–150° move to Server3 (270°)
Server3 becomes overloaded.

With vnodes:
Server2 had vnodes at 60° and 240°.
If Server2 fails, its small ranges (60°–90°, 240°–270°) are taken by next vnodes clockwise.

Some go to Server3, some to Server1.

Load is divided evenly across all remaining nodes.

Example after failure:
Range 60°–90° -> taken by Server3 (S3#1)
Range 240°–270° -> taken by Server1 (S1#3)

Now load is shared, no single node overloaded.

Another Example
3 servers, each with 3 vnodes
Total keys = 3000
Total vnodes = 9
Each vnode ≈ 3000 / 9 = 333 keys
Server1 has 3 vnodes → 333*3 = 999 keys ≈ 1/3(Maintaining 1/N) of total load

WHY THIS IS BALANCED

Vnodes spread each server’s ownership randomly around ring.
When one server fails, its vnodes are reassigned across many nodes.
Each node takes only a small fraction of the failed node’s data.
If new server added, it receives a few vnodes from each existing node, balancing automatically.
Since Hash Function will give different values that means say for 3 nodes A, B and C, they could be on the ring like
A B C or A C B or B C A or B A C. By this if one node goes down requests are equally distributed.

Optional Cassandra-style example (real-world):
M = 2^64
Hash = Murmur3
Each node = 256 vnodes
Each vnode = token range
Data stored in SSTables on disk
Replication = next 2 vnodes clockwise
