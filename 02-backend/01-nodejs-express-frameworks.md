# Node.js & Express Framework

## Overview

Node.js enables JavaScript on the server side, while Express is a minimal and flexible framework that provides a robust set of features for building web and mobile applications. Express is the de facto standard for Node.js web applications and follows a middleware-based architecture.

## What is Express?

Express is a fast, unopinionated, minimalist web framework for Node.js that provides:

- **Routing**: Simple and flexible routing system
- **Middleware**: Chain of functions for request/response handling
- **HTTP utilities**: Built on top of Node.js HTTP module
- **Template engine support**: Integration with various view engines
- **Content negotiation**: Easy request and response handling

## Practical Use Cases

- **RESTful APIs**: Building scalable backend services
- **Web applications**: Server-side rendered websites
- **Microservices**: Lightweight services
- **Real-time applications**: WebSocket and event-driven architectures
- **Authentication systems**: JWT, OAuth, session-based auth

## Step-by-Step Implementation

### 1. Setting Up an Express Project with TypeScript

```bash
# Create project directory
mkdir my-express-api
cd my-express-api

# Initialize project
npm init -y

# Install dependencies
npm install express cors helmet morgan dotenv
npm install express-validator bcrypt jsonwebtoken

# Install TypeScript and types
npm install -D typescript @types/express @types/node @types/cors
npm install -D @types/morgan @types/bcrypt @types/jsonwebtoken
npm install -D ts-node nodemon

# Install dev tools
npm install -D eslint prettier
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "moduleResolution": "node"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

```json
// package.json scripts
{
  "scripts": {
    "dev": "nodemon src/server.ts",
    "build": "tsc",
    "start": "node dist/server.js",
    "lint": "eslint src/**/*.ts",
    "format": "prettier --write src/**/*.ts"
  }
}
```

```typescript
// src/server.ts - Application entry point
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import dotenv from 'dotenv';
import { errorHandler } from './middleware/error.middleware';
import { notFoundHandler } from './middleware/notFound.middleware';
import userRoutes from './routes/user.routes';
import authRoutes from './routes/auth.routes';
import productRoutes from './routes/product.routes';

// Load environment variables
dotenv.config();

const app = express();
const PORT = process.env.PORT || 3000;

// Security middleware
app.use(helmet());

// CORS configuration
app.use(
  cors({
    origin: process.env.FRONTEND_URL || 'http://localhost:3000',
    credentials: true,
  })
);

// Request parsing
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Logging
app.use(morgan('dev'));

// API routes
app.use('/api/v1/users', userRoutes);
app.use('/api/v1/auth', authRoutes);
app.use('/api/v1/products', productRoutes);

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// 404 handler
app.use(notFoundHandler);

// Global error handler
app.use(errorHandler);

