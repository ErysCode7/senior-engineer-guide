# Authentication (JWT, OAuth, Session-Based)

## Overview

Authentication verifies user identity. This tutorial covers three main approaches: JWT (JSON Web Tokens) for stateless authentication, OAuth for third-party authentication, and session-based authentication for traditional server-side sessions.

## Authentication Strategies Comparison

| Feature         | JWT                             | OAuth                    | Session-Based            |
| --------------- | ------------------------------- | ------------------------ | ------------------------ |
| **Storage**     | Client (localStorage/cookie)    | Token stored client-side | Server (session store)   |
| **Scalability** | Excellent (stateless)           | Excellent                | Requires sticky sessions |
| **Security**    | Good (if implemented correctly) | Excellent                | Good                     |
| **Logout**      | Token expiration only           | Token revocation         | Server-side destroy      |
| **Use Case**    | APIs, SPAs                      | Third-party login        | Traditional web apps     |

## Step-by-Step Implementation

### 1. JWT Authentication with NestJS

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm install -D @types/passport-jwt @types/bcrypt
```

```typescript
// auth/auth.module.ts
import { Module } from "@nestjs/common";
import { JwtModule } from "@nestjs/jwt";
import { PassportModule } from "@nestjs/passport";
import { ConfigModule, ConfigService } from "@nestjs/config";
import { AuthService } from "./auth.service";
import { AuthController } from "./auth.controller";
import { JwtStrategy } from "./strategies/jwt.strategy";
import { LocalStrategy } from "./strategies/local.strategy";
import { UsersModule } from "../users/users.module";

@Module({
  imports: [
    UsersModule,
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get<string>("JWT_SECRET"),
        signOptions: {
          expiresIn: configService.get<string>("JWT_EXPIRES_IN", "15m"),
        },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy, LocalStrategy],
  exports: [AuthService],
})
export class AuthModule {}

// auth/auth.service.ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";
import { ConfigService } from "@nestjs/config";
import * as bcrypt from "bcrypt";
import { UsersService } from "../users/users.service";
import { LoginDto } from "./dto/login.dto";
import { RegisterDto } from "./dto/register.dto";

export interface JwtPayload {
  sub: string; // user id
  email: string;
  roles: string[];
}

export interface AuthTokens {
  accessToken: string;
  refreshToken: string;
}

@Injectable()
export class AuthService {
  constructor(
    private readonly usersService: UsersService,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService
  ) {}

  async validateUser(email: string, password: string) {
    const user = await this.usersService.findByEmail(email);

    if (!user) {
      throw new UnauthorizedException("Invalid credentials");
    }

    const isPasswordValid = await bcrypt.compare(password, user.password);

    if (!isPasswordValid) {
      throw new UnauthorizedException("Invalid credentials");
    }

    return user;
  }

  async login(loginDto: LoginDto): Promise<AuthTokens> {
    const user = await this.validateUser(loginDto.email, loginDto.password);
    return this.generateTokens(user);
  }

  async register(registerDto: RegisterDto) {
    const hashedPassword = await bcrypt.hash(registerDto.password, 10);

    const user = await this.usersService.create({
      ...registerDto,
      password: hashedPassword,
    });

    return this.generateTokens(user);
  }

  async generateTokens(user: any): Promise<AuthTokens> {
    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      roles: user.roles || [],
    };

    const accessToken = this.jwtService.sign(payload);

    const refreshToken = this.jwtService.sign(payload, {
      secret: this.configService.get("JWT_REFRESH_SECRET"),
      expiresIn: "7d",
    });

    // Store refresh token in database
    await this.usersService.updateRefreshToken(user.id, refreshToken);

    return {
      accessToken,
      refreshToken,
    };
  }

  async refreshTokens(refreshToken: string): Promise<AuthTokens> {
    try {
      const payload = this.jwtService.verify(refreshToken, {
        secret: this.configService.get("JWT_REFRESH_SECRET"),
      });

      const user = await this.usersService.findOne(payload.sub);

      // Verify refresh token matches stored token
      const isValid = await bcrypt.compare(refreshToken, user.refreshToken);

      if (!isValid) {
        throw new UnauthorizedException("Invalid refresh token");
      }

      return this.generateTokens(user);
    } catch (error) {
      throw new UnauthorizedException("Invalid refresh token");
    }
  }

  async logout(userId: string): Promise<void> {
    await this.usersService.updateRefreshToken(userId, null);
  }
}

