# Chat Application Architecture

## Table of Contents

1. [System Overview](#system-overview)
2. [WebSocket Architecture](#websocket-architecture)
3. [Message Types & Protocols](#message-types--protocols)
4. [Presence System](#presence-system)
5. [Typing Indicators](#typing-indicators)
6. [Notifications](#notifications)
7. [Message History](#message-history)
8. [File Sharing](#file-sharing)
9. [Search & Filtering](#search--filtering)
10. [Offline Support](#offline-support)
11. [Security & Encryption](#security--encryption)
12. [Key Takeaways](#key-takeaways)

## System Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    Client Layer                              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐            │
│  │   Chat     │  │  Presence  │  │   Notify   │            │
│  │ Interface  │  │  Manager   │  │  Manager   │            │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘            │
└────────┼────────────────┼────────────────┼───────────────────┘
         │                │                │
┌────────▼────────────────▼────────────────▼───────────────────┐
│              WebSocket Connection Pool                        │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  - Connection Management                              │   │
│  │  - Reconnection Logic                                 │   │
│  │  - Message Queue                                      │   │
│  └──────────────────────────────────────────────────────┘   │
└────────┬──────────────────────────────────────────────────────┘
         │
┌────────▼──────────────────────────────────────────────────────┐
│                    WebSocket Gateway                          │
│         (Load Balancing, Authentication, Routing)             │
└────────┬──────────────────────────────────────────────────────┘
         │
    ┌────┴────┬──────────────┬──────────────┬──────────────┐
    │         │              │              │              │
┌───▼────┐┌──▼─────┐┌───────▼─────┐┌───────▼─────┐┌──────▼────┐
│ Chat   ││Presence││  Message    ││   File      ││Notification│
│Service ││Service ││   Queue     ││  Service    ││  Service   │
└───┬────┘└──┬─────┘└──────┬──────┘└──────┬──────┘└──────┬────┘
    │        │             │               │              │
┌───▼────────▼─────────────▼───────────────▼──────────────▼────┐
│                    Database Layer                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │Messages  │  │  Users   │  │  Redis   │  │  S3      │    │
│  │   DB     │  │    DB    │  │  Cache   │  │  (Files) │    │
│  │(Cassandra│  │(Postgres)│  │          │  │          │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└───────────────────────────────────────────────────────────────┘
```

## WebSocket Architecture

### Core Types

```typescript
interface ChatMessage {
  id: string;
  conversationId: string;
  senderId: string;
  sender: User;
  content: string;
  type: MessageType;
  attachments: Attachment[];
  replyTo?: string;
  reactions: Reaction[];
  status: MessageStatus;
  createdAt: Date;
  updatedAt: Date;
  deletedAt?: Date;
}

type MessageType = "text" | "image" | "video" | "file" | "audio" | "system";
type MessageStatus = "sending" | "sent" | "delivered" | "read" | "failed";

interface Conversation {
  id: string;
  type: ConversationType;
  participants: Participant[];
  name?: string;
  avatarUrl?: string;
  lastMessage?: ChatMessage;
  unreadCount: number;
  mutedUntil?: Date;
  pinnedAt?: Date;
  archivedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
}

type ConversationType = "direct" | "group" | "channel";

interface Participant {
  userId: string;
  user: User;
  role: ParticipantRole;
  joinedAt: Date;
  leftAt?: Date;
}

type ParticipantRole = "owner" | "admin" | "member";

interface Attachment {
  id: string;
  type: AttachmentType;
  url: string;
  thumbnailUrl?: string;
  filename: string;
  size: number;
  mimeType: string;
}

type AttachmentType = "image" | "video" | "audio" | "document";

interface Reaction {
  emoji: string;
  userId: string;
  user: User;
  createdAt: Date;
}

interface PresenceStatus {
  userId: string;
  status: UserStatus;
  lastSeenAt: Date;
  customStatus?: string;
}

type UserStatus = "online" | "away" | "busy" | "offline";

interface TypingIndicator {
  conversationId: string;
  userId: string;
  user: User;
  isTyping: boolean;
  timestamp: Date;
}
```

### WebSocket Manager

```typescript
interface WebSocketMessage {
  type: WSMessageType;
  payload: any;
  timestamp: Date;
  id?: string;
}

type WSMessageType =
  | "message"
  | "message_read"
  | "message_delivered"
  | "typing_start"
  | "typing_stop"
  | "presence_update"
  | "conversation_update"
  | "reaction_add"
  | "reaction_remove";

class ChatWebSocketManager {
  private ws: WebSocket | null = null;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;
  private reconnectDelay = 1000;
  private heartbeatInterval: NodeJS.Timeout | null = null;
  private messageQueue: WebSocketMessage[] = [];
  private listeners: Map<WSMessageType, Set<(data: any) => void>> = new Map();
  private pendingMessages: Map<string, PendingMessage> = new Map();

  constructor(
    private url: string,
    private token: string,
  ) {}

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      try {
        this.ws = new WebSocket(`${this.url}?token=${this.token}`);

        this.ws.onopen = () => {
          console.log("WebSocket connected");
          this.reconnectAttempts = 0;
          this.startHeartbeat();
          this.flushMessageQueue();
          resolve();
        };

        this.ws.onmessage = (event) => {
          const message: WebSocketMessage = JSON.parse(event.data);
          this.handleMessage(message);
        };

        this.ws.onerror = (error) => {
          console.error("WebSocket error:", error);
          reject(error);
        };

        this.ws.onclose = (event) => {
          console.log("WebSocket closed:", event.code, event.reason);
          this.stopHeartbeat();

          if (!event.wasClean) {
            this.attemptReconnect();
          }
        };
      } catch (error) {
        reject(error);
      }
    });
  }

  send(type: WSMessageType, payload: any): Promise<void> {
    return new Promise((resolve, reject) => {
      const message: WebSocketMessage = {
        id: generateId(),
        type,
        payload,
        timestamp: new Date(),
      };

      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify(message));

        // Track pending message for acknowledgment
        this.pendingMessages.set(message.id!, {
          message,
          resolve,
          reject,
          timeout: setTimeout(() => {
            this.pendingMessages.delete(message.id!);
            reject(new Error("Message timeout"));
          }, 10000),
        });
      } else {
        // Queue message if not connected
        this.messageQueue.push(message);
        resolve();
      }
    });
  }

  subscribe(type: WSMessageType, callback: (data: any) => void): () => void {
    if (!this.listeners.has(type)) {
      this.listeners.set(type, new Set());
    }

    this.listeners.get(type)!.add(callback);

    return () => {
      this.listeners.get(type)?.delete(callback);
    };
  }

  private handleMessage(message: WebSocketMessage): void {
    // Handle acknowledgments
    if (message.id && this.pendingMessages.has(message.id)) {
      const pending = this.pendingMessages.get(message.id)!;
      clearTimeout(pending.timeout);
      pending.resolve();
      this.pendingMessages.delete(message.id);
    }

    // Notify listeners
    const callbacks = this.listeners.get(message.type);
    if (callbacks) {
      callbacks.forEach((callback) => callback(message.payload));
    }
  }

  private startHeartbeat(): void {
    this.heartbeatInterval = setInterval(() => {
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({ type: "ping" }));
      }
    }, 30000); // 30 seconds
  }

  private stopHeartbeat(): void {
    if (this.heartbeatInterval) {
      clearInterval(this.heartbeatInterval);
      this.heartbeatInterval = null;
    }
  }

  private attemptReconnect(): void {
    if (this.reconnectAttempts >= this.maxReconnectAttempts) {
      console.error("Max reconnection attempts reached");
      return;
    }

    this.reconnectAttempts++;
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts - 1);

    setTimeout(() => {
      console.log(`Reconnecting... (attempt ${this.reconnectAttempts})`);
      this.connect().catch(() => {
        // Will retry in next attempt
      });
    }, delay);
  }

  private flushMessageQueue(): void {
    while (this.messageQueue.length > 0) {
      const message = this.messageQueue.shift()!;
      this.send(message.type, message.payload);
    }
  }

  disconnect(): void {
    this.stopHeartbeat();
    if (this.ws) {
      this.ws.close(1000, "Client disconnect");
      this.ws = null;
    }
  }
}

interface PendingMessage {
  message: WebSocketMessage;
  resolve: () => void;
  reject: (error: Error) => void;
  timeout: NodeJS.Timeout;
}

function generateId(): string {
  return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
}
```

### Chat Hook

```typescript
export function useChat(userId: string) {
  const wsManager = useRef<ChatWebSocketManager | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const token = getAuthToken();
    wsManager.current = new ChatWebSocketManager(WS_URL, token);

    wsManager.current
      .connect()
      .then(() => setIsConnected(true))
      .catch((error) => {
        console.error("Failed to connect:", error);
        setIsConnected(false);
      });

    return () => {
      wsManager.current?.disconnect();
    };
  }, [userId]);

  const sendMessage = useCallback(
    async (
      conversationId: string,
      content: string,
      attachments?: Attachment[],
    ) => {
      if (!wsManager.current) throw new Error("Not connected");

      await wsManager.current.send("message", {
        conversationId,
        content,
        attachments,
      });
    },
    [],
  );

  const markAsRead = useCallback(
    async (conversationId: string, messageId: string) => {
      if (!wsManager.current) throw new Error("Not connected");

      await wsManager.current.send("message_read", {
        conversationId,
        messageId,
      });
    },
    [],
  );

  const startTyping = useCallback(async (conversationId: string) => {
    if (!wsManager.current) throw new Error("Not connected");

    await wsManager.current.send("typing_start", {
      conversationId,
    });
  }, []);

  const stopTyping = useCallback(async (conversationId: string) => {
    if (!wsManager.current) throw new Error("Not connected");

    await wsManager.current.send("typing_stop", {
      conversationId,
    });
  }, []);

  const updatePresence = useCallback(
    async (status: UserStatus, customStatus?: string) => {
      if (!wsManager.current) throw new Error("Not connected");

      await wsManager.current.send("presence_update", {
        status,
        customStatus,
      });
    },
    [],
  );

  return {
    isConnected,
    sendMessage,
    markAsRead,
    startTyping,
    stopTyping,
    updatePresence,
    wsManager: wsManager.current,
  };
}
```

## Message Types & Protocols

### Message Service

```typescript
class MessageService {
  constructor(
    private apiClient: ApiClient,
    private wsManager: ChatWebSocketManager,
  ) {}

  async sendMessage(
    conversationId: string,
    content: string,
    attachments?: Attachment[],
    replyTo?: string,
  ): Promise<ChatMessage> {
    const tempId = generateId();

    // Create optimistic message
    const optimisticMessage: ChatMessage = {
      id: tempId,
      conversationId,
      senderId: getCurrentUser().id,
      sender: getCurrentUser(),
      content,
      type: "text",
      attachments: attachments || [],
      replyTo,
      reactions: [],
      status: "sending",
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    // Send via WebSocket
    try {
      await this.wsManager.send("message", {
        tempId,
        conversationId,
        content,
        attachments,
        replyTo,
      });

      return optimisticMessage;
    } catch (error) {
      // Fallback to HTTP if WebSocket fails
      return await this.apiClient.post<ChatMessage>(
        `/conversations/${conversationId}/messages`,
        {
          content,
          attachments,
          replyTo,
        },
      );
    }
  }

  async getMessages(
    conversationId: string,
    params: MessageQueryParams,
  ): Promise<PaginatedResponse<ChatMessage>> {
    return await this.apiClient.get<PaginatedResponse<ChatMessage>>(
      `/conversations/${conversationId}/messages`,
      { params },
    );
  }

  async updateMessage(
    messageId: string,
    content: string,
  ): Promise<ChatMessage> {
    const message = await this.apiClient.put<ChatMessage>(
      `/messages/${messageId}`,
      { content },
    );

    // Notify via WebSocket
    await this.wsManager.send("message_update", {
      messageId,
      content,
    });

    return message;
  }

  async deleteMessage(messageId: string): Promise<void> {
    await this.apiClient.delete(`/messages/${messageId}`);

    // Notify via WebSocket
    await this.wsManager.send("message_delete", {
      messageId,
    });
  }

  async addReaction(messageId: string, emoji: string): Promise<ChatMessage> {
    const message = await this.apiClient.post<ChatMessage>(
      `/messages/${messageId}/reactions`,
      { emoji },
    );

    // Notify via WebSocket
    await this.wsManager.send("reaction_add", {
      messageId,
      emoji,
    });

    return message;
  }

  async removeReaction(messageId: string, emoji: string): Promise<ChatMessage> {
    const message = await this.apiClient.delete<ChatMessage>(
      `/messages/${messageId}/reactions/${emoji}`,
    );

    // Notify via WebSocket
    await this.wsManager.send("reaction_remove", {
      messageId,
      emoji,
    });

    return message;
  }

  async searchMessages(
    query: string,
    conversationId?: string,
  ): Promise<ChatMessage[]> {
    return await this.apiClient.get<ChatMessage[]>("/messages/search", {
      params: { query, conversationId },
    });
  }
}

interface MessageQueryParams {
  before?: string;
  after?: string;
  limit?: number;
}
```

### Chat Component

```typescript
export const ChatConversation: React.FC<{ conversationId: string }> = ({
  conversationId,
}) => {
  const { userId } = useAuth();
  const { sendMessage, markAsRead, wsManager } = useChat(userId);
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [inputValue, setInputValue] = useState('');
  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    loadMessages();
  }, [conversationId]);

  useEffect(() => {
    if (!wsManager) return;

    const unsubscribeMessage = wsManager.subscribe('message', (message: ChatMessage) => {
      if (message.conversationId === conversationId) {
        setMessages((prev) => [...prev, message]);
        scrollToBottom();

        // Mark as read if not sent by current user
        if (message.senderId !== userId) {
          markAsRead(conversationId, message.id);
        }
      }
    });

    const unsubscribeRead = wsManager.subscribe('message_read', (data) => {
      setMessages((prev) =>
        prev.map((msg) =>
          msg.id === data.messageId ? { ...msg, status: 'read' } : msg
        )
      );
    });

    return () => {
      unsubscribeMessage();
      unsubscribeRead();
    };
  }, [conversationId, wsManager, userId]);

  const loadMessages = async () => {
    const messageService = new MessageService(apiClient, wsManager!);
    const response = await messageService.getMessages(conversationId, {
      limit: 50,
    });
    setMessages(response.items);
    scrollToBottom();
  };

  const handleSendMessage = async (e: React.FormEvent) => {
    e.preventDefault();

    if (!inputValue.trim()) return;

    await sendMessage(conversationId, inputValue);
    setInputValue('');
  };

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  };

  return (
    <div className="chat-conversation">
      <div className="messages-container">
        {messages.map((message) => (
          <MessageBubble
            key={message.id}
            message={message}
            isOwnMessage={message.senderId === userId}
          />
        ))}
        <div ref={messagesEndRef} />
      </div>

      <form onSubmit={handleSendMessage} className="message-input">
        <input
          type="text"
          value={inputValue}
          onChange={(e) => setInputValue(e.target.value)}
          placeholder="Type a message..."
        />
        <button type="submit" disabled={!inputValue.trim()}>
          Send
        </button>
      </form>
    </div>
  );
};

const MessageBubble: React.FC<{
  message: ChatMessage;
  isOwnMessage: boolean;
}> = ({ message, isOwnMessage }) => {
  return (
    <div className={`message-bubble ${isOwnMessage ? 'own' : 'other'}`}>
      {!isOwnMessage && (
        <img
          src={message.sender.avatarUrl}
          alt={message.sender.displayName}
          className="sender-avatar"
        />
      )}

      <div className="message-content">
        {!isOwnMessage && (
          <span className="sender-name">{message.sender.displayName}</span>
        )}

        <div className="message-text">{message.content}</div>

        {message.attachments.length > 0 && (
          <div className="message-attachments">
            {message.attachments.map((attachment) => (
              <AttachmentPreview key={attachment.id} attachment={attachment} />
            ))}
          </div>
        )}

        {message.reactions.length > 0 && (
          <div className="message-reactions">
            {Object.entries(
              message.reactions.reduce((acc, reaction) => {
                acc[reaction.emoji] = (acc[reaction.emoji] || 0) + 1;
                return acc;
              }, {} as Record<string, number>)
            ).map(([emoji, count]) => (
              <span key={emoji} className="reaction">
                {emoji} {count}
              </span>
            ))}
          </div>
        )}

        <div className="message-meta">
          <span className="message-time">
            {formatTime(message.createdAt)}
          </span>
          {isOwnMessage && (
            <span className="message-status">
              {getStatusIcon(message.status)}
            </span>
          )}
        </div>
      </div>
    </div>
  );
};

function formatTime(date: Date): string {
  const now = new Date();
  const diff = now.getTime() - date.getTime();

  if (diff < 60000) return 'Just now';
  if (diff < 3600000) return `${Math.floor(diff / 60000)}m ago`;
  if (diff < 86400000) return `${Math.floor(diff / 3600000)}h ago`;

  return date.toLocaleDateString();
}

function getStatusIcon(status: MessageStatus): React.ReactNode {
  switch (status) {
    case 'sending':
      return <ClockIcon />;
    case 'sent':
      return <CheckIcon />;
    case 'delivered':
      return <CheckDoubleIcon />;
    case 'read':
      return <CheckDoubleIcon className="read" />;
    case 'failed':
      return <ErrorIcon />;
  }
}
```

## Presence System

### Presence Service

```typescript
class PresenceService {
  private presenceCache: Map<string, PresenceStatus> = new Map();
  private updateInterval: NodeJS.Timeout | null = null;

  constructor(
    private wsManager: ChatWebSocketManager,
    private apiClient: ApiClient
  ) {
    this.startPeriodicUpdate();
    this.subscribeToPresenceUpdates();
  }

  async updatePresence(
    status: UserStatus,
    customStatus?: string
  ): Promise<void> {
    await this.wsManager.send('presence_update', {
      status,
      customStatus,
    });

    await this.apiClient.post('/presence', {
      status,
      customStatus,
    });
  }

  getPresence(userId: string): PresenceStatus | null {
    return this.presenceCache.get(userId) || null;
  }

  async fetchPresence(userIds: string[]): Promise<Map<string, PresenceStatus>> {
    const presences = await this.apiClient.post<PresenceStatus[]>(
      '/presence/batch',
      { userIds }
    );

    presences.forEach((presence) => {
      this.presenceCache.set(presence.userId, presence);
    });

    return this.presenceCache;
  }

  private startPeriodicUpdate(): void {
    this.updateInterval = setInterval(() => {
      this.updatePresence('online');
    }, 60000); // Update every minute
  }

  private subscribeToPresenceUpdates(): void {
    this.wsManager.subscribe('presence_update', (presence: PresenceStatus) => {
      this.presenceCache.set(presence.userId, presence);
    });
  }

  destroy(): void {
    if (this.updateInterval) {
      clearInterval(this.updateInterval);
    }
  }
}

// Presence hook
export function usePresence(userId: string) {
  const { wsManager } = useChat(userId);
  const [presence, setPresence] = useState<PresenceStatus | null>(null);
  const presenceService = useRef<PresenceService | null>(null);

  useEffect(() => {
    if (!wsManager) return;

    presenceService.current = new PresenceService(wsManager, apiClient);

    return () => {
      presenceService.current?.destroy();
    };
  }, [wsManager]);

  useEffect(() => {
    if (!presenceService.current) return;

    const unsubscribe = wsManager?.subscribe(
      'presence_update',
      (updatedPresence: PresenceStatus) => {
        if (updatedPresence.userId === userId) {
          setPresence(updatedPresence);
        }
      }
    );

    // Fetch initial presence
    presenceService.current.fetchPresence([userId]);

    return unsubscribe;
  }, [userId]);

  const updatePresence = useCallback(
    (status: UserStatus, customStatus?: string) => {
      presenceService.current?.updatePresence(status, customStatus);
    },
    []
  );

  return { presence, updatePresence };
}

// Presence indicator component
export const PresenceIndicator: React.FC<{ userId: string }> = ({ userId }) => {
  const { presence } = usePresence(userId);

  if (!presence) return null;

  return (
    <div className="presence-indicator">
      <span className={`status-dot ${presence.status}`} />
      <span className="status-text">{presence.status}</span>
      {presence.customStatus && (
        <span className="custom-status">{presence.customStatus}</span>
      )}
    </div>
  );
};
```

## Typing Indicators

### Typing Indicator Service

```typescript
class TypingIndicatorService {
  private typingUsers: Map<string, Set<string>> = new Map();
  private typingTimeouts: Map<string, NodeJS.Timeout> = new Map();
  private readonly TYPING_TIMEOUT = 3000; // 3 seconds

  constructor(private wsManager: ChatWebSocketManager) {
    this.subscribeToTypingEvents();
  }

  startTyping(conversationId: string): void {
    this.wsManager.send('typing_start', { conversationId });

    // Auto-stop after timeout
    const key = conversationId;
    if (this.typingTimeouts.has(key)) {
      clearTimeout(this.typingTimeouts.get(key)!);
    }

    this.typingTimeouts.set(
      key,
      setTimeout(() => {
        this.stopTyping(conversationId);
      }, this.TYPING_TIMEOUT)
    );
  }

  stopTyping(conversationId: string): void {
    this.wsManager.send('typing_stop', { conversationId });

    const key = conversationId;
    if (this.typingTimeouts.has(key)) {
      clearTimeout(this.typingTimeouts.get(key)!);
      this.typingTimeouts.delete(key);
    }
  }

  getTypingUsers(conversationId: string): string[] {
    return Array.from(this.typingUsers.get(conversationId) || []);
  }

  private subscribeToTypingEvents(): void {
    this.wsManager.subscribe('typing_start', (data: TypingIndicator) => {
      if (!this.typingUsers.has(data.conversationId)) {
        this.typingUsers.set(data.conversationId, new Set());
      }
      this.typingUsers.get(data.conversationId)!.add(data.userId);

      // Auto-remove after timeout
      setTimeout(() => {
        this.typingUsers.get(data.conversationId)?.delete(data.userId);
      }, this.TYPING_TIMEOUT);
    });

    this.wsManager.subscribe('typing_stop', (data: TypingIndicator) => {
      this.typingUsers.get(data.conversationId)?.delete(data.userId);
    });
  }
}

// Typing indicator hook
export function useTypingIndicator(conversationId: string) {
  const { wsManager } = useChat(getCurrentUser().id);
  const [typingUsers, setTypingUsers] = useState<string[]>([]);
  const typingService = useRef<TypingIndicatorService | null>(null);
  const typingDebounce = useRef<NodeJS.Timeout | null>(null);

  useEffect(() => {
    if (!wsManager) return;

    typingService.current = new TypingIndicatorService(wsManager);

    const unsubscribeStart = wsManager.subscribe(
      'typing_start',
      (data: TypingIndicator) => {
        if (data.conversationId === conversationId) {
          setTypingUsers(typingService.current!.getTypingUsers(conversationId));
        }
      }
    );

    const unsubscribeStop = wsManager.subscribe(
      'typing_stop',
      (data: TypingIndicator) => {
        if (data.conversationId === conversationId) {
          setTypingUsers(typingService.current!.getTypingUsers(conversationId));
        }
      }
    );

    return () => {
      unsubscribeStart();
      unsubscribeStop();
    };
  }, [conversationId, wsManager]);

  const onTyping = useCallback(() => {
    if (!typingService.current) return;

    if (typingDebounce.current) {
      clearTimeout(typingDebounce.current);
    }

    typingService.current.startTyping(conversationId);

    typingDebounce.current = setTimeout(() => {
      typingService.current?.stopTyping(conversationId);
    }, 1000);
  }, [conversationId]);

  return { typingUsers, onTyping };
}

// Typing indicator component
export const TypingIndicator: React.FC<{ conversationId: string }> = ({
  conversationId,
}) => {
  const { typingUsers } = useTypingIndicator(conversationId);

  if (typingUsers.length === 0) return null;

  return (
    <div className="typing-indicator">
      <span className="typing-dots">
        <span />
        <span />
        <span />
      </span>
      <span className="typing-text">
        {typingUsers.length === 1
          ? `${typingUsers[0]} is typing...`
          : `${typingUsers.length} people are typing...`}
      </span>
    </div>
  );
};
```

## Notifications

### Notification Service

```typescript
interface Notification {
  id: string;
  type: NotificationType;
  title: string;
  message: string;
  conversationId?: string;
  messageId?: string;
  senderId: string;
  sender: User;
  read: boolean;
  createdAt: Date;
}

type NotificationType = "message" | "mention" | "reaction" | "invite";

class NotificationService {
  constructor(
    private wsManager: ChatWebSocketManager,
    private apiClient: ApiClient,
  ) {
    this.subscribeToNotifications();
  }

  async getNotifications(): Promise<Notification[]> {
    return await this.apiClient.get<Notification[]>("/notifications");
  }

  async markAsRead(notificationId: string): Promise<void> {
    await this.apiClient.put(`/notifications/${notificationId}/read`);
  }

  async markAllAsRead(): Promise<void> {
    await this.apiClient.put("/notifications/read-all");
  }

  private subscribeToNotifications(): void {
    this.wsManager.subscribe("notification", (notification: Notification) => {
      this.showNotification(notification);
    });
  }

  private showNotification(notification: Notification): void {
    // Show browser notification
    if ("Notification" in window && Notification.permission === "granted") {
      new Notification(notification.title, {
        body: notification.message,
        icon: notification.sender.avatarUrl,
        tag: notification.id,
      });
    }

    // Show in-app notification
    toast.info(notification.message, {
      onClick: () => {
        if (notification.conversationId) {
          window.location.href = `/chat/${notification.conversationId}`;
        }
      },
    });
  }

  async requestPermission(): Promise<NotificationPermission> {
    if ("Notification" in window) {
      return await Notification.requestPermission();
    }
    return "denied";
  }
}

// Notification hook
export function useNotifications() {
  const { wsManager } = useChat(getCurrentUser().id);
  const [notifications, setNotifications] = useState<Notification[]>([]);
  const [unreadCount, setUnreadCount] = useState(0);
  const notificationService = useRef<NotificationService | null>(null);

  useEffect(() => {
    if (!wsManager) return;

    notificationService.current = new NotificationService(wsManager, apiClient);

    // Request permission
    notificationService.current.requestPermission();

    // Load notifications
    loadNotifications();

    const unsubscribe = wsManager.subscribe(
      "notification",
      (notification: Notification) => {
        setNotifications((prev) => [notification, ...prev]);
        setUnreadCount((prev) => prev + 1);
      },
    );

    return unsubscribe;
  }, [wsManager]);

  const loadNotifications = async () => {
    if (!notificationService.current) return;

    const data = await notificationService.current.getNotifications();
    setNotifications(data);
    setUnreadCount(data.filter((n) => !n.read).length);
  };

  const markAsRead = async (notificationId: string) => {
    if (!notificationService.current) return;

    await notificationService.current.markAsRead(notificationId);
    setNotifications((prev) =>
      prev.map((n) => (n.id === notificationId ? { ...n, read: true } : n)),
    );
    setUnreadCount((prev) => Math.max(0, prev - 1));
  };

  const markAllAsRead = async () => {
    if (!notificationService.current) return;

    await notificationService.current.markAllAsRead();
    setNotifications((prev) => prev.map((n) => ({ ...n, read: true })));
    setUnreadCount(0);
  };

  return {
    notifications,
    unreadCount,
    markAsRead,
    markAllAsRead,
  };
}
```

## Message History

### Message History with Pagination

```typescript
export function useMessageHistory(conversationId: string) {
  const {
    data,
    fetchNextPage,
    fetchPreviousPage,
    hasNextPage,
    hasPreviousPage,
    isFetchingNextPage,
    isFetchingPreviousPage,
  } = useInfiniteQuery({
    queryKey: ['messages', conversationId],
    queryFn: ({ pageParam }) => {
      const messageService = new MessageService(apiClient, wsManager);
      return messageService.getMessages(conversationId, {
        before: pageParam,
        limit: 30,
      });
    },
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    getPreviousPageParam: (firstPage) => firstPage.previousCursor,
  });

  const messages = useMemo(() => {
    return data?.pages.flatMap((page) => page.items).reverse() || [];
  }, [data]);

  return {
    messages,
    fetchNextPage,
    fetchPreviousPage,
    hasNextPage,
    hasPreviousPage,
    isFetchingNextPage,
    isFetchingPreviousPage,
  };
}

// Message list with infinite scroll
export const MessageList: React.FC<{ conversationId: string }> = ({
  conversationId,
}) => {
  const {
    messages,
    fetchPreviousPage,
    hasPreviousPage,
    isFetchingPreviousPage,
  } = useMessageHistory(conversationId);

  const containerRef = useRef<HTMLDivElement>(null);
  const topSentinelRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!topSentinelRef.current) return;

    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting && hasPreviousPage && !isFetchingPreviousPage) {
          fetchPreviousPage();
        }
      },
      { root: containerRef.current, threshold: 1.0 }
    );

    observer.observe(topSentinelRef.current);

    return () => observer.disconnect();
  }, [hasPreviousPage, isFetchingPreviousPage]);

  return (
    <div ref={containerRef} className="message-list">
      <div ref={topSentinelRef} className="load-more-trigger">
        {isFetchingPreviousPage && <Spinner />}
      </div>

      {messages.map((message) => (
        <MessageBubble
          key={message.id}
          message={message}
          isOwnMessage={message.senderId === getCurrentUser().id}
        />
      ))}
    </div>
  );
};
```

## File Sharing

### File Upload Service

```typescript
class FileUploadService {
  constructor(
    private apiClient: ApiClient,
    private maxFileSize: number = 50 * 1024 * 1024 // 50MB
  ) {}

