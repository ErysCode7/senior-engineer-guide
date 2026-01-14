# Monitoring & Logging

## Overview

Monitoring and logging are essential practices for maintaining application health, performance, and reliability. Monitoring tracks system metrics and application behavior in real-time, while logging captures detailed records of events for debugging, auditing, and analysis.

**Key Components:**

- **Metrics**: Quantitative measurements (CPU, memory, request rates, latency)
- **Logs**: Detailed event records with context
- **Traces**: Request flow through distributed systems
- **Alerts**: Automated notifications for anomalies
- **Dashboards**: Visual representation of system health

## Practical Use Cases

### 1. **Performance Monitoring**

Track application performance

- Response time tracking
- Resource utilization
- Bottleneck identification
- Capacity planning

### 2. **Error Tracking**

Identify and debug issues

- Exception monitoring
- Error rate tracking
- Stack trace analysis
- User impact assessment

### 3. **Business Metrics**

Monitor business KPIs

- User engagement
- Conversion rates
- Revenue tracking
- Feature adoption

### 4. **Security Monitoring**

Detect security threats

- Failed login attempts
- Unusual access patterns
- API abuse
- Data breach detection

### 5. **Compliance & Auditing**

Maintain audit trails

- User activity logging
- Data access logs
- Regulatory compliance
- Forensic analysis

## Application Logging

### 1. Structured Logging with Winston

```typescript
// src/config/logger.ts
import winston from "winston";

const { combine, timestamp, json, printf, colorize } = winston.format;

// Custom format for console
const consoleFormat = printf(({ level, message, timestamp, ...metadata }) => {
  let msg = `${timestamp} [${level}]: ${message}`;

  if (Object.keys(metadata).length > 0) {
    msg += ` ${JSON.stringify(metadata)}`;
  }

  return msg;
});

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || "info",
  format: combine(timestamp({ format: "YYYY-MM-DD HH:mm:ss" }), json()),
  defaultMeta: {
    service: "myapp",
    environment: process.env.NODE_ENV,
  },
  transports: [
    // Console output
    new winston.transports.Console({
      format: combine(
        colorize(),
        timestamp({ format: "YYYY-MM-DD HH:mm:ss" }),
        consoleFormat
      ),
    }),

    // File output
    new winston.transports.File({
      filename: "logs/error.log",
      level: "error",
      maxsize: 5242880, // 5MB
      maxFiles: 5,
    }),
    new winston.transports.File({
      filename: "logs/combined.log",
      maxsize: 5242880,
      maxFiles: 5,
    }),
  ],

  // Handle uncaught exceptions
  exceptionHandlers: [
    new winston.transports.File({ filename: "logs/exceptions.log" }),
  ],

  // Handle unhandled promise rejections
  rejectionHandlers: [
    new winston.transports.File({ filename: "logs/rejections.log" }),
  ],
});

// src/middleware/request-logger.middleware.ts
import { Request, Response, NextFunction } from "express";
import { logger } from "../config/logger";

export function requestLoggerMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const start = Date.now();

  // Log request
  logger.info("Incoming request", {
    method: req.method,
    url: req.url,
    ip: req.ip,
    userAgent: req.get("user-agent"),
  });

  // Log response
  res.on("finish", () => {
    const duration = Date.now() - start;

    const logData = {
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      duration,
      contentLength: res.get("content-length"),
    };

    if (res.statusCode >= 400) {
      logger.error("Request failed", logData);
    } else {
      logger.info("Request completed", logData);
    }
  });

  next();
}
```

### 2. Context-Aware Logging

```typescript
// src/utils/logger.util.ts
import { logger as baseLogger } from "../config/logger";

export class ContextLogger {
  constructor(
    private context: string,
    private metadata: Record<string, any> = {}
  ) {}

  private log(level: string, message: string, meta: Record<string, any> = {}) {
    baseLogger.log(level, message, {
      context: this.context,
      ...this.metadata,
      ...meta,
    });
  }

  info(message: string, meta?: Record<string, any>) {
    this.log("info", message, meta);
  }

  error(message: string, error?: Error, meta?: Record<string, any>) {
    this.log("error", message, {
      ...meta,
      error: error
        ? {
            name: error.name,
            message: error.message,
            stack: error.stack,
          }
        : undefined,
    });
  }

  warn(message: string, meta?: Record<string, any>) {
    this.log("warn", message, meta);
  }

  debug(message: string, meta?: Record<string, any>) {
    this.log("debug", message, meta);
  }

  child(additionalMetadata: Record<string, any>) {
    return new ContextLogger(this.context, {
      ...this.metadata,
      ...additionalMetadata,
    });
  }
}

// Usage in service
export class UserService {
  private logger = new ContextLogger("UserService");

  async createUser(dto: CreateUserDto) {
    const requestLogger = this.logger.child({ userId: dto.email });

    requestLogger.info("Creating new user");

    try {
      const user = await this.userRepository.save(dto);
      requestLogger.info("User created successfully", { userId: user.id });
      return user;
    } catch (error) {
      requestLogger.error("Failed to create user", error);
      throw error;
    }
  }
}
```

## Application Monitoring

