# PostgreSQL Deep Dive

## Overview

PostgreSQL is a powerful, open-source relational database management system (RDBMS) known for its reliability, feature robustness, and performance. It supports advanced data types, complex queries, and extensive indexing options, making it ideal for enterprise applications.

**Key Features:**

- **ACID Compliance**: Atomicity, Consistency, Isolation, Durability
- **Advanced Data Types**: JSON/JSONB, Arrays, UUID, Custom types
- **Full-Text Search**: Built-in text search capabilities
- **Extensions**: PostGIS (geospatial), pg_stat_statements, pg_trgm
- **Window Functions**: Advanced analytical queries
- **CTEs**: Common Table Expressions for complex queries

## Practical Use Cases

### 1. **Enterprise Applications**

Mission-critical business systems

- ERP systems
- CRM platforms
- Financial applications
- Inventory management

### 2. **Web Applications**

High-traffic websites and APIs

- E-commerce platforms
- Social networks
- Content management systems
- SaaS applications

### 3. **Data Analytics**

Business intelligence and reporting

- Data warehousing
- Real-time analytics
- Aggregation pipelines
- Time-series data

### 4. **Geospatial Applications**

Location-based services

- Mapping applications
- Route planning
- Location tracking
- Geographic analysis (with PostGIS)

### 5. **Multi-tenant Systems**

SaaS applications with tenant isolation

- Row-level security
- Schema-per-tenant
- Shared database with tenant ID

## Step-by-Step Implementation

### 1. Install PostgreSQL

```bash
# macOS
brew install postgresql@15
brew services start postgresql@15

# Ubuntu/Debian
sudo apt update
sudo apt install postgresql postgresql-contrib

# Start service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Connect to PostgreSQL
psql -U postgres
```

### 2. Database Setup & Configuration

```sql
-- Create database
CREATE DATABASE myapp_production;

-- Create user with password
CREATE USER myapp_user WITH ENCRYPTED PASSWORD 'secure_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE myapp_production TO myapp_user;

-- Connect to database
\c myapp_production

-- Grant schema privileges
GRANT ALL ON SCHEMA public TO myapp_user;

-- Create extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm"; -- For fuzzy text search
CREATE EXTENSION IF NOT EXISTS "btree_gin"; -- For composite indexes
```

### 3. Advanced Table Design

```sql
-- Users table with advanced features
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  profile JSONB DEFAULT '{}'::jsonb,
  roles TEXT[] DEFAULT ARRAY['user']::TEXT[],
  preferences JSONB DEFAULT '{}'::jsonb,
  last_login_at TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  deleted_at TIMESTAMP WITH TIME ZONE, -- Soft delete

  -- Constraints
  CONSTRAINT email_format CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
  CONSTRAINT username_length CHECK (char_length(username) >= 3)
);

-- Products table with full-text search
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
  stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
  category VARCHAR(100),
  tags TEXT[],
  metadata JSONB DEFAULT '{}'::jsonb,
  search_vector tsvector, -- For full-text search
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Orders table with constraints
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  order_number VARCHAR(50) UNIQUE NOT NULL,
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  status VARCHAR(50) NOT NULL DEFAULT 'pending',
  total_amount DECIMAL(10, 2) NOT NULL CHECK (total_amount >= 0),
  shipping_address JSONB NOT NULL,
  items JSONB NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

  CONSTRAINT valid_status CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled'))
);

-- Order items table (normalized)
CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id INTEGER NOT NULL REFERENCES products(id),
  quantity INTEGER NOT NULL CHECK (quantity > 0),
  price_at_purchase DECIMAL(10, 2) NOT NULL CHECK (price_at_purchase >= 0),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,

  UNIQUE(order_id, product_id)
);
```

### 4. Indexes for Performance

```sql
-- B-tree indexes (default, for equality and range queries)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Partial index (for soft deletes)
CREATE INDEX idx_users_active ON users(id) WHERE deleted_at IS NULL;

-- Composite index
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- GIN index for JSONB
CREATE INDEX idx_users_profile ON users USING GIN (profile);
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- GIN index for array columns
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_users_roles ON users USING GIN (roles);

-- Full-text search index
CREATE INDEX idx_products_search ON products USING GIN (search_vector);

-- Unique partial index
CREATE UNIQUE INDEX idx_users_username_active
  ON users(username)
  WHERE deleted_at IS NULL;
```

### 5. Triggers & Functions

```sql
-- Update timestamp trigger function
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = CURRENT_TIMESTAMP;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply trigger to tables
CREATE TRIGGER update_users_updated_at
  BEFORE UPDATE ON users
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_products_updated_at
  BEFORE UPDATE ON products
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

-- Full-text search trigger
CREATE OR REPLACE FUNCTION update_search_vector()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', COALESCE(NEW.name, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_products_search_vector
  BEFORE INSERT OR UPDATE ON products
  FOR EACH ROW
  EXECUTE FUNCTION update_search_vector();

-- Prevent stock from going negative
CREATE OR REPLACE FUNCTION check_product_stock()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.stock < 0 THEN
    RAISE EXCEPTION 'Stock cannot be negative';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER validate_product_stock
  BEFORE UPDATE ON products
  FOR EACH ROW
  WHEN (OLD.stock IS DISTINCT FROM NEW.stock)
  EXECUTE FUNCTION check_product_stock();
```

