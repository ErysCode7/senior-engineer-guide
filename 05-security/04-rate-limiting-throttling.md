# Rate Limiting & Throttling

## Overview

Rate limiting and throttling are essential techniques for protecting APIs from abuse, preventing denial-of-service attacks, and ensuring fair resource allocation. Rate limiting restricts the number of requests a client can make in a given time period, while throttling controls the rate at which requests are processed. Together, they protect infrastructure, maintain service quality, and prevent malicious behavior.

**Key Concepts:**

- **Rate Limiting**: Maximum requests allowed in a time window
- **Throttling**: Controlling request processing speed
- **Token Bucket**: Algorithm for smooth rate limiting
- **Sliding Window**: More accurate rate limiting
- **Distributed Rate Limiting**: Coordinated limiting across servers

## Practical Use Cases

### 1. **API Protection**

Prevent API abuse

- Public API endpoints
- Third-party integrations
- Webhook receivers
- Search endpoints

### 2. **DDoS Prevention**

Mitigate attacks

- High-volume attacks
- Distributed attacks
- Slow attacks
- Layer 7 attacks

### 3. **Resource Management**

Fair resource allocation

- Database connection limits
- CPU-intensive operations
- External API calls
- File processing

### 4. **Authentication Protection**

Prevent brute force attacks

- Login endpoints
- Password reset
- OTP generation
- Token refresh

### 5. **Cost Control**

Manage cloud costs

- Third-party API usage
- SMS/email sending
- Cloud function executions
- Database queries

## NestJS Rate Limiting

### 1. Basic Throttler Setup

```typescript
// app.module.ts
import { Module } from "@nestjs/common";
import { ThrottlerModule, ThrottlerGuard } from "@nestjs/throttler";
import { APP_GUARD } from "@nestjs/core";

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        ttl: 60000, // Time window in milliseconds (1 minute)
        limit: 10, // Maximum requests per ttl
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}

// controllers/user.controller.ts
import { Controller, Get, Post, UseGuards } from "@nestjs/common";
import { Throttle, SkipThrottle } from "@nestjs/throttler";

@Controller("users")
export class UserController {
  // Use default rate limit (10 requests per minute)
  @Get()
  findAll() {
    return { message: "List users" };
  }

  // Custom rate limit for specific endpoint
  @Post("login")
  @Throttle({ default: { limit: 5, ttl: 60000 } }) // 5 requests per minute
  login() {
    return { message: "Login" };
  }

  // Skip rate limiting
  @Get("public")
  @SkipThrottle()
  publicData() {
    return { message: "Public data" };
  }

  // Different limits for different user types
  @Get("profile")
  @Throttle({ default: { limit: 20, ttl: 60000 } }) // Premium users get more
  getProfile() {
    return { message: "Profile" };
  }
}
```

### 2. Redis-Based Distributed Rate Limiting

```typescript
// config/throttler.config.ts
import { ThrottlerModuleOptions } from "@nestjs/throttler";
import { ThrottlerStorageRedisService } from "nestjs-throttler-storage-redis";
import Redis from "ioredis";

export const throttlerConfig: ThrottlerModuleOptions = {
  throttlers: [
    {
      ttl: 60000, // 1 minute
      limit: 10,
    },
  ],
  storage: new ThrottlerStorageRedisService(
    new Redis({
      host: process.env.REDIS_HOST || "localhost",
      port: parseInt(process.env.REDIS_PORT || "6379"),
      password: process.env.REDIS_PASSWORD,
    })
  ),
};

// app.module.ts
import { Module } from "@nestjs/common";
import { ThrottlerModule } from "@nestjs/throttler";
import { throttlerConfig } from "./config/throttler.config";

@Module({
  imports: [ThrottlerModule.forRoot(throttlerConfig)],
})
export class AppModule {}
```

### 3. Custom Rate Limiting by User ID

```typescript
// guards/user-throttler.guard.ts
import { Injectable, ExecutionContext } from "@nestjs/common";
import { ThrottlerGuard, ThrottlerException } from "@nestjs/throttler";
import { ThrottlerRequest } from "@nestjs/throttler/dist/throttler.guard.interface";

@Injectable()
export class UserThrottlerGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    // Rate limit by user ID if authenticated, otherwise by IP
    const user = req.user;
    return user ? `user:${user.id}` : `ip:${req.ip}`;
  }

  protected async getLimit(context: ExecutionContext): Promise<number> {
    const request = context.switchToHttp().getRequest();
    const user = request.user;

    // Premium users get higher limits
    if (user?.isPremium) {
      return 100;
    }

    // Authenticated users get standard limits
    if (user) {
      return 50;
    }

    // Anonymous users get lower limits
    return 10;
  }
}

// Usage
@Controller("api")
@UseGuards(UserThrottlerGuard)
export class ApiController {
  @Get("data")
  getData() {
    return { message: "Data" };
  }
}
```

