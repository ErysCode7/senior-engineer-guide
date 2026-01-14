# Monolithic vs Microservices Architecture

## Overview

Monolithic and microservices architectures represent two fundamental approaches to structuring applications. Monolithic architecture bundles all functionality into a single deployable unit, while microservices decompose applications into independent, loosely coupled services. Each approach has distinct trade-offs in terms of scalability, maintainability, deployment complexity, and team organization.

**Key Characteristics:**

**Monolithic:**

- Single codebase and deployment unit
- Shared database
- Tight coupling between components
- Simpler initial development
- Vertical scaling

**Microservices:**

- Multiple independent services
- Service-specific databases
- Loose coupling via APIs
- Complex infrastructure
- Horizontal scaling

## Practical Use Cases

### 1. **Monolithic - Startups & MVPs**

Rapid development and iteration

- New product validation
- Small teams
- Limited resources
- Simple deployment
- Fast time-to-market

### 2. **Microservices - Large Scale**

Enterprise applications

- High traffic systems
- Multiple teams
- Independent scaling
- Technology diversity
- Continuous deployment

### 3. **Hybrid Approach**

Modular monolith

- Domain-driven modules
- Clear boundaries
- Single deployment
- Migration path
- Gradual evolution

### 4. **Event-Driven Architecture**

Asynchronous communication

- Order processing
- Notification systems
- Data pipelines
- Real-time analytics
- Workflow orchestration

### 5. **Service-Oriented Architecture (SOA)**

Enterprise integration

- Legacy system integration
- B2B communication
- API gateways
- Service bus patterns
- Enterprise service bus

## Monolithic Architecture

### 1. Well-Structured Monolith

```typescript
// Modular monolith structure
src/
├── modules/
│   ├── users/
│   │   ├── users.module.ts
│   │   ├── users.service.ts
│   │   ├── users.controller.ts
│   │   ├── users.repository.ts
│   │   └── dto/
│   ├── products/
│   │   ├── products.module.ts
│   │   ├── products.service.ts
│   │   ├── products.controller.ts
│   │   └── products.repository.ts
│   ├── orders/
│   │   ├── orders.module.ts
│   │   ├── orders.service.ts
│   │   ├── orders.controller.ts
│   │   └── orders.repository.ts
│   └── payments/
│       ├── payments.module.ts
│       ├── payments.service.ts
│       └── payments.controller.ts
├── shared/
│   ├── database/
│   ├── config/
│   ├── guards/
│   └── utils/
├── app.module.ts
└── main.ts

// modules/orders/orders.module.ts
import { OrdersService } from './orders.service';
import { OrdersController } from './orders.controller';
import { OrdersRepository } from './orders.repository';
import { ProductsModule } from '../products/products.module';
import { PaymentsModule } from '../payments/payments.module';

  imports: [
    ProductsModule, // Import other modules
    PaymentsModule,
  ],
  controllers: [OrdersController],
  providers: [OrdersService, OrdersRepository],
  exports: [OrdersService], // Export for other modules
})
export class OrdersModule {}

// modules/orders/orders.service.ts
import { ProductsService } from '../products/products.service';
import { PaymentsService } from '../payments/payments.service';
import { OrdersRepository } from './orders.repository';

export class OrdersService {
  constructor(
    private ordersRepository: OrdersRepository,
    private productsService: ProductsService,
    private paymentsService: PaymentsService,
  ) {}

  async createOrder(userId: string, items: OrderItem[]) {
    // Validate products
    for (const item of items) {
      const product = await this.productsService.findById(item.productId);
      if (!product || product.stock < item.quantity) {
        throw new Error('Product unavailable');
      }
    }

    // Calculate total
    const total = await this.calculateTotal(items);

    // Create order
    const order = await this.ordersRepository.create({
      userId,
      items,
      total,
      status: 'pending',
    });

    // Process payment
    const payment = await this.paymentsService.processPayment({
      orderId: order.id,
      amount: total,
      userId,
    });

    // Update order status
    if (payment.status === 'success') {
      await this.ordersRepository.update(order.id, { status: 'confirmed' });

      // Update product stock
      for (const item of items) {
        await this.productsService.decrementStock(item.productId, item.quantity);
      }
    }

    return order;
  }

  private async calculateTotal(items: OrderItem[]): Promise<number> {
    let total = 0;
    for (const item of items) {
      const product = await this.productsService.findById(item.productId);
      total += product.price * item.quantity;
    }
    return total;
  }
}
```