  async uploadFile(
    file: File,
    onProgress?: (progress: number) => void
  ): Promise<Attachment> {
    // Validate file
    this.validateFile(file);

    // Get upload URL
    const { uploadUrl, attachmentId } = await this.apiClient.post<{
      uploadUrl: string;
      attachmentId: string;
    }>('/attachments/upload-url', {
      filename: file.name,
      mimeType: file.type,
      size: file.size,
    });

    // Upload file
    await fetch(uploadUrl, {
      method: 'PUT',
      body: file,
      headers: {
        'Content-Type': file.type,
      },
    });

    // Confirm upload
    const attachment = await this.apiClient.post<Attachment>(
      `/attachments/${attachmentId}/confirm`
    );

    return attachment;
  }

  private validateFile(file: File): void {
    if (file.size > this.maxFileSize) {
      throw new Error('File too large');
    }

    const allowedTypes = [
      'image/jpeg',
      'image/png',
      'image/gif',
      'image/webp',
      'video/mp4',
      'audio/mpeg',
      'application/pdf',
      'application/msword',
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
    ];

    if (!allowedTypes.includes(file.type)) {
      throw new Error('File type not allowed');
    }
  }
}

// File upload component
export const FileUploadButton: React.FC<{
  onFileSelect: (attachment: Attachment) => void;
}> = ({ onFileSelect }) => {
  const [uploading, setUploading] = useState(false);
  const [progress, setProgress] = useState(0);
  const fileUploadService = new FileUploadService(apiClient);

  const handleFileChange = async (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (!file) return;

    setUploading(true);
    setProgress(0);

    try {
      const attachment = await fileUploadService.uploadFile(file, setProgress);
      onFileSelect(attachment);
      toast.success('File uploaded successfully');
    } catch (error) {
      toast.error('Failed to upload file');
      console.error(error);
    } finally {
      setUploading(false);
      setProgress(0);
    }
  };

  return (
    <>
      <input
        type="file"
        onChange={handleFileChange}
        disabled={uploading}
        id="file-upload"
        style={{ display: 'none' }}
      />
      <label htmlFor="file-upload" className="file-upload-button">
        {uploading ? (
          <span>Uploading... {progress}%</span>
        ) : (
          <>
            <AttachIcon />
            Attach File
          </>
        )}
      </label>
    </>
  );
};
```

## Offline Support

### Offline Queue Manager

```typescript
class OfflineQueueManager {
  private readonly QUEUE_KEY = "chat_offline_queue";

