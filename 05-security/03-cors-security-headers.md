# CORS & Security Headers

## Overview

Cross-Origin Resource Sharing (CORS) and security headers are critical mechanisms for protecting web applications from various attacks. CORS controls which domains can access your API, while security headers provide additional protections against XSS, clickjacking, and other vulnerabilities. Properly configured, they form an essential layer of defense-in-depth security.

**Key Security Headers:**

- **CORS**: Controls cross-origin resource access
- **CSP**: Content Security Policy prevents XSS
- **HSTS**: Forces HTTPS connections
- **X-Frame-Options**: Prevents clickjacking
- **X-Content-Type-Options**: Prevents MIME-type sniffing

## Practical Use Cases

### 1. **API Security**

Control API access

- Restrict allowed origins
- Protect sensitive endpoints
- Prevent CSRF attacks
- Control request methods

### 2. **XSS Prevention**

Block malicious scripts

- Content Security Policy
- Script source restrictions
- Inline script blocking
- Trusted domains only

### 3. **Clickjacking Protection**

Prevent iframe embedding

- Frame options header
- Frame ancestors CSP
- Protect sensitive pages
- Login page protection

### 4. **Data Theft Prevention**

Protect sensitive data

- Referrer policy
- Cookie security
- HTTPS enforcement
- Resource isolation

### 5. **Browser Security**

Leverage browser protections

- XSS filters
- MIME-type validation
- Download restrictions
- Feature policies

## CORS Configuration

### 1. NestJS CORS Setup

```typescript
// src/app.ts
import express from "express";
import cors from "cors";

const app = express();

// Basic CORS configuration
app.use(
  cors({
    origin: process.env.ALLOWED_ORIGINS?.split(",") || [
      "https://example.com",
      "https://www.example.com",
    ],
    methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
    allowedHeaders: [
      "Content-Type",
      "Authorization",
      "X-Requested-With",
      "Accept",
    ],
    exposedHeaders: ["X-Total-Count", "X-Page-Number"],
    credentials: true,
    maxAge: 3600, // 1 hour preflight cache
  })
);

app.listen(3000, () => {
  console.log("Server running on port 3000");
});

// src/config/cors.config.ts
import { CorsOptions } from "cors";

export const corsConfig: CorsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(",") || [];

    // Allow requests with no origin (mobile apps, curl, etc.)
    if (!origin) {
      return callback(null, true);
    }

    // Check if origin is allowed
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error("Not allowed by CORS"));
    }
  },
  credentials: true,
  methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"],
  allowedHeaders: [
    "Content-Type",
    "Authorization",
    "X-Requested-With",
    "Accept",
    "X-CSRF-Token",
  ],
  exposedHeaders: ["X-Total-Count", "X-Page-Number", "X-Page-Size"],
  maxAge: 86400, // 24 hours
};
```

### 2. Dynamic CORS by Route

```typescript
// src/decorators/cors.decorator.ts

export const CORS_ORIGINS_KEY = "cors_origins";
export const CorsOrigins = (...origins: string[]) =>
  SetMetadata(CORS_ORIGINS_KEY, origins);

// src/guards/cors.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
import { CORS_ORIGINS_KEY } from "../decorators/cors.decorator";

export class CorsGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const allowedOrigins = this.reflector.get<string[]>(
      CORS_ORIGINS_KEY,
      context.getHandler()
    );

    if (!allowedOrigins) {
      return true; // No specific CORS restriction
    }

    const request = context.switchToHttp().getRequest();
    const origin = request.headers.origin;

    if (!origin || !allowedOrigins.includes(origin)) {
      throw new ForbiddenException("Origin not allowed");
    }

    return true;
  }
}

// src/controllers/user.controller.ts

export class UserController {
  @Get("public")
  @CorsOrigins("*") // Allow all origins
  getPublicData() {
    return { message: "Public data" };
  }

  @Post()
  @CorsOrigins("https://admin.example.com") // Specific origin only
  @UseGuards(CorsGuard)
  createUser() {
    return { message: "User created" };
  }
}
```

### 3. CORS Middleware

