# ORMs: Prisma & TypeORM

## Overview

Object-Relational Mapping (ORM) tools bridge the gap between object-oriented programming and relational databases, allowing you to interact with databases using your programming language's syntax instead of raw SQL. This tutorial covers two popular TypeScript ORMs: Prisma and TypeORM.

**Key Features:**

- **Type Safety**: Auto-generated types from schema
- **Migrations**: Database schema version control
- **Query Builder**: Fluent API for building queries
- **Relations**: Easy handling of table relationships
- **Transactions**: ACID-compliant operations

## Practical Use Cases

### 1. **Rapid Development**

Build features faster

- CRUD operations
- Prototyping
- MVP development
- Startups

### 2. **Type-Safe Database Access**

Prevent runtime errors

- TypeScript projects
- Enterprise applications
- Large teams
- Complex schemas

### 3. **Database Migrations**

Schema version control

- Team collaboration
- Deployment automation
- Schema evolution
- Rollback capability

### 4. **Multi-Database Support**

Database agnostic code

- PostgreSQL, MySQL, SQLite
- Easy database switching
- Testing with SQLite
- Production with PostgreSQL

### 5. **Complex Relationships**

Navigate related data easily

- Eager/lazy loading
- Deep nested queries
- Aggregations
- Joins

## Prisma

### 1. Installation & Setup

```bash
npm install prisma @prisma/client
# Prisma is framework-agnostic, no special Express package needed

# Initialize Prisma
npx prisma init

# This creates:
# - prisma/schema.prisma
# - .env file
```

### 2. Schema Definition

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
  id        String   @id @default(uuid())
  email     String   @unique
  username  String   @unique
  password  String
  role      Role     @default(USER)
  profile   Profile?
  posts     Post[]
  comments  Comment[]
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?

  @@index([email])
  @@index([username])
  @@map("users")
}

model Profile {
  id          String   @id @default(uuid())
  userId      String   @unique
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  firstName   String?
  lastName    String?
  bio         String?  @db.Text
  avatar      String?
  dateOfBirth DateTime?
  phone       String?

  @@map("profiles")
}

model Post {
  id          String    @id @default(uuid())
  title       String
  slug        String    @unique
  content     String    @db.Text
  published   Boolean   @default(false)
  authorId    String
  author      User      @relation(fields: [authorId], references: [id], onDelete: Cascade)
  categoryId  String?
  category    Category? @relation(fields: [categoryId], references: [id], onDelete: SetNull)
  tags        Tag[]     @relation("PostTags")
  comments    Comment[]
  viewCount   Int       @default(0)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  publishedAt DateTime?

  @@index([authorId])
  @@index([categoryId])
  @@index([slug])
  @@map("posts")
}

model Category {
  id          String     @id @default(uuid())
  name        String
  slug        String     @unique
  parentId    String?
  parent      Category?  @relation("CategoryHierarchy", fields: [parentId], references: [id], onDelete: Cascade)
  children    Category[] @relation("CategoryHierarchy")
  posts       Post[]
  createdAt   DateTime   @default(now())

  @@map("categories")
}

model Tag {
  id    String @id @default(uuid())
  name  String @unique
  slug  String @unique
  posts Post[] @relation("PostTags")

  @@map("tags")
}

model Comment {
  id        String   @id @default(uuid())
  content   String   @db.Text
  postId    String
  post      Post     @relation(fields: [postId], references: [id], onDelete: Cascade)
  authorId  String
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)
  parentId  String?
  parent    Comment? @relation("CommentReplies", fields: [parentId], references: [id], onDelete: Cascade)
  replies   Comment[] @relation("CommentReplies")
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@index([postId])
  @@index([authorId])
  @@map("comments")
}

enum Role {
  USER
  ADMIN
  MODERATOR
}
```

### 3. Generate Client & Run Migrations

```bash
# Generate Prisma Client
npx prisma generate

# Create migration
npx prisma migrate dev --name init

# Apply migrations in production
npx prisma migrate deploy

# Reset database (development only)
npx prisma migrate reset