### 4. Endpoint-Specific Rate Limits

```typescript
// decorators/rate-limit.decorator.ts
import { SetMetadata } from "@nestjs/common";

export const RATE_LIMIT_KEY = "rate_limit";

export interface RateLimitOptions {
  points: number; // Number of requests
  duration: number; // Time window in seconds
  blockDuration?: number; // How long to block after exceeding limit
}

export const RateLimit = (options: RateLimitOptions) =>
  SetMetadata(RATE_LIMIT_KEY, options);

// guards/custom-rate-limit.guard.ts
import { Injectable, CanActivate, ExecutionContext } from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { RateLimiterRedis, RateLimiterRes } from "rate-limiter-flexible";
import Redis from "ioredis";
import {
  RATE_LIMIT_KEY,
  RateLimitOptions,
} from "../decorators/rate-limit.decorator";

@Injectable()
export class CustomRateLimitGuard implements CanActivate {
  private redis: Redis;
  private limiters: Map<string, RateLimiterRedis> = new Map();

  constructor(private reflector: Reflector) {
    this.redis = new Redis({
      host: process.env.REDIS_HOST,
      port: parseInt(process.env.REDIS_PORT || "6379"),
    });
  }

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const rateLimitOptions = this.reflector.get<RateLimitOptions>(
      RATE_LIMIT_KEY,
      context.getHandler()
    );

    if (!rateLimitOptions) {
      return true; // No rate limit specified
    }

    const request = context.switchToHttp().getRequest();
    const key = `${request.ip}:${request.route.path}`;

    // Get or create rate limiter for this configuration
    const limiterKey = `${rateLimitOptions.points}:${rateLimitOptions.duration}`;
    let limiter = this.limiters.get(limiterKey);

    if (!limiter) {
      limiter = new RateLimiterRedis({
        storeClient: this.redis,
        keyPrefix: "ratelimit",
        points: rateLimitOptions.points,
        duration: rateLimitOptions.duration,
        blockDuration: rateLimitOptions.blockDuration || 0,
      });
      this.limiters.set(limiterKey, limiter);
    }

    try {
      await limiter.consume(key);
      return true;
    } catch (error) {
      if (error instanceof RateLimiterRes) {
        const response = context.switchToHttp().getResponse();
        response.setHeader("Retry-After", Math.ceil(error.msBeforeNext / 1000));
        response.setHeader("X-RateLimit-Limit", rateLimitOptions.points);
        response.setHeader("X-RateLimit-Remaining", error.remainingPoints);
        response.setHeader(
          "X-RateLimit-Reset",
          new Date(Date.now() + error.msBeforeNext).toISOString()
        );

        throw new Error(
          `Rate limit exceeded. Retry after ${Math.ceil(
            error.msBeforeNext / 1000
          )} seconds`
        );
      }
      throw error;
    }
  }
}

// Usage
@Controller("api")
export class ApiController {
  @Post("expensive-operation")
  @RateLimit({ points: 5, duration: 3600 }) // 5 requests per hour
  @UseGuards(CustomRateLimitGuard)
  expensiveOperation() {
    return { message: "Processing..." };
  }

  @Post("login")
  @RateLimit({ points: 5, duration: 900, blockDuration: 3600 }) // 5 attempts per 15 min, block 1 hour
  @UseGuards(CustomRateLimitGuard)
  login() {
    return { message: "Login" };
  }
}
```

## Advanced Rate Limiting Patterns

### 1. Sliding Window Rate Limiter

```typescript
// services/sliding-window-rate-limiter.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import Redis from "ioredis";

@Injectable()
export class SlidingWindowRateLimiterService {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async checkLimit(
    key: string,
    limit: number,
    windowMs: number
  ): Promise<{ allowed: boolean; remaining: number; resetAt: Date }> {
    const now = Date.now();
    const windowStart = now - windowMs;
    const redisKey = `ratelimit:sliding:${key}`;

    // Use Redis sorted set for sliding window
    const multi = this.redis.multi();

    // Remove old entries outside the window
    multi.zremrangebyscore(redisKey, 0, windowStart);

    // Count requests in current window
    multi.zcard(redisKey);

    // Add current request
    multi.zadd(redisKey, now, `${now}:${Math.random()}`);

    // Set expiration
    multi.expire(redisKey, Math.ceil(windowMs / 1000));

    const results = await multi.exec();
    const count = results![1][1] as number;

    const allowed = count < limit;
    const remaining = Math.max(0, limit - count - 1);
    const resetAt = new Date(now + windowMs);

    return { allowed, remaining, resetAt };
  }
}
```

