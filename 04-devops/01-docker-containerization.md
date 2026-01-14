# Docker Containerization

## Overview

Docker is a platform for developing, shipping, and running applications in containers. Containers package an application with all its dependencies, ensuring consistency across different environments. Docker enables rapid deployment, scaling, and isolation of applications.

**Key Concepts:**

- **Containers**: Lightweight, standalone executable packages
- **Images**: Read-only templates for creating containers
- **Dockerfile**: Instructions for building images
- **Docker Compose**: Multi-container orchestration
- **Volumes**: Persistent data storage

## Practical Use Cases

### 1. **Development Environment Consistency**

Same environment across team

- Eliminates "works on my machine" issues
- Quick onboarding for new developers
- Consistent development/production parity
- Easy dependency management

### 2. **Microservices Architecture**

Isolated service deployment

- Independent scaling
- Technology stack flexibility
- Service isolation
- Easy rollback

### 3. **CI/CD Pipelines**

Automated testing and deployment

- Consistent build environments
- Fast test execution
- Parallel testing
- Reproducible builds

### 4. **Application Portability**

Run anywhere Docker runs

- Cloud provider agnostic
- Local to production consistency
- Easy migration
- Hybrid cloud support

### 5. **Resource Efficiency**

Better hardware utilization

- Multiple containers per host
- Fast startup times
- Lower overhead than VMs
- Dynamic resource allocation

## Step-by-Step Implementation

### 1. Install Docker

```bash
# macOS (using Homebrew)
brew install --cask docker

# Or download Docker Desktop from docker.com

# Ubuntu/Debian
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Verify installation
docker --version
docker run hello-world
```

### 2. Basic Dockerfile for Node.js/Express

```dockerfile
# Dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Build application
RUN npm run build

# Expose port
EXPOSE 3000

# Start application
CMD ["node", "dist/main"]
```

### 3. Multi-Stage Build (Optimized)

```dockerfile
# Dockerfile.multistage
# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig*.json ./

# Install all dependencies (including dev)
RUN npm ci

# Copy source code
COPY src ./src

# Build application
RUN npm run build

# Stage 2: Production
FROM node:18-alpine AS production

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production && \
    npm cache clean --force

# Copy built application from builder stage
COPY --from=builder --chown=nestjs:nodejs /app/dist ./dist

# Switch to non-root user
USER nestjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Start application
CMD ["node", "dist/main"]
```

### 4. .dockerignore File

```
# .dockerignore
node_modules
npm-debug.log
dist
.git
.gitignore
.env
.env.local
.env.*.local
*.md
.vscode
.idea
coverage
.DS_Store
```

### 5. Docker Compose for Full Stack

```yaml
# docker-compose.yml
version: "3.8"

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: myapp-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-postgres}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "${DB_PORT:-5432}:5432"
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-redis}
    volumes:
      - redis_data:/data
    ports:
      - "${REDIS_PORT:-6379}:6379"
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Backend API
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    container_name: myapp-api
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      NODE_ENV: production
      PORT: 3000
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: ${DB_NAME:-myapp}
      DB_USER: ${DB_USER:-postgres}
      DB_PASSWORD: ${DB_PASSWORD:-postgres}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD:-redis}
      JWT_SECRET: ${JWT_SECRET}
    ports:
      - "${API_PORT:-3000}:3000"
    networks:
      - app-network
    volumes:
      - ./uploads:/app/uploads
    healthcheck:
      test:
        [
          "CMD",
          "wget",
          "--quiet",
          "--tries=1",
          "--spider",
          "http://localhost:3000/health",
        ]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: myapp-nginx
    restart: unless-stopped
    depends_on:
      - api
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
```

### 6. Development Docker Compose

```yaml
# docker-compose.dev.yml
version: "3.8"

services:
  postgres:
    extends:
      file: docker-compose.yml
      service: postgres
    ports:
      - "5432:5432"

  redis:
    extends:
      file: docker-compose.yml
      service: redis
    ports:
      - "6379:6379"

  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
      target: development
    container_name: myapp-api-dev
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
    environment:
      NODE_ENV: development
      PORT: 3000
      DB_HOST: postgres
      DB_PORT: 5432
      REDIS_HOST: redis
      REDIS_PORT: 6379
    ports:
      - "3000:3000"
      - "9229:9229" # Debug port
    networks:
      - app-network
    volumes:
      - .:/app
      - /app/node_modules
      - /app/dist
    command: npm run start:dev
```

### 7. Development Dockerfile with Hot Reload

```dockerfile
# Dockerfile.dev
FROM node:18-alpine AS development

WORKDIR /app

# Install dependencies for node-gyp
RUN apk add --no-cache python3 make g++

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev)
RUN npm install

# Copy source code
COPY . .

# Expose ports (app and debug)
EXPOSE 3000 9229

# Start with hot reload
CMD ["npm", "run", "start:dev"]
```

### 8. Docker Commands

```bash
# Build image
docker build -t myapp:latest .

# Build with specific Dockerfile
docker build -f Dockerfile.multistage -t myapp:prod .

# Run container
docker run -d -p 3000:3000 --name myapp myapp:latest

# Run with environment variables
docker run -d -p 3000:3000 \
  -e DB_HOST=postgres \
  -e DB_PASSWORD=secret \
  --name myapp myapp:latest

# Run with volume mount
docker run -d -p 3000:3000 \
  -v $(pwd)/uploads:/app/uploads \
  --name myapp myapp:latest

# View logs
docker logs myapp
docker logs -f myapp  # Follow logs

# Execute commands in running container
docker exec -it myapp sh
docker exec myapp npm run migration:run

# Stop and remove container
docker stop myapp
docker rm myapp

# Remove image
docker rmi myapp:latest

# Prune unused resources
docker system prune -a
docker volume prune
```