```typescript
// src/middleware/cors.middleware.ts
import { Request, Response, NextFunction } from "express";

export function corsMiddleware(
  req: Request,
  res: Response,
  next: NextFunction
) {
  const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(",") || [];
  const origin = req.headers.origin;

  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
    res.setHeader("Access-Control-Allow-Credentials", "true");
    res.setHeader(
      "Access-Control-Allow-Methods",
      "GET, POST, PUT, PATCH, DELETE, OPTIONS"
    );
    res.setHeader(
      "Access-Control-Allow-Headers",
      "Content-Type, Authorization, X-Requested-With"
    );
    res.setHeader(
      "Access-Control-Expose-Headers",
      "X-Total-Count, X-Page-Number"
    );
    res.setHeader("Access-Control-Max-Age", "86400");
  }

  // Handle preflight requests
  if (req.method === "OPTIONS") {
    return res.sendStatus(204);
  }

  next();
}
```

## Security Headers with Helmet

### 1. Helmet Configuration

```typescript
// src/app.ts
import express from "express";
import helmet from "helmet";

const app = express();

// Apply Helmet middleware
app.use(
  helmet({
    // Content Security Policy
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          scriptSrc: ["'self'", "'unsafe-inline'", "https://cdn.example.com"],
          styleSrc: [
            "'self'",
            "'unsafe-inline'",
            "https://fonts.googleapis.com",
          ],
          fontSrc: ["'self'", "https://fonts.gstatic.com"],
          imgSrc: ["'self'", "data:", "https:", "blob:"],
          connectSrc: ["'self'", "https://api.example.com"],
          frameSrc: ["'none'"],
          objectSrc: ["'none'"],
          upgradeInsecureRequests: [],
        },
      },

      // HTTP Strict Transport Security
      hsts: {
        maxAge: 31536000, // 1 year
        includeSubDomains: true,
        preload: true,
      },

      // X-Frame-Options
      frameguard: {
        action: "deny",
      },

      // X-Content-Type-Options
      noSniff: true,

      // X-XSS-Protection
      xssFilter: true,

      // Referrer-Policy
      referrerPolicy: {
        policy: "strict-origin-when-cross-origin",
      },

      // X-Permitted-Cross-Domain-Policies
      permittedCrossDomainPolicies: {
        permittedPolicies: "none",
      },

      // Hide X-Powered-By header
      hidePoweredBy: true,

      // X-DNS-Prefetch-Control
      dnsPrefetchControl: {
        allow: false,
      },
    })
  );

  await app.listen(3000);
}

bootstrap();
```

### 2. Custom Security Headers Middleware

```typescript
// src/middleware/security-headers.middleware.ts
import { Request, Response, NextFunction } from "express";

export class SecurityHeadersMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Content Security Policy
    res.setHeader(
      "Content-Security-Policy",
      [
        "default-src 'self'",
        "script-src 'self' 'unsafe-inline' https://cdn.example.com",
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
        "font-src 'self' https://fonts.gstatic.com",
        "img-src 'self' data: https: blob:",
        "connect-src 'self' https://api.example.com",
        "frame-src 'none'",
        "object-src 'none'",
        "base-uri 'self'",
        "form-action 'self'",
        "upgrade-insecure-requests",
      ].join("; ")
    );

    // HTTP Strict Transport Security
    if (process.env.NODE_ENV === "production") {
      res.setHeader(
        "Strict-Transport-Security",
        "max-age=31536000; includeSubDomains; preload"
      );
    }

    // X-Frame-Options
    res.setHeader("X-Frame-Options", "DENY");

    // X-Content-Type-Options
    res.setHeader("X-Content-Type-Options", "nosniff");

    // X-XSS-Protection
    res.setHeader("X-XSS-Protection", "1; mode=block");

    // Referrer-Policy
    res.setHeader("Referrer-Policy", "strict-origin-when-cross-origin");

    // Permissions-Policy (Feature-Policy)
    res.setHeader(
      "Permissions-Policy",
      [
        "camera=()",
        "microphone=()",
        "geolocation=()",
        "payment=()",
        "usb=()",
      ].join(", ")
    );

    // X-Permitted-Cross-Domain-Policies
    res.setHeader("X-Permitted-Cross-Domain-Policies", "none");

    // Remove X-Powered-By
    res.removeHeader("X-Powered-By");

    next();
  }
}

// Register middleware
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(SecurityHeadersMiddleware).forRoutes("*");
  }
}
```