// auth/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";
import { ConfigService } from "@nestjs/config";
import { UsersService } from "../../users/users.service";
import { JwtPayload } from "../auth.service";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, "jwt") {
  constructor(
    private readonly configService: ConfigService,
    private readonly usersService: UsersService
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get("JWT_SECRET"),
      ignoreExpiration: false,
    });
  }

  async validate(payload: JwtPayload) {
    const user = await this.usersService.findOne(payload.sub);

    if (!user || !user.isActive) {
      throw new UnauthorizedException();
    }

    return user; // Attached to request.user
  }
}

// auth/strategies/local.strategy.ts
import { Injectable } from "@nestjs/common";
import { PassportStrategy } from "@nestjs/passport";
import { Strategy } from "passport-local";
import { AuthService } from "../auth.service";

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy, "local") {
  constructor(private authService: AuthService) {
    super({
      usernameField: "email",
    });
  }

  async validate(email: string, password: string) {
    return this.authService.validateUser(email, password);
  }
}

// auth/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
import { Reflector } from "@nestjs/core";

@Injectable()
export class JwtAuthGuard extends AuthGuard("jwt") {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    const isPublic = this.reflector.getAllAndOverride<boolean>("isPublic", [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) {
      return true;
    }

    return super.canActivate(context);
  }
}

// auth/auth.controller.ts
import {
  Controller,
  Post,
  Body,
  UseGuards,
  HttpCode,
  HttpStatus,
  Req,
} from "@nestjs/common";
import { ApiTags, ApiOperation, ApiBearerAuth } from "@nestjs/swagger";
import { AuthService } from "./auth.service";
import { LoginDto } from "./dto/login.dto";
import { RegisterDto } from "./dto/register.dto";
import { RefreshTokenDto } from "./dto/refresh-token.dto";
import { JwtAuthGuard } from "./guards/jwt-auth.guard";
import { Public } from "./decorators/public.decorator";

@ApiTags("auth")
@Controller("auth")
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Public()
  @Post("register")
  @ApiOperation({ summary: "Register a new user" })
  register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }

  @Public()
  @Post("login")
  @HttpCode(HttpStatus.OK)
  @ApiOperation({ summary: "Login user" })
  login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }

  @Public()
  @Post("refresh")
  @HttpCode(HttpStatus.OK)
  @ApiOperation({ summary: "Refresh access token" })
  refresh(@Body() refreshTokenDto: RefreshTokenDto) {
    return this.authService.refreshTokens(refreshTokenDto.refreshToken);
  }

  @Post("logout")
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()
  @HttpCode(HttpStatus.OK)
  @ApiOperation({ summary: "Logout user" })
  logout(@Req() req) {
    return this.authService.logout(req.user.id);
  }
}
```

### 2. OAuth 2.0 Implementation (Google, GitHub)

```bash
npm install @nestjs/passport passport-google-oauth20 passport-github2
npm install -D @types/passport-google-oauth20 @types/passport-github2
```

```typescript
// auth/strategies/google.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy, VerifyCallback } from 'passport-google-oauth20';
import { ConfigService } from '@nestjs/config';
import { AuthService } from '../auth.service';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy, 'google') {
  constructor(
    private configService: ConfigService,
    private authService: AuthService,
  ) {
    super({
      clientID: configService.get('GOOGLE_CLIENT_ID'),
      clientSecret: configService.get('GOOGLE_CLIENT_SECRET'),
      callbackURL: configService.get('GOOGLE_CALLBACK_URL'),
      scope: ['email', 'profile'],
    });
  }

  async validate(
    accessToken: string,
    refreshToken: string,
    profile: any,
    done: VerifyCallback,
  ) {
    const { name, emails, photos } = profile;

    const user = {
      email: emails[0].value,
      firstName: name.givenName,
      lastName: name.familyName,
      picture: photos[0].value,
      provider: 'google',
      providerId: profile.id,
    };

    // Find or create user
    const dbUser = await this.authService.findOrCreateOAuthUser(user);

    done(null, dbUser);
  }
}

// auth/strategies/github.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-github2';
import { ConfigService } from '@nestjs/config';
import { AuthService } from '../auth.service';

@Injectable()
export class GithubStrategy extends PassportStrategy(Strategy, 'github') {
  constructor(
    private configService: ConfigService,
    private authService: AuthService,
  ) {
    super({
      clientID: configService.get('GITHUB_CLIENT_ID'),
      clientSecret: configService.get('GITHUB_CLIENT_SECRET'),
      callbackURL: configService.get('GITHUB_CALLBACK_URL'),
      scope: ['user:email'],
    });
  }

