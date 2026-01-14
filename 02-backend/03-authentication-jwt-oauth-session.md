# Authentication (JWT, OAuth, Session-Based)

## Overview

Authentication verifies user identity. This tutorial covers three main approaches with Express: JWT (JSON Web Tokens) for stateless authentication, OAuth for third-party authentication, and session-based authentication using express-session.

## Authentication Strategies Comparison

| Feature         | JWT                             | OAuth                    | Session-Based            |
| --------------- | ------------------------------- | ------------------------ | ------------------------ |
| **Storage**     | Client (localStorage/cookie)    | Token stored client-side | Server (session store)   |
| **Scalability** | Excellent (stateless)           | Excellent                | Requires sticky sessions |
| **Security**    | Good (if implemented correctly) | Excellent                | Good                     |
| **Logout**      | Token expiration only           | Token revocation         | Server-side destroy      |
| **Use Case**    | APIs, SPAs                      | Third-party login        | Traditional web apps     |

## JWT Authentication with Express & Passport

### 1. Setup

\`\`\`bash
npm install passport passport-jwt jsonwebtoken bcrypt
npm install -D @types/passport @types/passport-jwt @types/jsonwebtoken
\`\`\`

\`\`\`typescript
// src/config/passport.ts
import { Strategy as JwtStrategy, ExtractJwt, StrategyOptions } from 'passport-jwt';
import passport from 'passport';
import userService from '../services/user.service';

const jwtOptions: StrategyOptions = {
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  secretOrKey: process.env.JWT_SECRET || 'your-secret-key',
};

passport.use(
  new JwtStrategy(jwtOptions, async (payload, done) => {
    try {
      const user = await userService.findById(payload.sub);
      if (user) {
        return done(null, user);
      }
      return done(null, false);
    } catch (error) {
      return done(error, false);
    }
  })
);

export default passport;
\`\`\`

\`\`\`typescript
// src/services/auth.service.ts
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import { v4 as uuid } from 'uuid';
import userService from './user.service';
import { AppError } from '../utils/AppError';

export interface JwtPayload {
  sub: string;
  email: string;
  roles: string[];
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

export class AuthService {
  private readonly JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';
  private readonly JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET || 'refresh-secret';
  private readonly ACCESS_TOKEN_EXPIRY = '15m';
  private readonly REFRESH_TOKEN_EXPIRY = '7d';

  async register(email: string, password: string, firstName: string, lastName: string) {
    const existingUser = await userService.findByEmail(email);
    if (existingUser) {
      throw new AppError('Email already exists', 409);
    }

    const hashedPassword = await bcrypt.hash(password, 10);

    const user = await userService.create({
      email,
      password: hashedPassword,
      firstName,
      lastName,
      roles: ['user'],
    });

    return this.generateTokens(user);
  }

  async login(email: string, password: string): Promise<AuthTokens> {
    const user = await userService.findByEmail(email);

    if (!user) {
      throw new AppError('Invalid credentials', 401);
    }

    const isPasswordValid = await bcrypt.compare(password, user.password);

    if (!isPasswordValid) {
      throw new AppError('Invalid credentials', 401);
    }

    return this.generateTokens(user);
  }

  async generateTokens(user: any): Promise<AuthTokens> {
    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      roles: user.roles || [],
    };

    const accessToken = jwt.sign(payload, this.JWT_SECRET, {
      expiresIn: this.ACCESS_TOKEN_EXPIRY,
    });

    const refreshToken = jwt.sign(payload, this.JWT_REFRESH_SECRET, {
      expiresIn: this.REFRESH_TOKEN_EXPIRY,
    });

    // Store refresh token
    await userService.updateRefreshToken(user.id, refreshToken);

    return { accessToken, refreshToken };
  }

  async refreshTokens(refreshToken: string): Promise<AuthTokens> {
    try {
      const payload = jwt.verify(refreshToken, this.JWT_REFRESH_SECRET) as JwtPayload;

      const user = await userService.findById(payload.sub);

      if (!user || user.refreshToken !== refreshToken) {
        throw new AppError('Invalid refresh token', 401);
      }

      return this.generateTokens(user);
    } catch (error) {
      throw new AppError('Invalid refresh token', 401);
    }
  }

  async logout(userId: string) {
    await userService.updateRefreshToken(userId, null);
  }
}

export default new AuthService();
\`\`\`

\`\`\`typescript
// src/controllers/auth.controller.ts
import { Request, Response } from 'express';
import authService from '../services/auth.service';
import { asyncHandler } from '../utils/asyncHandler';

export class AuthController {
  register = asyncHandler(async (req: Request, res: Response) => {
    const { email, password, firstName, lastName } = req.body;
    const tokens = await authService.register(email, password, firstName, lastName);

    res.status(201).json({
      success: true,
      data: tokens,
    });
  });

  login = asyncHandler(async (req: Request, res: Response) => {
    const { email, password } = req.body;
    const tokens = await authService.login(email, password);

    res.json({
      success: true,
      data: tokens,
    });
  });

  refresh = asyncHandler(async (req: Request, res: Response) => {
    const { refreshToken } = req.body;
    const tokens = await authService.refreshTokens(refreshToken);

    res.json({
      success: true,
      data: tokens,
    });
  });

  logout = asyncHandler(async (req: Request, res: Response) => {
    await authService.logout(req.user!.id);

    res.json({
      success: true,
      message: 'Logged out successfully',
    });
  });

  me = asyncHandler(async (req: Request, res: Response) => {
    res.json({
      success: true,
      data: req.user,
    });
  });
}

export default new AuthController();
\`\`\`

\`\`\`typescript
// src/routes/auth.routes.ts
import { Router } from 'express';
import authController from '../controllers/auth.controller';
import { authenticate } from '../middleware/auth.middleware';
import { validateRegister, validateLogin } from '../middleware/validation.middleware';
import { authLimiter } from '../middleware/rateLimiter.middleware';

const router = Router();

router.post('/register', authLimiter, validateRegister, authController.register);
router.post('/login', authLimiter, validateLogin, authController.login);
router.post('/refresh', authController.refresh);
router.post('/logout', authenticate, authController.logout);
router.get('/me', authenticate, authController.me);

export default router;
\`\`\`

## OAuth Authentication

### 1. OAuth with Google

\`\`\`bash
npm install passport-google-oauth20
npm install -D @types/passport-google-oauth20
\`\`\`

\`\`\`typescript
// src/config/passport-google.ts
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';
import passport from 'passport';
import userService from '../services/user.service';

passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      callbackURL: process.env.GOOGLE_CALLBACK_URL || 'http://localhost:3000/api/v1/auth/google/callback',
    },
    async (accessToken, refreshToken, profile, done) => {
      try {
        let user = await userService.findByEmail(profile.emails![0].value);

        if (!user) {
          user = await userService.create({
            email: profile.emails![0].value,
            firstName: profile.name!.givenName,
            lastName: profile.name!.familyName,
            password: '', // OAuth users don't have password
            roles: ['user'],
          });
        }

        return done(null, user);
      } catch (error) {
        return done(error as Error, undefined);
      }
    }
  )
);
\`\`\`

\`\`\`typescript
// src/routes/auth.routes.ts (add OAuth routes)
import passport from '../config/passport-google';

// Google OAuth
router.get(
  '/google',
  passport.authenticate('google', {
    scope: ['profile', 'email'],
    session: false,
  })
);

router.get(
  '/google/callback',
  passport.authenticate('google', { session: false }),
  asyncHandler(async (req, res) => {
    const tokens = await authService.generateTokens(req.user);
    
    // Redirect to frontend with tokens
    res.redirect(\`\${process.env.FRONTEND_URL}/auth/callback?token=\${tokens.accessToken}\`);
  })
);
\`\`\`

### 2. OAuth with GitHub

\`\`\`bash
npm install passport-github2
npm install -D @types/passport-github2
\`\`\`

\`\`\`typescript
// src/config/passport-github.ts
import { Strategy as GitHubStrategy } from 'passport-github2';
import passport from 'passport';
import userService from '../services/user.service';

passport.use(
  new GitHubStrategy(
    {
      clientID: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      callbackURL: 'http://localhost:3000/api/v1/auth/github/callback',
    },
    async (accessToken: string, refreshToken: string, profile: any, done: any) => {
      try {
        const email = profile.emails[0].value;
        let user = await userService.findByEmail(email);

        if (!user) {
          user = await userService.create({
            email,
            firstName: profile.displayName.split(' ')[0],
            lastName: profile.displayName.split(' ')[1] || '',
            password: '',
            roles: ['user'],
          });
        }

        return done(null, user);
      } catch (error) {
        return done(error, undefined);
      }
    }
  )
);
\`\`\`

## Session-Based Authentication

### 1. Setup Express Session

\`\`\`bash
npm install express-session connect-pg-simple
npm install -D @types/express-session
\`\`\`

\`\`\`typescript
// src/config/session.ts
import session from 'express-session';
import connectPgSimple from 'connect-pg-simple';
import { Pool } from 'pg';

const PgSession = connectPgSimple(session);

const pgPool = new Pool({
  connectionString: process.env.DATABASE_URL,
});

export const sessionConfig = session({
  store: new PgSession({
    pool: pgPool,
    tableName: 'session',
  }),
  secret: process.env.SESSION_SECRET || 'session-secret',
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
    sameSite: 'lax',
  },
});
\`\`\`

\`\`\`typescript
// src/server.ts (add session middleware)
import { sessionConfig } from './config/session';

app.use(sessionConfig);
\`\`\`

\`\`\`typescript
// src/services/auth.service.ts (add session methods)
async loginWithSession(req: any, email: string, password: string) {
  const user = await this.validateUser(email, password);

  // Store user in session
  req.session.userId = user.id;
  req.session.user = {
    id: user.id,
    email: user.email,
    roles: user.roles,
  };

  return user;
}

logoutFromSession(req: any) {
  return new Promise((resolve, reject) => {
    req.session.destroy((err: any) => {
      if (err) reject(err);
      resolve(true);
    });
  });
}
\`\`\`

\`\`\`typescript
// src/middleware/session-auth.middleware.ts
import { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/AppError';

export const requireSession = (req: Request, res: Response, next: NextFunction) => {
  if (!req.session.userId) {
    throw new AppError('Not authenticated', 401);
  }

  req.user = req.session.user;
  next();
};
\`\`\`

## Security Best Practices

### 1. Password Hashing

\`\`\`typescript
// Strong password hashing with bcrypt
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 10;

async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
\`\`\`

### 2. Token Security

\`\`\`typescript
// Secure token generation
import crypto from 'crypto';

function generateSecureToken(bytes: number = 32): string {
  return crypto.randomBytes(bytes).toString('hex');
}

// Token blacklisting for logout
class TokenBlacklist {
  private tokens: Set<string> = new Set();

  add(token: string, expiresIn: number) {
    this.tokens.add(token);
    setTimeout(() => this.tokens.delete(token), expiresIn);
  }

  has(token: string): boolean {
    return this.tokens.has(token);
  }
}

export const tokenBlacklist = new TokenBlacklist();
\`\`\`

### 3. Rate Limiting

\`\`\`typescript
// src/middleware/rateLimiter.middleware.ts
import rateLimit from 'express-rate-limit';

export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts
  message: 'Too many authentication attempts, please try again later',
  standardHeaders: true,
  legacyHeaders: false,
  skipSuccessfulRequests: true,
});

export const registerLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 3, // 3 registrations per hour
  message: 'Too many accounts created, please try again later',
});
\`\`\`

### 4. CSRF Protection

\`\`\`bash
npm install csurf
\`\`\`

\`\`\`typescript
// src/middleware/csrf.middleware.ts
import csrf from 'csurf';

export const csrfProtection = csrf({ cookie: true });

// In routes
router.post('/login', csrfProtection, authController.login);

// Send CSRF token to client
router.get('/csrf-token', csrfProtection, (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});
\`\`\`

### 5. Password Reset Flow

\`\`\`typescript
// src/services/auth.service.ts
async requestPasswordReset(email: string) {
  const user = await userService.findByEmail(email);
  if (!user) {
    // Don't reveal if email exists
    return;
  }

  const resetToken = crypto.randomBytes(32).toString('hex');
  const hashedToken = crypto.createHash('sha256').update(resetToken).digest('hex');
  
  await userService.updateResetToken(user.id, hashedToken, Date.now() + 3600000); // 1 hour

  // Send email with resetToken
  await emailService.sendPasswordReset(email, resetToken);
}

async resetPassword(token: string, newPassword: string) {
  const hashedToken = crypto.createHash('sha256').update(token).digest('hex');
  
  const user = await userService.findByResetToken(hashedToken);
  
  if (!user || user.resetTokenExpiry < Date.now()) {
    throw new AppError('Invalid or expired reset token', 400);
  }

  const hashedPassword = await bcrypt.hash(newPassword, 10);
  await userService.updatePassword(user.id, hashedPassword);
  await userService.clearResetToken(user.id);
}
\`\`\`

## Testing

\`\`\`typescript
// src/__tests__/auth.test.ts
import request from 'supertest';
import app from '../server';

describe('Authentication', () => {
  describe('POST /api/v1/auth/register', () => {
    it('should register a new user', async () => {
      const res = await request(app)
        .post('/api/v1/auth/register')
        .send({
          email: 'newuser@example.com',
          password: 'SecurePass123!',
          firstName: 'John',
          lastName: 'Doe',
        });

      expect(res.status).toBe(201);
      expect(res.body.data).toHaveProperty('accessToken');
      expect(res.body.data).toHaveProperty('refreshToken');
    });

    it('should not register duplicate email', async () => {
      await request(app)
        .post('/api/v1/auth/register')
        .send({
          email: 'duplicate@example.com',
          password: 'password123',
          firstName: 'John',
          lastName: 'Doe',
        });

      const res = await request(app)
        .post('/api/v1/auth/register')
        .send({
          email: 'duplicate@example.com',
          password: 'password123',
          firstName: 'Jane',
          lastName: 'Doe',
        });

      expect(res.status).toBe(409);
    });
  });

  describe('POST /api/v1/auth/login', () => {
    it('should login with valid credentials', async () => {
      const res = await request(app)
        .post('/api/v1/auth/login')
        .send({
          email: 'test@example.com',
          password: 'password123',
        });

      expect(res.status).toBe(200);
      expect(res.body.data).toHaveProperty('accessToken');
    });

    it('should reject invalid credentials', async () => {
      const res = await request(app)
        .post('/api/v1/auth/login')
        .send({
          email: 'test@example.com',
          password: 'wrongpassword',
        });

      expect(res.status).toBe(401);
    });
  });

  describe('POST /api/v1/auth/refresh', () => {
    it('should refresh tokens', async () => {
      const loginRes = await request(app)
        .post('/api/v1/auth/login')
        .send({
          email: 'test@example.com',
          password: 'password123',
        });

      const refreshToken = loginRes.body.data.refreshToken;

      const res = await request(app)
        .post('/api/v1/auth/refresh')
        .send({ refreshToken });

      expect(res.status).toBe(200);
      expect(res.body.data).toHaveProperty('accessToken');
    });
  });
});
\`\`\`

## Key Takeaways

1. **JWT** is stateless and scalable for APIs
2. **OAuth** enables third-party authentication
3. **Sessions** work well for traditional web apps
4. **Passport.js** simplifies strategy implementation
5. **Rate limiting** protects against brute force
6. **Bcrypt** for secure password hashing
7. **Refresh tokens** enable secure long-term sessions
8. **CSRF protection** prevents cross-site attacks

Choose the right strategy based on your application needs and scale requirements.