## Content Security Policy (CSP)

### 1. Strict CSP Configuration

```typescript
// src/config/csp.config.ts
export const cspDirectives = {
  // Default fallback
  defaultSrc: ["'self'"],

  // Scripts
  scriptSrc: [
    "'self'",
    "'nonce-{NONCE}'", // Use nonce for inline scripts
    "https://cdn.example.com",
    "https://www.google-analytics.com",
  ],

  // Styles
  styleSrc: ["'self'", "'nonce-{NONCE}'", "https://fonts.googleapis.com"],

  // Fonts
  fontSrc: ["'self'", "https://fonts.gstatic.com"],

  // Images
  imgSrc: ["'self'", "data:", "https:", "blob:"],

  // AJAX/WebSocket
  connectSrc: ["'self'", "https://api.example.com", "wss://api.example.com"],

  // Media
  mediaSrc: ["'self'", "https://media.example.com"],

  // Frames
  frameSrc: ["'none'"],
  frameAncestors: ["'none'"],

  // Objects (Flash, etc.)
  objectSrc: ["'none'"],

  // Workers
  workerSrc: ["'self'", "blob:"],

  // Form submissions
  formAction: ["'self'"],

  // Base tag
  baseUri: ["'self'"],

  // Manifest
  manifestSrc: ["'self'"],

  // Upgrade insecure requests
  upgradeInsecureRequests: [],

  // Block all mixed content
  blockAllMixedContent: [],
};

// src/middleware/csp-nonce.middleware.ts
import { Request, Response, NextFunction } from "express";
import * as crypto from "crypto";

export class CspNonceMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Generate unique nonce for each request
    const nonce = crypto.randomBytes(16).toString("base64");
    res.locals.cspNonce = nonce;

    // Set CSP with nonce
    const csp = [
      "default-src 'self'",
      `script-src 'self' 'nonce-${nonce}'`,
      `style-src 'self' 'nonce-${nonce}'`,
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self'",
      "frame-src 'none'",
      "object-src 'none'",
    ].join("; ");

    res.setHeader("Content-Security-Policy", csp);

    next();
  }
}

// Usage in template
// <script nonce="<%= cspNonce %>">
//   // Inline script
// </script>
```

### 2. CSP Violation Reporting

```typescript
// src/controllers/csp-report.controller.ts

interface CspReport {
  "csp-report": {
    "document-uri": string;
    "violated-directive": string;
    "effective-directive": string;
    "original-policy": string;
    "blocked-uri": string;
    "status-code": number;
  };
}

export class CspReportController {
  private readonly logger = new Logger(CspReportController.name);

  @Post()
  async receiveReport(@Body() report: CspReport) {
    this.logger.warn("CSP Violation:", {
      documentUri: report["csp-report"]["document-uri"],
      violatedDirective: report["csp-report"]["violated-directive"],
      blockedUri: report["csp-report"]["blocked-uri"],
    });

    // Store in database or send to monitoring service

    return { status: "received" };
  }
}

// Add report-uri to CSP
const cspWithReporting = [
  "default-src 'self'",
  "script-src 'self'",
  "report-uri /csp-report",
  "report-to csp-endpoint",
].join("; ");

// Report-To header (newer standard)
res.setHeader(
  "Report-To",
  JSON.stringify({
    group: "csp-endpoint",
    max_age: 10886400,
    endpoints: [{ url: "https://example.com/csp-report" }],
  })
);
```

## Cookie Security

```typescript
// src/config/session.config.ts
import * as session from "express-session";
import * as RedisStore from "connect-redis";
import { createClient } from "redis";

const redisClient = createClient({
  url: process.env.REDIS_URL,
});

export const sessionConfig: session.SessionOptions = {
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  name: "sessionId", // Don't use default 'connect.sid'
  cookie: {
    httpOnly: true, // Prevent JavaScript access
    secure: process.env.NODE_ENV === "production", // HTTPS only
    sameSite: "strict", // CSRF protection
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    domain: process.env.COOKIE_DOMAIN,
    path: "/",
  },
};

// JWT Cookie
res.cookie("accessToken", token, {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "strict",
  maxAge: 15 * 60 * 1000, // 15 minutes
});
```