  async validate(accessToken: string, refreshToken: string, profile: any) {
    const { displayName, emails, photos } = profile;

    const user = {
      email: emails[0].value,
      firstName: displayName.split(' ')[0],
      lastName: displayName.split(' ')[1] || '',
      picture: photos[0].value,
      provider: 'github',
      providerId: profile.id,
    };

    return this.authService.findOrCreateOAuthUser(user);
  }
}

// auth/auth.service.ts (add OAuth methods)
async findOrCreateOAuthUser(oauthUser: any) {
  let user = await this.usersService.findByEmail(oauthUser.email);

  if (!user) {
    // Create new user from OAuth data
    user = await this.usersService.create({
      email: oauthUser.email,
      firstName: oauthUser.firstName,
      lastName: oauthUser.lastName,
      picture: oauthUser.picture,
      provider: oauthUser.provider,
      providerId: oauthUser.providerId,
      isEmailVerified: true, // OAuth emails are already verified
    });
  }

  return user;
}

// auth/guards/google-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class GoogleAuthGuard extends AuthGuard('google') {}

// auth/guards/github-auth.guard.ts
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class GithubAuthGuard extends AuthGuard('github') {}

// auth/auth.controller.ts (add OAuth routes)
@Get('google')
@UseGuards(GoogleAuthGuard)
@ApiOperation({ summary: 'Login with Google' })
googleAuth() {
  // Initiates Google OAuth flow
}

