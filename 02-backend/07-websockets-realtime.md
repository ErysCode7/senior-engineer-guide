# WebSockets & Real-time Communication

## Overview

WebSockets enable bidirectional, real-time communication between clients and servers. This guide covers integrating Socket.IO with Express for chat applications, live notifications, real-time dashboards, and collaborative features.

## Socket.IO Setup

```bash
npm install socket.io
npm install -D @types/socket.io
```

```typescript
// src/config/socket.ts
import { Server as HTTPServer } from 'http';
import { Server as SocketIOServer } from 'socket.io';

export const initializeSocket = (httpServer: HTTPServer) => {
  const io = new SocketIOServer(httpServer, {
    cors: {
      origin: process.env.CLIENT_URL || 'http://localhost:3000',
      credentials: true,
    },
  });

  return io;
};
```

```typescript
// src/server.ts
import express from 'express';
import http from 'http';
import { initializeSocket } from './config/socket';
import socketHandlers from './sockets';

const app = express();
const httpServer = http.createServer(app);
const io = initializeSocket(httpServer);

// Initialize socket handlers
socketHandlers(io);

// Express routes
app.use(express.json());

const PORT = process.env.PORT || 3000;
httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Authentication Middleware

```typescript
// src/middleware/socket-auth.middleware.ts
import { Socket } from 'socket.io';
import jwt from 'jsonwebtoken';
import { prisma } from '../config/database';

export interface AuthenticatedSocket extends Socket {
  userId?: string;
  user?: any;
}

export const socketAuthMiddleware = async (
  socket: AuthenticatedSocket,
  next: (err?: Error) => void
) => {
  try {
    const token = socket.handshake.auth.token || socket.handshake.headers.authorization;

    if (!token) {
      return next(new Error('Authentication required'));
    }

    const decoded = jwt.verify(token, process.env.JWT_SECRET!) as { id: string };

    const user = await prisma.user.findUnique({ where: { id: decoded.id } });

    if (!user) {
      return next(new Error('User not found'));
    }

    socket.userId = user.id;
    socket.user = user;
    next();
  } catch (error) {
    next(new Error('Invalid token'));
  }
};
```

## Socket Handlers

```typescript
// src/sockets/index.ts
import { Server } from 'socket.io';
import { socketAuthMiddleware } from '../middleware/socket-auth.middleware';
import chatHandler from './chat.handler';
import notificationHandler from './notification.handler';

export default (io: Server) => {
  // Apply authentication middleware
  io.use(socketAuthMiddleware);

  io.on('connection', (socket) => {
    console.log('Client connected:', socket.id);

    // Register handlers
    chatHandler(io, socket);
    notificationHandler(io, socket);

    socket.on('disconnect', () => {
      console.log('Client disconnected:', socket.id);
    });
  });
};
```

## Chat Implementation

```typescript
// src/sockets/chat.handler.ts
import { Server, Socket } from 'socket.io';
import { AuthenticatedSocket } from '../middleware/socket-auth.middleware';
import { prisma } from '../config/database';

export default (io: Server, socket: Socket) => {
  const authSocket = socket as AuthenticatedSocket;

  socket.on('chat:join', async (roomId: string) => {
    await socket.join(roomId);
    console.log(`User ${authSocket.userId} joined room ${roomId}`);

    // Notify others in the room
    socket.to(roomId).emit('chat:user_joined', {
      userId: authSocket.userId,
      username: authSocket.user.name,
    });
  });

  socket.on('chat:leave', async (roomId: string) => {
    await socket.leave(roomId);
    
    socket.to(roomId).emit('chat:user_left', {
      userId: authSocket.userId,
      username: authSocket.user.name,
    });
  });

  socket.on('chat:message', async (data: { roomId: string; message: string }) => {
    const { roomId, message } = data;

    // Save message to database
    const savedMessage = await prisma.message.create({
      data: {
        roomId,
        userId: authSocket.userId!,
        content: message,
      },
      include: {
        user: {
          select: { id: true, name: true, avatar: true },
        },
      },
    });

    // Broadcast to room
    io.to(roomId).emit('chat:message', savedMessage);
  });

  socket.on('chat:typing', (data: { roomId: string; isTyping: boolean }) => {
    socket.to(data.roomId).emit('chat:typing', {
      userId: authSocket.userId,
      username: authSocket.user.name,
      isTyping: data.isTyping,
    });
  });
};
```

## Notification Handler

```typescript
// src/sockets/notification.handler.ts
import { Server, Socket } from 'socket.io';
import { AuthenticatedSocket } from '../middleware/socket-auth.middleware';

export default (io: Server, socket: Socket) => {
  const authSocket = socket as AuthenticatedSocket;

  // Join user's personal notification room
  socket.join(`user:${authSocket.userId}`);

  socket.on('notification:mark_read', async (notificationId: string) => {
    await prisma.notification.update({
      where: { id: notificationId },
      data: { read: true },
    });

    socket.emit('notification:read', { notificationId });
  });
};

