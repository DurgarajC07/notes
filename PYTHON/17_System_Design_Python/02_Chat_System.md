# 💬 Chat System Design

## Overview

Design a scalable real-time chat system supporting 1-on-1 and group messaging with presence, typing indicators, and message history.

---

## Requirements

### Functional Requirements

- **1-on-1 messaging**: Direct messages between two users
- **Group messaging**: Channels with multiple participants
- **Real-time delivery**: Messages appear instantly
- **Message history**: Store and retrieve past messages
- **Presence**: Show online/offline status
- **Typing indicators**: Show when someone is typing
- **Read receipts**: Show when message is read
- **Media support**: Images, videos, files

### Non-Functional Requirements

- **Scalability**: 100M daily active users (DAU)
- **Availability**: 99.99% uptime
- **Low latency**: <100ms message delivery
- **Consistency**: Message ordering preserved
- **Durability**: Messages never lost

---

## Capacity Estimation

### Traffic

```python
"""
Users:
- 100M DAU (Daily Active Users)
- 40% online at peak (40M concurrent)
- Average 50 messages/day per user

Messages:
- Total: 100M × 50 = 5B messages/day
- Per second: 5B / 86400 ≈ 58K messages/sec
- Peak (3x): 174K messages/sec

Connections:
- WebSocket per online user: 40M concurrent connections
- Need horizontal scaling
"""
```

### Storage

```python
"""
Message Storage:
- Message size: ~500 bytes (text + metadata)
- Per day: 5B × 500 bytes = 2.5 TB/day
- Per year: 2.5 TB × 365 ≈ 900 TB/year
- With replication (3x): 2.7 PB/year

Media Storage:
- 10% messages have media (500M/day)
- Average media size: 1 MB
- Per day: 500M × 1 MB = 500 TB/day
- Use CDN + object storage (S3)

Bandwidth:
- Ingress: 58K req/sec × 500 bytes ≈ 30 MB/sec
- Egress: 2-5x (group chats, read receipts)
"""
```

---

## High-Level Architecture

```python
"""
Components:

1. Client (Web/Mobile)
   ├── WebSocket connection
   └── HTTP for history/media

2. Load Balancer
   └── Route to chat servers

3. Chat Servers
   ├── Maintain WebSocket connections
   ├── Route messages
   └── Handle presence

4. Message Queue (Kafka)
   ├── Buffer messages
   └── Decouple services

5. Message Service
   ├── Store messages
   └── Fanout to recipients

6. Presence Service
   ├── Track online status
   └── Heartbeat

7. Database
   ├── Cassandra (messages)
   └── Redis (online users, typing)

8. Object Storage (S3)
   └── Media files

9. CDN
   └── Deliver media
"""
```

---

## WebSocket Communication

### Connection Management

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect
from typing import Dict, Set
import asyncio
import json

app = FastAPI()

class ConnectionManager:
    """
    Manage WebSocket connections

    Responsibilities:
    - Track active connections per user
    - Send messages to connected users
    - Handle disconnections
    """

    def __init__(self):
        # user_id -> set of WebSocket connections
        # Multiple devices per user possible
        self.active_connections: Dict[int, Set[WebSocket]] = {}

    async def connect(self, user_id: int, websocket: WebSocket):
        """
        Connect user

        Accept WebSocket and add to active connections.
        """
        await websocket.accept()

        if user_id not in self.active_connections:
            self.active_connections[user_id] = set()

        self.active_connections[user_id].add(websocket)

        # Notify presence service
        await self._update_presence(user_id, online=True)

    def disconnect(self, user_id: int, websocket: WebSocket):
        """Disconnect user"""
        if user_id in self.active_connections:
            self.active_connections[user_id].discard(websocket)

            # Remove user if no more connections
            if not self.active_connections[user_id]:
                del self.active_connections[user_id]
                asyncio.create_task(self._update_presence(user_id, online=False))

    async def send_to_user(self, user_id: int, message: dict):
        """
        Send message to user (all devices)

        If user offline, message already in DB for later retrieval.
        """
        if user_id in self.active_connections:
            # Send to all user's devices
            disconnected = set()

            for websocket in self.active_connections[user_id]:
                try:
                    await websocket.send_json(message)
                except:
                    # Connection broken
                    disconnected.add(websocket)

            # Clean up broken connections
            for ws in disconnected:
                self.disconnect(user_id, ws)

    async def broadcast_to_group(self, user_ids: list, message: dict):
        """Broadcast message to multiple users"""
        tasks = [
            self.send_to_user(user_id, message)
            for user_id in user_ids
        ]
        await asyncio.gather(*tasks, return_exceptions=True)

    def is_online(self, user_id: int) -> bool:
        """Check if user is online"""
        return user_id in self.active_connections

    async def _update_presence(self, user_id: int, online: bool):
        """Update presence in Redis"""
        # Publish to presence service
        pass

