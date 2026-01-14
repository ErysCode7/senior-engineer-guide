# Query Optimization & Indexing

## Overview

Query optimization is the process of improving database query performance through better query writing, proper indexing, and understanding how the database executes queries. Even small optimizations can dramatically improve application speed and reduce server costs.

**Key Concepts:**

- **Query Execution Plans**: Understanding how databases execute queries
- **Indexes**: Data structures that speed up data retrieval
- **Query Rewriting**: Transforming slow queries into faster equivalents
- **Database Statistics**: Maintaining accurate data distribution info
- **Caching**: Storing frequently accessed data in memory

## Practical Use Cases

### 1. **Slow Dashboard Loading**

Complex analytics queries

- Reporting dashboards
- Business intelligence
- Real-time analytics
- Data aggregations

### 2. **API Performance Issues**

High-traffic endpoints

- Product listings
- Search functionality
- User feeds
- Recommendation engines

### 3. **Background Job Optimization**

Batch processing tasks

- Data exports
- Report generation
- Email campaigns
- Data synchronization

### 4. **Scalability Improvements**

Preparing for growth

- Handling increased traffic
- Larger datasets
- More concurrent users
- Complex queries

### 5. **Cost Reduction**

Infrastructure optimization

- Reducing database load
- Lower resource usage
- Fewer database replicas
- Smaller instance sizes

## Step-by-Step Implementation

### 1. Understanding Query Execution Plans

```sql
-- PostgreSQL: EXPLAIN and EXPLAIN ANALYZE
EXPLAIN
SELECT u.username, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON u.id = p.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.username
ORDER BY post_count DESC
LIMIT 10;

-- Output shows estimated costs and execution strategy
-- Seq Scan on users (cost=0.00..35.00 rows=1000 width=516)
-- Hash Left Join  (cost=...)
-- Sort (cost=...)

-- EXPLAIN ANALYZE runs the query and shows actual time
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category_id = 5
AND price BETWEEN 100 AND 500;

-- Output includes actual execution time
-- Planning Time: 0.123 ms
-- Execution Time: 45.678 ms

-- Format as JSON for programmatic analysis
EXPLAIN (FORMAT JSON, ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 'uuid-here';
```

**Key Metrics in Execution Plans:**

- **Seq Scan**: Full table scan (slow for large tables)
- **Index Scan**: Uses an index (fast)
- **Index Only Scan**: Uses covering index (fastest)
- **Nested Loop**: Joins small datasets
- **Hash Join**: Joins larger datasets
- **Merge Join**: Joins sorted datasets
- **cost=0.00..100.00**: Estimated cost units
- **rows=1000**: Estimated number of rows
- **width=516**: Average row size in bytes

### 2. Index Types & Usage

#### B-tree Index (Default)

```sql
-- Single column index
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters!)
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Works for:
-- WHERE user_id = 123 AND status = 'pending'
-- WHERE user_id = 123
-- But NOT for: WHERE status = 'pending' alone

-- Descending index for sorting
CREATE INDEX idx_posts_created_desc ON posts(created_at DESC);

-- Benefits queries like:
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10;
```

#### Partial Index

```sql
-- Index only active users
CREATE INDEX idx_users_active_email
ON users(email)
WHERE deleted_at IS NULL;

-- Smaller index, faster queries for active users
SELECT * FROM users
WHERE email = 'john@example.com'
AND deleted_at IS NULL;

-- Index only pending orders
CREATE INDEX idx_orders_pending
ON orders(user_id, created_at)
WHERE status = 'pending';
```

#### Covering Index (Index-Only Scan)

```sql
-- Include columns in index
CREATE INDEX idx_products_category_covering
ON products(category_id)
INCLUDE (name, price, stock);

-- Query can be satisfied entirely from index
SELECT name, price, stock
FROM products
WHERE category_id = 5;

-- No need to access table data!
```

#### GIN Index (for arrays, JSONB, full-text)

