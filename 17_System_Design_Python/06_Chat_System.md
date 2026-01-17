# System Design: Real-Time Chat System

## 📖 Problem Statement

Design a real-time chat system like WhatsApp, Slack, or Discord that:

- Supports one-on-one and group chats
- Delivers messages in real-time (< 1 second)
- Stores message history persistently
- Shows online/offline status and typing indicators
- Handles millions of concurrent users
- Provides read receipts and message delivery status
- Supports multimedia messages (images, files)

## 🎯 Requirements

### Functional Requirements

1. Send and receive messages in real-time
2. One-on-one and group chat rooms
3. Message history and search
4. Online presence and typing indicators
5. Read receipts (delivered, read status)
6. Push notifications for offline users
7. File attachments and media sharing

### Non-Functional Requirements

1. **Low Latency**: < 1 second message delivery
2. **High Availability**: 99.9% uptime
3. **Scalability**: Millions of concurrent connections
4. **Consistency**: Messages delivered in order
5. **Persistence**: Messages stored permanently
6. **Security**: End-to-end encryption (optional)

## 📊 Architecture

```
         ┌─────────────┐
         │   Client    │
         │ (WebSocket) │
         └──────┬──────┘
                │
                ▼
     ┌──────────────────┐
     │  Load Balancer   │
     └────────┬─────────┘
              │
       ┌──────┴──────┐
       │             │
       ▼             ▼
  ┌────────┐   ┌────────┐
  │  Chat  │   │  Chat  │
  │Server 1│   │Server 2│
  └────┬───┘   └───┬────┘
       │           │
       └─────┬─────┘
             │
        ┌────┴────┐
        │  Redis  │  ← Pub/Sub for cross-server messaging
        │ Pub/Sub │
        └─────────┘
             │
        ┌────┴────┐
        │Database │  ← PostgreSQL/MongoDB for persistence
        │ (Msgs)  │
        └─────────┘
```

## 🔑 Core Components

### 1. WebSocket Connection Manager

```python
import asyncio
import json
from typing import Dict, Set
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class Connection:
    """Represents a WebSocket connection"""

    def __init__(self, websocket, user_id: str):
        self.websocket = websocket
        self.user_id = user_id
        self.rooms: Set[str] = set()
        self.connected_at = datetime.utcnow()

class ConnectionManager:
    """
    Manages WebSocket connections

    - Tracks active connections per user
    - Handles connection lifecycle
    - Routes messages to appropriate connections
    """

    def __init__(self):
        """Initialize connection manager"""
        # user_id -> list of connections (multiple devices)
        self.active_connections: Dict[str, list[Connection]] = {}

        # room_id -> set of user_ids
        self.room_members: Dict[str, Set[str]] = {}

        self.lock = asyncio.Lock()

    async def connect(self, websocket, user_id: str) -> Connection:
        """
        Register new connection

        Args:
            websocket: WebSocket connection
            user_id: User ID

        Returns:
            Connection object
        """
        async with self.lock:
            connection = Connection(websocket, user_id)

            if user_id not in self.active_connections:
                self.active_connections[user_id] = []

            self.active_connections[user_id].append(connection)

            logger.info(f"User {user_id} connected (total: {len(self.active_connections[user_id])} devices)")

            return connection

    async def disconnect(self, connection: Connection):
        """
        Remove connection

        Args:
            connection: Connection to remove
        """
        async with self.lock:
            user_id = connection.user_id

            if user_id in self.active_connections:
                self.active_connections[user_id].remove(connection)

                if not self.active_connections[user_id]:
                    del self.active_connections[user_id]
                    logger.info(f"User {user_id} fully disconnected")

    async def join_room(self, user_id: str, room_id: str):
        """
        Add user to room

        Args:
            user_id: User ID
            room_id: Room ID
        """
        async with self.lock:
            if room_id not in self.room_members:
                self.room_members[room_id] = set()

            self.room_members[room_id].add(user_id)

            # Add room to all user's connections
            if user_id in self.active_connections:
                for conn in self.active_connections[user_id]:
                    conn.rooms.add(room_id)

            logger.info(f"User {user_id} joined room {room_id}")

    async def leave_room(self, user_id: str, room_id: str):
        """
        Remove user from room

        Args:
            user_id: User ID
            room_id: Room ID
        """
        async with self.lock:
            if room_id in self.room_members:
                self.room_members[room_id].discard(user_id)

            # Remove room from all user's connections
            if user_id in self.active_connections:
                for conn in self.active_connections[user_id]:
                    conn.rooms.discard(room_id)

    async def send_to_user(self, user_id: str, message: dict):
        """
        Send message to all user's devices

        Args:
            user_id: Target user ID
            message: Message to send
        """
        if user_id not in self.active_connections:
            return

        # Send to all user's devices
        for connection in self.active_connections[user_id]:
            try:
                await connection.websocket.send(json.dumps(message))
            except Exception as e:
                logger.error(f"Error sending to user {user_id}: {e}")

    async def send_to_room(self, room_id: str, message: dict, exclude_user: str = None):
        """
        Send message to all room members

        Args:
            room_id: Target room ID
            message: Message to send
            exclude_user: Optional user to exclude (sender)
        """
        if room_id not in self.room_members:
            return

        # Send to all room members
        for user_id in self.room_members[room_id]:
            if user_id != exclude_user:
                await self.send_to_user(user_id, message)

    def is_user_online(self, user_id: str) -> bool:
        """Check if user is online"""
        return user_id in self.active_connections

    def get_online_users(self, room_id: str) -> Set[str]:
        """Get online users in room"""
        if room_id not in self.room_members:
            return set()

        return {
            user_id
            for user_id in self.room_members[room_id]
            if self.is_user_online(user_id)
        }

# Global connection manager
manager = ConnectionManager()
```