manager = ConnectionManager()

@app.websocket("/ws/{user_id}")
async def websocket_endpoint(websocket: WebSocket, user_id: int):
    """
    WebSocket endpoint for chat

    Flow:
    1. Accept connection
    2. Receive messages from client
    3. Process and route messages
    4. Handle disconnection
    """
    await manager.connect(user_id, websocket)

    try:
        while True:
            # Receive message from client
            data = await websocket.receive_json()

            # Process message based on type
            await handle_message(user_id, data)

    except WebSocketDisconnect:
        manager.disconnect(user_id, websocket)
    except Exception as e:
        print(f"Error: {e}")
        manager.disconnect(user_id, websocket)

async def handle_message(sender_id: int, data: dict):
    """
    Handle incoming message

    Message types:
    - chat: Regular message
    - typing: Typing indicator
    - read: Read receipt
    """
    msg_type = data.get('type')

    if msg_type == 'chat':
        await handle_chat_message(sender_id, data)
    elif msg_type == 'typing':
        await handle_typing(sender_id, data)
    elif msg_type == 'read':
        await handle_read_receipt(sender_id, data)

async def handle_chat_message(sender_id: int, data: dict):
    """
    Handle chat message

    Steps:
    1. Validate message
    2. Save to database
    3. Route to recipient(s)
    4. Send acknowledgment to sender
    """
    recipient_id = data.get('recipient_id')
    group_id = data.get('group_id')
    content = data.get('content')

    # Create message record
    message = {
        'id': generate_message_id(),
        'sender_id': sender_id,
        'recipient_id': recipient_id,
        'group_id': group_id,
        'content': content,
        'timestamp': int(time.time() * 1000),
        'type': 'text'
    }

    # Save to database
    await save_message(message)

    # Route to recipients
    if recipient_id:
        # 1-on-1 message
        await manager.send_to_user(recipient_id, {
            'type': 'message',
            'data': message
        })
    elif group_id:
        # Group message - fanout
        members = await get_group_members(group_id)
        await manager.broadcast_to_group(members, {
            'type': 'message',
            'data': message
        })

    # Acknowledge to sender
    await manager.send_to_user(sender_id, {
        'type': 'ack',
        'message_id': message['id']
    })

async def handle_typing(sender_id: int, data: dict):
    """
    Handle typing indicator

    Ephemeral - not stored, just forwarded.
    """
    recipient_id = data.get('recipient_id')
    is_typing = data.get('is_typing')

    await manager.send_to_user(recipient_id, {
        'type': 'typing',
        'user_id': sender_id,
        'is_typing': is_typing
    })

async def handle_read_receipt(sender_id: int, data: dict):
    """Handle read receipt"""
    message_id = data.get('message_id')

    # Update message as read
    await mark_message_read(message_id, sender_id)

    # Notify original sender
    message = await get_message(message_id)
    await manager.send_to_user(message['sender_id'], {
        'type': 'read_receipt',
        'message_id': message_id,
        'read_by': sender_id
    })
```

---

## Message Storage

### Database Schema (Cassandra)

```python
"""
Messages Table:

CREATE TABLE messages (
    conversation_id UUID,      -- Partition key (1-on-1 or group)
    timestamp TIMESTAMP,       -- Clustering key (ordering)
    message_id UUID,
    sender_id BIGINT,
    content TEXT,
    media_url TEXT,
    status TEXT,               -- sent/delivered/read
    PRIMARY KEY (conversation_id, timestamp, message_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Query pattern: Get messages for conversation
SELECT * FROM messages
WHERE conversation_id = ?
ORDER BY timestamp DESC
LIMIT 50;

Why Cassandra?
✅ Horizontal scaling
✅ High write throughput
✅ Time-series data (messages)
✅ Partition by conversation
✅ Fast range queries

Conversations Table:

CREATE TABLE conversations (
    user_id BIGINT,           -- Partition key
    conversation_id UUID,     -- Clustering key
    last_message_time TIMESTAMP,
    unread_count INT,
    PRIMARY KEY (user_id, last_message_time, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_time DESC);

-- Query: Get user's conversations ordered by recent
SELECT * FROM conversations
WHERE user_id = ?
ORDER BY last_message_time DESC
LIMIT 20;
"""