### 2. Monolith Advantages

```typescript
// Simple transaction management

export class OrdersService {
  constructor(
    private dataSource: DataSource,
    private ordersRepository: OrdersRepository,
    private productsRepository: ProductsRepository,
    private paymentsRepository: PaymentsRepository
  ) {}

  async createOrder(data: CreateOrderDto) {
    // Single database transaction across all entities
    return this.dataSource.transaction(async (manager) => {
      // Create order
      const order = await manager.save(Order, {
        userId: data.userId,
        items: data.items,
        total: data.total,
      });

      // Update product stock
      for (const item of data.items) {
        await manager.decrement(
          Product,
          { id: item.productId },
          "stock",
          item.quantity
        );
      }

      // Create payment
      const payment = await manager.save(Payment, {
        orderId: order.id,
        amount: data.total,
        status: "pending",
      });

      return { order, payment };
    });
  }
}

// Simple debugging and testing

export class OrdersService {
  // Direct method calls, easy to trace
  async createOrder(data: CreateOrderDto) {
    const product = await this.productsService.getProduct(data.productId);
    const user = await this.usersService.getUser(data.userId);
    const order = await this.ordersRepository.save({ ...data });
    await this.notificationsService.sendOrderConfirmation(order);
    return order;
  }
}
```

## Microservices Architecture

### 1. Service Structure

```typescript
// Service 1: User Service
// user-service/src/main.ts
import { AppModule } from "./app.module";

async function bootstrap() {
  // HTTP API
  const app = express();
  app.enableCors();
  await app.listen(3001);

  // Message queue for async communication
  const microservice = app.connectMicroservice<MicroserviceOptions>({
    transport: Transport.RMQ,
    options: {
      urls: [process.env.RABBITMQ_URL],
      queue: "user_queue",
      queueOptions: { durable: true },
    },
  });

  await app.startAllMicroservices();
}

bootstrap();

// user-service/src/users/users.controller.ts

export class UsersController {
  constructor(private usersService: UsersService) {}

  // HTTP endpoint
  @Get(":id")
  async getUser(@Param("id") id: string) {
    return this.usersService.findById(id);
  }

  // Message pattern for inter-service communication
  @MessagePattern("user.get")
  async getUserByMessage(@Payload() data: { id: string }) {
    return this.usersService.findById(data.id);
  }

  @MessagePattern("user.validate")
  async validateUser(@Payload() data: { email: string; password: string }) {
    return this.usersService.validateCredentials(data.email, data.password);
  }
}

// Service 2: Order Service
// order-service/src/orders/orders.service.ts
import { firstValueFrom } from "rxjs";

export class OrdersService {
  constructor(
    @Inject("USER_SERVICE") private userClient: ClientProxy,
    @Inject("PRODUCT_SERVICE") private productClient: ClientProxy,
    @Inject("PAYMENT_SERVICE") private paymentClient: ClientProxy,
    private ordersRepository: OrdersRepository
  ) {}

  async createOrder(data: CreateOrderDto) {
    // Call user service to validate user
    const user = await firstValueFrom(
      this.userClient.send("user.get", { id: data.userId })
    );

    if (!user) {
      throw new Error("User not found");
    }

    // Call product service to validate products and get prices
    const products = await firstValueFrom(
      this.productClient.send("product.getMultiple", {
        ids: data.items.map((item) => item.productId),
      })
    );

    // Calculate total
    let total = 0;
    for (const item of data.items) {
      const product = products.find((p) => p.id === item.productId);
      if (!product || product.stock < item.quantity) {
        throw new Error(`Product ${item.productId} unavailable`);
      }
      total += product.price * item.quantity;
    }

    // Create order in local database
    const order = await this.ordersRepository.create({
      userId: data.userId,
      items: data.items,
      total,
      status: "pending",
    });

    // Call payment service
    const payment = await firstValueFrom(
      this.paymentClient.send("payment.process", {
        orderId: order.id,
        amount: total,
        userId: data.userId,
      })
    );

    // Update order status based on payment
    if (payment.status === "success") {
      await this.ordersRepository.update(order.id, { status: "confirmed" });

      // Update product stock (async message)
      for (const item of data.items) {
        this.productClient.emit("product.decrementStock", {
          productId: item.productId,
          quantity: item.quantity,
        });
      }

      // Send notification (async)
      this.userClient.emit("notification.send", {
        userId: data.userId,
        type: "order_confirmed",
        orderId: order.id,
      });
    }

    return order;
  }
}
```

