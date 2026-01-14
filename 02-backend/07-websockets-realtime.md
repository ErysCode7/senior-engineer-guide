# WebSockets & Real-time Communication

## Overview

WebSockets provide full-duplex, bidirectional communication between clients and servers over a single TCP connection. Unlike traditional HTTP requests, WebSockets maintain a persistent connection, enabling real-time data exchange with minimal latency and overhead.

**Key Technologies:**

- **Socket.io**: Popular WebSocket library with fallbacks and enhanced features
- **@nestjs/websockets**: NestJS integration for WebSocket gateways
- **@nestjs/platform-socket.io**: Socket.io adapter for NestJS
- **Redis Adapter**: For scaling WebSocket servers across multiple instances

## Practical Use Cases

### 1. **Chat Applications**

Real-time messaging platforms (Slack, Discord, WhatsApp Web)

- Instant message delivery
- Typing indicators
- Read receipts
- Online presence

### 2. **Live Dashboards**

Analytics and monitoring dashboards

- Stock price updates
- Server metrics
- Social media feeds
- Live sports scores

### 3. **Collaborative Tools**

Multi-user editing and collaboration

- Google Docs-style editing
- Project management boards (Trello, Asana)
- Whiteboarding tools (Miro, Figma)
- Code pair programming

### 4. **Notifications**

Push notifications and alerts

- Order status updates
- System alerts
- Social notifications
- IoT device updates

### 5. **Gaming**

Real-time multiplayer games

- Player movement synchronization
- Game state updates
- Matchmaking
- Leaderboards

## Step-by-Step Implementation

### 1. Install Dependencies

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
npm install -D @types/socket.io

# For Redis adapter (scaling)
npm install @socket.io/redis-adapter redis
```

### 2. Basic WebSocket Gateway

```typescript
// chat.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect,
  OnGatewayInit,
} from "@nestjs/websockets";
import { Server, Socket } from "socket.io";
import { Logger } from "@nestjs/common";

interface ChatMessage {
  username: string;
  message: string;
  room?: string;
  timestamp: Date;
}

@WebSocketGateway({
  cors: {
    origin: ["http://localhost:3000", "https://yourdomain.com"],
    credentials: true,
  },
  namespace: "/chat", // Optional namespace
})
export class ChatGateway
  implements OnGatewayInit, OnGatewayConnection, OnGatewayDisconnect
{
  @WebSocketServer()
  server: Server;

  private logger: Logger = new Logger("ChatGateway");
  private connectedUsers: Map<string, string> = new Map(); // socketId -> username

  afterInit(server: Server) {
    this.logger.log("WebSocket Gateway initialized");
  }

  handleConnection(client: Socket) {
    this.logger.log(`Client connected: ${client.id}`);

    // Send welcome message
    client.emit("welcome", {
      message: "Welcome to the chat!",
      timestamp: new Date(),
    });
  }

  handleDisconnect(client: Socket) {
    const username = this.connectedUsers.get(client.id);
    this.connectedUsers.delete(client.id);

    this.logger.log(`Client disconnected: ${client.id} (${username})`);

    // Notify others
    if (username) {
      this.server.emit("userLeft", {
        username,
        timestamp: new Date(),
      });
    }
  }

  @SubscribeMessage("joinChat")
  handleJoinChat(
    @MessageBody() data: { username: string },
    @ConnectedSocket() client: Socket
  ) {
    this.connectedUsers.set(client.id, data.username);

    // Notify all clients
    this.server.emit("userJoined", {
      username: data.username,
      timestamp: new Date(),
    });

    // Send list of online users to the new client
    const onlineUsers = Array.from(this.connectedUsers.values());
    client.emit("onlineUsers", { users: onlineUsers });

    return { success: true, username: data.username };
  }

  @SubscribeMessage("sendMessage")
  handleMessage(
    @MessageBody() data: { message: string },
    @ConnectedSocket() client: Socket
  ) {
    const username = this.connectedUsers.get(client.id);

    if (!username) {
      return { error: "You must join the chat first" };
    }

    const chatMessage: ChatMessage = {
      username,
      message: data.message,
      timestamp: new Date(),
    };

    // Broadcast to all connected clients
    this.server.emit("newMessage", chatMessage);

    return { success: true };
  }

  @SubscribeMessage("typing")
  handleTyping(
    @MessageBody() data: { isTyping: boolean },
    @ConnectedSocket() client: Socket
  ) {
    const username = this.connectedUsers.get(client.id);

    if (username) {
      // Broadcast to all except sender
      client.broadcast.emit("userTyping", {
        username,
        isTyping: data.isTyping,
      });
    }
  }
}
```

### 3. Room-Based Communication

```typescript
// rooms.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
} from "@nestjs/websockets";
import { Server, Socket } from "socket.io";
import { Logger } from "@nestjs/common";