app.listen(PORT, () => {
  console.log(\`🚀 Server running on http://localhost:\${PORT}\`);
  console.log(\`📝 Environment: \${process.env.NODE_ENV || 'development'}\`);
});

export default app;
```


### 2. Project Structure

```
src/
├── config/
│   ├── database.ts
│   └── env.ts
├── controllers/
│   ├── user.controller.ts
│   ├── auth.controller.ts
│   └── product.controller.ts
├── middleware/
│   ├── auth.middleware.ts
│   ├── validation.middleware.ts
│   ├── error.middleware.ts
│   ├── notFound.middleware.ts
│   └── rateLimiter.middleware.ts
├── models/
│   ├── user.model.ts
│   └── product.model.ts
├── routes/
│   ├── user.routes.ts
│   ├── auth.routes.ts
│   └── product.routes.ts
├── services/
│   ├── user.service.ts
│   ├── auth.service.ts
│   └── product.service.ts
├── types/
│   ├── express.d.ts
│   └── index.d.ts
├── utils/
│   ├── AppError.ts
│   ├── asyncHandler.ts
│   └── response.ts
└── server.ts
```

### 3. Creating a Complete User Module

```typescript
// src/types/express.d.ts - Extend Express types
declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        email: string;
        roles: string[];
      };
    }
  }
}

export {};
```

```typescript
// src/models/user.model.ts
export interface User {
  id: string;
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  isActive: boolean;
  roles: string[];
  refreshToken?: string;
  createdAt: Date;
  updatedAt: Date;
}

export interface CreateUserDto {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  roles?: string[];
}

export interface UpdateUserDto {
  firstName?: string;
  lastName?: string;
  isActive?: boolean;
}

export interface UserResponse extends Omit<User, 'password' | 'refreshToken'> {}
```

```typescript
// src/utils/AppError.ts
export class AppError extends Error {
  statusCode: number;
  isOperational: boolean;

  constructor(message: string, statusCode: number = 500) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
    Error.captureStackTrace(this, this.constructor);
  }
}
```

```typescript
// src/utils/asyncHandler.ts
import { Request, Response, NextFunction } from 'express';

export const asyncHandler = (
  fn: (req: Request, res: Response, next: NextFunction) => Promise<any>
) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};
```

```typescript
// src/services/user.service.ts
import bcrypt from 'bcrypt';
import { v4 as uuid } from 'uuid';
import { User, CreateUserDto, UpdateUserDto } from '../models/user.model';
import { AppError } from '../utils/AppError';

// In-memory storage (replace with database in production)
const users: User[] = [];

export class UserService {
  async create(data: CreateUserDto): Promise<Omit<User, 'password'>> {
    const existing = users.find((u) => u.email === data.email);
    if (existing) {
      throw new AppError('Email already exists', 409);
    }

    const hashedPassword = await bcrypt.hash(data.password, 10);

    const user: User = {
      id: uuid(),
      email: data.email,
      password: hashedPassword,
      firstName: data.firstName,
      lastName: data.lastName,
      isActive: true,
      roles: data.roles || ['user'],
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    users.push(user);

    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }

  async findAll(page = 1, limit = 10) {
    const start = (page - 1) * limit;
    const end = start + limit;

    const paginatedUsers = users.slice(start, end).map((user) => {
      const { password, ...userWithoutPassword } = user;
      return userWithoutPassword;
    });

    return {
      data: paginatedUsers,
      total: users.length,
    };
  }

  async findById(id: string): Promise<Omit<User, 'password'>> {
    const user = users.find((u) => u.id === id);
    if (!user) {
      throw new AppError(`User with ID ${id} not found`, 404);
    }

    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }

  async findByEmail(email: string): Promise<User | undefined> {
    return users.find((u) => u.email === email);
  }

  async update(id: string, data: UpdateUserDto): Promise<Omit<User, 'password'>> {
    const user = users.find((u) => u.id === id);
    if (!user) {
      throw new AppError(`User with ID ${id} not found`, 404);
    }

    Object.assign(user, data, { updatedAt: new Date() });

    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }

  async delete(id: string): Promise<void> {
    const index = users.findIndex((u) => u.id === id);
    if (index === -1) {
      throw new AppError(`User with ID ${id} not found`, 404);
    }

    users.splice(index, 1);
  }

  async updateRefreshToken(id: string, refreshToken: string | null): Promise<void> {
    const user = users.find((u) => u.id === id);
    if (user) {
      user.refreshToken = refreshToken || undefined;
    }
  }
}

export default new UserService();
```

```typescript
// src/controllers/user.controller.ts
import { Request, Response } from 'express';
import userService from '../services/user.service';
import { asyncHandler } from '../utils/asyncHandler';

export class UserController {
  create = asyncHandler(async (req: Request, res: Response) => {
    const user = await userService.create(req.body);
    res.status(201).json({
      success: true,
      data: user,
    });
  });

  findAll = asyncHandler(async (req: Request, res: Response) => {
    const page = parseInt(req.query.page as string) || 1;
    const limit = parseInt(req.query.limit as string) || 10;

    const result = await userService.findAll(page, limit);

    res.json({
      success: true,
      data: result.data,
      pagination: {
        page,
        limit,
        total: result.total,
        totalPages: Math.ceil(result.total / limit),
      },
    });
  });

  findOne = asyncHandler(async (req: Request, res: Response) => {
    const user = await userService.findById(req.params.id);
    res.json({
      success: true,
      data: user,
    });
  });

  update = asyncHandler(async (req: Request, res: Response) => {
    const user = await userService.update(req.params.id, req.body);
    res.json({
      success: true,
      data: user,
    });
  });

  delete = asyncHandler(async (req: Request, res: Response) => {
    await userService.delete(req.params.id);
    res.status(204).send();
  });
}

export default new UserController();
```

```typescript
// src/routes/user.routes.ts
import { Router } from 'express';
import userController from '../controllers/user.controller';
import { authenticate } from '../middleware/auth.middleware';
import { authorize } from '../middleware/roles.middleware';
import { validateCreateUser, validateUpdateUser } from '../middleware/validation.middleware';

const router = Router();

router.use(authenticate); // All routes require authentication

router.post('/', authorize('admin'), validateCreateUser, userController.create);
router.get('/', authorize('admin'), userController.findAll);
router.get('/:id', userController.findOne);
router.patch('/:id', validateUpdateUser, userController.update);
router.delete('/:id', authorize('admin'), userController.delete);

export default router;
```

### 4. Middleware Implementation

```typescript
// src/middleware/auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { AppError } from '../utils/AppError';
import { asyncHandler } from '../utils/asyncHandler';

export interface JwtPayload {
  sub: string;
  email: string;
  roles: string[];
}

export const authenticate = asyncHandler(
  async (req: Request, res: Response, next: NextFunction) => {
    const token = req.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      throw new AppError('No token provided', 401);
    }

    try {
      const decoded = jwt.verify(
        token,
        process.env.JWT_SECRET || 'your-secret-key'
      ) as JwtPayload;

      req.user = {
        id: decoded.sub,
        email: decoded.email,
        roles: decoded.roles,
      };
      
      next();
    } catch (error) {
      throw new AppError('Invalid token', 401);
    }
  }
);
```

```typescript
// src/middleware/roles.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/AppError';

export const authorize = (...roles: string[]) => {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) {
      throw new AppError('Not authenticated', 401);
    }

    const hasRole = roles.some((role) => req.user!.roles.includes(role));

    if (!hasRole) {
      throw new AppError('Insufficient permissions', 403);
    }

    next();
  };
};
```

```typescript
// src/middleware/validation.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { body, validationResult } from 'express-validator';
import { AppError } from '../utils/AppError';

export const validateCreateUser = [
  body('email').isEmail().withMessage('Invalid email address'),
  body('password')
    .isLength({ min: 8 })
    .withMessage('Password must be at least 8 characters'),
  body('firstName').notEmpty().withMessage('First name is required'),
  body('lastName').notEmpty().withMessage('Last name is required'),
  (req: Request, res: Response, next: NextFunction) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      throw new AppError(
        errors.array().map((e) => e.msg).join(', '),
        400
      );
    }
    next();
  },
];

export const validateUpdateUser = [
  body('email').optional().isEmail().withMessage('Invalid email address'),
  body('firstName').optional().notEmpty().withMessage('First name cannot be empty'),
  body('lastName').optional().notEmpty().withMessage('Last name cannot be empty'),
  (req: Request, res: Response, next: NextFunction) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      throw new AppError(
        errors.array().map((e) => e.msg).join(', '),
        400
      );
    }
    next();
  },
];
```

```typescript
// src/middleware/error.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/AppError';

export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      success: false,
      message: err.message,
      ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
    });
  }

  console.error('ERROR 💥:', err);

  res.status(500).json({
    success: false,
    message: 'Internal server error',
    ...(process.env.NODE_ENV === 'development' && {
      error: err.message,
      stack: err.stack,
    }),
  });
};
```

```typescript
// src/middleware/notFound.middleware.ts
import { Request, Response } from 'express';

export const notFoundHandler = (req: Request, res: Response) => {
  res.status(404).json({
    success: false,
    message: `Route ${req.originalUrl} not found`,
  });
};
```

### 5. Rate Limiting

```bash
npm install express-rate-limit
```

```typescript
// src/middleware/rateLimiter.middleware.ts
import rateLimit from 'express-rate-limit';

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: 'Too many requests from this IP, please try again later.',
  standardHeaders: true,
  legacyHeaders: false,
});

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  message: 'Too many login attempts, please try again later.',
  skipSuccessfulRequests: true,
});
```

### 6. Database Integration with Prisma

```bash
npm install @prisma/client
npm install -D prisma
npx prisma init
```

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id           String   @id @default(uuid())
  email        String   @unique
  password     String
  firstName    String
  lastName     String
  isActive     Boolean  @default(true)
  roles        String[]
  refreshToken String?
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  @@map("users")
}
```

```typescript
// src/config/database.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
});