### 2. API Gateway Pattern

```typescript
// api-gateway/src/main.ts
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = express();

  // Global prefix
  app.setGlobalPrefix("api");

  // CORS
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(","),
    credentials: true,
  });

  await app.listen(3000);
}

bootstrap();

// api-gateway/src/gateway.module.ts

  imports: [
    ClientsModule.register([
      {
        name: "USER_SERVICE",
        transport: Transport.TCP,
        options: { host: "user-service", port: 3001 },
      },
      {
        name: "PRODUCT_SERVICE",
        transport: Transport.TCP,
        options: { host: "product-service", port: 3002 },
      },
      {
        name: "ORDER_SERVICE",
        transport: Transport.TCP,
        options: { host: "order-service", port: 3003 },
      },
    ]),
  ],
})
export class GatewayModule {}

// api-gateway/src/controllers/orders.controller.ts
import { firstValueFrom } from "rxjs";

export class OrdersController {
  constructor(@Inject("ORDER_SERVICE") private orderClient: ClientProxy) {}

  @Post()
  async createOrder(@Body() data: CreateOrderDto) {
    return firstValueFrom(this.orderClient.send("order.create", data));
  }

  @Get(":id")
  async getOrder(@Param("id") id: string) {
    return firstValueFrom(this.orderClient.send("order.get", { id }));
  }
}
```

### 3. Event-Driven Microservices

```typescript
// shared/events/order.events.ts
export class OrderCreatedEvent {
  constructor(
    public readonly orderId: string,
    public readonly userId: string,
    public readonly total: number,
    public readonly items: OrderItem[]
  ) {}
}

export class PaymentProcessedEvent {
  constructor(
    public readonly orderId: string,
    public readonly status: "success" | "failed",
    public readonly amount: number
  ) {}
}

// order-service/src/orders/orders.service.ts

export class OrdersService {
  constructor(
    private ordersRepository: OrdersRepository,
    private eventEmitter: EventEmitter2
  ) {}

  async createOrder(data: CreateOrderDto) {
    const order = await this.ordersRepository.create(data);

    // Emit event
    this.eventEmitter.emit(
      "order.created",
      new OrderCreatedEvent(order.id, order.userId, order.total, order.items)
    );

    return order;
  }
}

// payment-service/src/payments/payments.listener.ts
import { OrderCreatedEvent, PaymentProcessedEvent } from "@shared/events";

export class PaymentsListener {
  constructor(
    private paymentsService: PaymentsService,
    private eventEmitter: EventEmitter2
  ) {}

  @OnEvent("order.created")
  async handleOrderCreated(event: OrderCreatedEvent) {
    try {
      const payment = await this.paymentsService.processPayment({
        orderId: event.orderId,
        amount: event.total,
      });

      // Emit payment result
      this.eventEmitter.emit(
        "payment.processed",
        new PaymentProcessedEvent(event.orderId, payment.status, payment.amount)
      );
    } catch (error) {
      this.eventEmitter.emit(
        "payment.processed",
        new PaymentProcessedEvent(event.orderId, "failed", event.total)
      );
    }
  }
}

// notification-service/src/notifications/notifications.listener.ts

export class NotificationsListener {
  constructor(private notificationsService: NotificationsService) {}

  @OnEvent("payment.processed")
  async handlePaymentProcessed(event: PaymentProcessedEvent) {
    if (event.status === "success") {
      await this.notificationsService.sendEmail({
        type: "order_confirmed",
        orderId: event.orderId,
      });
    } else {
      await this.notificationsService.sendEmail({
        type: "payment_failed",
        orderId: event.orderId,
      });
    }
  }
}
```

## Saga Pattern for Distributed Transactions