interface RoomMessage {
  username: string;
  room: string;
  message: string;
  timestamp: Date;
}

@WebSocketGateway({ namespace: "/rooms" })
export class RoomsGateway {
  @WebSocketServer()
  server: Server;

  private logger: Logger = new Logger("RoomsGateway");

  @SubscribeMessage("joinRoom")
  async handleJoinRoom(
    @MessageBody() data: { room: string; username: string },
    @ConnectedSocket() client: Socket
  ) {
    // Leave all previous rooms
    const rooms = Array.from(client.rooms).filter((room) => room !== client.id);
    rooms.forEach((room) => client.leave(room));

    // Join new room
    await client.join(data.room);

    this.logger.log(`${data.username} joined room: ${data.room}`);

    // Notify room members
    this.server.to(data.room).emit("userJoinedRoom", {
      username: data.username,
      room: data.room,
      timestamp: new Date(),
    });

    // Get room size
    const roomSize =
      this.server.sockets.adapter.rooms.get(data.room)?.size || 0;

    return {
      success: true,
      room: data.room,
      members: roomSize,
    };
  }

  @SubscribeMessage("leaveRoom")
  async handleLeaveRoom(
    @MessageBody() data: { room: string; username: string },
    @ConnectedSocket() client: Socket
  ) {
    await client.leave(data.room);

    this.logger.log(`${data.username} left room: ${data.room}`);

    // Notify room members
    this.server.to(data.room).emit("userLeftRoom", {
      username: data.username,
      room: data.room,
      timestamp: new Date(),
    });

    return { success: true };
  }

  @SubscribeMessage("sendRoomMessage")
  handleRoomMessage(
    @MessageBody() data: { room: string; username: string; message: string },
    @ConnectedSocket() client: Socket
  ) {
    const roomMessage: RoomMessage = {
      username: data.username,
      room: data.room,
      message: data.message,
      timestamp: new Date(),
    };

    // Send only to room members
    this.server.to(data.room).emit("newRoomMessage", roomMessage);

    return { success: true };
  }

  @SubscribeMessage("getRoomMembers")
  async handleGetRoomMembers(
    @MessageBody() data: { room: string },
    @ConnectedSocket() client: Socket
  ) {
    const room = this.server.sockets.adapter.rooms.get(data.room);

    if (!room) {
      return { members: [] };
    }

    // Get socket IDs in the room
    const socketIds = Array.from(room);

    return {
      room: data.room,
      memberCount: socketIds.length,
      socketIds,
    };
  }
}
```

### 4. Authentication with WebSockets

```typescript
// auth.gateway.ts
import {
  WebSocketGateway,
  WebSocketServer,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from "@nestjs/websockets";
import { Server, Socket } from "socket.io";
import { Logger, UnauthorizedException } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";

interface AuthenticatedSocket extends Socket {
  user?: {
    id: string;
    email: string;
    username: string;
  };
}

@WebSocketGateway({
  cors: {
    origin: process.env.FRONTEND_URL,
    credentials: true,
  },
})
export class AuthGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private logger: Logger = new Logger("AuthGateway");

  constructor(private jwtService: JwtService) {}

  async handleConnection(client: AuthenticatedSocket) {
    try {
      // Extract token from handshake
      const token = this.extractToken(client);

      if (!token) {
        throw new UnauthorizedException("No token provided");
      }

      // Verify JWT token
      const payload = await this.jwtService.verifyAsync(token);

      // Attach user to socket
      client.user = {
        id: payload.sub,
        email: payload.email,
        username: payload.username,
      };

      // Join user-specific room for targeted messages
      await client.join(`user:${payload.sub}`);

      this.logger.log(
        `Authenticated user connected: ${client.user.username} (${client.id})`
      );

      // Send authentication success
      client.emit("authenticated", {
        user: client.user,
        timestamp: new Date(),
      });
    } catch (error) {
      this.logger.error(`Authentication failed: ${error.message}`);

      client.emit("authError", {
        message: "Authentication failed",
        error: error.message,
      });

      client.disconnect();
    }
  }

  handleDisconnect(client: AuthenticatedSocket) {
    if (client.user) {
      this.logger.log(
        `Authenticated user disconnected: ${client.user.username}`
      );
    }
  }

  private extractToken(client: Socket): string | null {
    // Method 1: From query parameters
    const queryToken = client.handshake.query.token as string;
    if (queryToken) return queryToken;

    // Method 2: From headers
    const authHeader = client.handshake.headers.authorization;
    if (authHeader?.startsWith("Bearer ")) {
      return authHeader.substring(7);
    }

    // Method 3: From auth object
    const authToken = client.handshake.auth?.token;
    if (authToken) return authToken;

    return null;
  }

  // Send message to specific user
  sendToUser(userId: string, event: string, data: any) {
    this.server.to(`user:${userId}`).emit(event, data);
  }

  // Broadcast to all authenticated users
  broadcastToAuthenticated(event: string, data: any) {
    this.server.emit(event, data);
  }
}
```

### 5. Redis Adapter for Scaling

```typescript
// websocket.adapter.ts
import { IoAdapter } from "@nestjs/platform-socket.io";
import { ServerOptions } from "socket.io";
import { createAdapter } from "@socket.io/redis-adapter";
import { createClient } from "redis";

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor: ReturnType<typeof createAdapter>;

  async connectToRedis(): Promise<void> {
    const pubClient = createClient({
      url: process.env.REDIS_URL || "redis://localhost:6379",
    });
    const subClient = pubClient.duplicate();

    await Promise.all([pubClient.connect(), subClient.connect()]);

    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): any {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor);
    return server;
  }
}
```

```typescript
// main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { RedisIoAdapter } from "./websocket.adapter";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Use Redis adapter for WebSocket scaling
  const redisIoAdapter = new RedisIoAdapter(app);
  await redisIoAdapter.connectToRedis();
  app.useWebSocketAdapter(redisIoAdapter);

  await app.listen(3000);
}
bootstrap();
```

### 6. Client-Side Implementation (TypeScript/React)

```typescript
// useSocket.ts
import { useEffect, useState, useCallback } from "react";
import { io, Socket } from "socket.io-client";

