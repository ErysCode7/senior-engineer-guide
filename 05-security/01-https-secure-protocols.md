# HTTPS & Secure Protocols

## Overview

HTTPS (HTTP over TLS/SSL) encrypts data transmitted between clients and servers, protecting against eavesdropping, tampering, and man-in-the-middle attacks. Secure protocols are fundamental to modern web security, ensuring data confidentiality, integrity, and authenticity.

**Key Concepts:**

- **TLS/SSL**: Cryptographic protocols for secure communication
- **Certificates**: Digital documents that verify server identity
- **Encryption**: Scrambling data to prevent unauthorized access
- **Certificate Authorities (CAs)**: Trusted entities that issue certificates
- **HSTS**: HTTP Strict Transport Security for enforcing HTTPS

## Practical Use Cases

### 1. **E-Commerce & Payments**

Protect financial transactions

- Payment processing
- Shopping cart security
- Customer data protection
- PCI-DSS compliance

### 2. **User Authentication**

Secure login systems

- Password transmission
- Session cookies
- OAuth flows
- API token exchange

### 3. **API Security**

Protect API communications

- RESTful APIs
- GraphQL endpoints
- Webhook delivery
- Microservices communication

### 4. **Compliance Requirements**

Meet regulatory standards

- GDPR requirements
- HIPAA for healthcare
- SOC 2 compliance
- Data protection laws

### 5. **SEO & Trust**

Improve rankings and credibility

- Search engine rankings
- Browser trust indicators
- User confidence
- Professional appearance

## SSL/TLS Certificates

### 1. Obtain Free Certificate with Let's Encrypt

```bash
# Install Certbot
sudo apt-get update
sudo apt-get install certbot python3-certbot-nginx

# Obtain certificate for Nginx
sudo certbot --nginx -d example.com -d www.example.com

# Automatic renewal (cron)
sudo certbot renew --dry-run

# Add to crontab for automatic renewal
0 0 * * * certbot renew --quiet --post-hook "systemctl reload nginx"
```

### 2. Manual Certificate Generation

```bash
# Generate private key
openssl genrsa -out private.key 2048

# Generate CSR (Certificate Signing Request)
openssl req -new -key private.key -out request.csr

# Self-signed certificate (for development)
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes

# View certificate details
openssl x509 -in cert.pem -text -noout
```

### 3. Nginx HTTPS Configuration

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    # SSL Certificate
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';

    # SSL Session Cache
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    ssl_session_tickets off;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;

    # Security Headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Application
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### 4. Node.js HTTPS Server

```typescript
// src/server.ts
import https from "https";
import http from "http";
import fs from "fs";
import express from "express";

const app = express();

// Middleware
app.use(express.json());

// Routes
app.get("/health", (req, res) => {
  res.json({ status: "ok" });
});

// Development: HTTP
if (process.env.NODE_ENV === "development") {
  const httpServer = http.createServer(app);
  httpServer.listen(3000, () => {
    console.log("HTTP Server running on http://localhost:3000");
  });
}

// Production: HTTPS
if (process.env.NODE_ENV === "production") {
  const httpsOptions = {
    key: fs.readFileSync(
      process.env.SSL_KEY_PATH || "/etc/ssl/private/key.pem"
    ),
    cert: fs.readFileSync(
      process.env.SSL_CERT_PATH || "/etc/ssl/certs/cert.pem"
    ),
    ca: fs.readFileSync(process.env.SSL_CA_PATH || "/etc/ssl/certs/ca.pem"), // Optional
  };

  const httpsServer = https.createServer(httpsOptions, app);
  httpsServer.listen(443, () => {
    console.log("HTTPS Server running on https://localhost:443");
  });

  // Redirect HTTP to HTTPS
  const httpServer = http.createServer((req, res) => {
    res.writeHead(301, { Location: `https://${req.headers.host}${req.url}` });
    res.end();
  });
  httpServer.listen(80);
}

export default app;
```

## NestJS HTTPS Configuration

```typescript
// src/main.ts
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe } from "@nestjs/common";
import * as fs from "fs";
import * as helmet from "helmet";

async function bootstrap() {
  const httpsOptions =
    process.env.NODE_ENV === "production"
      ? {
          key: fs.readFileSync(process.env.SSL_KEY_PATH!),
          cert: fs.readFileSync(process.env.SSL_CERT_PATH!),
        }
      : undefined;

  const app = await NestFactory.create(AppModule, {
    httpsOptions,
  });

  // Security middleware
  app.use(
    helmet({
      hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true,
      },
      contentSecurityPolicy: {
        directives: {
          defaultSrc: ["'self'"],
          styleSrc: ["'self'", "'unsafe-inline'"],
          scriptSrc: ["'self'"],
          imgSrc: ["'self'", "data:", "https:"],
        },
      },
    })
  );

  // Global validation pipe
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    })
  );

  // Enable CORS with secure settings
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(",") || ["https://example.com"],
    credentials: true,
    methods: ["GET", "POST", "PUT", "DELETE", "PATCH"],
    allowedHeaders: ["Content-Type", "Authorization"],
  });

  await app.listen(process.env.PORT || 3000);

  console.log(`Application is running on: ${await app.getUrl()}`);
}