### 2. Message Model and Storage

```python
from dataclasses import dataclass, asdict
from typing import Optional
import uuid
from datetime import datetime

@dataclass
class Message:
    """Chat message"""

    id: str
    room_id: str
    sender_id: str
    content: str
    timestamp: datetime
    message_type: str = "text"  # text, image, file
    reply_to: Optional[str] = None  # Reply to message ID

    @classmethod
    def create(cls, room_id: str, sender_id: str, content: str, **kwargs) -> 'Message':
        """Create new message"""
        return cls(
            id=str(uuid.uuid4()),
            room_id=room_id,
            sender_id=sender_id,
            content=content,
            timestamp=datetime.utcnow(),
            **kwargs
        )

    def to_dict(self) -> dict:
        """Convert to dictionary"""
        data = asdict(self)
        data['timestamp'] = self.timestamp.isoformat()
        return data

class MessageStore:
    """
    Message persistence layer

    In production: Use PostgreSQL, MongoDB, or Cassandra
    """

    def __init__(self):
        """Initialize message store"""
        # In-memory storage (use database in production)
        self.messages: Dict[str, Message] = {}
        self.room_messages: Dict[str, list[str]] = {}  # room_id -> message_ids

    async def save_message(self, message: Message):
        """
        Save message to storage

        Args:
            message: Message to save
        """
        self.messages[message.id] = message

        if message.room_id not in self.room_messages:
            self.room_messages[message.room_id] = []

        self.room_messages[message.room_id].append(message.id)

        logger.info(f"Saved message {message.id} to room {message.room_id}")

    async def get_message(self, message_id: str) -> Optional[Message]:
        """Get message by ID"""
        return self.messages.get(message_id)

    async def get_room_messages(
        self,
        room_id: str,
        limit: int = 50,
        before: Optional[str] = None
    ) -> list[Message]:
        """
        Get messages for room

        Args:
            room_id: Room ID
            limit: Maximum messages to return
            before: Get messages before this message ID (pagination)

        Returns:
            List of messages
        """
        if room_id not in self.room_messages:
            return []

        message_ids = self.room_messages[room_id]

        # Find starting position
        if before:
            try:
                before_index = message_ids.index(before)
                message_ids = message_ids[:before_index]
            except ValueError:
                pass

        # Get last N messages
        message_ids = message_ids[-limit:]

        messages = [
            self.messages[msg_id]
            for msg_id in message_ids
            if msg_id in self.messages
        ]

        return messages

    async def search_messages(self, room_id: str, query: str) -> list[Message]:
        """
        Search messages in room

        Args:
            room_id: Room ID
            query: Search query

        Returns:
            Matching messages
        """
        if room_id not in self.room_messages:
            return []

        messages = []

        for msg_id in self.room_messages[room_id]:
            message = self.messages.get(msg_id)

            if message and query.lower() in message.content.lower():
                messages.append(message)

        return messages

# Global message store
message_store = MessageStore()
```