interface UseSocketOptions {
  namespace?: string;
  token?: string;
}

export function useSocket(options: UseSocketOptions = {}) {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [isConnected, setIsConnected] = useState(false);

  useEffect(() => {
    const socketInstance = io(
      `${process.env.NEXT_PUBLIC_WS_URL || "http://localhost:3000"}${
        options.namespace || ""
      }`,
      {
        auth: {
          token: options.token,
        },
        transports: ["websocket", "polling"], // Fallback to polling
      }
    );

    socketInstance.on("connect", () => {
      console.log("Socket connected:", socketInstance.id);
      setIsConnected(true);
    });

    socketInstance.on("disconnect", () => {
      console.log("Socket disconnected");
      setIsConnected(false);
    });

    socketInstance.on("connect_error", (error) => {
      console.error("Connection error:", error);
    });

    setSocket(socketInstance);

    return () => {
      socketInstance.disconnect();
    };
  }, [options.namespace, options.token]);

  const emit = useCallback(
    (event: string, data?: any) => {
      if (socket) {
        socket.emit(event, data);
      }
    },
    [socket]
  );

  const on = useCallback(
    (event: string, handler: (...args: any[]) => void) => {
      if (socket) {
        socket.on(event, handler);
        return () => socket.off(event, handler);
      }
    },
    [socket]
  );

  return {
    socket,
    isConnected,
    emit,
    on,
  };
}
```

```typescript
// ChatComponent.tsx
import React, { useState, useEffect } from "react";
import { useSocket } from "./useSocket";

interface Message {
  username: string;
  message: string;
  timestamp: Date;
}

export function ChatComponent() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [inputMessage, setInputMessage] = useState("");
  const [username, setUsername] = useState("");
  const [hasJoined, setHasJoined] = useState(false);

  const { socket, isConnected, emit, on } = useSocket({
    namespace: "/chat",
  });

  useEffect(() => {
    if (!socket) return;

    const unsubscribe = on("newMessage", (message: Message) => {
      setMessages((prev) => [...prev, message]);
    });

    return unsubscribe;
  }, [socket, on]);

  const handleJoin = () => {
    if (username.trim()) {
      emit("joinChat", { username });
      setHasJoined(true);
    }
  };

  const handleSendMessage = () => {
    if (inputMessage.trim()) {
      emit("sendMessage", { message: inputMessage });
      setInputMessage("");
    }
  };

  if (!hasJoined) {
    return (
      <div className="p-4">
        <input
          type="text"
          placeholder="Enter username"
          value={username}
          onChange={(e) => setUsername(e.target.value)}
          className="border p-2 rounded"
        />
        <button
          onClick={handleJoin}
          className="ml-2 bg-blue-500 text-white px-4 py-2 rounded"
        >
          Join Chat
        </button>
      </div>
    );
  }

  return (
    <div className="flex flex-col h-screen p-4">
      <div className="flex-1 overflow-y-auto border p-4 mb-4">
        {messages.map((msg, index) => (
          <div key={index} className="mb-2">
            <strong>{msg.username}:</strong> {msg.message}
            <span className="text-xs text-gray-500 ml-2">
              {new Date(msg.timestamp).toLocaleTimeString()}
            </span>
          </div>
        ))}
      </div>

      <div className="flex gap-2">
        <input
          type="text"
          value={inputMessage}
          onChange={(e) => setInputMessage(e.target.value)}
          onKeyPress={(e) => e.key === "Enter" && handleSendMessage()}
          placeholder="Type a message..."
          className="flex-1 border p-2 rounded"
          disabled={!isConnected}
        />
        <button
          onClick={handleSendMessage}
          disabled={!isConnected}
          className="bg-blue-500 text-white px-4 py-2 rounded disabled:opacity-50"
        >
          Send
        </button>
      </div>

      <div className="text-sm mt-2">
        Status: {isConnected ? "🟢 Connected" : "🔴 Disconnected"}
      </div>
    </div>
  );
}
```

## Best Practices

### 1. **Connection Management**

```typescript
// Implement reconnection logic
const socket = io(url, {
  reconnection: true,
  reconnectionDelay: 1000,
  reconnectionDelayMax: 5000,
  reconnectionAttempts: 5,
});

