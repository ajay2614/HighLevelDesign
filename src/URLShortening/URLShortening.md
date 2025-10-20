URL SHORTENING – COMPLETE NOTES

URL shortening means converting a long URL into a short unique alias which redirects to the original link.
For example:
https://www.amazon.com/products/item123?ref=abc
→ https://short.ly/k9L2bXy

Goals are scalability, uniqueness, decentralization, and fast lookups.

Short URLs usually consist of a short domain and a Base62-encoded ID.
Base62 uses characters 0-9, a-z, A-Z which gives 62 possibilities per character.
So with 6 characters → 62^6 ≈ 5.68 × 10^10 combinations (about 56 billion).
If we increase to 7 or 8 characters, combinations grow massively.

WHY WE CALCULATE URLS PER DAY OR PER YEAR

When designing a shortener, we estimate how many URLs will be created per day or per year to decide
how many unique IDs we need. For example, if 10 million new URLs are shortened every day, then
we need 10 million unique IDs daily. Over a year → about 3.65 billion.
This helps decide how many characters (in Base62) we must support.
It does not mean the same URL is shortened multiple times per hit — it means new unique URLs are shortened.

WHY BIG COMPANIES HAVE LONG LINKS

Websites like Amazon or Flipkart often have long URLs because they include query parameters for
tracking, session data, recommendations, referral IDs, user data, and filters.
The shortener just hides all that complexity by creating a short alias that redirects to the long URL.

HOW BASE62 IS USED

Each shortened URL stores an integer ID. That ID is converted to Base62 to make it short and URL-safe.
Example:
ID 100000 → Base62 = q0U
The Base62 string is appended to the domain: short.ly/q0U

When user visits short.ly/q0U → decode Base62 back to integer ID → look up in database → redirect to original URL.

HASHING APPROACHES

Hashing the original long URL using MD5 or SHA-1 gives a fixed 128-bit or 160-bit hash.
Convert that hash into Base62 and take the first few characters (e.g., 6).
Gives a short code that looks random and consistent for the same URL.
Collision can occur due to truncation → DB lookup required.

Problems:

Two different URLs may produce the same truncated hash → collision
Requires DB check
Not sequential or time-ordered → harder for sharding

SNOWFLAKE ID APPROACH

Snowflake IDs generate unique 64-bit numbers across nodes without central coordination.
Structure (64 bits):
41 bits timestamp in ms
10 bits node ID
12 bits sequence number per ms

Sequence number per ms allows multiple IDs per millisecond per node (up to 4096 IDs/ms).
Each node can generate IDs independently → fully decentralized.

Example: timestamp=1697664000000, node=12, seq=7 → 240897563456789 → Base62 → k9L2bXy
Base62 encoding of Snowflake IDs ≈ 11–12 chars → no collision, no DB lookup needed.

Optional hashing/truncation:
To get 6-char codes, hash or take lower bits → check DB for collisions.

Full Snowflake → Base62 → 11–12 chars → unique, no collision.

SEQUENCE NUMBER PER MILLISECOND
Ensures no duplicate IDs even when multiple IDs are generated in same ms on a node.
Sequence resets to 0 on next millisecond.
Combined with timestamp and node ID → globally unique IDs.

TICKET SERVER VS ZOOKEEPER

Ticket Server:

Centralized server that hands out blocks of IDs to different application servers.
Workflow: server requests N IDs → ticket server returns a block → server uses IDs locally.
Pros: simple, ensures uniqueness within blocks.
Cons: centralized → single point of failure. If ticket server is down, new IDs cannot be allocated.

Zookeeper:

Distributed, replicated coordination service.
Manages ID allocation using ephemeral nodes and sequences.
Servers request ID ranges from Zookeeper → Zookeeper ensures uniqueness and availability.

Pros: fault-tolerant, highly available, prevents single-point-of-failure.
Cons: more complex, requires Zookeeper cluster.

Both approaches preallocate ranges to avoid collisions.
Early ranges may cause hotspot load on DB shards, but Zookeeper mitigates centralization risk compared to a single ticket server.

Usually Snowflake or Zookeeper are preferred approaches for distributed systems

KEY TRADE-OFFS

Short codes of 6 chars → collision possible → DB lookup required.
Full Snowflake → 11–12 chars → no collision → no lookup needed.
Hashing or truncation helps shorten code length but adds DB checking overhead.
Ticket server → simple, centralized, may fail.
Zookeeper → distributed, fault-tolerant, more complex.
Snowflake → fully decentralized, scalable, deterministic, high throughput.

FINAL SUMMARY

URL shorteners map long URLs to short unique Base62 codes.
Base62 keeps URLs short and readable.
Daily URL volume helps decide required character length.
Hashing gives deterministic short codes but may collide if truncated.
Ticket server allocates ID blocks centrally, Zookeeper allocates ranges in a distributed, fault-tolerant way.
Snowflake generates unique 64-bit IDs using timestamp, node ID, and sequence per ms.
Full Snowflake ID → Base62 ≈ 11–12 chars → unique, no collisions.
Truncated Snowflake / hashed ID → 6 chars → collisions possible → DB lookup needed.
Sharding must consider sequential IDs → hash-based sharding avoids hotspots.
Snowflake is widely used by systems like Twitter and Bitly.
Zookeeper/ticket servers are used in legacy or preallocated-range systems.