bootstrap();
```

## HTTP Strict Transport Security (HSTS)

```typescript
// src/middleware/hsts.middleware.ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class HSTSMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Only set HSTS in production
    if (process.env.NODE_ENV === "production") {
      res.setHeader(
        "Strict-Transport-Security",
        "max-age=31536000; includeSubDomains; preload"
      );
    }

    next();
  }
}

// src/middleware/force-https.middleware.ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class ForceHTTPSMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Skip in development
    if (process.env.NODE_ENV !== "production") {
      return next();
    }

    // Check if request is already HTTPS
    const isSecure = req.secure || req.headers["x-forwarded-proto"] === "https";

    if (!isSecure) {
      return res.redirect(301, `https://${req.headers.host}${req.url}`);
    }

    next();
  }
}
```

## Certificate Pinning (Mobile Apps)

```typescript
// certificate-pinning.ts
import axios, { AxiosInstance } from "axios";
import https from "https";
import * as tls from "tls";
import * as crypto from "crypto";

// Expected certificate fingerprints (SHA-256)
const TRUSTED_FINGERPRINTS = [
  "AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99",
  // Backup certificate
  "11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00:11:22:33:44:55:66:77:88:99:AA:BB:CC:DD:EE:FF:00",
];

function getCertificateFingerprint(cert: tls.PeerCertificate): string {
  const der = cert.raw;
  const hash = crypto.createHash("sha256");
  hash.update(der);
  return hash.digest("hex").toUpperCase().match(/.{2}/g)!.join(":");
}

function createPinnedHttpsAgent(): https.Agent {
  return new https.Agent({
    checkServerIdentity: (hostname, cert) => {
      const fingerprint = getCertificateFingerprint(cert);

      if (!TRUSTED_FINGERPRINTS.includes(fingerprint)) {
        throw new Error(
          `Certificate fingerprint mismatch. Expected: ${TRUSTED_FINGERPRINTS[0]}, Got: ${fingerprint}`
        );
      }

      // Perform standard hostname verification
      return tls.checkServerIdentity(hostname, cert);
    },
  });
}

// Create axios instance with certificate pinning
export const pinnedAxios: AxiosInstance = axios.create({
  httpsAgent: createPinnedHttpsAgent(),
  timeout: 10000,
});

// Usage
async function makeSecureRequest() {
  try {
    const response = await pinnedAxios.get("https://api.example.com/data");
    return response.data;
  } catch (error) {
    console.error("Secure request failed:", error);
    throw error;
  }
}
```

## Mutual TLS (mTLS)

```typescript
// mtls-server.ts
import express from "express";
import https from "https";
import fs from "fs";

const app = express();

// Middleware to verify client certificate
app.use((req, res, next) => {
  const cert = (req as any).socket.getPeerCertificate();

  if (!cert || !Object.keys(cert).length) {
    return res.status(401).json({ error: "Client certificate required" });
  }

  // Verify certificate subject
  if (cert.subject.CN !== "authorized-client") {
    return res.status(403).json({ error: "Invalid client certificate" });
  }

  // Add client info to request
  (req as any).clientCert = cert;
  next();
});

app.get("/secure-data", (req, res) => {
  res.json({
    message: "Secure data",
    client: (req as any).clientCert.subject,
  });
});

const httpsOptions = {
  key: fs.readFileSync("server-key.pem"),
  cert: fs.readFileSync("server-cert.pem"),
  ca: fs.readFileSync("ca-cert.pem"),
  requestCert: true,
  rejectUnauthorized: true,
};

https.createServer(httpsOptions, app).listen(443, () => {
  console.log("mTLS server running on https://localhost:443");
});

// mtls-client.ts
import axios from "axios";
import https from "https";
import fs from "fs";

const httpsAgent = new https.Agent({
  cert: fs.readFileSync("client-cert.pem"),
  key: fs.readFileSync("client-key.pem"),
  ca: fs.readFileSync("ca-cert.pem"),
});

async function makeSecureRequest() {
  try {
    const response = await axios.get("https://localhost:443/secure-data", {
      httpsAgent,
    });
    console.log(response.data);
  } catch (error) {
    console.error("Request failed:", error);
  }
}

makeSecureRequest();
```

## WebSocket over TLS (WSS)

```typescript
// wss-server.ts
import { WebSocketServer } from "ws";
import https from "https";
import fs from "fs";
import express from "express";

const app = express();

const httpsServer = https.createServer(
  {
    cert: fs.readFileSync("cert.pem"),
    key: fs.readFileSync("key.pem"),
  },
  app
);