export default prisma;
```

```typescript
// src/services/user.service.ts (with Prisma)
import bcrypt from 'bcrypt';
import prisma from '../config/database';
import { CreateUserDto, UpdateUserDto } from '../models/user.model';
import { AppError } from '../utils/AppError';

export class UserService {
  async create(data: CreateUserDto) {
    const existing = await prisma.user.findUnique({
      where: { email: data.email },
    });

    if (existing) {
      throw new AppError('Email already exists', 409);
    }

    const hashedPassword = await bcrypt.hash(data.password, 10);

    const user = await prisma.user.create({
      data: {
        ...data,
        password: hashedPassword,
        roles: data.roles || ['user'],
      },
      select: {
        id: true,
        email: true,
        firstName: true,
        lastName: true,
        roles: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      },
    });

    return user;
  }

  async findAll(page = 1, limit = 10) {
    const skip = (page - 1) * limit;

    const [data, total] = await Promise.all([
      prisma.user.findMany({
        skip,
        take: limit,
        orderBy: { createdAt: 'desc' },
        select: {
          id: true,
          email: true,
          firstName: true,
          lastName: true,
          roles: true,
          isActive: true,
          createdAt: true,
        },
      }),
      prisma.user.count(),
    ]);

    return { data, total };
  }

  async findById(id: string) {
    const user = await prisma.user.findUnique({
      where: { id },
      select: {
        id: true,
        email: true,
        firstName: true,
        lastName: true,
        roles: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      },
    });

    if (!user) {
      throw new AppError(`User with ID ${id} not found`, 404);
    }

    return user;
  }