# View database in browser
npx prisma studio
```

### 4. Express Integration

```typescript
// src/database/prisma.service.ts
import { PrismaClient } from "@prisma/client";

export class PrismaService extends PrismaClient {
  constructor() {
    super({
      log:
        process.env.NODE_ENV === "development"
          ? ["query", "error", "warn"]
          : ["error"],
    });
  }

  async connect() {
    await this.$connect();
    console.log("✓ Database connected");
  }

  async disconnect() {
    await this.$disconnect();
    console.log("✓ Database disconnected");
  }

  // Custom methods
  async cleanDatabase() {
    if (process.env.NODE_ENV === "production") {
      throw new Error("Cannot clean production database");
    }

    const models = Reflect.ownKeys(this).filter((key) => key[0] !== "_");
    return Promise.all(
      models.map((modelKey) => (this as any)[modelKey].deleteMany())
    );
  }
}

// Singleton instance
export const prisma = new PrismaService();

// Graceful shutdown
process.on("SIGINT", async () => {
  await prisma.disconnect();
  process.exit(0);
});

process.on("SIGTERM", async () => {
  await prisma.disconnect();
  process.exit(0);
});
```

### 5. Prisma CRUD Operations

```typescript
// src/services/user.service.ts
import { prisma } from "../database/prisma.service";
import { User, Prisma } from "@prisma/client";

export class UserService {
  private prisma = prisma;

  // Create
  async create(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
      include: {
        profile: true,
      },
    });
  }

  // Create with nested relations
  async createWithProfile(
    email: string,
    username: string,
    password: string,
    firstName: string,
    lastName: string
  ): Promise<User> {
    return this.prisma.user.create({
      data: {
        email,
        username,
        password,
        profile: {
          create: {
            firstName,
            lastName,
          },
        },
      },
      include: {
        profile: true,
      },
    });
  }

  // Read - Find one
  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id },
      include: {
        profile: true,
        posts: {
          where: { published: true },
          take: 10,
          orderBy: { createdAt: "desc" },
        },
      },
    });
  }

  // Read - Find many with filters
  async findAll(params: {
    skip?: number;
    take?: number;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
  }): Promise<User[]> {
    const { skip, take, where, orderBy } = params;

    return this.prisma.user.findMany({
      skip,
      take,
      where,
      orderBy,
      include: {
        profile: true,
      },
    });
  }

  // Update
  async update(id: string, data: Prisma.UserUpdateInput): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data,
      include: {
        profile: true,
      },
    });
  }

  // Soft delete
  async softDelete(id: string): Promise<User> {
    return this.prisma.user.update({
      where: { id },
      data: {
        deletedAt: new Date(),
      },
    });
  }

  // Hard delete
  async delete(id: string): Promise<User> {
    return this.prisma.user.delete({
      where: { id },
    });
  }

  // Complex queries
  async getUserStatistics() {
    const [totalUsers, activeUsers, adminCount] = await Promise.all([
      this.prisma.user.count(),
      this.prisma.user.count({
        where: {
          deletedAt: null,
          posts: {
            some: {
              createdAt: {
                gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000),
              },
            },
          },
        },
      }),
      this.prisma.user.count({
        where: { role: "ADMIN" },
      }),
    ]);

    return { totalUsers, activeUsers, adminCount };
  }

  // Aggregations
  async getPostStatsByUser(userId: string) {
    return this.prisma.post.aggregate({
      where: { authorId: userId },
      _count: { id: true },
      _sum: { viewCount: true },
      _avg: { viewCount: true },
    });
  }

  // Raw SQL (when needed)
  async getUsersWithPostCount() {
    return this.prisma.$queryRaw<
      Array<{ username: string; postCount: bigint }>
    >`
      SELECT u.username, COUNT(p.id) as "postCount"
      FROM users u
      LEFT JOIN posts p ON u.id = p."authorId"
      GROUP BY u.id, u.username
      ORDER BY "postCount" DESC
      LIMIT 10
    `;
  }
}
```

### 6. Prisma Transactions

```typescript
// Multiple operations in a transaction
async createPostWithTags(userId: string, title: string, content: string, tagNames: string[]) {
  return this.prisma.$transaction(async (tx) => {
    // Create post
    const post = await tx.post.create({
      data: {
        title,
        content,
        slug: this.generateSlug(title),
        authorId: userId,
      },
    });

    // Create or connect tags
    const tags = await Promise.all(
      tagNames.map(name =>
        tx.tag.upsert({
          where: { name },
          create: { name, slug: this.generateSlug(name) },
          update: {},
        })
      )
    );

    // Connect tags to post
    await tx.post.update({
      where: { id: post.id },
      data: {
        tags: {
          connect: tags.map(tag => ({ id: tag.id })),
        },
      },
    });

    return post;
  });
}