### 3. Presence and Typing Indicators

```python
import time
from typing import Dict

class PresenceManager:
    """
    Manages user presence (online/offline status)

    In production: Use Redis with TTL
    """

    def __init__(self):
        """Initialize presence manager"""
        # user_id -> last_seen timestamp
        self.last_seen: Dict[str, float] = {}

        # (user_id, room_id) -> typing_started timestamp
        self.typing: Dict[tuple, float] = {}

        self.typing_timeout = 5  # seconds

    async def mark_online(self, user_id: str):
        """Mark user as online"""
        self.last_seen[user_id] = time.time()

    async def mark_offline(self, user_id: str):
        """Mark user as offline"""
        if user_id in self.last_seen:
            del self.last_seen[user_id]

    def is_online(self, user_id: str) -> bool:
        """Check if user is online (active in last 60 seconds)"""
        if user_id not in self.last_seen:
            return False

        return time.time() - self.last_seen[user_id] < 60

    async def start_typing(self, user_id: str, room_id: str):
        """
        Mark user as typing in room

        Args:
            user_id: User ID
            room_id: Room ID
        """
        self.typing[(user_id, room_id)] = time.time()

        # Broadcast typing indicator
        await manager.send_to_room(
            room_id,
            {
                "type": "typing",
                "user_id": user_id,
                "room_id": room_id,
                "is_typing": True
            },
            exclude_user=user_id
        )

    async def stop_typing(self, user_id: str, room_id: str):
        """
        Mark user as stopped typing

        Args:
            user_id: User ID
            room_id: Room ID
        """
        key = (user_id, room_id)

        if key in self.typing:
            del self.typing[key]

        # Broadcast stop typing
        await manager.send_to_room(
            room_id,
            {
                "type": "typing",
                "user_id": user_id,
                "room_id": room_id,
                "is_typing": False
            },
            exclude_user=user_id
        )

    def is_typing(self, user_id: str, room_id: str) -> bool:
        """Check if user is typing"""
        key = (user_id, room_id)

        if key not in self.typing:
            return False

        # Check timeout
        return time.time() - self.typing[key] < self.typing_timeout

    def get_typing_users(self, room_id: str) -> list[str]:
        """Get users currently typing in room"""
        typing_users = []

        for (user_id, rid), timestamp in self.typing.items():
            if rid == room_id and time.time() - timestamp < self.typing_timeout:
                typing_users.append(user_id)

        return typing_users

# Global presence manager
presence = PresenceManager()
```

### 4. FastAPI WebSocket Server

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Depends
from fastapi.responses import HTMLResponse
import asyncio

app = FastAPI(title="Chat System")

async def get_current_user(websocket: WebSocket) -> str:
    """
    Get authenticated user from WebSocket

    In production: Validate JWT token from query params or headers
    """
    # For demo: use query parameter
    return websocket.query_params.get("user_id", "anonymous")