  async findByEmail(email: string) {
    return prisma.user.findUnique({
      where: { email },
    });
  }

  async update(id: string, data: UpdateUserDto) {
    const user = await prisma.user.update({
      where: { id },
      data,
      select: {
        id: true,
        email: true,
        firstName: true,
        lastName: true,
        roles: true,
        isActive: true,
        createdAt: true,
        updatedAt: true,
      },
    });

    return user;
  }

  async delete(id: string) {
    await prisma.user.delete({
      where: { id },
    });
  }

  async updateRefreshToken(id: string, refreshToken: string | null) {
    await prisma.user.update({
      where: { id },
      data: { refreshToken },
    });
  }
}

export default new UserService();
```

### 7. Testing with Jest and Supertest

```bash
npm install -D jest ts-jest @types/jest supertest @types/supertest
```

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.interface.ts',
  ],
};
```

```typescript
// src/__tests__/user.service.test.ts
import userService from '../services/user.service';
import { AppError } from '../utils/AppError';

describe('UserService', () => {
  describe('create', () => {
    it('should create a new user', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'password123',
        firstName: 'John',
        lastName: 'Doe',
      };

      const user = await userService.create(userData);

      expect(user).toHaveProperty('id');
      expect(user.email).toBe(userData.email);
      expect(user).not.toHaveProperty('password');
    });

    it('should throw error if email exists', async () => {
      const userData = {
        email: 'existing@example.com',
        password: 'password123',
        firstName: 'John',
        lastName: 'Doe',
      };

      await userService.create(userData);

      await expect(userService.create(userData)).rejects.toThrow(AppError);
    });
  });

  describe('findById', () => {
    it('should throw error if user not found', async () => {
      await expect(userService.findById('invalid-id')).rejects.toThrow(AppError);
    });
  });
});
```