// Interactive transactions
async transferCredits(fromUserId: string, toUserId: string, amount: number) {
  return this.prisma.$transaction(async (tx) => {
    // Deduct from sender
    const sender = await tx.user.update({
      where: { id: fromUserId },
      data: {
        credits: {
          decrement: amount,
        },
      },
    });

    if (sender.credits < 0) {
      throw new Error('Insufficient credits');
    }

    // Add to receiver
    await tx.user.update({
      where: { id: toUserId },
      data: {
        credits: {
          increment: amount,
        },
      },
    });
  });
}
```

## TypeORM

### 1. Installation & Setup

```bash
npm install typeorm reflect-metadata pg

# For migrations
npm install -D ts-node
```

```typescript
// src/database/data-source.ts
import "reflect-metadata";
import { DataSource } from "typeorm";
import * as dotenv from "dotenv";

dotenv.config();

export const AppDataSource = new DataSource({
  type: "postgres",
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT || "5432"),
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
  entities: [__dirname + "/../entities/**/*.entity{.ts,.js}"],
  synchronize: false, // Never true in production!
  logging: process.env.NODE_ENV === "development",
  migrations: [__dirname + "/migrations/**/*{.ts,.js}"],
  subscribers: [],
});

// Initialize connection
export async function initializeDatabase() {
  try {
    await AppDataSource.initialize();
    console.log("✓ Database connection established");
  } catch (error) {
    console.error("Database connection failed:", error);
    process.exit(1);
  }
}

// Graceful shutdown
process.on("SIGINT", async () => {
  await AppDataSource.destroy();
  process.exit(0);
});

process.on("SIGTERM", async () => {
  await AppDataSource.destroy();
  process.exit(0);
});
```

### 2. Entity Definition

```typescript
// entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  OneToOne,
  OneToMany,
  CreateDateColumn,
  UpdateDateColumn,
  DeleteDateColumn,
  Index,
  JoinColumn,
} from "typeorm";
import { Profile } from "./profile.entity";
import { Post } from "./post.entity";
import { Comment } from "./comment.entity";

export enum UserRole {
  USER = "user",
  ADMIN = "admin",
  MODERATOR = "moderator",
}

@Entity("users")
@Index(["email"])
@Index(["username"])
export class User {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column({ unique: true })
  email: string;

  @Column({ unique: true, length: 50 })
  username: string;

  @Column()
  password: string;

  @Column({
    type: "enum",
    enum: UserRole,
    default: UserRole.USER,
  })
  role: UserRole;

  @OneToOne(() => Profile, (profile) => profile.user, { cascade: true })
  profile: Profile;

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @OneToMany(() => Comment, (comment) => comment.author)
  comments: Comment[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @DeleteDateColumn()
  deletedAt: Date;
}
```

```typescript
// entities/profile.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  OneToOne,
  JoinColumn,
} from "typeorm";
import { User } from "./user.entity";

@Entity("profiles")
export class Profile {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column()
  userId: string;

  @OneToOne(() => User, (user) => user.profile, { onDelete: "CASCADE" })
  @JoinColumn()
  user: User;

  @Column({ nullable: true })
  firstName: string;

  @Column({ nullable: true })
  lastName: string;

  @Column({ type: "text", nullable: true })
  bio: string;

  @Column({ nullable: true })
  avatar: string;

  @Column({ type: "date", nullable: true })
  dateOfBirth: Date;