@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    user_id: str = Depends(get_current_user)
):
    """
    WebSocket endpoint for chat

    Args:
        websocket: WebSocket connection
        user_id: Authenticated user ID
    """
    await websocket.accept()

    # Register connection
    connection = await manager.connect(websocket, user_id)

    # Mark user as online
    await presence.mark_online(user_id)

    try:
        while True:
            # Receive message from client
            data = await websocket.receive_json()

            message_type = data.get("type")

            if message_type == "join_room":
                # Join chat room
                room_id = data["room_id"]
                await manager.join_room(user_id, room_id)

                # Send room history
                messages = await message_store.get_room_messages(room_id, limit=50)
                await websocket.send_json({
                    "type": "history",
                    "room_id": room_id,
                    "messages": [msg.to_dict() for msg in messages]
                })

                # Notify room members
                online_users = manager.get_online_users(room_id)
                await manager.send_to_room(room_id, {
                    "type": "user_joined",
                    "user_id": user_id,
                    "room_id": room_id,
                    "online_users": list(online_users)
                })

            elif message_type == "leave_room":
                # Leave chat room
                room_id = data["room_id"]
                await manager.leave_room(user_id, room_id)

                # Notify room members
                await manager.send_to_room(room_id, {
                    "type": "user_left",
                    "user_id": user_id,
                    "room_id": room_id
                })

            elif message_type == "message":
                # Send chat message
                room_id = data["room_id"]
                content = data["content"]

                # Create and save message
                message = Message.create(
                    room_id=room_id,
                    sender_id=user_id,
                    content=content,
                    message_type=data.get("message_type", "text"),
                    reply_to=data.get("reply_to")
                )

                await message_store.save_message(message)

                # Broadcast to room
                await manager.send_to_room(room_id, {
                    "type": "message",
                    "message": message.to_dict()
                })

            elif message_type == "typing":
                # Typing indicator
                room_id = data["room_id"]
                is_typing = data.get("is_typing", True)

                if is_typing:
                    await presence.start_typing(user_id, room_id)
                else:
                    await presence.stop_typing(user_id, room_id)

            elif message_type == "ping":
                # Heartbeat
                await presence.mark_online(user_id)
                await websocket.send_json({"type": "pong"})

    except WebSocketDisconnect:
        # Client disconnected
        await manager.disconnect(connection)
        await presence.mark_offline(user_id)

        # Notify rooms user was in
        for room_id in connection.rooms:
            await manager.send_to_room(room_id, {
                "type": "user_left",
                "user_id": user_id,
                "room_id": room_id
            })

@app.get("/rooms/{room_id}/messages")
async def get_messages(room_id: str, limit: int = 50, before: str = None):
    """
    Get message history for room

    Args:
        room_id: Room ID
        limit: Maximum messages
        before: Get messages before this ID (pagination)

    Returns:
        List of messages
    """
    messages = await message_store.get_room_messages(room_id, limit, before)

    return {
        "room_id": room_id,
        "messages": [msg.to_dict() for msg in messages]
    }

@app.get("/rooms/{room_id}/search")
async def search_messages(room_id: str, query: str):
    """
    Search messages in room

    Args:
        room_id: Room ID
        query: Search query

    Returns:
        Matching messages
    """
    messages = await message_store.search_messages(room_id, query)

    return {
        "room_id": room_id,
        "query": query,
        "results": [msg.to_dict() for msg in messages]
    }

@app.get("/")
async def get():
    """Demo chat client"""
    html = """
    <!DOCTYPE html>
    <html>
    <head>
        <title>Chat</title>
    </head>
    <body>
        <h1>Chat System</h1>
        <div>
            <input id="userId" placeholder="User ID" value="user1" />
            <input id="roomId" placeholder="Room ID" value="room1" />
            <button onclick="connect()">Connect</button>
        </div>
        <div>
            <input id="message" placeholder="Message" />
            <button onclick="sendMessage()">Send</button>
        </div>
        <div id="messages"></div>

        <script>
            let ws;
            let userId;
            let roomId;

            function connect() {
                userId = document.getElementById('userId').value;
                roomId = document.getElementById('roomId').value;

                ws = new WebSocket(`ws://localhost:8000/ws?user_id=${userId}`);

                ws.onmessage = (event) => {
                    const data = JSON.parse(event.data);
                    displayMessage(JSON.stringify(data, null, 2));
                };

                ws.onopen = () => {
                    // Join room
                    ws.send(JSON.stringify({
                        type: 'join_room',
                        room_id: roomId
                    }));
                };
            }

            function sendMessage() {
                const content = document.getElementById('message').value;

                ws.send(JSON.stringify({
                    type: 'message',
                    room_id: roomId,
                    content: content
                }));

                document.getElementById('message').value = '';
            }

            function displayMessage(msg) {
                const div = document.createElement('div');
                div.textContent = msg;
                document.getElementById('messages').appendChild(div);
            }
        </script>
    </body>
    </html>
    """
    return HTMLResponse(html)

