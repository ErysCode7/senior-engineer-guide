# API Design Best Practices

## Overview

API design is a critical skill for senior engineers. Well-designed APIs are intuitive, maintainable, and scalable. This tutorial covers RESTful and GraphQL API design principles, versioning strategies, documentation practices, and best practices for building production-grade APIs.

## Use Cases

- **Public APIs**: Providing third-party access to your platform
- **Microservices Communication**: Inter-service communication patterns
- **Mobile/Web Applications**: Backend APIs for client applications
- **Partner Integrations**: B2B API integrations
- **Internal Tools**: Admin dashboards and internal services

## RESTful API Design Principles

### Resource-Based URL Design

```typescript
// ❌ Bad: Action-based URLs
POST /api/createUser
GET /api/getUserById?id=123
POST /api/deleteUser

// ✅ Good: Resource-based URLs
POST /api/users
GET /api/users/123
DELETE /api/users/123

// Nested resources
GET /api/users/123/orders
GET /api/users/123/orders/456
POST /api/users/123/orders

// Collections and filtering
GET /api/users?role=admin&status=active
GET /api/orders?date_from=2024-01-01&date_to=2024-12-31
```

### HTTP Methods and Status Codes

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Patch,
  Delete,
  Body,
  Param,
  HttpStatus,
  HttpException,
} from "@nestjs/common";

@Controller("api/users")
export class UsersController {
  // GET - Retrieve resource(s)
  @Get()
  async findAll() {
    return {
      statusCode: HttpStatus.OK,
      data: await this.usersService.findAll(),
    };
  }

  @Get(":id")
  async findOne(@Param("id") id: string) {
    const user = await this.usersService.findOne(id);

    if (!user) {
      throw new HttpException("User not found", HttpStatus.NOT_FOUND); // 404
    }

    return {
      statusCode: HttpStatus.OK, // 200
      data: user,
    };
  }

  // POST - Create new resource
  @Post()
  async create(@Body() createUserDto: CreateUserDto) {
    const user = await this.usersService.create(createUserDto);

    return {
      statusCode: HttpStatus.CREATED, // 201
      data: user,
    };
  }

  // PUT - Full update (replace entire resource)
  @Put(":id")
  async update(@Param("id") id: string, @Body() updateUserDto: UpdateUserDto) {
    const user = await this.usersService.update(id, updateUserDto);

    return {
      statusCode: HttpStatus.OK, // 200
      data: user,
    };
  }

  // PATCH - Partial update
  @Patch(":id")
  async partialUpdate(
    @Param("id") id: string,
    @Body() partialDto: Partial<UpdateUserDto>
  ) {
    const user = await this.usersService.partialUpdate(id, partialDto);

    return {
      statusCode: HttpStatus.OK, // 200
      data: user,
    };
  }

  // DELETE - Remove resource
  @Delete(":id")
  async remove(@Param("id") id: string) {
    await this.usersService.remove(id);

    return {
      statusCode: HttpStatus.NO_CONTENT, // 204
    };
  }
}
```

### Pagination and Filtering

```typescript
import { Controller, Get, Query } from "@nestjs/common";

interface PaginationQuery {
  page?: number;
  limit?: number;
  sortBy?: string;
  order?: "asc" | "desc";
}

interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

@Controller("api/products")
export class ProductsController {
  @Get()
  async findAll(
    @Query("page") page: number = 1,
    @Query("limit") limit: number = 10,
    @Query("sortBy") sortBy: string = "createdAt",
    @Query("order") order: "asc" | "desc" = "desc",
    @Query("category") category?: string,
    @Query("minPrice") minPrice?: number,
    @Query("maxPrice") maxPrice?: number
  ): Promise<PaginatedResponse<Product>> {
    // Validate pagination parameters
    const validatedPage = Math.max(1, page);
    const validatedLimit = Math.min(100, Math.max(1, limit)); // Max 100 items

    const filters = {
      category,
      minPrice,
      maxPrice,
    };

    const [products, total] = await this.productsService.findAllWithPagination({
      page: validatedPage,
      limit: validatedLimit,
      sortBy,
      order,
      filters,
    });

    return {
      data: products,
      pagination: {
        page: validatedPage,
        limit: validatedLimit,
        total,
        totalPages: Math.ceil(total / validatedLimit),
      },
    };
  }
}