  @Column({ nullable: true, length: 20 })
  phone: string;
}
```

```typescript
// entities/post.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  ManyToOne,
  ManyToMany,
  OneToMany,
  JoinTable,
  Index,
  CreateDateColumn,
  UpdateDateColumn,
} from "typeorm";
import { User } from "./user.entity";
import { Category } from "./category.entity";
import { Tag } from "./tag.entity";
import { Comment } from "./comment.entity";

@Entity("posts")
@Index(["slug"], { unique: true })
@Index(["authorId"])
export class Post {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column()
  title: string;

  @Column({ unique: true })
  slug: string;

  @Column({ type: "text" })
  content: string;

  @Column({ default: false })
  published: boolean;

  @Column()
  authorId: string;

  @ManyToOne(() => User, (user) => user.posts, { onDelete: "CASCADE" })
  author: User;

  @Column({ nullable: true })
  categoryId: string;

  @ManyToOne(() => Category, (category) => category.posts, {
    onDelete: "SET NULL",
  })
  category: Category;

  @ManyToMany(() => Tag, (tag) => tag.posts)
  @JoinTable({ name: "post_tags" })
  tags: Tag[];

  @OneToMany(() => Comment, (comment) => comment.post)
  comments: Comment[];

  @Column({ default: 0 })
  viewCount: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @Column({ type: "timestamp", nullable: true })
  publishedAt: Date;
}
```

### 3. TypeORM Migrations

```bash
# Generate migration from entities
npx typeorm migration:generate -d src/data-source.ts -n InitialMigration

# Create empty migration
npx typeorm migration:create src/migrations/AddIndexes

# Run migrations
npx typeorm migration:run -d src/data-source.ts

# Revert last migration
npx typeorm migration:revert -d src/data-source.ts
```

```typescript
// migrations/1705123456789-InitialMigration.ts
import { MigrationInterface, QueryRunner } from "typeorm";