// Handle reconnection events
socket.on("reconnect", (attemptNumber) => {
  console.log(`Reconnected after ${attemptNumber} attempts`);
  // Re-join rooms, re-subscribe to events
});
```

### 2. **Rate Limiting**

```typescript
// Implement rate limiting for WebSocket events
import { RateLimiterMemory } from 'rate-limiter-flexible';

const rateLimiter = new RateLimiterMemory({
  points: 10, // 10 messages
  duration: 1, // per 1 second
});

@SubscribeMessage('sendMessage')
async handleMessage(
  @MessageBody() data: any,
  @ConnectedSocket() client: Socket,
) {
  try {
    await rateLimiter.consume(client.id);
    // Process message
  } catch {
    client.emit('error', { message: 'Rate limit exceeded' });
  }
}
```

### 3. **Message Validation**

```typescript
// Use DTOs for WebSocket messages
import { IsString, IsNotEmpty, MaxLength } from 'class-validator';

export class SendMessageDto {
  @IsString()
  @IsNotEmpty()
  @MaxLength(500)
  message: string;
}

// Validate in gateway
@UsePipes(new ValidationPipe())
@SubscribeMessage('sendMessage')
handleMessage(@MessageBody() dto: SendMessageDto) {
  // Message is validated
}
```

### 4. **Error Handling**

```typescript
// Comprehensive error handling
@SubscribeMessage('sendMessage')
handleMessage(@MessageBody() data: any, @ConnectedSocket() client: Socket) {
  try {
    // Process message
    return { success: true };
  } catch (error) {
    this.logger.error(`Error processing message: ${error.message}`);
    client.emit('error', {
      message: 'Failed to send message',
      code: 'MESSAGE_SEND_ERROR',
    });
    return { success: false, error: error.message };
  }
}
```

### 5. **Memory Management**

```typescript
// Clean up on disconnect
handleDisconnect(client: Socket) {
  // Remove from tracking maps
  this.connectedUsers.delete(client.id);
  this.userSessions.delete(client.id);

  // Leave all rooms
  const rooms = Array.from(client.rooms);
  rooms.forEach(room => client.leave(room));

  // Clear any timers or intervals
  this.clearUserTimers(client.id);
}
```

### 6. **Security Considerations**

- Always validate and sanitize incoming messages
- Implement authentication for sensitive operations
- Use CORS properly to restrict origins
- Rate limit connections and messages
- Encrypt sensitive data in transit (use WSS)
- Validate room access permissions

### 7. **Performance Optimization**

- Use binary data when possible (ArrayBuffer, Buffer)
- Implement message batching for high-frequency updates
- Use compression for large messages
- Monitor and limit room sizes
- Implement connection pooling

### 8. **Monitoring and Debugging**

```typescript
// Add comprehensive logging
afterInit(server: Server) {
  this.logger.log('WebSocket server initialized');

  // Monitor connections
  setInterval(() => {
    const clientsCount = this.server.sockets.sockets.size;
    const roomsCount = this.server.sockets.adapter.rooms.size;
    this.logger.debug(`Clients: ${clientsCount}, Rooms: ${roomsCount}`);
  }, 60000); // Every minute
}
```

## Key Takeaways

✅ **WebSockets enable real-time bidirectional communication**  
✅ **Use Socket.io for enhanced features and fallbacks**  
✅ **Implement authentication to secure connections**  
✅ **Use rooms for targeted message broadcasting**  
✅ **Scale with Redis adapter across multiple servers**  
✅ **Always validate and rate-limit incoming messages**  
✅ **Handle reconnection gracefully on client side**  
✅ **Monitor connection counts and performance metrics**  
✅ **Clean up resources properly on disconnect**  
✅ **Use namespaces to separate different features**

WebSockets are essential for modern real-time applications, providing instant updates and interactive experiences that traditional HTTP polling cannot match.