// Usage example:
// GET /api/products?page=2&limit=20&sortBy=price&order=asc&category=electronics&minPrice=100
```

### Error Handling

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from "@nestjs/common";

interface ErrorResponse {
  statusCode: number;
  timestamp: string;
  path: string;
  message: string;
  errors?: any[];
}

@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse();
    const request = ctx.getRequest();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = "Internal server error";
    let errors = undefined;

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      if (typeof exceptionResponse === "object") {
        message = exceptionResponse["message"] || message;
        errors = exceptionResponse["errors"];
      } else {
        message = exceptionResponse;
      }
    }

    const errorResponse: ErrorResponse = {
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message,
      ...(errors && { errors }),
    };

    response.status(status).json(errorResponse);
  }
}

// Validation error example
import { IsEmail, IsNotEmpty, MinLength } from "class-validator";

export class CreateUserDto {
  @IsNotEmpty({ message: "Name is required" })
  name: string;

  @IsEmail({}, { message: "Invalid email format" })
  email: string;

  @MinLength(8, { message: "Password must be at least 8 characters" })
  password: string;
}

// Custom business logic exception
export class InsufficientFundsException extends HttpException {
  constructor(availableBalance: number, requiredAmount: number) {
    super(
      {
        statusCode: HttpStatus.PAYMENT_REQUIRED,
        message: "Insufficient funds",
        errors: [
          {
            field: "balance",
            available: availableBalance,
            required: requiredAmount,
            shortfall: requiredAmount - availableBalance,
          },
        ],
      },
      HttpStatus.PAYMENT_REQUIRED
    );
  }
}
```

## GraphQL API Design

### Schema Design Best Practices

```typescript
// src/graphql/schema.graphql

# ✅ Good: Clear, descriptive types with proper relationships
type User {
  id: ID!
  email: String!
  name: String!
  role: UserRole!
  createdAt: DateTime!
  updatedAt: DateTime!

  # Relationships
  posts: [Post!]!
  comments: [Comment!]!
  profile: UserProfile
}

enum UserRole {
  ADMIN
  USER
  MODERATOR
}

type Post {
  id: ID!
  title: String!
  content: String!
  published: Boolean!
  author: User!
  comments: [Comment!]!
  tags: [String!]!
  createdAt: DateTime!
  updatedAt: DateTime!
}

# Input types for mutations
input CreatePostInput {
  title: String!
  content: String!
  published: Boolean = false
  tags: [String!]
}

input UpdatePostInput {
  title: String
  content: String
  published: Boolean
  tags: [String!]
}

# Queries
type Query {
  # Single resource
  user(id: ID!): User
  post(id: ID!): Post

  # Collections with pagination
  users(
    page: Int = 1
    limit: Int = 10
    role: UserRole
  ): UserConnection!

  posts(
    page: Int = 1
    limit: Int = 10
    published: Boolean
    authorId: ID
  ): PostConnection!
}

# Pagination types
type UserConnection {
  edges: [UserEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type UserEdge {
  node: User!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

# Mutations
type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!

  createPost(input: CreatePostInput!): Post!
  updatePost(id: ID!, input: UpdatePostInput!): Post!
  deletePost(id: ID!): Boolean!
}

# Subscriptions for real-time updates
type Subscription {
  postCreated: Post!
  postUpdated(id: ID!): Post!
  commentAdded(postId: ID!): Comment!
}

scalar DateTime
```

### GraphQL Resolvers