```typescript
// Choreography-based saga
// order-service/src/sagas/order-saga.ts

export class OrderSaga {
  constructor(
    private ordersService: OrdersService,
    private eventEmitter: EventEmitter2
  ) {}

  @OnEvent("payment.processed")
  async handlePaymentProcessed(event: PaymentProcessedEvent) {
    if (event.status === "success") {
      await this.ordersService.confirmOrder(event.orderId);
      this.eventEmitter.emit("order.confirmed", { orderId: event.orderId });
    } else {
      await this.ordersService.cancelOrder(event.orderId);
      this.eventEmitter.emit("order.cancelled", { orderId: event.orderId });
    }
  }

  @OnEvent("inventory.reserved-failed")
  async handleInventoryFailure(event: { orderId: string }) {
    await this.ordersService.cancelOrder(event.orderId);
    // Compensating transaction: refund payment
    this.eventEmitter.emit("payment.refund", { orderId: event.orderId });
  }
}
```

## Service Mesh & Circuit Breaker

```typescript
// Circuit breaker for service-to-service calls
import * as CircuitBreaker from "opossum";

export class ResilientOrdersService {
  private userServiceBreaker: CircuitBreaker;
  private productServiceBreaker: CircuitBreaker;

  constructor(
    @Inject("USER_SERVICE") private userClient: ClientProxy,
    @Inject("PRODUCT_SERVICE") private productClient: ClientProxy
  ) {
    // Configure circuit breaker
    this.userServiceBreaker = new CircuitBreaker(
      (userId: string) => this.callUserService(userId),
      {
        timeout: 3000, // 3 seconds
        errorThresholdPercentage: 50,
        resetTimeout: 30000, // 30 seconds
      }
    );

    this.userServiceBreaker.fallback(() => {
      // Return cached data or default
      return { id: "unknown", name: "Guest" };
    });

    this.userServiceBreaker.on("open", () => {
      console.error("User service circuit breaker opened");
    });
  }

  private async callUserService(userId: string) {
    return firstValueFrom(this.userClient.send("user.get", { id: userId }));
  }

  async getUser(userId: string) {
    return this.userServiceBreaker.fire(userId);
  }
}
```

## Decision Framework

### When to Use Monolithic

✅ Small to medium applications
✅ Small team (< 10 developers)
✅ Rapid prototyping/MVP
✅ Limited operational expertise
✅ Simple deployment requirements
✅ Tight coupling acceptable
✅ Shared database preferred
✅ Strong consistency required

### When to Use Microservices

✅ Large, complex applications
✅ Multiple teams
✅ Independent scaling needs
✅ Technology diversity required
✅ Continuous deployment
✅ High availability critical
✅ Domain-driven design
✅ Eventual consistency acceptable

### Migration Strategy

```typescript
// Start monolithic, extract services gradually
// 1. Identify bounded contexts
// 2. Create API boundaries within monolith
// 3. Extract to separate service
// 4. Replace synchronous calls with async messaging
// 5. Migrate data incrementally

// Strangler Fig Pattern

export class OrdersService {
  constructor(
    private ordersRepository: OrdersRepository,
    @Inject("NEW_ORDER_SERVICE") private newOrderService?: ClientProxy
  ) {}

  async createOrder(data: CreateOrderDto) {
    // Feature flag for gradual migration
    if (process.env.USE_NEW_ORDER_SERVICE === "true") {
      return this.newOrderService.send("order.create", data);
    }

    // Legacy monolith code
    return this.ordersRepository.create(data);
  }
}
```

## Best Practices

### Monolithic

- Clear module boundaries
- Dependency injection
- Separate business logic
- Single database per domain
- Modular architecture

### Microservices

- One service, one responsibility
- API-first design
- Independent deployment
- Service per database
- Async communication
- Circuit breakers
- Centralized logging
- Distributed tracing

## Key Takeaways

✅ **Monoliths are simpler to develop and deploy initially**  
✅ **Microservices enable independent scaling and deployment**  
✅ **Modular monoliths provide middle ground**  
✅ **Microservices require significant operational overhead**  
✅ **Start monolithic, migrate to microservices when needed**  
✅ **Use API gateway for microservices**  
✅ **Implement circuit breakers for resilience**  
✅ **Event-driven architecture decouples services**  
✅ **Distributed transactions are complex**  
✅ **Choose architecture based on team size and requirements**

There's no one-size-fits-all answer—choose the architecture that matches your team's capabilities and application requirements.