### 6. Advanced Queries

```sql
-- Full-text search with ranking
SELECT
  id,
  name,
  description,
  ts_rank(search_vector, query) AS rank
FROM
  products,
  to_tsquery('english', 'laptop & gaming') AS query
WHERE
  search_vector @@ query
ORDER BY
  rank DESC
LIMIT 10;

-- JSON queries
-- Find users with specific role
SELECT id, email, roles
FROM users
WHERE roles @> ARRAY['admin']::TEXT[];

-- Query JSONB fields
SELECT id, email, profile->>'city' AS city
FROM users
WHERE profile->>'country' = 'USA';

-- JSONB containment
SELECT id, name
FROM products
WHERE metadata @> '{"featured": true}';

-- Window functions - Running total
SELECT
  id,
  order_number,
  total_amount,
  SUM(total_amount) OVER (
    ORDER BY created_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS running_total
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY created_at;

-- Rank products by sales
SELECT
  p.id,
  p.name,
  COUNT(oi.id) AS times_sold,
  SUM(oi.quantity) AS total_quantity,
  RANK() OVER (ORDER BY SUM(oi.quantity) DESC) AS sales_rank
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id, p.name
ORDER BY sales_rank;

-- Common Table Expressions (CTEs)
WITH monthly_sales AS (
  SELECT
    DATE_TRUNC('month', created_at) AS month,
    SUM(total_amount) AS revenue,
    COUNT(*) AS order_count
  FROM orders
  WHERE status = 'delivered'
  GROUP BY DATE_TRUNC('month', created_at)
),
sales_growth AS (
  SELECT
    month,
    revenue,
    order_count,
    LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
    revenue - LAG(revenue) OVER (ORDER BY month) AS revenue_growth
  FROM monthly_sales
)
SELECT
  month,
  revenue,
  order_count,
  ROUND((revenue_growth / NULLIF(prev_month_revenue, 0) * 100), 2) AS growth_percentage
FROM sales_growth
ORDER BY month DESC;
```

### 7. Transactions & Concurrency

```sql
-- Basic transaction
BEGIN;

-- Deduct stock
UPDATE products
SET stock = stock - 5
WHERE id = 123 AND stock >= 5;

-- Create order
INSERT INTO orders (order_number, user_id, status, total_amount)
VALUES ('ORD-2024-001', 'user-uuid', 'pending', 99.99)
RETURNING id;

COMMIT;
-- Or ROLLBACK on error

-- Transaction with isolation level
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Perform operations
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;

-- Optimistic locking with version
UPDATE products
SET
  stock = stock - 1,
  version = version + 1
WHERE
  id = 123
  AND version = 5  -- Check version hasn't changed
  AND stock >= 1;

-- Row-level locking
BEGIN;

SELECT * FROM products
WHERE id = 123
FOR UPDATE; -- Exclusive lock

-- Perform updates
UPDATE products SET stock = stock - 1 WHERE id = 123;

COMMIT;

-- Advisory locks (application-level)
SELECT pg_advisory_lock(123); -- Acquire lock
-- Perform operations
SELECT pg_advisory_unlock(123); -- Release lock
```

### 8. PostgreSQL with TypeORM (NestJS)

```typescript
// entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  DeleteDateColumn,
  Index,
} from "typeorm";

@Entity("users")
@Index(["email"], { unique: true, where: "deleted_at IS NULL" })
export class User {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column({ type: "varchar", length: 255, unique: true })
  @Index()
  email: string;

  @Column({ type: "varchar", length: 50 })
  username: string;

  @Column({ type: "varchar", length: 255 })
  password_hash: string;

  @Column({ type: "jsonb", default: {} })
  profile: Record<string, any>;

  @Column({ type: "text", array: true, default: ["user"] })
  roles: string[];

  @Column({ type: "jsonb", default: {} })
  preferences: Record<string, any>;

  @Column({ type: "timestamp with time zone", nullable: true })
  last_login_at: Date;

  @CreateDateColumn({ type: "timestamp with time zone" })
  created_at: Date;

  @UpdateDateColumn({ type: "timestamp with time zone" })
  updated_at: Date;

  @DeleteDateColumn({ type: "timestamp with time zone" })
  deleted_at: Date;
}
```

