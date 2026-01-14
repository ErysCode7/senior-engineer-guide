# Secure Authentication Practices

## Overview

Secure authentication is the foundation of application security. It verifies user identity while protecting credentials, sessions, and sensitive data. Implementing authentication correctly prevents unauthorized access, protects user privacy, and maintains system integrity. Modern authentication combines multiple layers of security including strong password policies, secure session management, multi-factor authentication, and token-based systems.

**Key Components:**

- **Password Security**: Hashing, salting, complexity requirements
- **Session Management**: Secure cookies, session expiration, token refresh
- **Multi-Factor Authentication (MFA)**: Additional verification layers
- **Token Security**: JWT best practices, token rotation
- **Account Protection**: Brute force prevention, account lockout

## Practical Use Cases

### 1. **User Authentication**

Verify user identity

- Login systems
- Single sign-on (SSO)
- Social authentication
- Passwordless authentication

### 2. **API Authentication**

Secure API access

- API keys
- OAuth 2.0
- JWT tokens
- Service-to-service auth

### 3. **Mobile Apps**

App-specific authentication

- Biometric authentication
- Device fingerprinting
- Certificate pinning
- Refresh tokens

### 4. **Enterprise Systems**

Corporate authentication

- LDAP/Active Directory
- SAML integration
- Multi-factor authentication
- Role-based access control

### 5. **Sensitive Operations**

Step-up authentication

- Financial transactions
- Account changes
- Admin operations
- Data exports

## Password Security

### 1. Password Hashing with Argon2

```typescript
// services/password.service.ts
import * as argon2 from "argon2";
import { Injectable } from "@nestjs/common";

@Injectable()
export class PasswordService {
  // Hash password using Argon2id (recommended)
  async hash(password: string): Promise<string> {
    return argon2.hash(password, {
      type: argon2.argon2id, // Hybrid of argon2i and argon2d
      memoryCost: 65536, // 64 MB
      timeCost: 3, // Iterations
      parallelism: 4, // Threads
    });
  }

  // Verify password
  async verify(hash: string, password: string): Promise<boolean> {
    try {
      return await argon2.verify(hash, password);
    } catch (error) {
      return false;
    }
  }

  // Check if hash needs rehashing (outdated parameters)
  needsRehash(hash: string): boolean {
    return argon2.needsRehash(hash, {
      type: argon2.argon2id,
      memoryCost: 65536,
      timeCost: 3,
      parallelism: 4,
    });
  }
}

// Alternative: bcrypt (also secure but slower for attackers)
import * as bcrypt from "bcrypt";

@Injectable()
export class BcryptPasswordService {
  private readonly saltRounds = 12; // Higher = more secure but slower

  async hash(password: string): Promise<string> {
    return bcrypt.hash(password, this.saltRounds);
  }

  async verify(hash: string, password: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }
}
```

### 2. Password Policy Validation