// Helper function to send notification to specific user
export const sendNotificationToUser = (io: Server, userId: string, notification: any) => {
  io.to(`user:${userId}`).emit('notification:new', notification);
};
```

## Service Integration

```typescript
// src/services/realtime.service.ts
import { Server } from 'socket.io';
import { sendNotificationToUser } from '../sockets/notification.handler';

export class RealtimeService {
  private io: Server | null = null;

  setIO(io: Server) {
    this.io = io;
  }

  async sendNotification(userId: string, notification: any) {
    if (!this.io) {
      console.error('Socket.IO not initialized');
      return;
    }

    sendNotificationToUser(this.io, userId, notification);
  }

  async broadcastToRoom(roomId: string, event: string, data: any) {
    if (!this.io) {
      console.error('Socket.IO not initialized');
      return;
    }

    this.io.to(roomId).emit(event, data);
  }

  async sendToUser(userId: string, event: string, data: any) {
    if (!this.io) {
      console.error('Socket.IO not initialized');
      return;
    }

    this.io.to(`user:${userId}`).emit(event, data);
  }
}

export default new RealtimeService();
```

```typescript
// Update server.ts to inject IO into service
import realtimeService from './services/realtime.service';

const io = initializeSocket(httpServer);
socketHandlers(io);
realtimeService.setIO(io);
```

## Live Dashboard Example

```typescript
// src/sockets/dashboard.handler.ts
import { Server, Socket } from 'socket.io';

export default (io: Server, socket: Socket) => {
  socket.on('dashboard:subscribe', (metrics: string[]) => {
    metrics.forEach((metric) => {
      socket.join(`metric:${metric}`);
    });
  });

  socket.on('dashboard:unsubscribe', (metrics: string[]) => {
    metrics.forEach((metric) => {
      socket.leave(`metric:${metric}`);
    });
  });
};

// Function to push metric updates
export const updateMetric = (io: Server, metricName: string, value: any) => {
  io.to(`metric:${metricName}`).emit('dashboard:update', {
    metric: metricName,
    value,
    timestamp: new Date(),
  });
};
```

## Collaborative Editing

```typescript
// src/sockets/collaboration.handler.ts
import { Server, Socket } from 'socket.io';
import { AuthenticatedSocket } from '../middleware/socket-auth.middleware';

interface CursorPosition {
  userId: string;
  username: string;
  x: number;
  y: number;
}

export default (io: Server, socket: Socket) => {
  const authSocket = socket as AuthenticatedSocket;

  socket.on('collab:join_document', async (documentId: string) => {
    await socket.join(`doc:${documentId}`);

    // Send current active users
    const sockets = await io.in(`doc:${documentId}`).fetchSockets();
    const activeUsers = sockets.map((s: any) => ({
      userId: s.userId,
      username: s.user.name,
    }));

    socket.emit('collab:active_users', activeUsers);

    // Notify others
    socket.to(`doc:${documentId}`).emit('collab:user_joined', {
      userId: authSocket.userId,
      username: authSocket.user.name,
    });
  });

  socket.on('collab:edit', async (data: { documentId: string; changes: any }) => {
    const { documentId, changes } = data;

    // Broadcast changes to others
    socket.to(`doc:${documentId}`).emit('collab:changes', {
      userId: authSocket.userId,
      changes,
    });

    // Optionally save to database
    await prisma.documentVersion.create({
      data: {
        documentId,
        userId: authSocket.userId!,
        changes: JSON.stringify(changes),
      },
    });
  });

  socket.on('collab:cursor_move', (data: { documentId: string; x: number; y: number }) => {
    socket.to(`doc:${data.documentId}`).emit('collab:cursor_update', {
      userId: authSocket.userId,
      username: authSocket.user.name,
      x: data.x,
      y: data.y,
    });
  });
};
```

## Presence System

```typescript
// src/services/presence.service.ts
import { Server } from 'socket.io';
import { redis } from '../config/redis';

export class PresenceService {
  async setUserOnline(userId: string, socketId: string) {
    await redis.sadd(`user:${userId}:sockets`, socketId);
    await redis.set(`user:${userId}:status`, 'online');
    await redis.expire(`user:${userId}:status`, 3600);
  }

  async setUserOffline(userId: string, socketId: string) {
    await redis.srem(`user:${userId}:sockets`, socketId);
    
    const remainingSockets = await redis.smembers(`user:${userId}:sockets`);
    
    if (remainingSockets.length === 0) {
      await redis.set(`user:${userId}:status`, 'offline');
      await redis.set(`user:${userId}:last_seen`, Date.now());
    }
  }

  async getUserStatus(userId: string): Promise<'online' | 'offline'> {
    const status = await redis.get(`user:${userId}:status`);
    return (status as 'online' | 'offline') || 'offline';
  }