# Run with: uvicorn chat_system:app --reload
```

## 🚀 Scaling Considerations

### 1. Cross-Server Messaging with Redis Pub/Sub

```python
import redis.asyncio as redis
import json

class RedisPubSub:
    """
    Redis pub/sub for cross-server messaging

    - Multiple chat servers share messages
    - User on Server 1 can message user on Server 2
    """

    def __init__(self, redis_url: str = "redis://localhost"):
        """
        Args:
            redis_url: Redis connection URL
        """
        self.redis = redis.from_url(redis_url)
        self.pubsub = self.redis.pubsub()

    async def subscribe(self, channel: str, callback):
        """
        Subscribe to channel

        Args:
            channel: Channel name (e.g., room ID)
            callback: Async function to call on message
        """
        await self.pubsub.subscribe(channel)

        # Listen for messages
        async for message in self.pubsub.listen():
            if message["type"] == "message":
                data = json.loads(message["data"])
                await callback(data)

    async def publish(self, channel: str, data: dict):
        """
        Publish message to channel

        Args:
            channel: Channel name
            data: Message data
        """
        await self.redis.publish(channel, json.dumps(data))

# Usage in chat server
redis_pubsub = RedisPubSub()

async def handle_redis_message(data: dict):
    """Handle message from Redis pub/sub"""
    room_id = data["room_id"]
    await manager.send_to_room(room_id, data)

# Subscribe to room channels
asyncio.create_task(redis_pubsub.subscribe("room:*", handle_redis_message))
```

## ❓ Interview Questions

### Q1: How to ensure message ordering?

**Answer**:

1. **Single partition**: Route all messages for chat to same partition
2. **Sequence numbers**: Assign incremental IDs per room
3. **Timestamps**: Use server timestamp, not client
4. **Causal ordering**: Track dependencies between messages

### Q2: How to handle offline users?

**Answer**:

1. **Store messages**: Save to database
2. **Push notifications**: Send via FCM/APNS
3. **Unread counts**: Track in Redis
4. **Message queue**: Queue messages for delivery when online

### Q3: WebSocket vs HTTP polling?

**Answer**:

- **WebSocket**: Real-time, bidirectional, lower latency
- **Polling**: Simpler, works everywhere, higher latency
- **Long polling**: Better than short polling, still higher overhead
- **Server-Sent Events**: One-way push from server

### Q4: How to scale to millions of connections?

**Answer**:

1. **Horizontal scaling**: Multiple chat servers
2. **Redis pub/sub**: Cross-server messaging
3. **Load balancing**: Sticky sessions for WebSocket
4. **Connection limits**: 10K-100K per server
5. **Database sharding**: Partition by room_id or user_id

## 📚 Summary

**Key Takeaways**:

1. **WebSocket**: Real-time bidirectional communication
2. **Connection Manager**: Track active connections
3. **Message Storage**: PostgreSQL/MongoDB for persistence
4. **Presence**: Online/offline status with heartbeats
5. **Typing Indicators**: Real-time feedback
6. **Redis Pub/Sub**: Cross-server messaging
7. **Horizontal Scaling**: Multiple chat servers
8. **Read Receipts**: Track message delivery/read status
9. **Push Notifications**: For offline users
10. **Security**: Authentication, rate limiting, encryption

Real-time chat requires careful design for scale!