```typescript
// validators/password.validator.ts
import {
  registerDecorator,
  ValidationOptions,
  ValidatorConstraint,
  ValidatorConstraintInterface,
} from "class-validator";

@ValidatorConstraint({ async: false })
export class IsStrongPasswordConstraint
  implements ValidatorConstraintInterface
{
  validate(password: string): boolean {
    if (!password || password.length < 12) {
      return false;
    }

    // At least one uppercase
    if (!/[A-Z]/.test(password)) {
      return false;
    }

    // At least one lowercase
    if (!/[a-z]/.test(password)) {
      return false;
    }

    // At least one number
    if (!/\d/.test(password)) {
      return false;
    }

    // At least one special character
    if (!/[!@#$%^&*()_+\-=\[\]{};':"\\|,.<>\/?]/.test(password)) {
      return false;
    }

    // Check against common passwords
    const commonPasswords = [
      "Password123!",
      "Admin123!",
      "Welcome123!",
      // Load from file or service
    ];

    if (commonPasswords.includes(password)) {
      return false;
    }

    // No sequential characters
    if (/(.)\1{2,}/.test(password)) {
      return false;
    }

    return true;
  }

  defaultMessage(): string {
    return "Password must be at least 12 characters with uppercase, lowercase, number, and special character";
  }
}

export function IsStrongPassword(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      constraints: [],
      validator: IsStrongPasswordConstraint,
    });
  };
}

// services/password-breach-checker.service.ts
import { Injectable } from "@nestjs/common";
import * as crypto from "crypto";
import axios from "axios";

@Injectable()
export class PasswordBreachCheckerService {
  // Check password against Have I Been Pwned API
  async isBreached(password: string): Promise<boolean> {
    const hash = crypto
      .createHash("sha1")
      .update(password)
      .digest("hex")
      .toUpperCase();
    const prefix = hash.substring(0, 5);
    const suffix = hash.substring(5);

    try {
      const response = await axios.get(
        `https://api.pwnedpasswords.com/range/${prefix}`,
        { timeout: 5000 }
      );

      const hashes = response.data.split("\n");
      return hashes.some((line: string) => line.startsWith(suffix));
    } catch (error) {
      // Fail open - don't block user if API is down
      console.error("Password breach check failed:", error);
      return false;
    }
  }
}
```

## JWT Token Security

### 1. Secure JWT Implementation

```typescript
// services/jwt.service.ts
import { Injectable } from "@nestjs/common";
import { JwtService as NestJwtService } from "@nestjs/jwt";
import { ConfigService } from "@nestjs/config";
import * as crypto from "crypto";

export interface TokenPayload {
  sub: string; // User ID
  email: string;
  roles: string[];
  iat?: number;
  exp?: number;
  jti?: string; // JWT ID for token revocation
}

@Injectable()
export class JwtService {
  constructor(
    private jwtService: NestJwtService,
    private configService: ConfigService
  ) {}

  // Generate access token (short-lived)
  generateAccessToken(payload: TokenPayload): string {
    return this.jwtService.sign(
      {
        ...payload,
        jti: crypto.randomUUID(),
        type: "access",
      },
      {
        secret: this.configService.get("JWT_ACCESS_SECRET"),
        expiresIn: "15m", // Short expiration
        issuer: "myapp",
        audience: "myapp-users",
      }
    );
  }

  // Generate refresh token (long-lived)
  generateRefreshToken(userId: string): string {
    return this.jwtService.sign(
      {
        sub: userId,
        jti: crypto.randomUUID(),
        type: "refresh",
      },
      {
        secret: this.configService.get("JWT_REFRESH_SECRET"),
        expiresIn: "7d",
        issuer: "myapp",
        audience: "myapp-users",
      }
    );
  }

  // Verify access token
  async verifyAccessToken(token: string): Promise<TokenPayload> {
    try {
      const payload = this.jwtService.verify(token, {
        secret: this.configService.get("JWT_ACCESS_SECRET"),
        issuer: "myapp",
        audience: "myapp-users",
      });

      // Check if token is revoked
      const isRevoked = await this.isTokenRevoked(payload.jti);
      if (isRevoked) {
        throw new Error("Token has been revoked");
      }

      return payload;
    } catch (error) {
      throw new Error("Invalid token");
    }
  }

  // Verify refresh token
  async verifyRefreshToken(token: string): Promise<TokenPayload> {
    try {
      return this.jwtService.verify(token, {
        secret: this.configService.get("JWT_REFRESH_SECRET"),
        issuer: "myapp",
        audience: "myapp-users",
      });
    } catch (error) {
      throw new Error("Invalid refresh token");
    }
  }

  // Token revocation (store in Redis)
  async revokeToken(jti: string, expiresIn: number): Promise<void> {
    await this.redis.setex(`revoked:${jti}`, expiresIn, "1");
  }

  // Check if token is revoked
  async isTokenRevoked(jti: string): Promise<boolean> {
    const result = await this.redis.get(`revoked:${jti}`);
    return result === "1";
  }
}
```

### 2. Token Rotation Strategy

```typescript
// services/auth.service.ts
import { Injectable, UnauthorizedException } from "@nestjs/common";

@Injectable()
export class AuthService {
  constructor(
    private jwtService: JwtService,
    private refreshTokenRepository: RefreshTokenRepository
  ) {}

