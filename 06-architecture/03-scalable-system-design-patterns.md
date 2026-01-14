# Scalable System Design Patterns

## Overview

Scalable system design patterns provide proven solutions for building systems that can handle growing loads efficiently. These patterns address challenges in distributed systems including data consistency, availability, performance, and reliability. Understanding and applying these patterns is essential for building production-ready applications that scale from hundreds to millions of users.

**Key Pattern Categories:**

- **Data Patterns**: Sharding, replication, caching
- **Communication Patterns**: Load balancing, message queues, pub/sub
- **Resilience Patterns**: Circuit breakers, retry, fallback
- **Performance Patterns**: CDN, lazy loading, pagination
- **Consistency Patterns**: CAP theorem, eventual consistency

## Practical Use Cases

### 1. **High-Traffic Applications**

Handle millions of requests

- E-commerce platforms
- Social media apps
- Streaming services
- Gaming platforms

### 2. **Global Distribution**

Serve users worldwide

- Multi-region deployment
- CDN for static assets
- Geo-routing
- Edge computing

### 3. **Real-Time Systems**

Low-latency requirements

- Chat applications
- Live updates
- Gaming
- Financial trading

### 4. **Data-Intensive Applications**

Process large datasets

- Analytics platforms
- Data pipelines
- Machine learning
- Big data processing

### 5. **Fault-Tolerant Systems**

High availability requirements

- Banking systems
- Healthcare applications
- Critical infrastructure
- SaaS platforms

## Caching Patterns

### 1. Cache-Aside Pattern

```typescript
// Cache-aside (Lazy Loading)
class UserService {
  constructor(
    private userRepository: UserRepository,
    private cacheClient: Redis
  ) {}

  async getUser(userId: string): Promise<User> {
    const cacheKey = `user:${userId}`;

    // 1. Try cache first
    const cached = await this.cacheClient.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // 2. Cache miss - fetch from database
    const user = await this.userRepository.findById(userId);

    if (!user) {
      throw new NotFoundException("User not found");
    }

    // 3. Store in cache for next time
    await this.cacheClient.setex(
      cacheKey,
      3600, // 1 hour TTL
      JSON.stringify(user)
    );

    return user;
  }

  async updateUser(userId: string, data: UpdateUserDto): Promise<User> {
    // Update database
    const user = await this.userRepository.update(userId, data);

    // Invalidate cache
    await this.cacheClient.del(`user:${userId}`);

    return user;
  }
}
```

### 2. Write-Through Cache

```typescript
// Write-through: Write to cache and database simultaneously
class ProductService {
  constructor(
    private productRepository: ProductRepository,
    private cacheClient: Redis
  ) {}

  async updateProduct(
    productId: string,
    data: UpdateProductDto
  ): Promise<Product> {
    // 1. Update database
    const product = await this.productRepository.update(productId, data);

    // 2. Update cache immediately
    const cacheKey = `product:${productId}`;
    await this.cacheClient.setex(cacheKey, 3600, JSON.stringify(product));

    return product;
  }
}
```

### 3. Read-Through Cache

```typescript
// Read-through: Cache automatically loads from database
class CacheService {
  constructor(private redis: Redis, private db: Database) {}

  async get<T>(
    key: string,
    loader: () => Promise<T>,
    ttl: number = 3600
  ): Promise<T> {
    // Try cache
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached) as T;
    }

    // Load from source
    const data = await loader();

    // Store in cache
    await this.redis.setex(key, ttl, JSON.stringify(data));

    return data;
  }
}

// Usage
const product = await cacheService.get(
  `product:${productId}`,
  () => productRepository.findById(productId),
  3600
);
```

## Database Sharding