@Get('google/callback')
@UseGuards(GoogleAuthGuard)
@ApiOperation({ summary: 'Google OAuth callback' })
async googleAuthCallback(@Req() req, @Res() res) {
  const tokens = await this.authService.generateTokens(req.user);

  // Redirect to frontend with tokens
  res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${tokens.accessToken}`);
}

@Get('github')
@UseGuards(GithubAuthGuard)
@ApiOperation({ summary: 'Login with GitHub' })
githubAuth() {
  // Initiates GitHub OAuth flow
}

@Get('github/callback')
@UseGuards(GithubAuthGuard)
@ApiOperation({ summary: 'GitHub OAuth callback' })
async githubAuthCallback(@Req() req, @Res() res) {
  const tokens = await this.authService.generateTokens(req.user);
  res.redirect(`${process.env.FRONTEND_URL}/auth/callback?token=${tokens.accessToken}`);
}
```

### 3. Session-Based Authentication

```bash
npm install express-session connect-redis redis
npm install -D @types/express-session @types/connect-redis
```

```typescript
// main.ts
import * as session from 'express-session';
import { createClient } from 'redis';
import RedisStore from 'connect-redis';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Redis client
  const redisClient = createClient({
    url: process.env.REDIS_URL,
  });
  await redisClient.connect();

  // Session middleware
  app.use(
    session({
      store: new RedisStore({ client: redisClient }),
      secret: process.env.SESSION_SECRET,
      resave: false,
      saveUninitialized: false,
      cookie: {
        maxAge: 1000 * 60 * 60 * 24, // 1 day
        httpOnly: true,
        secure: process.env.NODE_ENV === 'production', // HTTPS only in production
        sameSite: 'lax',
      },
    }),
  );

  await app.listen(3000);
}

// auth/strategies/session.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AuthService } from '../auth.service';

@Injectable()
export class SessionStrategy extends PassportStrategy(Strategy, 'session') {
  constructor(private authService: AuthService) {
    super({
      usernameField: 'email',
      passReqToCallback: false,
    });
  }

  async validate(email: string, password: string) {
    return this.authService.validateUser(email, password);
  }
}

// auth/serializer/session.serializer.ts
import { Injectable } from '@nestjs/common';
import { PassportSerializer } from '@nestjs/passport';
import { UsersService } from '../../users/users.service';

@Injectable()
export class SessionSerializer extends PassportSerializer {
  constructor(private usersService: UsersService) {
    super();
  }

  serializeUser(user: any, done: (err: Error, user: any) => void) {
    done(null, { id: user.id });
  }

  async deserializeUser(payload: any, done: (err: Error, user: any) => void) {
    const user = await this.usersService.findOne(payload.id);
    done(null, user);
  }
}

// auth/guards/session-auth.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class SessionAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    return request.isAuthenticated();
  }
}

// auth/auth.controller.ts (session endpoints)
@Post('session/login')
@UseGuards(LocalAuthGuard)
@HttpCode(HttpStatus.OK)
sessionLogin(@Req() req) {
  return { user: req.user };
}

@Post('session/logout')
@UseGuards(SessionAuthGuard)
@HttpCode(HttpStatus.OK)
sessionLogout(@Req() req, @Res() res) {
  req.logout((err) => {
    if (err) {
      return res.status(500).json({ message: 'Logout failed' });
    }
    res.json({ message: 'Logged out successfully' });
  });
}

@Get('session/me')
@UseGuards(SessionAuthGuard)
getSessionUser(@Req() req) {
  return req.user;
}
```

### 4. Multi-Factor Authentication (MFA)

```bash
npm install otplib qrcode
npm install -D @types/qrcode
```

```typescript
// auth/mfa.service.ts
import { Injectable } from '@nestjs/common';
import { authenticator } from 'otplib';
import * as QRCode from 'qrcode';
import { UsersService } from '../users/users.service';

@Injectable()
export class MfaService {
  constructor(private usersService: UsersService) {}

  async generateSecret(userId: string) {
    const user = await this.usersService.findOne(userId);
    const secret = authenticator.generateSecret();

    const otpauth = authenticator.keyuri(
      user.email,
      'MyApp',
      secret,
    );

    // Generate QR code
    const qrCode = await QRCode.toDataURL(otpauth);

    // Save secret to user (encrypted)
    await this.usersService.updateMfaSecret(userId, secret);

    return {
      secret,
      qrCode,
    };
  }

  async verifyToken(userId: string, token: string): Promise<boolean> {
    const user = await this.usersService.findOne(userId);

    if (!user.mfaSecret) {
      return false;
    }

    return authenticator.verify({
      token,
      secret: user.mfaSecret,
    });
  }

  async enableMfa(userId: string, token: string): Promise<boolean> {
    const isValid = await this.verifyToken(userId, token);

    if (isValid) {
      await this.usersService.enableMfa(userId);
      return true;
    }

    return false;
  }

  async disableMfa(userId: string, token: string): Promise<boolean> {
    const isValid = await this.verifyToken(userId, token);

    if (isValid) {
      await this.usersService.disableMfa(userId);
      return true;
    }

    return false;
  }
}

// auth/auth.controller.ts (MFA endpoints)
@Post('mfa/setup')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
setupMfa(@CurrentUser('id') userId: string) {
  return this.mfaService.generateSecret(userId);
}

@Post('mfa/enable')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
enableMfa(@CurrentUser('id') userId: string, @Body() body: { token: string }) {
  return this.mfaService.enableMfa(userId, body.token);
}

@Post('mfa/verify')
@UseGuards(JwtAuthGuard)
@ApiBearerAuth()
verifyMfa(@CurrentUser('id') userId: string, @Body() body: { token: string }) {
  return this.mfaService.verifyToken(userId, body.token);
}
```

## Best Practices

### 1. Password Security

```typescript
// Use bcrypt with appropriate rounds
const saltRounds = 10; // 10-12 rounds recommended
const hashedPassword = await bcrypt.hash(password, saltRounds);

// Implement password strength requirements
import * as zxcvbn from "zxcvbn";

function validatePasswordStrength(password: string) {
  const result = zxcvbn(password);
  return result.score >= 3; // 0-4 scale
}
```

### 2. Token Security

```typescript
// Store refresh tokens securely
- Hash refresh tokens before storing in database
- Use httpOnly cookies for tokens when possible
- Implement token rotation
- Set appropriate expiration times (15min access, 7d refresh)
```

### 3. Rate Limiting

```typescript
// Implement rate limiting on auth endpoints
@Throttle({ default: { limit: 3, ttl: 300000 } }) // 3 attempts per 5 min
@Post('login')
login(@Body() loginDto: LoginDto) {}
```

## Key Takeaways

1. **JWT**: Stateless, scalable, perfect for APIs and SPAs
2. **OAuth**: Delegate authentication to trusted providers
3. **Sessions**: Traditional, server-side, good for web apps
4. **MFA**: Add extra security layer for sensitive operations
5. **Hash passwords**: Always use bcrypt or argon2
6. **Short-lived tokens**: 15min access, longer refresh tokens
7. **Secure storage**: httpOnly cookies or secure localStorage
8. **Rate limiting**: Prevent brute force attacks

Choose the authentication strategy that best fits your application's needs and security requirements.