  async login(email: string, password: string) {
    // Verify credentials
    const user = await this.validateUser(email, password);

    // Generate tokens
    const accessToken = this.jwtService.generateAccessToken({
      sub: user.id,
      email: user.email,
      roles: user.roles,
    });

    const refreshToken = this.jwtService.generateRefreshToken(user.id);

    // Store refresh token in database
    await this.refreshTokenRepository.save({
      userId: user.id,
      token: await this.hashToken(refreshToken),
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
      userAgent: request.headers["user-agent"],
      ipAddress: request.ip,
    });

    return {
      accessToken,
      refreshToken,
      expiresIn: 900, // 15 minutes
    };
  }

  async refresh(refreshToken: string) {
    // Verify refresh token
    const payload = await this.jwtService.verifyRefreshToken(refreshToken);

    // Check if refresh token exists in database
    const hashedToken = await this.hashToken(refreshToken);
    const storedToken = await this.refreshTokenRepository.findOne({
      where: {
        userId: payload.sub,
        token: hashedToken,
      },
    });

    if (!storedToken || storedToken.expiresAt < new Date()) {
      throw new UnauthorizedException("Invalid refresh token");
    }

    // Revoke old refresh token (rotation)
    await this.refreshTokenRepository.delete(storedToken.id);

    // Generate new tokens
    const user = await this.userRepository.findById(payload.sub);
    const newAccessToken = this.jwtService.generateAccessToken({
      sub: user.id,
      email: user.email,
      roles: user.roles,
    });

    const newRefreshToken = this.jwtService.generateRefreshToken(user.id);

    // Store new refresh token
    await this.refreshTokenRepository.save({
      userId: user.id,
      token: await this.hashToken(newRefreshToken),
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });

    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken,
      expiresIn: 900,
    };
  }

  async logout(userId: string, refreshToken: string) {
    // Revoke refresh token
    const hashedToken = await this.hashToken(refreshToken);
    await this.refreshTokenRepository.delete({
      userId,
      token: hashedToken,
    });

    // Optionally revoke all user tokens
    // await this.refreshTokenRepository.delete({ userId });
  }

  private async hashToken(token: string): Promise<string> {
    return crypto.createHash("sha256").update(token).digest("hex");
  }
}
```

## Multi-Factor Authentication (MFA)

### 1. TOTP-Based MFA

```typescript
// services/mfa.service.ts
import { Injectable } from "@nestjs/common";
import * as speakeasy from "speakeasy";
import * as QRCode from "qrcode";

@Injectable()
export class MfaService {
  // Generate MFA secret
  generateSecret(email: string): { secret: string; qrCode: string } {
    const secret = speakeasy.generateSecret({
      name: `MyApp (${email})`,
      issuer: "MyApp",
      length: 32,
    });

    return {
      secret: secret.base32,
      qrCode: secret.otpauth_url!,
    };
  }

  // Generate QR code image
  async generateQRCode(otpauthUrl: string): Promise<string> {
    return QRCode.toDataURL(otpauthUrl);
  }

  // Verify TOTP code
  verifyToken(secret: string, token: string): boolean {
    return speakeasy.totp.verify({
      secret,
      encoding: "base32",
      token,
      window: 2, // Allow 2 time steps before/after (60 seconds)
    });
  }

  // Generate backup codes
  generateBackupCodes(count: number = 10): string[] {
    const codes: string[] = [];
    for (let i = 0; i < count; i++) {
      const code = crypto.randomBytes(4).toString("hex").toUpperCase();
      codes.push(code);
    }
    return codes;
  }
}

// controllers/mfa.controller.ts
@Controller("mfa")
export class MfaController {
  constructor(
    private mfaService: MfaService,
    private userService: UserService
  ) {}

  @Post("enable")
  @UseGuards(JwtAuthGuard)
  async enableMfa(@CurrentUser() user: User) {
    // Generate secret
    const { secret, qrCode } = this.mfaService.generateSecret(user.email);

    // Store secret temporarily (not yet enabled)
    await this.userService.storePendingMfaSecret(user.id, secret);

    // Generate QR code image
    const qrCodeImage = await this.mfaService.generateQRCode(qrCode);

    return {
      qrCode: qrCodeImage,
      secret, // Show to user for manual entry
    };
  }