  async getOnlineUsers(userIds: string[]): Promise<string[]> {
    const pipeline = redis.pipeline();
    userIds.forEach((id) => pipeline.get(`user:${id}:status`));
    
    const results = await pipeline.exec();
    
    return userIds.filter((_, index) => results?.[index]?.[1] === 'online');
  }
}

export default new PresenceService();
```

## Rate Limiting

```typescript
// src/middleware/socket-rate-limit.middleware.ts
import { Socket } from 'socket.io';
import { RateLimiterMemory } from 'rate-limiter-flexible';

const rateLimiter = new RateLimiterMemory({
  points: 10, // 10 requests
  duration: 1, // per 1 second
});

export const socketRateLimitMiddleware = async (
  socket: Socket,
  next: (err?: Error) => void
) => {
  try {
    await rateLimiter.consume(socket.id);
    next();
  } catch (error) {
    next(new Error('Rate limit exceeded'));
  }
};
```

## Client Example (React)

```typescript
// client/hooks/useSocket.ts
import { useEffect, useState } from 'react';
import { io, Socket } from 'socket.io-client';

export const useSocket = (token: string) => {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const newSocket = io(process.env.NEXT_PUBLIC_API_URL!, {
      auth: { token },
    });

    newSocket.on('connect', () => {
      console.log('Connected to server');
      setConnected(true);
    });

    newSocket.on('disconnect', () => {
      console.log('Disconnected from server');
      setConnected(false);
    });

    setSocket(newSocket);

    return () => {
      newSocket.close();
    };
  }, [token]);

  return { socket, connected };
};
```

```typescript
// client/components/Chat.tsx
import { useEffect, useState } from 'react';
import { useSocket } from '../hooks/useSocket';

export const Chat = ({ roomId, token }: { roomId: string; token: string }) => {
  const { socket } = useSocket(token);
  const [messages, setMessages] = useState<any[]>([]);
  const [input, setInput] = useState('');

  useEffect(() => {
    if (!socket) return;

    socket.emit('chat:join', roomId);

    socket.on('chat:message', (message) => {
      setMessages((prev) => [...prev, message]);
    });

    return () => {
      socket.emit('chat:leave', roomId);
      socket.off('chat:message');
    };
  }, [socket, roomId]);

  const sendMessage = () => {
    if (socket && input.trim()) {
      socket.emit('chat:message', { roomId, message: input });
      setInput('');
    }
  };

  return (
    <div>
      <div>
        {messages.map((msg, i) => (
          <div key={i}>
            <strong>{msg.user.name}:</strong> {msg.content}
          </div>
        ))}
      </div>
      <input value={input} onChange={(e) => setInput(e.target.value)} />
      <button onClick={sendMessage}>Send</button>
    </div>
  );
};
```

## Testing

```typescript
// src/__tests__/socket.test.ts
import { io as Client, Socket } from 'socket.io-client';
import { Server } from 'http';
import app from '../server';

describe('WebSocket Tests', () => {
  let httpServer: Server;
  let clientSocket: Socket;
  let token: string;

  beforeAll((done) => {
    httpServer = app.listen(() => {
      const port = (httpServer.address() as any).port;
      clientSocket = Client(`http://localhost:${port}`, {
        auth: { token },
      });

      clientSocket.on('connect', done);
    });
  });

  afterAll(() => {
    clientSocket.close();
    httpServer.close();
  });

  it('should connect and authenticate', (done) => {
    expect(clientSocket.connected).toBe(true);
    done();
  });

  it('should join a room and receive messages', (done) => {
    const roomId = 'test-room';

    clientSocket.emit('chat:join', roomId);

    clientSocket.on('chat:message', (message) => {
      expect(message.content).toBe('Hello');
      done();
    });

    clientSocket.emit('chat:message', { roomId, message: 'Hello' });
  });
});
```

## Best Practices

1. **Authenticate connections** before allowing socket operations
2. **Use rooms** for efficient broadcasting
3. **Implement rate limiting** to prevent abuse
4. **Handle reconnection** gracefully on the client
5. **Validate all incoming data** from sockets
6. **Store connection state** in Redis for scaling
7. **Use namespaces** to organize different features
8. **Emit acknowledgments** for critical messages
9. **Monitor socket connections** and memory usage
10. **Implement backpressure** for high-throughput scenarios

## Key Takeaways

1. **Socket.IO** simplifies WebSocket implementation
2. **Authentication** is critical for security
3. **Rooms** enable efficient group communication
4. **Presence systems** track online/offline status
5. **Rate limiting** prevents abuse
6. **Redis** helps scale across multiple servers
7. **Client reconnection** improves reliability
8. **Testing** ensures real-time features work correctly

Real-time features dramatically improve user experience when implemented correctly.