```sql
-- For array columns
CREATE INDEX idx_users_roles ON users USING GIN (roles);

-- Query arrays
SELECT * FROM users WHERE roles @> ARRAY['admin'];

-- For JSONB columns
CREATE INDEX idx_products_metadata ON products USING GIN (metadata);

-- Query JSONB
SELECT * FROM products
WHERE metadata @> '{"featured": true}';

SELECT * FROM products
WHERE metadata->>'brand' = 'Apple';
```

#### GiST Index (for geometric/range data)

```sql
-- For range types
CREATE INDEX idx_bookings_period ON bookings USING GIST (period);

-- For geometric data (PostGIS)
CREATE INDEX idx_locations_point ON locations USING GIST (coordinates);
```

#### Hash Index (equality only)

```sql
-- For exact matches only (no ranges or sorting)
CREATE INDEX idx_sessions_token ON sessions USING HASH (session_token);

-- Works for:
SELECT * FROM sessions WHERE session_token = 'abc123';

-- NOT for:
SELECT * FROM sessions WHERE session_token LIKE 'abc%';
```

### 3. Query Optimization Techniques

#### Avoid SELECT \*

```sql
-- ❌ Bad: Retrieves unnecessary data
SELECT * FROM users WHERE id = 123;

-- ✅ Good: Select only needed columns
SELECT id, username, email FROM users WHERE id = 123;

-- Reduces data transfer and allows covering indexes
```

#### Use LIMIT

```sql
-- ❌ Bad: Retrieves all rows
SELECT * FROM posts ORDER BY created_at DESC;

-- ✅ Good: Limit results
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20;

-- Even better with offset for pagination
SELECT * FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;  -- Page 3
```

#### Avoid Functions on Indexed Columns

```sql
-- ❌ Bad: Function prevents index usage
SELECT * FROM users
WHERE LOWER(email) = 'john@example.com';

-- ✅ Good: Store lowercase, or use functional index
SELECT * FROM users
WHERE email = 'john@example.com';  -- Assuming email stored as lowercase

-- Or create functional index
CREATE INDEX idx_users_email_lower
ON users(LOWER(email));
```

#### Use EXISTS Instead of COUNT

```sql
-- ❌ Bad: Counts all rows
SELECT * FROM users u
WHERE (SELECT COUNT(*) FROM posts WHERE user_id = u.id) > 0;

-- ✅ Good: Stops at first match
SELECT * FROM users u
WHERE EXISTS (SELECT 1 FROM posts WHERE user_id = u.id);
```

#### Optimize JOINs

```sql
-- ❌ Bad: Multiple separate queries (N+1 problem)
-- In application code:
SELECT * FROM orders;  -- 100 orders
-- Then for each order:
SELECT * FROM users WHERE id = ?;  -- 100 queries

-- ✅ Good: Single JOIN query
SELECT
  o.*,
  u.username,
  u.email
FROM orders o
JOIN users u ON o.user_id = u.id;

-- ✅ Better: Add indexes on join columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(id);  -- Usually exists as PK
```

#### Use UNION ALL Instead of UNION

```sql
-- ❌ Bad: UNION removes duplicates (expensive)
SELECT name FROM products WHERE category_id = 1
UNION
SELECT name FROM products WHERE category_id = 2;

-- ✅ Good: UNION ALL keeps duplicates (faster)
SELECT name FROM products WHERE category_id = 1
UNION ALL
SELECT name FROM products WHERE category_id = 2;

-- Use UNION only when you need to remove duplicates
```

#### Batch INSERT/UPDATE

```sql
-- ❌ Bad: Multiple individual inserts
INSERT INTO products (name, price) VALUES ('Product 1', 10.99);
INSERT INTO products (name, price) VALUES ('Product 2', 20.99);
INSERT INTO products (name, price) VALUES ('Product 3', 30.99);

-- ✅ Good: Batch insert
INSERT INTO products (name, price) VALUES
  ('Product 1', 10.99),
  ('Product 2', 20.99),
  ('Product 3', 30.99);

-- Even better: Use COPY for large datasets
COPY products (name, price)
FROM '/path/to/data.csv'
WITH (FORMAT csv, HEADER true);
```