  @Post("verify")
  @UseGuards(JwtAuthGuard)
  async verifyAndEnableMfa(
    @CurrentUser() user: User,
    @Body("token") token: string
  ) {
    // Get pending secret
    const secret = await this.userService.getPendingMfaSecret(user.id);

    if (!secret) {
      throw new BadRequestException("MFA setup not initiated");
    }

    // Verify token
    const isValid = this.mfaService.verifyToken(secret, token);

    if (!isValid) {
      throw new BadRequestException("Invalid verification code");
    }

    // Generate backup codes
    const backupCodes = this.mfaService.generateBackupCodes();
    const hashedBackupCodes = await Promise.all(
      backupCodes.map((code) => this.passwordService.hash(code))
    );

    // Enable MFA
    await this.userService.enableMfa(user.id, secret, hashedBackupCodes);

    return {
      message: "MFA enabled successfully",
      backupCodes, // Show once to user
    };
  }

  @Post("verify-login")
  async verifyMfaLogin(
    @Body("userId") userId: string,
    @Body("token") token: string
  ) {
    const user = await this.userService.findById(userId);

    if (!user.mfaEnabled) {
      throw new BadRequestException("MFA not enabled");
    }

    // Verify TOTP
    let isValid = this.mfaService.verifyToken(user.mfaSecret, token);

    // If TOTP fails, try backup codes
    if (!isValid) {
      isValid = await this.verifyBackupCode(user, token);
    }

    if (!isValid) {
      throw new UnauthorizedException("Invalid MFA code");
    }

    // Generate tokens
    return this.authService.generateTokens(user);
  }

  private async verifyBackupCode(user: User, code: string): Promise<boolean> {
    for (const hashedCode of user.mfaBackupCodes) {
      const isMatch = await this.passwordService.verify(hashedCode, code);
      if (isMatch) {
        // Remove used backup code
        await this.userService.removeBackupCode(user.id, hashedCode);
        return true;
      }
    }
    return false;
  }
}
```

## Session Management

### 1. Secure Session Configuration

```typescript
// config/session.config.ts
import * as session from "express-session";
import * as RedisStore from "connect-redis";
import { createClient } from "redis";

const redisClient = createClient({
  url: process.env.REDIS_URL,
  socket: {
    reconnectStrategy: (retries) => Math.min(retries * 50, 500),
  },
});

redisClient.connect();

export const sessionConfig: session.SessionOptions = {
  store: new RedisStore({
    client: redisClient,
    prefix: "sess:",
    ttl: 86400, // 24 hours
  }),
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  name: "sid", // Custom session cookie name
  cookie: {
    httpOnly: true, // Prevent XSS
    secure: process.env.NODE_ENV === "production", // HTTPS only
    sameSite: "lax", // CSRF protection
    maxAge: 24 * 60 * 60 * 1000, // 24 hours
    domain: process.env.COOKIE_DOMAIN,
  },
  rolling: true, // Reset expiry on each request
};

// middleware/session-security.middleware.ts
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class SessionSecurityMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    if (req.session && req.session.userId) {
      // Check session metadata
      const userAgent = req.headers["user-agent"];
      const ipAddress = req.ip;

      // Detect session hijacking
      if (req.session.userAgent && req.session.userAgent !== userAgent) {
        req.session.destroy(() => {});
        return res.status(401).json({ error: "Session invalid" });
      }

      // Track IP changes (optional: allow but log)
      if (req.session.ipAddress && req.session.ipAddress !== ipAddress) {
        console.warn(`IP change detected for session ${req.session.id}`);
        // Optionally require re-authentication
      }

      // Store metadata on first request
      if (!req.session.userAgent) {
        req.session.userAgent = userAgent;
        req.session.ipAddress = ipAddress;
      }

      // Update last activity
      req.session.lastActivity = Date.now();
    }

    next();
  }
}
```

## Account Protection

### 1. Brute Force Protection

```typescript
// services/brute-force-protection.service.ts
import { Injectable } from "@nestjs/common";
import { InjectRedis } from "@nestjs-modules/ioredis";
import Redis from "ioredis";