### 9. Docker Compose Commands

```bash
# Start all services
docker-compose up

# Start in detached mode
docker-compose up -d

# Start specific service
docker-compose up postgres redis

# Use different compose file
docker-compose -f docker-compose.dev.yml up

# View logs
docker-compose logs
docker-compose logs -f api

# Execute command in service
docker-compose exec api npm run migration:run
docker-compose exec postgres psql -U postgres -d myapp

# Stop services
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop and remove with volumes
docker-compose down -v

# Rebuild services
docker-compose up --build

# Scale services
docker-compose up -d --scale api=3
```

### 10. Nginx Configuration

```nginx
# nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream api {
        server api:3000;
    }

    server {
        listen 80;
        server_name localhost;

        # Rate limiting
        limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

        # API proxy
        location /api {
            limit_req zone=api_limit burst=20 nodelay;

            proxy_pass http://api;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;

            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }

        # Health check
        location /health {
            proxy_pass http://api/health;
            access_log off;
        }

        # Static files
        location /uploads {
            alias /app/uploads;
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }
}
```

### 11. Health Check Endpoint

```typescript
// src/routes/health.routes.ts
import { Router, Request, Response } from "express";
import { AppDataSource } from "../database/data-source"; // TypeORM example

const router = Router();

router.get("/health", async (req: Request, res: Response) => {
  try {
    const checks = {
      status: "ok",
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      database: "unknown",
      memory: {
        used: process.memoryUsage().heapUsed,
        total: process.memoryUsage().heapTotal,
        limit: 150 * 1024 * 1024,
      },
    };

    // Database health check
    try {
      await AppDataSource.query("SELECT 1");
      checks.database = "healthy";
    } catch (error) {
      checks.database = "unhealthy";
      checks.status = "degraded";
    }

    // Memory check
    if (checks.memory.used > checks.memory.limit) {
      checks.status = "degraded";
    }

    const statusCode = checks.status === "ok" ? 200 : 503;
    res.status(statusCode).json(checks);
  } catch (error) {
    res.status(503).json({
      status: "error",
      error: error instanceof Error ? error.message : "Unknown error",
    });
  }
});

export default router;

      // Memory RSS check
      () => this.memory.checkRSS("memory_rss", 150 * 1024 * 1024),

      // Disk storage check
      () =>
        this.disk.checkStorage("storage", { path: "/", thresholdPercent: 0.9 }),
    ]);
  }
}
```

### 12. Production-Ready Dockerfile

```dockerfile
# Dockerfile.production
# Stage 1: Dependencies
FROM node:18-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Builder
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Runner
FROM node:18-alpine AS runner
WORKDIR /app

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nodejs

# Copy dependencies and build artifacts
COPY --from=deps --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist
COPY --from=builder --chown=nodejs:nodejs /app/package*.json ./

# Set environment
ENV NODE_ENV=production
ENV PORT=3000

USER nestjs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

CMD ["node", "dist/main.js"]
```

## Best Practices

### 1. **Use Multi-Stage Builds**

Reduces final image size significantly:

```dockerfile
# Builder stage has dev dependencies
FROM node:18-alpine AS builder
RUN npm ci
RUN npm run build

# Production stage only has runtime dependencies
FROM node:18-alpine AS production
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
```

### 2. **Use .dockerignore**

Exclude unnecessary files from build context:

```
node_modules
.git
.env
*.md
coverage
```

### 3. **Layer Caching Optimization**

Order commands from least to most frequently changed:

```dockerfile
# 1. Base image (rarely changes)
FROM node:18-alpine

# 2. Dependencies (changes occasionally)
COPY package*.json ./
RUN npm ci

# 3. Source code (changes frequently)
COPY . .
RUN npm run build
```

### 4. **Use Specific Image Tags**

```dockerfile
# ❌ Bad - unpredictable
FROM node:latest

# ✅ Good - specific version
FROM node:18.19.0-alpine3.19
```

### 5. **Run as Non-Root User**

```dockerfile
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs
```

### 6. **Health Checks**

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1
```

### 7. **Environment Variables**

```dockerfile
# Use ARG for build-time variables
ARG NODE_VERSION=18

# Use ENV for runtime variables
ENV NODE_ENV=production
ENV PORT=3000
```

### 8. **Security Scanning**

```bash
# Scan images for vulnerabilities
docker scan myapp:latest

# Use Trivy
trivy image myapp:latest
```

## Key Takeaways

✅ **Containers ensure consistency across environments**  
✅ **Multi-stage builds reduce image size dramatically**  
✅ **Use .dockerignore to optimize build context**  
✅ **Always run containers as non-root users**  
✅ **Implement health checks for reliability**  
✅ **Use Docker Compose for multi-container apps**  
✅ **Tag images with specific versions**  
✅ **Layer caching speeds up builds**  
✅ **Volumes persist data beyond container lifecycle**  
✅ **Regular security scanning prevents vulnerabilities**

Docker containerization is essential for modern application deployment, providing consistency, portability, and efficiency across all environments.