### 4. Advanced Optimization Patterns

#### Window Functions for Ranking

```sql
-- ❌ Bad: Subquery for each row
SELECT
  p.*,
  (SELECT COUNT(*) FROM products p2
   WHERE p2.category_id = p.category_id
   AND p2.price < p.price) + 1 AS price_rank
FROM products p;

-- ✅ Good: Window function
SELECT
  p.*,
  RANK() OVER (
    PARTITION BY category_id
    ORDER BY price
  ) AS price_rank
FROM products p;
```

#### CTEs for Readability and Performance

```sql
-- Common Table Expression
WITH active_users AS (
  SELECT id, username, created_at
  FROM users
  WHERE deleted_at IS NULL
  AND last_login_at > CURRENT_DATE - INTERVAL '30 days'
),
user_stats AS (
  SELECT
    user_id,
    COUNT(*) AS post_count,
    MAX(created_at) AS last_post_at
  FROM posts
  WHERE user_id IN (SELECT id FROM active_users)
  GROUP BY user_id
)
SELECT
  u.username,
  s.post_count,
  s.last_post_at
FROM active_users u
LEFT JOIN user_stats s ON u.id = s.user_id
ORDER BY s.post_count DESC
LIMIT 20;

-- Recursive CTE for hierarchical data
WITH RECURSIVE category_tree AS (
  -- Base case: root categories
  SELECT id, name, parent_id, 0 AS level, name AS path
  FROM categories
  WHERE parent_id IS NULL

  UNION ALL

  -- Recursive case: child categories
  SELECT c.id, c.name, c.parent_id, ct.level + 1, ct.path || ' > ' || c.name
  FROM categories c
  JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT * FROM category_tree
ORDER BY path;
```

#### Materialized Views

```sql
-- Create materialized view for expensive aggregations
CREATE MATERIALIZED VIEW product_statistics AS
SELECT
  p.id,
  p.name,
  COUNT(DISTINCT oi.order_id) AS times_ordered,
  SUM(oi.quantity) AS total_sold,
  AVG(r.rating) AS avg_rating,
  COUNT(r.id) AS review_count
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN reviews r ON p.id = r.product_id
GROUP BY p.id, p.name;

-- Create index on materialized view
CREATE INDEX idx_product_stats_times_ordered
ON product_statistics(times_ordered DESC);

-- Refresh periodically (in cron job or scheduled task)
REFRESH MATERIALIZED VIEW CONCURRENTLY product_statistics;

-- Query is now instant
SELECT * FROM product_statistics
WHERE times_ordered > 100
ORDER BY times_ordered DESC;
```

#### Partitioning for Large Tables

```sql
-- Create partitioned table (PostgreSQL 10+)
CREATE TABLE orders (
  id SERIAL,
  user_id INTEGER NOT NULL,
  total DECIMAL(10, 2),
  created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2023 PARTITION OF orders
FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Queries automatically use correct partition
SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';
-- Only scans orders_2024 partition
```

### 5. Monitoring & Maintenance

```sql
-- Find slow queries (PostgreSQL)
SELECT
  query,
  calls,
  total_time / 1000 AS total_seconds,
  mean_time / 1000 AS avg_seconds,
  max_time / 1000 AS max_seconds
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 20;

-- Enable pg_stat_statements (postgresql.conf)
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- Find missing indexes
SELECT
  schemaname,
  tablename,
  attname,
  n_distinct,
  correlation
FROM pg_stats
WHERE schemaname = 'public'
AND n_distinct > 100  -- High cardinality
AND correlation < 0.1;  -- Low correlation (good for indexing)

-- Find unused indexes
SELECT
  schemaname,
  tablename,
  indexname,
  idx_scan,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Update table statistics
ANALYZE users;
ANALYZE VERBOSE products;  -- With details

-- Vacuum to reclaim space
VACUUM ANALYZE users;

-- Check bloat
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
  pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS indexes_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

### 6. Application-Level Optimization

```typescript
// Connection pooling
import { Pool } from "pg";