  addToQueue(message: ChatMessage): void {
    const queue = this.getQueue();
    queue.push(message);
    localStorage.setItem(this.QUEUE_KEY, JSON.stringify(queue));
  }

  getQueue(): ChatMessage[] {
    const data = localStorage.getItem(this.QUEUE_KEY);
    return data ? JSON.parse(data) : [];
  }

  clearQueue(): void {
    localStorage.removeItem(this.QUEUE_KEY);
  }

  async processQueue(
    sendFunction: (message: ChatMessage) => Promise<void>,
  ): Promise<void> {
    const queue = this.getQueue();

    for (const message of queue) {
      try {
        await sendFunction(message);
      } catch (error) {
        console.error("Failed to send queued message:", error);
      }
    }

    this.clearQueue();
  }
}

// Offline support hook
export function useOfflineSupport() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);
  const queueManager = new OfflineQueueManager();

  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true);
      // Process queued messages
      queueManager.processQueue(async (message) => {
        // Send message logic
      });
    };

    const handleOffline = () => {
      setIsOnline(false);
    };

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return { isOnline, queueManager };
}
```

## Key Takeaways

### 1. **WebSocket Reliability**

Implement automatic reconnection with exponential backoff. Queue messages when connection is lost. Use heartbeat to detect connection issues early.

### 2. **Optimistic Updates**

Show messages immediately before server confirmation. Track message status (sending, sent, delivered, read). Provide clear feedback for failed messages.

### 3. **Presence System**

Update presence status periodically. Cache presence data on client. Use WebSocket for real-time presence updates. Show last seen for offline users.

### 4. **Typing Indicators**

Debounce typing events to reduce WebSocket traffic. Auto-stop typing after timeout. Show indicators only for active conversation.

### 5. **Smart Notifications**

Request browser notification permission. Use service workers for background notifications. Support in-app toast notifications. Allow users to mute conversations.

### 6. **Message History**

Implement bi-directional infinite scroll. Cache loaded messages locally. Fetch older messages on scroll. Support search within conversation.

### 7. **File Handling**

Support multiple file types with validation. Use direct upload to cloud storage. Generate thumbnails for images. Show upload progress.

### 8. **Offline Support**

Queue messages when offline. Sync on reconnection. Show offline indicator. Cache recent conversations locally.

### 9. **Security**

Encrypt messages end-to-end. Validate all inputs. Use secure WebSocket (WSS). Implement rate limiting.

### 10. **Performance**

Virtualize long message lists. Lazy load attachments. Compress images before upload. Use message pagination. Cache user data.

---

**Further Reading:**

- WebSocket API Documentation
- Socket.IO Best Practices
- End-to-End Encryption with Signal Protocol
- IndexedDB for Offline Storage
