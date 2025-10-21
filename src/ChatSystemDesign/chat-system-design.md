# Chat System Design - WhatsApp Architecture

## Table of Contents
1. [Back of Envelope Estimation](#back-of-envelope-estimation)
2. [Core Functional Requirements](#core-functional-requirements)
3. [Non-Functional Requirements](#non-functional-requirements)
4. [How Messages Work](#how-messages-work)
5. [How Group Chats Work](#how-group-chats-work)
6. [How Read Receipts Work](#how-read-receipts-work)
7. [Offline Use Cases](#offline-use-cases)
8. [Communication Protocols](#communication-protocols)
9. [Local Message IDs](#local-message-ids)
10. [Chat Server Architecture](#chat-server-architecture)
11. [Service Discovery - ZooKeeper](#service-discovery-zookeeper)
12. [Database Design](#database-design)
13. [High Level Design](#high-level-design)

---

## Back of Envelope Estimation

### Assumptions
- **Seconds in a day**: ~100,000 (actual: 86,400)
- **Daily Active Users (DAU)**: 500 million
- **Messages per user per day**: 50
- **Average message size**: 1 KB
- **Media files**: 5% of messages (average 10 KB each)

### Traffic Estimation

**Total messages per day**:
- 500M users × 50 messages = 25 billion messages/day

**Messages per second (QPS)**:
- 25 billion / 100,000 seconds = 250,000 messages/sec

**Peak QPS** (assume 2x average):
- 250,000 × 2 = 500,000 messages/sec

### Storage Estimation

**Text messages per day**:
- 25 billion × 1 KB = 25,000 GB = 25 TB/day

**Media files per day**:
- 25 billion × 5% = 1.25 billion media files
- 1.25 billion × 10 KB = 12,500 GB = 12.5 TB/day

**Total storage per day**: 25 + 12.5 = ~40 TB/day

**Annual storage**: 40 TB × 365 = ~15 PB/year

### Server Estimation

**WebSocket connections**:
- Each server can handle ~60K concurrent connections (65K ports - 5K reserved)
- Active connections needed: 500M DAU
- Servers required: 500M / 60K = ~8,500 servers

**With 20% buffer**: ~10,000 WebSocket servers

**Message processing servers**:
- Using formula: (Messages/sec × Latency) / Connections per server
- (500,000 × 0.020) / 100,000 = 100 servers
- **With redundancy**: ~150 servers

### Memory/Cache Estimation

**Assumption**: Cache last 10 messages per active chat for each user
- Average active chats per user: 20
- Messages to cache: 20 × 10 = 200 messages
- Size per cache: 200 × 1 KB = 200 KB per user
- Total cache: 500M × 200 KB = 100,000 GB = **100 TB RAM**

---

## Core Functional Requirements

### Must-Have Features
1. **One-on-one messaging**: Users can send text messages to each other
2. **Group chats**: Support conversations with up to 256 participants
3. **Message delivery acknowledgment**: Sent, delivered, and read receipts
4. **Media sharing**: Images, videos, audio files, documents
5. **Offline message storage**: Messages stored until successful delivery (30 days)
6. **Push notifications**: Notify offline users when they come online
7. **Last seen**: Show when user was last active
8. **Typing indicators**: Show when someone is typing

---

## Non-Functional Requirements

### Performance
- **Low latency**: Messages delivered within 100-200ms
- **High availability**: 99.99% uptime
- **Scalability**: Handle billions of messages per day

### Security
- **End-to-end encryption**: Only sender and receiver can read messages
- **Data privacy**: No intermediary can access message content

### Consistency
- **Message ordering**: Messages delivered in the order they were sent
- **Eventual consistency**: Accept eventual consistency for availability

---

## How Messages Work

### 1-to-1 Message Flow

```
User A (Sender) → WebSocket Handler → Message Service → Message Queue
                                                              ↓
                                                    Inbox Table (if offline)
                                                              ↓
User B (Receiver) ← WebSocket Handler ← Message Service ← Message Queue
```

### Detailed Steps

**Step 1: Sending a Message**
1. User A types message in client app
2. Client generates **local message ID** (timestamp + random)
3. Message sent to WebSocket Handler via persistent connection
4. Message shows **single tick** (sent) immediately with local ID

**Step 2: Server Processing**
1. WebSocket Handler receives message
2. Forwards to Message Service
3. Message Service generates **global message ID** (unique across system)
4. Stores message in **Message Table** (Cassandra/Database)
5. Returns acknowledgment to User A
6. User A sees **double tick** (delivered to server)

**Step 3: Recipient Delivery**
1. Message Service checks if User B is online (lookup in Redis)
2. **If online**:
   - Find User B's WebSocket Handler (using Redis mapping)
   - Push message through WebSocket
   - User B receives message
   - User B sends ACK back
   - Delete from Inbox table
   - User A sees **double blue tick** (read)
   
3. **If offline**:
   - Store in **Inbox Table** with User B's ID
   - When User B comes online, pull undelivered messages
   - Deliver all pending messages
   - Send ACK and update read status

---

## How Group Chats Work

### Group Message Flow

```
User A → WebSocket Handler → Group Service
                                    ↓
                          Message Fanout Service
                          /        |         \
                    User B    User C    User D (Inbox if offline)
```

### Database Schema for Groups

**ChatGroup Table**:
```
- group_id (PK)
- group_name
- created_at
- created_by
- last_message_id
```

**ChatGroupMember Table**:
```
- group_id (PK)
- user_id (PK)
- role (admin/member)
- joined_at
- last_read_message_id
```

**GroupMessage Table**:
```
- message_id (PK)
- group_id (indexed)
- sender_id
- message_content
- created_at
- message_type (text/media)
```

### Group Message Steps

1. **User A sends message to group** (group_id: 123)
2. Group Service fetches all members from ChatGroupMember table
3. **Fanout Service** creates individual delivery tasks for each member
4. For each online member:
   - Lookup WebSocket connection in Redis
   - Push message via WebSocket
5. For each offline member:
   - Write entry to **Inbox Table**
6. Store single message in GroupMessage table
7. Update `last_message_id` in ChatGroup table

### Optimization: Message Fanout
- **Write fanout at send time**: Create inbox entries immediately (WhatsApp approach)
- **Read fanout at fetch time**: Query group messages when user opens chat (Slack approach)
- WhatsApp uses **write fanout** for better read performance

---

## How Read Receipts Work

### Three States
1. **Single tick (✓)**: Message sent to server successfully
2. **Double tick (✓✓)**: Message delivered to recipient's device
3. **Blue double tick (✓✓)**: Message read by recipient

### Implementation

**Message Status Table**:
```
- message_id (PK)
- sender_id
- receiver_id
- status (SENT/DELIVERED/READ)
- sent_at
- delivered_at
- read_at
```

### Flow
1. User A sends message → status = SENT
2. Message reaches User B's device → status = DELIVERED (send ACK)
3. User B opens chat and views message → client calls `markAsRead()`
4. Server updates status = READ, records `read_at` timestamp
5. Server pushes read receipt to User A via WebSocket
6. User A sees blue ticks

### Group Chat Read Receipts
- Store read status per user in **GroupMessageStatus** table
- Show "Read by N people" count
- Display individual read receipts when tapped

```
GroupMessageStatus Table:
- message_id (PK)
- user_id (PK)
- read_at
```

---

## Offline Use Cases

### Problem
User B is offline when User A sends message. How to ensure delivery?

### Solution: Inbox Pattern

**Inbox Table Structure**:
```
- user_id (PK) - recipient
- message_id (SK) - sortable by timestamp
- sender_id
- chat_id (group or 1-1)
- message_content
- created_at
- ttl (30 days)
```

### Offline Message Flow

**When sending to offline user**:
1. Message Service checks Redis: `GET user:B:connection` → NULL
2. Store message in Inbox Table with `user_id = B`
3. Message persisted for 30 days (TTL)
4. When User B comes online:
   - Client connects to WebSocket Handler
   - Send sync request: "Give me undelivered messages"
   - Server queries: `SELECT * FROM Inbox WHERE user_id = B ORDER BY created_at`
   - Push all messages to User B
   - User B sends ACK for each message
   - Delete from Inbox table

**Push Notification**:
1. When message stored in Inbox, trigger push notification service
2. Use FCM (Firebase Cloud Messaging) for Android
3. Use APNS (Apple Push Notification Service) for iOS
4. User sees notification even when app is closed

### Optimization
- **Pagination**: Fetch 100 messages at a time
- **Priority delivery**: Deliver recent messages first
- **Compression**: Compress message batches

---

## Communication Protocols

### Why NOT HTTP Polling?

**Regular Polling**:
```
Client → Server: "Any new messages?" (every 1 sec)
Server → Client: "No" | "Yes, here: [messages]"
```

**Problems**:
- Wastes bandwidth (90% of requests return empty)
- High server load (millions of clients polling every second)
- Battery drain on mobile devices
- Latency: Up to 1 second delay

**Cost**: 500M users × 1 request/sec = 500M requests/sec (unnecessary!)

### Why NOT Long Polling?

**Long Polling Flow**:
```
Client → Server: "Any new messages?"
Server: (holds connection open for 30-60 seconds)
        (new message arrives)
Server → Client: "Yes, here: [message]"
Client → Server: (immediately reconnects) "Any new messages?"
```

**Problems**:
- Still requires reconnection after each response
- Doesn't support true bidirectional communication
- Server resources held open (threads/connections)
- Reconnection overhead

**Why it's better than regular polling**: Reduces unnecessary requests, but still not ideal.

### Why WebSockets WIN?

**WebSocket Flow**:
```
Client ←→ Server: (single persistent bidirectional connection)
```

**Advantages**:
1. **Persistent connection**: No reconnection overhead
2. **Bidirectional**: Server can push messages anytime
3. **Low latency**: Real-time delivery (<100ms)
4. **Efficient**: Single connection per user
5. **Less overhead**: No HTTP headers on every message

**Protocol Upgrade**:
```
Client → Server: HTTP GET with "Upgrade: websocket" header
Server → Client: HTTP 101 Switching Protocols
Client ←→ Server: WebSocket connection established
```

**Message Format** (lightweight):
```
{
  "type": "message",
  "chat_id": "123",
  "content": "Hello",
  "timestamp": 1698765432
}
```

---

## Local Message IDs

### Problem
User sends message but network is slow. How to show message immediately in chat?

### Solution: Optimistic UI with Local IDs

**Local Message ID Generation**:
```
local_id = timestamp + device_id + random_number
Example: 1698765432_device123_9876
```

**Flow**:
1. User types message and hits send
2. **Client generates local_id immediately**
3. Display message in chat UI with local_id (single tick)
4. Send message to server via WebSocket
5. Server generates **global message_id** (UUID or Snowflake ID)
6. Server responds with mapping: `local_id → global_id`
7. Client updates message in UI: replace local_id with global_id
8. Now message has global_id (double tick)

**Why important?**:
- **Instant feedback**: User sees message immediately
- **Offline support**: Can queue messages with local IDs
- **Idempotency**: Prevents duplicate messages (client can deduplicate by local_id)

### Message ID Generation (Server-side)

**Snowflake ID** (Twitter-style):
```
64 bits total:
- 41 bits: Timestamp (milliseconds)
- 10 bits: Machine ID
- 12 bits: Sequence number

Example: 1234567890123456789
```

**Benefits**:
- Globally unique
- Sortable by time
- Decentralized generation
- No database lookup needed

---

## Chat Server Architecture

### Why Separate Chat Servers?

**Traditional Monolith**: Single server handles everything
```
Client → [Web Server] (handles auth, messages, media, everything)
```

**Problems**:
- Can't scale WebSocket connections independently
- Business logic mixed with connection handling
- Hard to maintain

### Separation of Concerns

**Gateway/WebSocket Server**:
- Lightweight
- Handles ONLY WebSocket connections
- Keeps connections alive
- Routes messages

**Chat Service** (Business Logic):
- Stateless
- Handles message validation
- Stores messages in database
- Manages read receipts
- Can scale independently

```
Client ←→ [WebSocket Gateway] ←→ [Message Queue] ←→ [Chat Service] ←→ [Database]
        |                                              |
        |                                              |
   Connection                                   Business Logic
    Management                                    (Stateless)
```

### Benefits
1. **Scalability**: Scale WebSocket servers and business logic independently
2. **Resilience**: If Chat Service crashes, connections stay alive
3. **Load balancing**: Distribute connections across multiple gateways
4. **Simplicity**: Each component has single responsibility

### WebSocket Handler Responsibilities
- Accept WebSocket connections
- Maintain mapping: `user_id → connection_id`
- Store mapping in Redis for fast lookup
- Forward incoming messages to Message Queue
- Push outgoing messages from Message Queue to clients

### Message Queue (Kafka/RabbitMQ)
- Decouples WebSocket Gateway from Chat Service
- Provides buffering during traffic spikes
- Enables asynchronous processing
- Supports retry logic

---

## Service Discovery - ZooKeeper

### Problem
- Multiple WebSocket servers (10,000+)
- User A on Server 1, User B on Server 5000
- How does system find which server User B is on?

### ZooKeeper Solution

**What is ZooKeeper?**
- Distributed coordination service
- Maintains configuration and naming registry
- Provides distributed synchronization
- Used for service discovery

### User-to-Server Mapping

**Approach 1: Store in Redis** (WhatsApp approach)
```
Redis Key-Value:
user:123 → websocket_server:5000:port:8080
```

When User A sends message to User B:
1. Chat Service queries Redis: `GET user:123`
2. Gets: `websocket_server:5000:port:8080`
3. Routes message to WebSocket Server 5000

**Approach 2: ZooKeeper for Server Registry**

ZooKeeper maintains:
```
/websocket-servers
    /server-1 (ephemeral node)
        - host: 192.168.1.10
        - port: 8080
        - load: 45000 connections
    /server-2 (ephemeral node)
        - host: 192.168.1.11
        - port: 8080
        - load: 50000 connections
```

**Benefits of Ephemeral Nodes**:
- If server crashes, node auto-deleted
- System knows server is down immediately
- Can redirect new connections to other servers

### Service Discovery Flow

**When WebSocket Server starts**:
1. Register with ZooKeeper (create ephemeral node)
2. Register with Service Registry

**When user connects**:
1. Load Balancer queries ZooKeeper for available servers
2. Gets least loaded server
3. Directs user to that server
4. Server updates Redis: `SET user:123 server:5`

**When sending message**:
1. Chat Service queries Redis for receiver's server
2. If found, route to that server
3. If not found (offline), store in Inbox table

### ZooKeeper vs Redis

**ZooKeeper** (for server registry):
- Service discovery
- Health checks
- Distributed consensus
- Server coordination

**Redis** (for user-server mapping):
- Fast key-value lookups
- User session data
- Online/offline status
- Temporary data (with TTL)

**Combined approach**: Use both!
- ZooKeeper: Manages which servers exist
- Redis: Maps users to servers

---

## Database Design

### Database Choice

**User Data & Metadata**: PostgreSQL/MySQL
- User profiles
- Group metadata
- Relationships (contacts)
- ACID compliance needed

**Message Storage**: Cassandra / ScyllaDB
- High write throughput
- Horizontal scalability
- Time-series data (messages)
- No complex joins needed

**Cache**: Redis
- User-server mapping
- Online status
- Recent messages
- Session data

### Core Tables

**Users Table** (PostgreSQL):
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY,
    username VARCHAR(50) UNIQUE,
    phone_number VARCHAR(15) UNIQUE,
    password_hash VARCHAR(255),
    profile_picture_url TEXT,
    created_at TIMESTAMP,
    last_seen TIMESTAMP
);
```

**Chats Table** (PostgreSQL):
```sql
CREATE TABLE chats (
    chat_id UUID PRIMARY KEY,
    chat_type ENUM('ONE_TO_ONE', 'GROUP'),
    created_at TIMESTAMP,
    created_by UUID REFERENCES users(user_id)
);
```

**ChatParticipants Table** (PostgreSQL):
```sql
CREATE TABLE chat_participants (
    chat_id UUID REFERENCES chats(chat_id),
    user_id UUID REFERENCES users(user_id),
    joined_at TIMESTAMP,
    last_read_message_id UUID,
    PRIMARY KEY (chat_id, user_id)
);
```

**Messages Table** (Cassandra):
```
CREATE TABLE messages (
    message_id UUID,
    chat_id UUID,
    sender_id UUID,
    message_content TEXT,
    message_type VARCHAR(20),
    created_at TIMESTAMP,
    PRIMARY KEY (chat_id, created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

**Why Cassandra?**:
- Partition by `chat_id` (distributes load)
- Cluster by `created_at` (time-series ordering)
- Fast writes (append-only)
- Easy to scale horizontally

**Inbox Table** (Cassandra):
```
CREATE TABLE inbox (
    user_id UUID,
    message_id UUID,
    chat_id UUID,
    sender_id UUID,
    message_content TEXT,
    created_at TIMESTAMP,
    ttl INT DEFAULT 2592000, -- 30 days
    PRIMARY KEY (user_id, created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

**MessageStatus Table** (Cassandra):
```
CREATE TABLE message_status (
    message_id UUID,
    user_id UUID,
    status VARCHAR(20), -- SENT/DELIVERED/READ
    updated_at TIMESTAMP,
    PRIMARY KEY (message_id, user_id)
);
```

### Redis Data Structures

**User-Server Mapping**:
```
Key: user:{user_id}:connection
Value: {server_id}:{port}
TTL: Session duration
```

**Online Users Set**:
```
Key: online_users
Type: SET
Members: [user_id_1, user_id_2, ...]
```

**User Session**:
```
Key: session:{user_id}
Type: HASH
Fields: {
    server_id: "ws-server-5",
    connection_id: "conn_12345",
    last_active: "1698765432",
    device_type: "mobile"
}
```

**Recent Messages Cache**:
```
Key: chat:{chat_id}:recent_messages
Type: LIST
Values: [message_json_1, message_json_2, ...]
Limit: 50 messages
TTL: 24 hours
```

---

## High Level Design

### Complete Architecture

```
                                [Load Balancer]
                                      |
                    +----------------+----------------+
                    |                                 |
            [API Gateway]                    [WebSocket Gateway]
                    |                                 |
                    |                         [Connection Manager]
                    |                                 |
        +-----------+-----------+                     |
        |           |           |                     |
   [Auth      [User      [Group                 [Redis Cluster]
   Service]   Service]   Service]               (user-server mapping)
                    |           |                     |
                    +-----+-----+                     |
                          |                           |
                    [Message Service] ←---------------+
                          |
                    [Message Queue]
                    (Kafka/RabbitMQ)
                          |
        +-----------------+-----------------+
        |                 |                 |
   [Delivery         [Notification    [Media
    Service]          Service]         Service]
        |                 |                 |
        |                 |                 |
   [PostgreSQL]      [FCM/APNS]        [S3/CDN]
   [Cassandra]       
   [Redis]
```

### Component Descriptions

**Load Balancer**:
- Distributes HTTP/WebSocket traffic
- Health checks on servers
- SSL termination

**API Gateway**:
- REST APIs for non-real-time operations
- Authentication/Authorization
- Rate limiting

**WebSocket Gateway** (Stateful):
- Maintains WebSocket connections
- Lightweight message routing
- Updates Redis with connection info

**Connection Manager**:
- Tracks active connections
- Heartbeat/ping-pong for connection health
- Graceful disconnection handling

**Auth Service**:
- User registration/login
- JWT token generation
- Phone number verification

**User Service**:
- User profile management
- Contact list
- Last seen updates

**Group Service**:
- Create/delete groups
- Add/remove members
- Group metadata

**Message Service** (Core):
- Message validation
- Store in Cassandra
- Fanout logic for groups
- Read receipt updates

**Delivery Service**:
- Looks up recipient's server
- Pushes message via WebSocket
- Handles inbox for offline users

**Notification Service**:
- Push notifications (FCM/APNS)
- Email notifications
- SMS notifications

**Media Service**:
- Upload/download media files
- Image/video compression
- Store in S3 or CDN

**Message Queue** (Kafka):
- Decouples components
- Event streaming
- Replay capability

---

## Additional Features

### Typing Indicators

**Implementation**:
1. User starts typing → emit `typing_start` event
2. Client sends event to server (debounced every 3 sec)
3. Server broadcasts to other chat participants
4. Other users show "User A is typing..."
5. After 5 sec of no typing → emit `typing_stop`

**WebSocket Events**:
```json
// Typing start
{
  "type": "typing",
  "chat_id": "123",
  "user_id": "456",
  "status": "typing"
}

// Typing stop
{
  "type": "typing",
  "chat_id": "123",
  "user_id": "456",
  "status": "stopped"
}
```

**Optimization**: Use **volatile events** (not stored, can be dropped)

### Last Seen

**Update Strategy**:
1. Client sends heartbeat every 30 seconds
2. Server updates `last_seen` timestamp in Redis
3. On disconnect, persist to database

**Privacy Settings**:
```
last_seen_privacy:
  - EVERYONE
  - CONTACTS_ONLY
  - NOBODY
```

**Redis Structure**:
```
Key: user:{user_id}:last_seen
Value: timestamp
TTL: 7 days
```

---

## Key Takeaways

1. **Use WebSockets** for real-time bidirectional communication
2. **Separate concerns**: WebSocket Gateway (stateful) + Message Service (stateless)
3. **Redis for speed**: User-server mapping, online status, cache
4. **Cassandra for scale**: Message storage with time-series ordering
5. **Inbox pattern**: Reliable offline message delivery
6. **Local message IDs**: Optimistic UI for instant feedback
7. **ZooKeeper**: Service discovery and coordination
8. **Message Queue**: Decouple components, handle spikes
9. **Fanout on write**: Better read performance for groups
10. **Back of envelope**: Always estimate scale before designing

---

## References
- WhatsApp handles 100+ billion messages per day
- Each WebSocket server: ~60K concurrent connections
- Message latency: <100ms for online users
- Storage: Cassandra for high write throughput
- End-to-end encryption: Signal Protocol