```typescript
// src/__tests__/user.routes.test.ts
import request from 'supertest';
import app from '../server';

describe('User Routes', () => {
  let authToken: string;

  beforeAll(async () => {
    const res = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'admin@example.com', password: 'password123' });
    
    authToken = res.body.data.accessToken;
  });

  describe('GET /api/v1/users', () => {
    it('should return users list', async () => {
      const res = await request(app)
        .get('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`);

      expect(res.status).toBe(200);
      expect(res.body).toHaveProperty('data');
      expect(Array.isArray(res.body.data)).toBe(true);
    });

    it('should return 401 without token', async () => {
      const res = await request(app).get('/api/v1/users');
      expect(res.status).toBe(401);
    });
  });

  describe('POST /api/v1/users', () => {
    it('should create a new user', async () => {
      const userData = {
        email: 'newuser@example.com',
        password: 'password123',
        firstName: 'New',
        lastName: 'User',
      };

      const res = await request(app)
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send(userData);

      expect(res.status).toBe(201);
      expect(res.body.data.email).toBe(userData.email);
    });

    it('should return 400 for invalid data', async () => {
      const res = await request(app)
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ email: 'invalid' });

      expect(res.status).toBe(400);
    });
  });
});
```

## Best Practices

### 1. Use TypeScript

Always use TypeScript for type safety and better developer experience.

### 2. Implement Proper Error Handling

Create custom error classes and use a global error handler middleware.

### 3. Use Async/Await with Error Boundaries

Wrap async route handlers with `asyncHandler` to catch errors automatically.

### 4. Organize Code by Feature

Group related files (routes, controllers, services, models) by feature/domain.

### 5. Implement Authentication & Authorization

Use middleware for JWT verification and role-based access control.

### 6. Validate Input Data

Use `express-validator` for comprehensive request validation.

### 7. Rate Limiting

Protect your API from abuse with rate limiting middleware.

### 8. Security Headers

Use `helmet` to set security-related HTTP headers.

### 9. Environment Variables

Store sensitive data and configuration in environment variables.

### 10. Write Tests

Unit test services and integration test routes for reliability.

## Express vs NestJS

| Feature            | Express                      | NestJS                          |
|--------------------|------------------------------|----------------------------------|
| **Learning Curve** | Easy to learn                | Steeper, requires OOP knowledge |
| **Flexibility**    | Very flexible, unopinionated | Opinionated, structured         |
| **Boilerplate**    | Minimal                      | More boilerplate                |
| **Type Safety**    | Optional (with TypeScript)   | Built-in with TypeScript        |
| **Architecture**   | Design your own              | Enforced architecture           |
| **DI**             | Manual or libraries          | Built-in                        |
| **Best For**       | Beginners, small-medium apps | Enterprise applications         |
| **Community**      | Massive, mature              | Growing, modern                 |

## Key Takeaways

1. **Express is simple**: Minimalist framework, easy to learn and use
2. **Middleware-based**: Chain functions to handle requests
3. **Flexible**: Design your own architecture and patterns
4. **TypeScript support**: Use TypeScript for better code quality
5. **Rich ecosystem**: Thousands of middleware packages available
6. **Performance**: Lightweight and fast
7. **Community**: Large community and extensive documentation
8. **Perfect for beginners**: Start simple, add complexity as needed

Express is ideal for developers who want control over their application architecture and prefer a minimalist approach to building APIs. It's the perfect starting point for learning backend development before moving to more opinionated frameworks like NestJS.