### 1. Prometheus Metrics

```typescript
// src/config/metrics.ts
import { register, Counter, Histogram, Gauge } from "prom-client";

// Enable default metrics (CPU, memory, etc.)
register.setDefaultLabels({
  app: "myapp",
  environment: process.env.NODE_ENV,
});

// HTTP request metrics
export const httpRequestDuration = new Histogram({
  name: "http_request_duration_seconds",
  help: "Duration of HTTP requests in seconds",
  labelNames: ["method", "route", "status_code"],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
});

export const httpRequestTotal = new Counter({
  name: "http_requests_total",
  help: "Total number of HTTP requests",
  labelNames: ["method", "route", "status_code"],
});

export const httpRequestErrors = new Counter({
  name: "http_request_errors_total",
  help: "Total number of HTTP request errors",
  labelNames: ["method", "route", "error_type"],
});

// Business metrics
export const userSignups = new Counter({
  name: "user_signups_total",
  help: "Total number of user signups",
  labelNames: ["source"],
});

export const activeUsers = new Gauge({
  name: "active_users",
  help: "Number of currently active users",
});

export const paymentTransactions = new Counter({
  name: "payment_transactions_total",
  help: "Total number of payment transactions",
  labelNames: ["status", "payment_method"],
});

// Database metrics
export const dbQueryDuration = new Histogram({
  name: "db_query_duration_seconds",
  help: "Duration of database queries in seconds",
  labelNames: ["operation", "table"],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 3, 5],
});

export const dbConnectionsActive = new Gauge({
  name: "db_connections_active",
  help: "Number of active database connections",
});

// src/middleware/metrics.middleware.ts
import { Request, Response, NextFunction } from "express";
import {
  httpRequestDuration,
  httpRequestTotal,
  httpRequestErrors,
} from "../config/metrics";

export function metricsMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const start = Date.now();

  res.on("finish", () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || req.path;

    const labels = {
      method: req.method,
      route,
      status_code: res.statusCode.toString(),
    };

    httpRequestDuration.observe(labels, duration);
    httpRequestTotal.inc(labels);

    if (res.statusCode >= 400) {
      httpRequestErrors.inc({
        method: req.method,
        route,
        error_type: res.statusCode >= 500 ? "server_error" : "client_error",
      });
    }
  });

  next();
}

// src/routes/metrics.routes.ts
import { Router, Request, Response } from "express";
import { register } from "prom-client";

const router = Router();

router.get("/metrics", async (req: Request, res: Response) => {
  res.setHeader("Content-Type", register.contentType);
  res.send(await register.metrics());
});

export default router;
```

### 2. Health Checks

```typescript
// src/routes/health.routes.ts
import { Router, Request, Response } from "express";
import { AppDataSource } from "../database/data-source";
import { redisClient } from "../config/redis";

const router = Router();

router.get("/health", async (req: Request, res: Response) => {
  const checks = {
    status: "ok",
    timestamp: new Date().toISOString(),
    checks: {
      database: { status: "unknown" },
      redis: { status: "unknown" },
      disk: { status: "unknown" },
    private memory: MemoryHealthIndicator
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck("database"),
      () => this.redis.isHealthy("redis"),
      () =>
        this.disk.checkStorage("storage", { path: "/", thresholdPercent: 0.9 }),
      () => this.memory.checkHeap("memory_heap", 300 * 1024 * 1024),
      () => this.memory.checkRSS("memory_rss", 300 * 1024 * 1024),
    ]);
  }

  @Get("ready")
  @HealthCheck()
  ready() {
    return this.health.check([
      () => this.db.pingCheck("database"),
      () => this.redis.isHealthy("redis"),
    ]);
  }

  @Get("live")
  live() {
    return { status: "ok" };
  }
}

// src/health/redis-health.indicator.ts
import {
  HealthIndicator,
  HealthIndicatorResult,
  HealthCheckError,
import Redis from "ioredis";
import { Redis } from "ioredis";

export class RedisHealthIndicator extends HealthIndicator {
  constructor(@InjectRedis() private readonly redis: Redis) {
    super();
  }

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    try {
      await this.redis.ping();
      return this.getStatus(key, true);
    } catch (error) {
      throw new HealthCheckError(
        "Redis check failed",
        this.getStatus(key, false)
      );
    }
  }
}
```

## AWS CloudWatch Integration

```typescript
// src/config/cloudwatch.ts
import {
  CloudWatchClient,
  PutMetricDataCommand,
} from "@aws-sdk/client-cloudwatch";

const cloudwatch = new CloudWatchClient({ region: process.env.AWS_REGION });

export async function publishMetric(
  metricName: string,
  value: number,
  unit: string = "Count",
  dimensions: { Name: string; Value: string }[] = []
) {
  const params = {
    Namespace: "MyApp",
    MetricData: [
      {
        MetricName: metricName,
        Value: value,
        Unit: unit,
        Timestamp: new Date(),
        Dimensions: dimensions,
      },
    ],
  };

  try {
    await cloudwatch.send(new PutMetricDataCommand(params));
  } catch (error) {
    console.error("Failed to publish metric to CloudWatch:", error);
  }
}

// Usage
publishMetric("UserSignup", 1, "Count", [
  { Name: "Environment", Value: process.env.NODE_ENV || "development" },
  { Name: "Source", Value: "web" },
]);
```