export class InitialMigration1705123456789 implements MigrationInterface {
  name = "InitialMigration1705123456789";

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE "users" (
        "id" uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
        "email" varchar NOT NULL UNIQUE,
        "username" varchar(50) NOT NULL UNIQUE,
        "password" varchar NOT NULL,
        "role" varchar NOT NULL DEFAULT 'user',
        "createdAt" TIMESTAMP NOT NULL DEFAULT now(),
        "updatedAt" TIMESTAMP NOT NULL DEFAULT now(),
        "deletedAt" TIMESTAMP
      )
    `);

    await queryRunner.query(`
      CREATE INDEX "IDX_users_email" ON "users" ("email")
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP INDEX "IDX_users_email"`);
    await queryRunner.query(`DROP TABLE "users"`);
  }
}
```

### 4. TypeORM CRUD Operations

```typescript
// src/services/user.service.ts
import { Repository, In, Like, MoreThan } from "typeorm";
import { AppDataSource } from "../database/data-source";
import { User } from "../entities/user.entity";

export class UserService {
  private userRepository: Repository<User>;

  constructor() {
    this.userRepository = AppDataSource.getRepository(User);
  }

  // Create
  async create(
    email: string,
    username: string,
    password: string
  ): Promise<User> {
    const user = this.userRepository.create({
      email,
      username,
      password,
    });
    return this.userRepository.save(user);
  }

  // Read
  async findById(id: string): Promise<User | null> {
    return this.userRepository.findOne({
      where: { id },
      relations: ["profile", "posts"],
    });
  }

  // Find with conditions
  async findAll(
    skip: number = 0,
    take: number = 10
  ): Promise<[User[], number]> {
    return this.userRepository.findAndCount({
      where: { deletedAt: null },
      relations: ["profile"],
      skip,
      take,
      order: { createdAt: "DESC" },
    });
  }

  // Query Builder
  async searchUsers(query: string): Promise<User[]> {
    return this.userRepository
      .createQueryBuilder("user")
      .leftJoinAndSelect("user.profile", "profile")
      .where("user.email ILIKE :query", { query: `%${query}%` })
      .orWhere("user.username ILIKE :query", { query: `%${query}%` })
      .andWhere("user.deletedAt IS NULL")
      .take(20)
      .getMany();
  }

  // Update
  async update(id: string, data: Partial<User>): Promise<User> {
    await this.userRepository.update(id, data);
    return this.findById(id);
  }

  // Soft delete
  async softDelete(id: string): Promise<void> {
    await this.userRepository.softDelete(id);
  }

  // Hard delete
  async delete(id: string): Promise<void> {
    await this.userRepository.delete(id);
  }

  // Transactions
  async createUserWithProfile(
    email: string,
    username: string,
    password: string,
    firstName: string,
    lastName: string
  ): Promise<User> {
    return this.userRepository.manager.transaction(async (manager) => {
      const user = manager.create(User, {
        email,
        username,
        password,
      });

      const savedUser = await manager.save(user);

      const profile = manager.create(Profile, {
        userId: savedUser.id,
        firstName,
        lastName,
      });

      await manager.save(profile);

      return manager.findOne(User, {
        where: { id: savedUser.id },
        relations: ["profile"],
      });
    });
  }

  // Raw queries
  async getUsersWithPostCount() {
    return this.userRepository.query(`
      SELECT u.id, u.username, COUNT(p.id) as "postCount"
      FROM users u
      LEFT JOIN posts p ON u.id = p."authorId"
      GROUP BY u.id, u.username
      ORDER BY "postCount" DESC
      LIMIT 10
    `);
  }
}
```

## Comparison: Prisma vs TypeORM

| Feature               | Prisma                      | TypeORM               |
| --------------------- | --------------------------- | --------------------- |
| **Type Safety**       | Excellent (generated types) | Good (decorators)     |
| **Learning Curve**    | Easy                        | Moderate              |
| **Schema Definition** | Prisma Schema Language      | TypeScript decorators |
| **Migrations**        | Automatic + manual          | Manual generation     |
| **Query Builder**     | Simple, intuitive           | Powerful, flexible    |
| **Raw SQL**           | Supported                   | Supported             |
| **Performance**       | Very fast                   | Fast                  |
| **Community**         | Growing rapidly             | Mature, large         |
| **MongoDB Support**   | Yes                         | Yes                   |
| **Active Record**     | No                          | Yes (optional)        |

## Best Practices

### 1. **Never Use synchronize in Production**

```typescript
// ❌ Bad
TypeOrmModule.forRoot({
  synchronize: true, // NEVER in production!
});

// ✅ Good
TypeOrmModule.forRoot({
  synchronize: false,
  migrations: [...],
});
```

### 2. **Use Transactions for Related Operations**

```typescript
// ✅ Good: Atomic operations
await this.prisma.$transaction([
  this.prisma.user.create({ data: userData }),
  this.prisma.profile.create({ data: profileData }),
]);
```

### 3. **Avoid N+1 Queries**

```typescript
// ❌ Bad: N+1 problem
const users = await this.userRepository.find();
for (const user of users) {
  const posts = await this.postRepository.find({
    where: { authorId: user.id },
  });
}

// ✅ Good: Eager loading
const users = await this.userRepository.find({
  relations: ["posts"],
});
```

### 4. **Use Indexes**

```typescript
// Prisma
@@index([email])
@@index([username, createdAt])

// TypeORM
@Index(['email'])
@Index(['username', 'createdAt'])
```

### 5. **Handle Soft Deletes**

```typescript
// Prisma: Custom implementation
where: { deletedAt: null }

// TypeORM: Built-in
@DeleteDateColumn()
deletedAt: Date;
```

## Key Takeaways

✅ **ORMs provide type-safe database access**  
✅ **Prisma offers superior developer experience with generated types**  
✅ **TypeORM provides more flexibility and control**  
✅ **Use migrations for schema changes, not synchronize**  
✅ **Transactions ensure data consistency**  
✅ **Eager loading prevents N+1 query problems**  
✅ **Indexes are still crucial for performance**  
✅ **Raw SQL available when ORM limitations arise**  
✅ **Choose ORM based on project needs and team preference**  
✅ **Test migrations thoroughly before production deployment**

ORMs significantly improve development speed and code maintainability while maintaining type safety in TypeScript applications.