```typescript
// Horizontal sharding by user ID
class ShardedUserRepository {
  private shards: Database[];

  constructor(shards: Database[]) {
    this.shards = shards;
  }

  private getShard(userId: string): Database {
    // Hash-based sharding
    const hash = this.hashString(userId);
    const shardIndex = hash % this.shards.length;
    return this.shards[shardIndex];
  }

  private hashString(str: string): number {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = (hash << 5) - hash + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }

  async findById(userId: string): Promise<User | null> {
    const shard = this.getShard(userId);
    return shard.query("SELECT * FROM users WHERE id = $1", [userId]);
  }

  async save(user: User): Promise<void> {
    const shard = this.getShard(user.id);
    await shard.query(
      "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
      [user.id, user.name, user.email]
    );
  }
}

// Consistent hashing for better distribution
class ConsistentHashSharding {
  private ring: Map<number, Database> = new Map();
  private virtualNodes = 150;

  constructor(shards: Database[]) {
    // Create virtual nodes for each shard
    shards.forEach((shard, index) => {
      for (let i = 0; i < this.virtualNodes; i++) {
        const hash = this.hash(`shard-${index}-vnode-${i}`);
        this.ring.set(hash, shard);
      }
    });
  }

  private hash(key: string): number {
    // Simple hash function (use crypto.createHash in production)
    let hash = 0;
    for (let i = 0; i < key.length; i++) {
      hash = (hash << 5) - hash + key.charCodeAt(i);
    }
    return Math.abs(hash);
  }

  getShard(key: string): Database {
    const keyHash = this.hash(key);
    const sortedHashes = Array.from(this.ring.keys()).sort((a, b) => a - b);

    // Find the first node >= keyHash
    for (const hash of sortedHashes) {
      if (hash >= keyHash) {
        return this.ring.get(hash)!;
      }
    }

    // Wrap around to first node
    return this.ring.get(sortedHashes[0])!;
  }
}
```

## Load Balancing

```typescript
// Round-robin load balancer
class RoundRobinLoadBalancer {
  private currentIndex = 0;

  constructor(private servers: string[]) {}

  getNextServer(): string {
    const server = this.servers[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.servers.length;
    return server;
  }
}

// Weighted load balancer
class WeightedLoadBalancer {
  private servers: Array<{ url: string; weight: number }>;
  private totalWeight: number;

  constructor(servers: Array<{ url: string; weight: number }>) {
    this.servers = servers;
    this.totalWeight = servers.reduce((sum, s) => sum + s.weight, 0);
  }

  getNextServer(): string {
    const random = Math.random() * this.totalWeight;
    let cumulative = 0;

    for (const server of this.servers) {
      cumulative += server.weight;
      if (random < cumulative) {
        return server.url;
      }
    }

    return this.servers[0].url;
  }
}

// Least connections load balancer
class LeastConnectionsLoadBalancer {
  private connections: Map<string, number> = new Map();

  constructor(private servers: string[]) {
    servers.forEach((server) => this.connections.set(server, 0));
  }

  getNextServer(): string {
    let minConnections = Infinity;
    let selectedServer = this.servers[0];

    this.servers.forEach((server) => {
      const connections = this.connections.get(server) || 0;
      if (connections < minConnections) {
        minConnections = connections;
        selectedServer = server;
      }
    });

    return selectedServer;
  }

  incrementConnections(server: string): void {
    const current = this.connections.get(server) || 0;
    this.connections.set(server, current + 1);
  }

  decrementConnections(server: string): void {
    const current = this.connections.get(server) || 0;
    this.connections.set(server, Math.max(0, current - 1));
  }
}
```

## Circuit Breaker Pattern

```typescript
// Circuit breaker for external service calls
enum CircuitState {
  CLOSED = "CLOSED",
  OPEN = "OPEN",
  HALF_OPEN = "HALF_OPEN",
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failureCount = 0;
  private successCount = 0;
  private lastFailureTime: number | null = null;

  constructor(
    private threshold: number = 5,
    private timeout: number = 60000, // 1 minute
    private halfOpenSuccessThreshold: number = 2
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (this.shouldAttemptReset()) {
        this.state = CircuitState.HALF_OPEN;
      } else {
        throw new Error("Circuit breaker is OPEN");
      }
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private shouldAttemptReset(): boolean {
    return (
      this.lastFailureTime !== null &&
      Date.now() - this.lastFailureTime >= this.timeout
    );
  }

  private onSuccess(): void {
    this.failureCount = 0;

    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.halfOpenSuccessThreshold) {
        this.state = CircuitState.CLOSED;
        this.successCount = 0;
      }
    }
  }

  private onFailure(): void {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    this.successCount = 0;

    if (this.failureCount >= this.threshold) {
      this.state = CircuitState.OPEN;
    }
  }

  getState(): CircuitState {
    return this.state;
  }
}

// Usage with external API
class PaymentService {
  private circuitBreaker = new CircuitBreaker(5, 60000);

  async processPayment(data: PaymentData): Promise<PaymentResult> {
    try {
      return await this.circuitBreaker.execute(async () => {
        return this.stripeClient.charge(data);
      });
    } catch (error) {
      // Fallback to queuing for later processing
      await this.queuePayment(data);
      throw error;
    }
  }
}
```

## Message Queue Pattern