## CSRF Protection

```typescript
// src/middleware/csrf.middleware.ts
import { Request, Response, NextFunction } from "express";
import * as csrf from "csurf";

export class CsrfMiddleware implements NestMiddleware {
  private csrfProtection = csrf({
    cookie: {
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "strict",
    },
  });

  use(req: Request, res: Response, next: NextFunction) {
    this.csrfProtection(req, res, (err) => {
      if (err) {
        throw new ForbiddenException("Invalid CSRF token");
      }

      // Make token available to views
      res.locals.csrfToken = req.csrfToken();
      next();
    });
  }
}

// Frontend usage
// <form method="POST">
//   <input type="hidden" name="_csrf" value="<%= csrfToken %>">
// </form>

// AJAX requests
// headers: { 'X-CSRF-Token': csrfToken }
```

## Testing Security Headers

```typescript
// test/security-headers.e2e-spec.ts
import * as request from "supertest";
import { AppModule } from "../src/app.module";

describe("Security Headers (e2e)", () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it("should set HSTS header", () => {
    return request(app.getHttpServer())
      .get("/")
      .expect((res) => {
        expect(res.headers["strict-transport-security"]).toBeDefined();
      });
  });

  it("should set X-Frame-Options", () => {
    return request(app.getHttpServer())
      .get("/")
      .expect((res) => {
        expect(res.headers["x-frame-options"]).toBe("DENY");
      });
  });

  it("should set Content-Security-Policy", () => {
    return request(app.getHttpServer())
      .get("/")
      .expect((res) => {
        expect(res.headers["content-security-policy"]).toBeDefined();
      });
  });

  it("should not expose X-Powered-By", () => {
    return request(app.getHttpServer())
      .get("/")
      .expect((res) => {
        expect(res.headers["x-powered-by"]).toBeUndefined();
      });
  });

  it("should handle CORS preflight", () => {
    return request(app.getHttpServer())
      .options("/api/users")
      .set("Origin", "https://example.com")
      .expect(204)
      .expect((res) => {
        expect(res.headers["access-control-allow-origin"]).toBe(
          "https://example.com"
        );
      });
  });
});
```

## Best Practices

### 1. **CORS Configuration**

- Use specific origins, not wildcards
- Enable credentials carefully
- Set appropriate maxAge
- Validate origins dynamically
- Handle preflight requests

### 2. **Content Security Policy**

- Start with strict policy
- Use nonces for inline scripts
- Avoid 'unsafe-inline' and 'unsafe-eval'
- Test thoroughly before deployment
- Monitor violation reports

### 3. **Cookie Security**

- Always set HttpOnly
- Use Secure in production
- Set appropriate SameSite
- Use short expiration times
- Don't store sensitive data

### 4. **HSTS**

- Enable in production only
- Use long max-age
- Include subdomains
- Consider preload list

### 5. **Frame Protection**

- Use DENY for sensitive pages
- Use SAMEORIGIN when needed
- Implement frame-ancestors CSP
- Test iframe embedding

### 6. **Regular Audits**

- Test with security scanners
- Use Mozilla Observatory
- Check securityheaders.com
- Review CSP violations
- Update configurations

### 7. **Environment-Specific**

- Different settings for dev/prod
- Relaxed CSP in development
- HTTPS enforcement in production
- Appropriate CORS origins

### 8. **Documentation**

- Document allowed origins
- Explain CSP directives
- Note security exceptions
- Maintain security policies

## Key Takeaways

✅ **CORS restricts cross-origin access to APIs**  
✅ **Use specific origins instead of wildcards**  
✅ **Content Security Policy prevents XSS attacks**  
✅ **HSTS enforces HTTPS connections**  
✅ **Set HttpOnly and Secure flags on cookies**  
✅ **X-Frame-Options prevents clickjacking**  
✅ **Use helmet middleware for automatic header management**  
✅ **Implement CSP violation reporting**  
✅ **Test security headers with automated tools**  
✅ **Regular audits and updates are essential**

Properly configured CORS and security headers are fundamental to web application security and should be implemented from day one.