```typescript
// src/resolvers/user.resolver.ts
import {
  Resolver,
  Query,
  Mutation,
  Args,
  ResolveField,
  Parent,
} from "@nestjs/graphql";
import { UseGuards } from "@nestjs/common";

@Resolver("User")
export class UserResolver {
  constructor(
    private readonly usersService: UsersService,
    private readonly postsService: PostsService
  ) {}

  // Query resolvers
  @Query("user")
  async getUser(@Args("id") id: string) {
    return this.usersService.findOne(id);
  }

  @Query("users")
  async getUsers(
    @Args("page") page: number = 1,
    @Args("limit") limit: number = 10,
    @Args("role") role?: string
  ) {
    const [users, total] = await this.usersService.findAll({
      page,
      limit,
      role,
    });

    return {
      edges: users.map((user) => ({
        node: user,
        cursor: Buffer.from(user.id).toString("base64"),
      })),
      pageInfo: {
        hasNextPage: page * limit < total,
        hasPreviousPage: page > 1,
      },
      totalCount: total,
    };
  }

  // Mutation resolvers
  @Mutation("createUser")
  @UseGuards(AuthGuard)
  async createUser(@Args("input") input: CreateUserInput) {
    return this.usersService.create(input);
  }

  @Mutation("updateUser")
  @UseGuards(AuthGuard)
  async updateUser(
    @Args("id") id: string,
    @Args("input") input: UpdateUserInput
  ) {
    return this.usersService.update(id, input);
  }

  // Field resolvers (resolve relationships)
  @ResolveField("posts")
  async getPosts(@Parent() user: User) {
    return this.postsService.findByAuthor(user.id);
  }

  @ResolveField("profile")
  async getProfile(@Parent() user: User) {
    return this.usersService.getProfile(user.id);
  }
}
```

### DataLoader for N+1 Problem

```typescript
// src/dataloader/user.loader.ts
import * as DataLoader from "dataloader";

export class UserLoader {
  constructor(private readonly usersService: UsersService) {}

  // Batch loading users
  createLoader(): DataLoader<string, User> {
    return new DataLoader<string, User>(async (userIds: string[]) => {
      const users = await this.usersService.findByIds(userIds);

      // Map users to match the order of userIds
      const userMap = new Map(users.map((user) => [user.id, user]));
      return userIds.map((id) => userMap.get(id));
    });
  }
}

// Usage in resolver
@Resolver("Post")
export class PostResolver {
  @ResolveField("author")
  async getAuthor(
    @Parent() post: Post,
    @Context()
    { loaders }: { loaders: { userLoader: DataLoader<string, User> } }
  ) {
    return loaders.userLoader.load(post.authorId);
  }
}
```

## API Versioning Strategies

### 1. URL Path Versioning

```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { RouterModule } from "@nestjs/core";

@Module({
  imports: [
    // Version 1
    UsersV1Module,
    PostsV1Module,

    // Version 2
    UsersV2Module,
    PostsV2Module,

    RouterModule.register([
      {
        path: "api/v1",
        children: [
          { path: "users", module: UsersV1Module },
          { path: "posts", module: PostsV1Module },
        ],
      },
      {
        path: "api/v2",
        children: [
          { path: "users", module: UsersV2Module },
          { path: "posts", module: PostsV2Module },
        ],
      },
    ]),
  ],
})
export class AppModule {}

// Usage:
// GET /api/v1/users - Version 1
// GET /api/v2/users - Version 2
```

### 2. Header Versioning

```typescript
// src/common/decorators/api-version.decorator.ts
import { SetMetadata } from "@nestjs/common";

export const API_VERSION_KEY = "apiVersion";
export const ApiVersion = (version: string) =>
  SetMetadata(API_VERSION_KEY, version);

// src/common/guards/api-version.guard.ts
import { Injectable, CanActivate, ExecutionContext } from "@nestjs/common";
import { Reflector } from "@nestjs/core";

@Injectable()
export class ApiVersionGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredVersion = this.reflector.get<string>(
      API_VERSION_KEY,
      context.getHandler()
    );

    if (!requiredVersion) {
      return true;
    }

    const request = context.switchToHttp().getRequest();
    const apiVersion =
      request.headers["api-version"] || request.headers["x-api-version"];

    return apiVersion === requiredVersion;
  }
}

// Usage in controller
@Controller("api/users")
export class UsersController {
  @Get()
  @ApiVersion("1.0")
  @UseGuards(ApiVersionGuard)
  async findAllV1() {
    // Version 1 implementation
  }

  @Get()
  @ApiVersion("2.0")
  @UseGuards(ApiVersionGuard)
  async findAllV2() {
    // Version 2 implementation with breaking changes
  }
}

// Client usage:
// GET /api/users
// Headers: { 'api-version': '2.0' }
```