```typescript
// user.repository.ts
import { Injectable } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository, ILike } from "typeorm";
import { User } from "./entities/user.entity";

@Injectable()
export class UserRepository {
  constructor(
    @InjectRepository(User)
    private readonly repository: Repository<User>
  ) {}

  async findByEmail(email: string): Promise<User | null> {
    return this.repository.findOne({
      where: { email },
    });
  }

  async findActiveUsers(limit: number = 100): Promise<User[]> {
    return this.repository.find({
      where: { deleted_at: null },
      order: { created_at: "DESC" },
      take: limit,
    });
  }

  async searchUsers(query: string): Promise<User[]> {
    return this.repository.find({
      where: [
        { email: ILike(`%${query}%`) },
        { username: ILike(`%${query}%`) },
      ],
    });
  }

  async findUsersWithRole(role: string): Promise<User[]> {
    return this.repository
      .createQueryBuilder("user")
      .where(":role = ANY(user.roles)", { role })
      .getMany();
  }

  async updateLastLogin(userId: string): Promise<void> {
    await this.repository.update(userId, {
      last_login_at: new Date(),
    });
  }

  async getUserStatistics() {
    return this.repository
      .createQueryBuilder("user")
      .select([
        "COUNT(*) as total_users",
        "COUNT(*) FILTER (WHERE created_at > CURRENT_DATE - INTERVAL '30 days') as new_users_30d",
        "COUNT(*) FILTER (WHERE last_login_at > CURRENT_DATE - INTERVAL '7 days') as active_users_7d",
      ])
      .getRawOne();
  }

  async softDelete(userId: string): Promise<void> {
    await this.repository.softDelete(userId);
  }
}
```

### 9. Database Backup & Restore

```bash
# Backup entire database
pg_dump -U myapp_user -d myapp_production -F c -f backup.dump

# Backup with compression
pg_dump -U myapp_user -d myapp_production -F c -Z 9 -f backup.dump

# Backup specific tables
pg_dump -U myapp_user -d myapp_production -t users -t orders -f tables_backup.sql

# Backup with plain SQL
pg_dump -U myapp_user -d myapp_production -f backup.sql

# Restore from custom format
pg_restore -U myapp_user -d myapp_production -c -v backup.dump

# Restore from SQL file
psql -U myapp_user -d myapp_production -f backup.sql

# Automated backup script
#!/bin/bash
BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
FILENAME="backup_$DATE.dump"

pg_dump -U myapp_user -d myapp_production -F c -f "$BACKUP_DIR/$FILENAME"

# Keep only last 7 days of backups
find $BACKUP_DIR -name "backup_*.dump" -mtime +7 -delete
```

## Best Practices

### 1. **Connection Pooling**

```typescript
// TypeORM configuration
{
  type: 'postgres',
  host: process.env.DB_HOST,
  port: 5432,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  extra: {
    max: 20, // Maximum connections in pool
    min: 5,  // Minimum connections
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
  },
}
```

### 2. **Use Prepared Statements**

Prevents SQL injection and improves performance:

```sql
-- Prepared statement
PREPARE get_user (UUID) AS
  SELECT * FROM users WHERE id = $1;

EXECUTE get_user('uuid-here');

-- In TypeORM (automatically uses prepared statements)
await repository.findOne({ where: { id: userId } });
```

### 3. **Analyze Query Performance**

```sql
-- Explain query plan
EXPLAIN ANALYZE
SELECT u.*, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- Check slow queries
SELECT
  query,
  calls,
  total_time,
  mean_time,
  max_time
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 10;
```

### 4. **Use Appropriate Data Types**

- Use `UUID` for distributed systems
- Use `TIMESTAMP WITH TIME ZONE` for timestamps
- Use `JSONB` instead of `JSON` (faster, indexable)
- Use `TEXT` instead of `VARCHAR` for variable-length text
- Use `DECIMAL` for monetary values

### 5. **Implement Row-Level Security (RLS)**

```sql
-- Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Create policy
CREATE POLICY user_orders_policy ON orders
  FOR ALL
  TO myapp_user
  USING (user_id = current_setting('app.user_id')::UUID);

-- Set user context
SET app.user_id = 'user-uuid';
```

### 6. **Regular Maintenance**

```sql
-- Vacuum to reclaim space
VACUUM ANALYZE users;

-- Auto-vacuum configuration (postgresql.conf)
autovacuum = on
autovacuum_max_workers = 3

-- Reindex
REINDEX INDEX idx_users_email;

-- Update statistics
ANALYZE users;
```

### 7. **Monitor Performance**

```sql
-- Check database size
SELECT
  pg_size_pretty(pg_database_size('myapp_production')) as db_size;

-- Check table sizes
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Check index usage
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  idx_tup_read,
  idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

## Key Takeaways

✅ **PostgreSQL provides ACID compliance and data integrity**  
✅ **Use appropriate indexes (B-tree, GIN, partial) for performance**  
✅ **JSONB enables flexible schema with indexing support**  
✅ **Triggers automate database-level logic**  
✅ **Window functions enable complex analytical queries**  
✅ **CTEs improve query readability and reusability**  
✅ **Transactions ensure data consistency**  
✅ **Connection pooling optimizes resource usage**  
✅ **Regular maintenance (VACUUM, ANALYZE) maintains performance**  
✅ **Monitor and analyze slow queries continuously**

PostgreSQL is a robust, feature-rich database that scales from small projects to enterprise applications with demanding requirements.