## Error Tracking with Sentry

```typescript
// src/config/sentry.ts
import * as Sentry from "@sentry/node";
import { ProfilingIntegration } from "@sentry/profiling-node";

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === "production" ? 0.1 : 1.0,
  profilesSampleRate: 0.1,
  integrations: [new ProfilingIntegration()],
  beforeSend(event, hint) {
    // Filter out sensitive data
    if (event.request) {
      delete event.request.cookies;
      delete event.request.headers?.authorization;
    }

    return event;
  },
});

// src/filters/sentry-exception.filter.ts
import * as Sentry from "@sentry/node";

@Catch()
export class SentryExceptionFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest();

    // Capture exception in Sentry
    Sentry.withScope((scope) => {
      scope.setUser({
        id: request.user?.id,
        email: request.user?.email,
      });

      scope.setContext("request", {
        method: request.method,
        url: request.url,
        headers: request.headers,
        body: request.body,
      });

      if (exception instanceof HttpException) {
        scope.setLevel(exception.getStatus() >= 500 ? "error" : "warning");
      }

      Sentry.captureException(exception);
    });

    super.catch(exception, host);
  }
}
```

## Distributed Tracing

```typescript
// src/config/tracing.ts
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-http";
import { Resource } from "@opentelemetry/resources";
import { SemanticResourceAttributes } from "@opentelemetry/semantic-conventions";

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: "myapp",
    [SemanticResourceAttributes.SERVICE_VERSION]:
      process.env.APP_VERSION || "1.0.0",
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV,
  }),
  traceExporter: new OTLPTraceExporter({
    url:
      process.env.OTEL_EXPORTER_OTLP_ENDPOINT ||
      "http://localhost:4318/v1/traces",
  }),
  instrumentations: [
    getNodeAutoInstrumentations({
      "@opentelemetry/instrumentation-fs": {
        enabled: false,
      },
    }),
  ],
});

sdk.start();

// Graceful shutdown
process.on("SIGTERM", () => {
  sdk
    .shutdown()
    .then(() => console.log("Tracing terminated"))
    .catch((error) => console.log("Error terminating tracing", error))
    .finally(() => process.exit(0));
});
```

## Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Application Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ]
      },
      {
        "title": "Response Time (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "{{route}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "rate(http_request_errors_total[5m])",
            "legendFormat": "{{error_type}}"
          }
        ]
      },
      {
        "title": "Active Users",
        "targets": [
          {
            "expr": "active_users"
          }
        ]
      },
      {
        "title": "Database Query Duration",
        "targets": [
          {
            "expr": "rate(db_query_duration_seconds_sum[5m]) / rate(db_query_duration_seconds_count[5m])",
            "legendFormat": "{{operation}}"
          }
        ]
      }
    ]
  }
}
```

## Alerting Rules

```yaml
# prometheus-alerts.yaml
groups:
  - name: application
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_request_errors_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors/sec"

      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s"

      - alert: DatabaseConnectionIssue
        expr: db_connections_active == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No active database connections"
          description: "Database may be down or unreachable"

      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes > 500000000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanize }}B"
```

## Best Practices

### 1. **Log Levels**

- `ERROR`: Something failed
- `WARN`: Something unexpected but handled
- `INFO`: Normal operation events
- `DEBUG`: Detailed diagnostic information

### 2. **Structured Logging**

- Use JSON format
- Include context (user ID, request ID, etc.)
- Consistent field names
- Include timestamps

### 3. **Sensitive Data**

- Never log passwords
- Mask credit card numbers
- Sanitize PII
- Filter authentication tokens

### 4. **Metrics Best Practices**

- Use appropriate metric types (Counter, Gauge, Histogram)
- Keep cardinality low
- Use consistent naming
- Include relevant labels

### 5. **Alert Fatigue**

- Set appropriate thresholds
- Avoid duplicate alerts
- Group related alerts
- Include actionable information

### 6. **Performance Impact**

- Async logging when possible
- Sample high-volume metrics
- Buffer log writes
- Monitor monitoring overhead

### 7. **Retention Policies**

- Define retention periods
- Archive old logs
- Compress stored logs
- Automate cleanup

### 8. **Dashboard Design**

- Focus on actionable metrics
- Use appropriate visualizations
- Group related metrics
- Include SLA indicators

## Key Takeaways

✅ **Structured logging provides better searchability and analysis**  
✅ **Monitor both technical and business metrics**  
✅ **Implement health checks for all critical dependencies**  
✅ **Use distributed tracing for microservices**  
✅ **Set up alerts for critical issues only**  
✅ **Never log sensitive data like passwords or tokens**  
✅ **Create actionable dashboards for different audiences**  
✅ **Implement proper log retention and rotation**  
✅ **Use error tracking services for better debugging**  
✅ **Regular review and refinement of monitoring strategy**

Effective monitoring and logging are essential for maintaining reliable, performant, and secure applications in production environments.