### 3. Content Negotiation Versioning

```typescript
// src/common/interceptors/content-negotiation.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { map } from "rxjs/operators";

@Injectable()
export class ContentNegotiationInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const acceptHeader = request.headers["accept"];

    // Parse version from Accept header
    // Accept: application/vnd.myapp.v2+json
    const versionMatch = acceptHeader?.match(/vnd\.myapp\.v(\d+)\+json/);
    const version = versionMatch ? versionMatch[1] : "1";

    request.apiVersion = version;

    return next.handle().pipe(
      map((data) => {
        // Transform data based on version
        if (version === "1") {
          return this.transformV1(data);
        } else if (version === "2") {
          return this.transformV2(data);
        }
        return data;
      })
    );
  }

  private transformV1(data: any) {
    // Legacy format
    return data;
  }

  private transformV2(data: any) {
    // New format with additional fields
    return {
      ...data,
      metadata: {
        version: "v2",
        timestamp: new Date().toISOString(),
      },
    };
  }
}
```

## API Documentation with OpenAPI/Swagger

### Setting Up Swagger

```typescript
// src/main.ts
import { NestFactory } from "@nestjs/core";
import { SwaggerModule, DocumentBuilder } from "@nestjs/swagger";
import { ValidationPipe } from "@nestjs/common";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Swagger configuration
  const config = new DocumentBuilder()
    .setTitle("My API")
    .setDescription("Comprehensive API documentation")
    .setVersion("1.0")
    .addBearerAuth(
      {
        type: "http",
        scheme: "bearer",
        bearerFormat: "JWT",
      },
      "JWT-auth"
    )
    .addTag("users", "User management endpoints")
    .addTag("posts", "Blog post endpoints")
    .addServer("https://api.production.com", "Production")
    .addServer("https://api.staging.com", "Staging")
    .addServer("http://localhost:3000", "Development")
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup("api/docs", app, document, {
    customSiteTitle: "API Documentation",
    customCss: ".swagger-ui .topbar { display: none }",
  });

  app.useGlobalPipes(new ValidationPipe());

  await app.listen(3000);
  console.log(`API Documentation: http://localhost:3000/api/docs`);
}
bootstrap();
```

### Documenting Endpoints

```typescript
// src/controllers/users.controller.ts
import { Controller, Get, Post, Body, Param } from "@nestjs/common";
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiParam,
  ApiBody,
  ApiBearerAuth,
} from "@nestjs/swagger";

@ApiTags("users")
@Controller("api/users")
export class UsersController {
  @Get()
  @ApiOperation({
    summary: "Get all users",
    description: "Retrieve a paginated list of users with optional filters",
  })
  @ApiResponse({
    status: 200,
    description: "Successfully retrieved users",
    schema: {
      example: {
        data: [
          {
            id: "1",
            email: "user@example.com",
            name: "John Doe",
            role: "USER",
          },
        ],
        pagination: {
          page: 1,
          limit: 10,
          total: 100,
          totalPages: 10,
        },
      },
    },
  })
  async findAll() {
    // Implementation
  }

  @Get(":id")
  @ApiOperation({ summary: "Get user by ID" })
  @ApiParam({
    name: "id",
    description: "User unique identifier",
    example: "123e4567-e89b-12d3-a456-426614174000",
  })
  @ApiResponse({ status: 200, description: "User found" })
  @ApiResponse({ status: 404, description: "User not found" })
  async findOne(@Param("id") id: string) {
    // Implementation
  }