const pool = new Pool({
  host: process.env.DB_HOST,
  port: 5432,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  max: 20, // Maximum connections
  min: 5, // Minimum connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Use prepared statements
const result = await pool.query("SELECT * FROM users WHERE email = $1", [
  "john@example.com",
]);

// Batch queries
const client = await pool.connect();
try {
  await client.query("BEGIN");

  for (const user of users) {
    await client.query("INSERT INTO users (email, username) VALUES ($1, $2)", [
      user.email,
      user.username,
    ]);
  }

  await client.query("COMMIT");
} catch (error) {
  await client.query("ROLLBACK");
  throw error;
} finally {
  client.release();
}

// Query result caching
import { CACHE_MANAGER } from "@nestjs/cache-manager";
import { Inject } from "@nestjs/common";

@Injectable()
export class ProductService {
  constructor(
    @Inject(CACHE_MANAGER) private cacheManager: Cache,
    private productRepository: Repository<Product>
  ) {}

  async findById(id: number): Promise<Product> {
    const cacheKey = `product:${id}`;

    // Check cache
    let product = await this.cacheManager.get<Product>(cacheKey);

    if (!product) {
      // Query database
      product = await this.productRepository.findOne({ where: { id } });

      // Store in cache for 5 minutes
      await this.cacheManager.set(cacheKey, product, 300);
    }

    return product;
  }
}
```

## Best Practices

### 1. **Index Strategy**

- Index foreign keys
- Index columns in WHERE, ORDER BY, GROUP BY clauses
- Use composite indexes for multiple conditions
- Don't over-index (slows writes)
- Use partial indexes when appropriate
- Consider covering indexes for common queries

### 2. **Query Writing**

- Select only needed columns
- Use LIMIT for pagination
- Avoid SELECT DISTINCT when possible
- Use JOINs instead of subqueries when possible
- Batch operations when possible
- Use prepared statements

### 3. **Monitoring**

```sql
-- Set up monitoring
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Regular maintenance schedule
-- Daily: ANALYZE heavily updated tables
-- Weekly: VACUUM ANALYZE all tables
-- Monthly: REINDEX if needed

-- Alert on slow queries (> 1 second)
-- Alert on connection pool exhaustion
-- Monitor cache hit ratio (aim for > 95%)
```

### 4. **Testing**

```sql
-- Test with realistic data volume
-- Production: 10 million users
-- Test with at least 10 million users

-- Test query performance
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';

-- Benchmark before and after optimization
-- Use tools like pgbench, Apache Bench
```

### 5. **Documentation**

```sql
-- Document complex queries
COMMENT ON INDEX idx_users_email IS 'Speeds up user lookup by email (login)';

-- Document why indexes exist
COMMENT ON INDEX idx_orders_user_status IS 'For user order history filtered by status';
```

## Key Takeaways

✅ **Use EXPLAIN ANALYZE to understand query execution**  
✅ **Index foreign keys and frequently queried columns**  
✅ **Avoid SELECT \* and retrieve only needed columns**  
✅ **Use LIMIT for pagination to reduce data transfer**  
✅ **Batch operations to reduce round trips**  
✅ **Monitor slow queries and optimize regularly**  
✅ **Use covering indexes for index-only scans**  
✅ **Keep database statistics up to date with ANALYZE**  
✅ **Consider caching for frequently accessed data**  
✅ **Test queries with production-like data volumes**

Query optimization is an ongoing process. Regular monitoring and maintenance ensure your database performs well as your application scales.