```typescript
// Producer-consumer with Bull queue
import { Queue, Worker, Job } from "bullmq";

// Producer
class OrderService {
  private orderQueue: Queue;

  constructor() {
    this.orderQueue = new Queue("orders", {
      connection: {
        host: "localhost",
        port: 6379,
      },
    });
  }

  async createOrder(orderData: CreateOrderDto): Promise<Order> {
    // Save order in database
    const order = await this.orderRepository.save(orderData);

    // Add to queue for async processing
    await this.orderQueue.add(
      "process-order",
      {
        orderId: order.id,
        userId: orderData.userId,
        items: orderData.items,
      },
      {
        attempts: 3,
        backoff: {
          type: "exponential",
          delay: 2000,
        },
      }
    );

    return order;
  }
}

// Consumer
class OrderProcessor {
  private worker: Worker;

  constructor(
    private paymentService: PaymentService,
    private inventoryService: InventoryService,
    private notificationService: NotificationService
  ) {
    this.worker = new Worker(
      "orders",
      async (job: Job) => {
        return this.processOrder(job.data);
      },
      {
        connection: {
          host: "localhost",
          port: 6379,
        },
        concurrency: 5, // Process 5 orders simultaneously
      }
    );

    this.worker.on("completed", (job) => {
      console.log(`Order ${job.data.orderId} processed successfully`);
    });

    this.worker.on("failed", (job, error) => {
      console.error(`Order ${job?.data.orderId} failed:`, error);
    });
  }

  private async processOrder(data: any): Promise<void> {
    // Reserve inventory
    await this.inventoryService.reserveItems(data.items);

    // Process payment
    const payment = await this.paymentService.processPayment({
      orderId: data.orderId,
      amount: data.total,
    });

    if (payment.status !== "success") {
      // Release inventory on payment failure
      await this.inventoryService.releaseItems(data.items);
      throw new Error("Payment failed");
    }

    // Send confirmation
    await this.notificationService.sendOrderConfirmation(data.orderId);
  }
}
```

## Event Sourcing Pattern

```typescript
// Event store
interface DomainEvent {
  eventId: string;
  aggregateId: string;
  eventType: string;
  data: any;
  timestamp: Date;
  version: number;
}

class EventStore {
  constructor(private db: Database) {}

  async saveEvent(event: DomainEvent): Promise<void> {
    await this.db.query(
      `INSERT INTO events (event_id, aggregate_id, event_type, data, timestamp, version)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [
        event.eventId,
        event.aggregateId,
        event.eventType,
        JSON.stringify(event.data),
        event.timestamp,
        event.version,
      ]
    );
  }

  async getEvents(aggregateId: string): Promise<DomainEvent[]> {
    const result = await this.db.query(
      `SELECT * FROM events WHERE aggregate_id = $1 ORDER BY version ASC`,
      [aggregateId]
    );
    return result.rows;
  }
}

// Aggregate
class OrderAggregate {
  private version = 0;
  private events: DomainEvent[] = [];

  constructor(
    public id: string,
    public status: string = "pending",
    public items: OrderItem[] = [],
    public total: number = 0
  ) {}

  static fromEvents(events: DomainEvent[]): OrderAggregate {
    const order = new OrderAggregate(events[0].aggregateId);

    events.forEach((event) => {
      order.apply(event);
    });

    return order;
  }

  createOrder(items: OrderItem[], total: number): void {
    const event: DomainEvent = {
      eventId: uuid(),
      aggregateId: this.id,
      eventType: "OrderCreated",
      data: { items, total },
      timestamp: new Date(),
      version: ++this.version,
    };

    this.apply(event);
    this.events.push(event);
  }

  confirmPayment(): void {
    if (this.status !== "pending") {
      throw new Error("Order is not in pending state");
    }

    const event: DomainEvent = {
      eventId: uuid(),
      aggregateId: this.id,
      eventType: "PaymentConfirmed",
      data: {},
      timestamp: new Date(),
      version: ++this.version,
    };

    this.apply(event);
    this.events.push(event);
  }

  private apply(event: DomainEvent): void {
    switch (event.eventType) {
      case "OrderCreated":
        this.items = event.data.items;
        this.total = event.data.total;
        this.status = "pending";
        break;
      case "PaymentConfirmed":
        this.status = "confirmed";
        break;
      case "OrderShipped":
        this.status = "shipped";
        break;
      case "OrderCancelled":
        this.status = "cancelled";
        break;
    }
    this.version = event.version;
  }

  getUncommittedEvents(): DomainEvent[] {
    return this.events;
  }