@Injectable()
export class BruteForceProtectionService {
  constructor(@InjectRedis() private readonly redis: Redis) {}

  async recordFailedAttempt(identifier: string): Promise<void> {
    const key = `auth:failed:${identifier}`;
    await this.redis.incr(key);
    await this.redis.expire(key, 3600); // 1 hour window
  }

  async getFailedAttempts(identifier: string): Promise<number> {
    const key = `auth:failed:${identifier}`;
    const count = await this.redis.get(key);
    return count ? parseInt(count) : 0;
  }

  async resetFailedAttempts(identifier: string): Promise<void> {
    const key = `auth:failed:${identifier}`;
    await this.redis.del(key);
  }

  async isLocked(identifier: string): Promise<boolean> {
    const attempts = await this.getFailedAttempts(identifier);
    return attempts >= 5; // Lock after 5 failed attempts
  }

  async lockAccount(
    identifier: string,
    durationSeconds: number = 3600
  ): Promise<void> {
    const key = `auth:locked:${identifier}`;
    await this.redis.setex(key, durationSeconds, "1");
  }

  async calculateDelay(attempts: number): Promise<number> {
    // Progressive delay: 2^attempts seconds
    return Math.min(Math.pow(2, attempts), 300); // Max 5 minutes
  }
}

// guards/brute-force.guard.ts
@Injectable()
export class BruteForceGuard implements CanActivate {
  constructor(private bruteForceService: BruteForceProtectionService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const identifier = request.body.email || request.ip;

    // Check if locked
    const isLocked = await this.bruteForceService.isLocked(identifier);
    if (isLocked) {
      throw new TooManyRequestsException(
        "Account temporarily locked. Try again later."
      );
    }

    // Add delay based on failed attempts
    const attempts = await this.bruteForceService.getFailedAttempts(identifier);
    if (attempts > 0) {
      const delay = await this.bruteForceService.calculateDelay(attempts);
      await new Promise((resolve) => setTimeout(resolve, delay * 1000));
    }

    return true;
  }
}
```

## Best Practices

### 1. **Password Security**

- Use Argon2id or bcrypt for hashing
- Enforce strong password policies (12+ characters)
- Check against breached passwords
- Never store plaintext passwords
- Implement password history

### 2. **Token Management**

- Short-lived access tokens (15 minutes)
- Long-lived refresh tokens (7 days)
- Token rotation on refresh
- Store refresh tokens securely
- Implement token revocation

### 3. **Session Security**

- Use secure, httpOnly cookies
- Implement session timeout
- Detect session hijacking
- Allow session termination
- Track active sessions

### 4. **Multi-Factor Authentication**

- Offer MFA for all users
- Support multiple MFA methods
- Provide backup codes
- Secure MFA secret storage
- Allow MFA recovery

### 5. **Account Protection**

- Rate limit login attempts
- Progressive delay after failures
- Account lockout after threshold
- Email notifications for suspicious activity
- IP-based restrictions

### 6. **Security Monitoring**

- Log all authentication events
- Monitor failed login attempts
- Alert on suspicious patterns
- Track token usage
- Regular security audits

### 7. **Secure Communication**

- Always use HTTPS
- Implement certificate pinning
- Validate SSL/TLS certificates
- Use secure WebSocket (WSS)
- Encrypt sensitive data

### 8. **Compliance**

- GDPR compliance for EU users
- PCI-DSS for payment data
- HIPAA for healthcare data
- Regular security assessments
- Documentation and policies

## Key Takeaways

✅ **Use Argon2id for password hashing**  
✅ **Implement JWT token rotation strategy**  
✅ **Short-lived access tokens, long-lived refresh tokens**  
✅ **Enable multi-factor authentication**  
✅ **Protect against brute force attacks**  
✅ **Secure session management with Redis**  
✅ **Monitor and log all authentication events**  
✅ **Implement progressive delays for failed attempts**  
✅ **Store sensitive data encrypted at rest**  
✅ **Regular security audits and updates**

Secure authentication is not optional—it's the foundation of application security and user trust.