### 2. Token Bucket Rate Limiter

```typescript
// services/token-bucket-rate-limiter.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import Redis from "ioredis";

interface BucketState {
  tokens: number;
  lastRefill: number;
}

@Injectable()
export class TokenBucketRateLimiterService {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async consume(
    key: string,
    capacity: number,
    refillRate: number, // tokens per second
    cost: number = 1
  ): Promise<{ allowed: boolean; tokensRemaining: number }> {
    const redisKey = `ratelimit:bucket:${key}`;
    const now = Date.now();

    // Get current bucket state
    const stateStr = await this.redis.get(redisKey);
    let state: BucketState;

    if (stateStr) {
      state = JSON.parse(stateStr);

      // Refill tokens based on time elapsed
      const elapsedSeconds = (now - state.lastRefill) / 1000;
      const tokensToAdd = elapsedSeconds * refillRate;
      state.tokens = Math.min(capacity, state.tokens + tokensToAdd);
      state.lastRefill = now;
    } else {
      // Initialize bucket
      state = {
        tokens: capacity,
        lastRefill: now,
      };
    }

    // Try to consume tokens
    if (state.tokens >= cost) {
      state.tokens -= cost;
      await this.redis.setex(redisKey, 3600, JSON.stringify(state));
      return { allowed: true, tokensRemaining: state.tokens };
    } else {
      await this.redis.setex(redisKey, 3600, JSON.stringify(state));
      return { allowed: false, tokensRemaining: state.tokens };
    }
  }
}
```

### 3. Leaky Bucket Rate Limiter

```typescript
// services/leaky-bucket-rate-limiter.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import Redis from "ioredis";

@Injectable()
export class LeakyBucketRateLimiterService {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async enqueue(
    key: string,
    capacity: number,
    leakRate: number // requests per second
  ): Promise<{ queued: boolean; queueSize: number }> {
    const redisKey = `ratelimit:leaky:${key}`;
    const now = Date.now();

    // Lua script for atomic operation
    const script = `
      local key = KEYS[1]
      local capacity = tonumber(ARGV[1])
      local leakRate = tonumber(ARGV[2])
      local now = tonumber(ARGV[3])
      
      -- Get bucket state
      local state = redis.call('GET', key)
      local size, lastLeak
      
      if state then
        local decoded = cjson.decode(state)
        size = decoded.size
        lastLeak = decoded.lastLeak
      else
        size = 0
        lastLeak = now
      end
      
      -- Leak requests based on time elapsed
      local elapsedSeconds = (now - lastLeak) / 1000
      local leaked = math.floor(elapsedSeconds * leakRate)
      size = math.max(0, size - leaked)
      
      -- Try to add request
      if size < capacity then
        size = size + 1
        local newState = cjson.encode({size = size, lastLeak = now})
        redis.call('SETEX', key, 3600, newState)
        return {1, size}
      else
        return {0, size}
      end
    `;

    const result = (await this.redis.eval(
      script,
      1,
      redisKey,
      capacity.toString(),
      leakRate.toString(),
      now.toString()
    )) as [number, number];

    return {
      queued: result[0] === 1,
      queueSize: result[1],
    };
  }
}
```

## Rate Limit Headers

```typescript
// interceptors/rate-limit-headers.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";

@Injectable()
export class RateLimitHeadersInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const response = context.switchToHttp().getResponse();

    // Get rate limit info from request (set by guard)
    const rateLimit = request.rateLimit;

    if (rateLimit) {
      response.setHeader("X-RateLimit-Limit", rateLimit.limit);
      response.setHeader("X-RateLimit-Remaining", rateLimit.remaining);
      response.setHeader("X-RateLimit-Reset", rateLimit.reset);
    }

    return next.handle();
  }
}
```

## Progressive Rate Limiting