from cassandra.cluster import Cluster
from cassandra.query import SimpleStatement
import uuid
from datetime import datetime

class MessageStore:
    """Message storage using Cassandra"""

    def __init__(self):
        self.cluster = Cluster(['cassandra1', 'cassandra2', 'cassandra3'])
        self.session = self.cluster.connect('chat')

    async def save_message(self, message: dict):
        """
        Save message to Cassandra

        Partition by conversation for efficient queries.
        """
        query = """
        INSERT INTO messages
        (conversation_id, timestamp, message_id, sender_id, content, media_url, status)
        VALUES (?, ?, ?, ?, ?, ?, ?)
        """

        conversation_id = self._get_conversation_id(
            message.get('recipient_id'),
            message.get('group_id'),
            message.get('sender_id')
        )

        self.session.execute(query, (
            conversation_id,
            datetime.fromtimestamp(message['timestamp'] / 1000),
            uuid.UUID(message['id']),
            message['sender_id'],
            message['content'],
            message.get('media_url'),
            'sent'
        ))

        # Update conversation
        await self._update_conversation(
            message['sender_id'],
            conversation_id,
            message['timestamp']
        )

    async def get_messages(
        self,
        conversation_id: str,
        limit: int = 50,
        before_timestamp: int = None
    ) -> list:
        """
        Get messages for conversation

        Pagination: Use before_timestamp for older messages.
        """
        if before_timestamp:
            query = """
            SELECT * FROM messages
            WHERE conversation_id = ? AND timestamp < ?
            ORDER BY timestamp DESC
            LIMIT ?
            """
            params = (
                uuid.UUID(conversation_id),
                datetime.fromtimestamp(before_timestamp / 1000),
                limit
            )
        else:
            query = """
            SELECT * FROM messages
            WHERE conversation_id = ?
            ORDER BY timestamp DESC
            LIMIT ?
            """
            params = (uuid.UUID(conversation_id), limit)

        rows = self.session.execute(query, params)

        return [
            {
                'id': str(row.message_id),
                'sender_id': row.sender_id,
                'content': row.content,
                'timestamp': int(row.timestamp.timestamp() * 1000),
                'status': row.status
            }
            for row in rows
        ]

    def _get_conversation_id(
        self,
        recipient_id: int,
        group_id: int,
        sender_id: int
    ) -> uuid.UUID:
        """
        Generate conversation ID

        1-on-1: Sort IDs for consistency (A->B and B->A same conversation)
        Group: Use group ID
        """
        if group_id:
            return uuid.uuid5(uuid.NAMESPACE_DNS, f"group_{group_id}")
        else:
            user_ids = sorted([sender_id, recipient_id])
            return uuid.uuid5(uuid.NAMESPACE_DNS, f"dm_{user_ids[0]}_{user_ids[1]}")
```

---

## Presence Service

### Redis for Online Status

```python
import redis
from typing import Set

class PresenceService:
    """
    Track user online status

    Using Redis:
    - SET: online_users (set of user IDs)
    - HASH: user:{id} -> {last_seen, status}
    - TTL: Heartbeat mechanism
    """

    def __init__(self):
        self.redis = redis.Redis(host='redis', port=6379)
        self.HEARTBEAT_INTERVAL = 30  # seconds
        self.ONLINE_THRESHOLD = 45    # Consider offline after 45s

    async def user_online(self, user_id: int):
        """
        Mark user as online

        Set with TTL - auto-expire if no heartbeat.
        """
        # Add to online users set
        self.redis.sadd('online_users', user_id)

        # Set user status with TTL
        self.redis.hset(f'user:{user_id}', 'status', 'online')
        self.redis.hset(f'user:{user_id}', 'last_seen', int(time.time()))
        self.redis.expire(f'user:{user_id}', self.ONLINE_THRESHOLD)

    async def user_offline(self, user_id: int):
        """Mark user as offline"""
        self.redis.srem('online_users', user_id)
        self.redis.hset(f'user:{user_id}', 'status', 'offline')
        self.redis.hset(f'user:{user_id}', 'last_seen', int(time.time()))

    async def heartbeat(self, user_id: int):
        """
        Heartbeat to keep user online

        Client sends every 30s.
        """
        await self.user_online(user_id)

    def is_online(self, user_id: int) -> bool:
        """Check if user is online"""
        return self.redis.sismember('online_users', user_id)

    async def get_online_users(self, user_ids: list) -> Set[int]:
        """Get which users from list are online"""
        if not user_ids:
            return set()

        # Use pipeline for efficiency
        pipeline = self.redis.pipeline()

        for user_id in user_ids:
            pipeline.sismember('online_users', user_id)

        results = pipeline.execute()

        return {
            user_id
            for user_id, is_online in zip(user_ids, results)
            if is_online
        }