  @Post()
  @ApiBearerAuth("JWT-auth")
  @ApiOperation({ summary: "Create new user" })
  @ApiBody({
    description: "User creation data",
    schema: {
      example: {
        email: "newuser@example.com",
        name: "Jane Doe",
        password: "SecurePass123!",
      },
    },
  })
  @ApiResponse({ status: 201, description: "User created successfully" })
  @ApiResponse({ status: 400, description: "Invalid input" })
  @ApiResponse({ status: 401, description: "Unauthorized" })
  async create(@Body() createUserDto: CreateUserDto) {
    // Implementation
  }
}

// src/dto/create-user.dto.ts
import { ApiProperty } from "@nestjs/swagger";
import { IsEmail, IsNotEmpty, MinLength } from "class-validator";

export class CreateUserDto {
  @ApiProperty({
    description: "User email address",
    example: "user@example.com",
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: "User full name",
    example: "John Doe",
  })
  @IsNotEmpty()
  name: string;

  @ApiProperty({
    description: "User password (min 8 characters)",
    example: "SecurePass123!",
    minLength: 8,
  })
  @MinLength(8)
  password: string;
}
```

## Rate Limiting and Throttling

```typescript
// src/common/guards/throttle.guard.ts
import { Injectable } from "@nestjs/common";
import { ThrottlerGuard } from "@nestjs/throttler";

@Injectable()
export class CustomThrottlerGuard extends ThrottlerGuard {
  protected async getTracker(req: Record<string, any>): Promise<string> {
    // Track by user ID if authenticated, otherwise by IP
    return req.user?.id || req.ip;
  }
}

// src/app.module.ts
import { Module } from "@nestjs/common";
import { ThrottlerModule } from "@nestjs/throttler";
import { APP_GUARD } from "@nestjs/core";

@Module({
  imports: [
    ThrottlerModule.forRoot({
      ttl: 60, // Time window in seconds
      limit: 10, // Max requests per ttl
    }),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: CustomThrottlerGuard,
    },
  ],
})
export class AppModule {}

// Per-endpoint throttling
import { Throttle } from "@nestjs/throttler";

@Controller("api/auth")
export class AuthController {
  @Post("login")
  @Throttle(5, 60) // 5 requests per minute
  async login(@Body() credentials: LoginDto) {
    // Implementation
  }

  @Post("register")
  @Throttle(3, 3600) // 3 registrations per hour
  async register(@Body() userData: RegisterDto) {
    // Implementation
  }
}
```

## Best Practices

### 1. **Consistent Response Format**

```typescript
interface ApiResponse<T> {
  statusCode: number;
  message?: string;
  data?: T;
  errors?: any[];
  metadata?: {
    timestamp: string;
    version: string;
  };
}
```

### 2. **Use DTOs for Validation**

Always validate input using Data Transfer Objects with class-validator.

### 3. **Implement HATEOAS (for REST)**

```typescript
{
  "id": "123",
  "name": "Product",
  "_links": {
    "self": { "href": "/api/products/123" },
    "reviews": { "href": "/api/products/123/reviews" },
    "related": { "href": "/api/products/123/related" }
  }
}
```

### 4. **Version Early**

Plan for API evolution from the start.

### 5. **Rate Limit All Endpoints**

Protect your API from abuse and ensure fair usage.

### 6. **Document Everything**

Keep OpenAPI/Swagger documentation up-to-date.

### 7. **Use Consistent Naming**

- REST: Plural nouns (`/users`, `/posts`)
- GraphQL: Descriptive queries (`getUser`, `listPosts`)

### 8. **Implement Proper Error Codes**

- 200: Success
- 201: Created
- 400: Bad Request
- 401: Unauthorized
- 403: Forbidden
- 404: Not Found
- 422: Unprocessable Entity
- 429: Too Many Requests
- 500: Internal Server Error

## Summary

Well-designed APIs are crucial for successful applications. Focus on:

- Clear, resource-based URL structures
- Proper HTTP methods and status codes
- Comprehensive error handling
- Thoughtful versioning strategy
- Complete documentation
- Rate limiting and security
- Consistent response formats

Following these best practices ensures your APIs are intuitive, maintainable, and scalable for years to come.