```typescript
// services/progressive-rate-limiter.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import Redis from "ioredis";

interface ProgressiveLimit {
  duration: number; // seconds
  limit: number;
}

@Injectable()
export class ProgressiveRateLimiterService {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async checkLimits(
    key: string,
    limits: ProgressiveLimit[]
  ): Promise<{ allowed: boolean; limitType: string }> {
    const now = Date.now();

    // Check each tier
    for (const limit of limits) {
      const tierKey = `ratelimit:progressive:${key}:${limit.duration}`;
      const windowStart = now - limit.duration * 1000;

      // Count requests in this window
      const count = await this.redis.zcount(tierKey, windowStart, now);

      if (count >= limit.limit) {
        return {
          allowed: false,
          limitType: `${limit.limit} requests per ${limit.duration} seconds`,
        };
      }
    }

    // Add request to all tiers
    for (const limit of limits) {
      const tierKey = `ratelimit:progressive:${key}:${limit.duration}`;
      const multi = this.redis.multi();
      multi.zadd(tierKey, now, `${now}:${Math.random()}`);
      multi.expire(tierKey, limit.duration);
      await multi.exec();
    }

    return { allowed: true, limitType: "none" };
  }
}

// Usage - progressive limits: 10/second, 100/minute, 1000/hour
const limits: ProgressiveLimit[] = [
  { duration: 1, limit: 10 },
  { duration: 60, limit: 100 },
  { duration: 3600, limit: 1000 },
];

const result = await this.progressiveRateLimiter.checkLimits(userId, limits);
```

## Dynamic Rate Limiting

```typescript
// services/dynamic-rate-limiter.service.ts
import { Injectable } from "@nestjs/common";

@Injectable()
export class DynamicRateLimiterService {
  private baseLimit = 100;
  private currentLoad = 0;

  getLimit(userId: string): number {
    // Reduce limits during high load
    if (this.currentLoad > 80) {
      return Math.floor(this.baseLimit * 0.5);
    } else if (this.currentLoad > 60) {
      return Math.floor(this.baseLimit * 0.75);
    }

    return this.baseLimit;
  }

  updateLoad(cpuUsage: number, memoryUsage: number) {
    this.currentLoad = Math.max(cpuUsage, memoryUsage);
  }
}
```

## Cost-Based Rate Limiting

```typescript
// Different operations have different costs
const OPERATION_COSTS = {
  "GET /api/users": 1,
  "POST /api/users": 5,
  "GET /api/search": 3,
  "POST /api/export": 10,
};

@Injectable()
export class CostBasedRateLimiterGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const operation = `${request.method} ${request.route.path}`;
    const cost = OPERATION_COSTS[operation] || 1;

    const userId = request.user?.id || request.ip;

    // Check if user has enough credits
    const result = await this.tokenBucketService.consume(
      userId,
      100, // capacity
      1, // 1 token per second refill
      cost // consume based on operation cost
    );

    return result.allowed;
  }
}
```

## Best Practices

### 1. **Choose Appropriate Algorithm**

- Fixed window: Simple, less accurate
- Sliding window: More accurate, higher overhead
- Token bucket: Smooth traffic, allows bursts
- Leaky bucket: Constant rate, no bursts

### 2. **Set Realistic Limits**

- Monitor actual usage patterns
- Different limits for different endpoints
- Higher limits for authenticated users
- Premium tier with elevated limits

### 3. **Informative Error Responses**

- Include Retry-After header
- Provide clear error messages
- Show remaining quota
- Indicate reset time

### 4. **Distributed Systems**

- Use Redis for coordination
- Consider eventual consistency
- Handle Redis failures gracefully
- Monitor rate limiter performance

### 5. **User Experience**

- Warn before blocking
- Show usage statistics
- Provide upgrade path
- Clear documentation

### 6. **Security Considerations**

- Rate limit authentication endpoints
- Block after repeated violations
- Log suspicious patterns
- Implement progressive delays

### 7. **Monitoring & Alerting**

- Track rate limit hits
- Monitor false positives
- Alert on unusual patterns
- Regular limit review

### 8. **Testing**

- Load test with limits
- Test edge cases
- Verify Redis failover
- Test concurrent requests

## Key Takeaways

✅ **Rate limiting protects APIs from abuse and DDoS attacks**  
✅ **Use Redis for distributed rate limiting**  
✅ **Different endpoints need different limits**  
✅ **Token bucket allows controlled bursts**  
✅ **Sliding window provides accurate limiting**  
✅ **Include rate limit headers in responses**  
✅ **Block aggressive violators progressively**  
✅ **Monitor and adjust limits based on usage**  
✅ **Cost-based limiting for expensive operations**  
✅ **Clear documentation helps users avoid limits**

Rate limiting is essential for API stability, security, and fair resource allocation in production systems.