  clearEvents(): void {
    this.events = [];
  }
}
```

## CQRS Pattern

```typescript
// Command side (Write model)
class CreateOrderCommand {
  constructor(
    public userId: string,
    public items: OrderItem[],
    public total: number
  ) {}
}

class OrderCommandHandler {
  constructor(
    private orderRepository: OrderRepository,
    private eventBus: EventBus
  ) {}

  async handle(command: CreateOrderCommand): Promise<string> {
    const order = await this.orderRepository.create({
      userId: command.userId,
      items: command.items,
      total: command.total,
      status: "pending",
    });

    // Publish event for read model
    await this.eventBus.publish(new OrderCreatedEvent(order));

    return order.id;
  }
}

// Query side (Read model)
class OrderReadModel {
  constructor(
    public id: string,
    public userId: string,
    public status: string,
    public total: number,
    public itemCount: number,
    public createdAt: Date
  ) {}
}

class OrderQueryHandler {
  constructor(private readDb: Database) {}

  async getOrderById(orderId: string): Promise<OrderReadModel> {
    const result = await this.readDb.query(
      "SELECT * FROM order_read_model WHERE id = $1",
      [orderId]
    );
    return result.rows[0];
  }

  async getUserOrders(userId: string): Promise<OrderReadModel[]> {
    const result = await this.readDb.query(
      "SELECT * FROM order_read_model WHERE user_id = $1 ORDER BY created_at DESC",
      [userId]
    );
    return result.rows;
  }
}

// Read model projector
class OrderProjector {
  constructor(private readDb: Database) {}

  async onOrderCreated(event: OrderCreatedEvent): Promise<void> {
    await this.readDb.query(
      `INSERT INTO order_read_model (id, user_id, status, total, item_count, created_at)
       VALUES ($1, $2, $3, $4, $5, $6)`,
      [
        event.orderId,
        event.userId,
        event.status,
        event.total,
        event.items.length,
        event.timestamp,
      ]
    );
  }

  async onOrderStatusChanged(event: OrderStatusChangedEvent): Promise<void> {
    await this.readDb.query(
      "UPDATE order_read_model SET status = $1 WHERE id = $2",
      [event.newStatus, event.orderId]
    );
  }
}
```

## Database Replication

```typescript
// Read replica pattern
class DatabaseService {
  constructor(private master: Database, private replicas: Database[]) {}

  // Writes go to master
  async write(query: string, params: any[]): Promise<any> {
    return this.master.query(query, params);
  }

  // Reads use replicas (load balanced)
  async read(query: string, params: any[]): Promise<any> {
    const replica = this.getReadReplica();
    return replica.query(query, params);
  }

  private getReadReplica(): Database {
    // Round-robin selection
    const index = Math.floor(Math.random() * this.replicas.length);
    return this.replicas[index];
  }
}

// Usage
class UserRepository {
  constructor(private db: DatabaseService) {}

  async findById(id: string): Promise<User> {
    return this.db.read("SELECT * FROM users WHERE id = $1", [id]);
  }

  async save(user: User): Promise<void> {
    await this.db.write(
      "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
      [user.id, user.name, user.email]
    );
  }
}
```

## Best Practices

### 1. **Caching**

- Cache frequently accessed data
- Set appropriate TTL values
- Implement cache invalidation
- Use cache-aside for flexibility
- Monitor cache hit rates

### 2. **Database Scaling**

- Use read replicas for reads
- Shard large datasets
- Index frequently queried columns
- Use connection pooling
- Monitor query performance

### 3. **Asynchronous Processing**

- Use message queues for long tasks
- Implement retry logic
- Handle failures gracefully
- Monitor queue depth
- Scale consumers independently

### 4. **Resilience**

- Implement circuit breakers
- Use timeouts
- Have fallback strategies
- Graceful degradation
- Health checks

### 5. **Monitoring**

- Track key metrics
- Set up alerts
- Log important events
- Distributed tracing
- Performance profiling

## Key Takeaways

✅ **Caching reduces database load and improves performance**  
✅ **Sharding enables horizontal database scaling**  
✅ **Load balancing distributes traffic across servers**  
✅ **Circuit breakers prevent cascading failures**  
✅ **Message queues enable async processing**  
✅ **Event sourcing provides complete audit trail**  
✅ **CQRS separates read and write models**  
✅ **Database replication improves read performance**  
✅ **Consistent hashing enables smooth scaling**  
✅ **Monitor everything to identify bottlenecks**

Scalable system design is about making intentional trade-offs between consistency, availability, and performance based on your specific requirements.