```

---

## Scalability Considerations

### Sharding Strategy

```python
"""
Problem: 40M concurrent WebSocket connections

Solution: Horizontal scaling

1. Consistent Hashing:
   - Hash user_id -> chat server
   - User always connects to same server
   - Server stores user's WebSocket connection

2. Service Discovery:
   - Load balancer tracks which server has which user
   - Redis: user_id -> server_address
   - When sending message, lookup server and forward

3. Server-to-Server Communication:
   - If recipient on different server, use RPC/message queue
   - Server A receives message for user on Server B
   - Server A publishes to Kafka topic
   - Server B consumes and delivers to user

Implementation:

class MessageRouter:
    def __init__(self):
        self.redis = redis.Redis()

    async def route_message(self, recipient_id: int, message: dict):
        # Check if user on this server
        if manager.is_online(recipient_id):
            await manager.send_to_user(recipient_id, message)
        else:
            # Check if user online on another server
            server = self.redis.get(f'user_server:{recipient_id}')

            if server:
                # Forward to other server via message queue
                await self.publish_to_queue(server, recipient_id, message)
            # else: user offline, will get from DB later

    async def publish_to_queue(self, server: str, user_id: int, message: dict):
        # Publish to Kafka topic for target server
        pass
"""
```

### Message Ordering

```python
"""
Challenge: Preserve message order in distributed system

Solutions:

1. Timestamp + Sequence:
   - Each message has timestamp + sequence number
   - Sequence per sender per conversation
   - Sort by (timestamp, sequence) on client

2. Lamport Timestamps:
   - Logical clocks for causality
   - Ensures happened-before relationship

3. Single Writer:
   - All messages for conversation go through same partition
   - Kafka partition by conversation_id
   - Preserves order within partition

Example:

message = {
    'id': str(uuid.uuid4()),
    'sender_id': sender_id,
    'conversation_id': conversation_id,
    'timestamp': int(time.time() * 1000),
    'sequence': get_next_sequence(sender_id, conversation_id),
    'content': content
}

def get_next_sequence(sender_id, conversation_id):
    # Atomic increment in Redis
    key = f'seq:{sender_id}:{conversation_id}'
    return redis.incr(key)
"""
```

---

## Interview Questions

### Q1: How do you handle message delivery guarantees?

**Answer**:

- **At-most-once**: Send and forget - may lose messages
- **At-least-once**: Retry until ack - may duplicate
- **Exactly-once**: Idempotent + deduplication
- **Implementation**: Message ID, store sent messages, retry with exponential backoff
- **Trade-off**: Exactly-once complex, at-least-once acceptable with dedup

### Q2: How do you scale WebSocket connections?

**Answer**:

- **Problem**: 40M concurrent connections
- **Solution**: Horizontal scaling with load balancing
- **Challenges**: Routing messages between servers
- **Approach**: Consistent hashing, service discovery, server-to-server communication
- **Message queue**: Kafka for async communication between servers

### Q3: Why Cassandra for message storage?

**Answer**:

- **Write-heavy**: Chat is write-intensive
- **Time-series**: Messages ordered by time
- **Partitioning**: By conversation_id for locality
- **Scalability**: Horizontal scaling, no SPOF
- **Query pattern**: Range queries within partition efficient

### Q4: How to handle read receipts at scale?

**Answer**:

- **Challenge**: Update all group members
- **Solution**: Batch updates, eventual consistency
- **Optimization**: Only show "seen by X people", not each individually
- **Database**: Update counter, not individual records
- **Real-time**: WebSocket notification, DB update async

### Q5: How do you ensure message ordering?

**Answer**:

- **Timestamp**: Server timestamp (not client to avoid clock skew)
- **Sequence number**: Per sender per conversation
- **Kafka partitioning**: Same conversation same partition
- **Client sorting**: Sort by (timestamp, sequence)
- **Idempotency**: Handle duplicates with message ID

---

## Summary

Chat system design:

- **WebSocket**: Real-time bidirectional communication
- **Cassandra**: Scalable message storage
- **Redis**: Presence and caching
- **Kafka**: Message queue for server communication
- **Horizontal scaling**: Consistent hashing, service discovery
- **Ordering**: Timestamp + sequence number
- **Reliability**: Ack, retry, idempotency

Design for scale from day one! 💬