const wss = new WebSocketServer({ server: httpsServer });

wss.on("connection", (ws, req) => {
  console.log("Client connected via WSS");

  ws.on("message", (message) => {
    console.log("Received:", message.toString());
    ws.send(`Echo: ${message}`);
  });

  ws.on("close", () => {
    console.log("Client disconnected");
  });
});

httpsServer.listen(443, () => {
  console.log("WSS server running on wss://localhost:443");
});

// wss-client.ts
import WebSocket from "ws";

const ws = new WebSocket("wss://localhost:443", {
  rejectUnauthorized: true, // Verify server certificate
});

ws.on("open", () => {
  console.log("Connected to WSS server");
  ws.send("Hello, secure world!");
});

ws.on("message", (data) => {
  console.log("Received:", data.toString());
});

ws.on("error", (error) => {
  console.error("WebSocket error:", error);
});
```

## Certificate Monitoring

```typescript
// certificate-monitor.ts
import https from "https";
import { URL } from "url";

interface CertificateInfo {
  subject: string;
  issuer: string;
  validFrom: Date;
  validTo: Date;
  daysRemaining: number;
}

async function checkCertificate(urlString: string): Promise<CertificateInfo> {
  const url = new URL(urlString);

  return new Promise((resolve, reject) => {
    const options = {
      host: url.hostname,
      port: 443,
      method: "GET",
    };

    const req = https.request(options, (res) => {
      const cert = res.socket.getPeerCertificate();

      if (!cert || !Object.keys(cert).length) {
        reject(new Error("No certificate found"));
        return;
      }

      const validTo = new Date(cert.valid_to);
      const now = new Date();
      const daysRemaining = Math.floor(
        (validTo.getTime() - now.getTime()) / (1000 * 60 * 60 * 24)
      );

      resolve({
        subject: cert.subject.CN,
        issuer: cert.issuer.CN,
        validFrom: new Date(cert.valid_from),
        validTo,
        daysRemaining,
      });
    });

    req.on("error", reject);
    req.end();
  });
}

// Monitor certificate expiration
async function monitorCertificates(urls: string[]) {
  for (const url of urls) {
    try {
      const info = await checkCertificate(url);

      console.log(`\nCertificate for ${url}:`);
      console.log(`Subject: ${info.subject}`);
      console.log(`Issuer: ${info.issuer}`);
      console.log(`Valid from: ${info.validFrom.toISOString()}`);
      console.log(`Valid to: ${info.validTo.toISOString()}`);
      console.log(`Days remaining: ${info.daysRemaining}`);

      if (info.daysRemaining < 30) {
        console.warn(`⚠️  Certificate expiring soon!`);
      }
    } catch (error) {
      console.error(`Error checking ${url}:`, error);
    }
  }
}

// Run daily
setInterval(() => {
  monitorCertificates(["https://example.com", "https://api.example.com"]);
}, 24 * 60 * 60 * 1000);
```

## Best Practices

### 1. **Use Strong TLS Versions**

- TLS 1.2 minimum (preferably TLS 1.3)
- Disable SSL 2.0, SSL 3.0, TLS 1.0, TLS 1.1
- Use strong cipher suites
- Prefer forward secrecy

### 2. **Certificate Management**

- Use certificates from trusted CAs
- Automate certificate renewal
- Monitor expiration dates
- Use wildcard certificates carefully
- Store private keys securely

### 3. **HSTS Configuration**

- Enable HSTS in production
- Use long max-age (1 year+)
- Include subdomains
- Consider HSTS preloading

### 4. **Redirect HTTP to HTTPS**

- Use 301 permanent redirects
- Apply at load balancer or web server
- Redirect before application logic
- Test redirects thoroughly

### 5. **Certificate Pinning**

- Pin intermediate or leaf certificates
- Include backup pins
- Have a recovery plan
- Monitor pin expiration

### 6. **Security Headers**

- Set all security headers
- Test header configuration
- Use CSP to prevent XSS
- Enable HSTS

### 7. **Testing**

- Use SSL Labs for testing
- Regular security audits
- Test certificate renewal
- Monitor certificate validity

### 8. **Performance**

- Enable HTTP/2
- Use OCSP stapling
- Configure session resumption
- Optimize cipher suites

## Key Takeaways

✅ **HTTPS is mandatory for production applications**  
✅ **Use Let's Encrypt for free, automated certificates**  
✅ **TLS 1.2+ only, disable older protocols**  
✅ **Enable HSTS to enforce HTTPS**  
✅ **Redirect all HTTP traffic to HTTPS**  
✅ **Monitor certificate expiration and automate renewal**  
✅ **Use strong cipher suites with forward secrecy**  
✅ **Certificate pinning adds extra security for mobile apps**  
✅ **mTLS provides mutual authentication for sensitive APIs**  
✅ **Regular security audits and SSL Labs testing**

HTTPS is the foundation of web security, protecting user data and building trust in your application.
