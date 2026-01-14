# Database Design & Relationships

## Overview

Database design is the process of organizing data according to a database model to ensure data integrity, minimize redundancy, and optimize query performance. Proper design is crucial for scalability, maintainability, and application success.

**Key Concepts:**

- **Normalization**: Organizing data to reduce redundancy
- **Relationships**: Defining how entities relate to each other
- **Constraints**: Enforcing data integrity rules
- **ER Diagrams**: Visual representation of database structure
- **Denormalization**: Strategic redundancy for performance

## Practical Use Cases

### 1. **E-commerce Platform**

Complex multi-entity system

- Products, orders, customers
- Inventory management
- Order history and tracking
- Product categories and variants

### 2. **Social Media Application**

User-generated content platform

- Users, posts, comments
- Friendships and followers
- Likes, shares, notifications
- Media attachments

### 3. **Learning Management System**

Educational content delivery

- Courses, lessons, quizzes
- Students, instructors, enrollments
- Progress tracking
- Certificates and achievements

### 4. **Healthcare System**

Patient management

- Patients, doctors, appointments
- Medical records and prescriptions
- Lab results and diagnoses
- Insurance and billing

### 5. **Project Management Tool**

Team collaboration

- Projects, tasks, subtasks
- Team members and roles
- Time tracking
- File attachments and comments

## Step-by-Step Implementation

### 1. Entity-Relationship (ER) Diagram

```
┌─────────────┐         ┌──────────────┐         ┌──────────────┐
│   Users     │         │   Orders     │         │   Products   │
├─────────────┤         ├──────────────┤         ├──────────────┤
│ id (PK)     │────┐    │ id (PK)      │    ┌────│ id (PK)      │
│ email       │    │    │ user_id (FK) │────│    │ name         │
│ username    │    │    │ status       │    │    │ price        │
│ password    │    │    │ total        │    │    │ stock        │
│ created_at  │    │    │ created_at   │    │    │ category_id  │
└─────────────┘    │    └──────────────┘    │    └──────────────┘
                   │                         │            │
                   │    ┌──────────────┐    │            │
                   │    │ Order Items  │    │            │
                   │    ├──────────────┤    │            │
                   │    │ id (PK)      │    │            │
                   └────│ order_id (FK)│    │            │
                        │ product_id   │────┘            │
                        │ quantity     │                 │
                        │ price        │                 │
                        └──────────────┘                 │
                                                         │
                        ┌──────────────┐                │
                        │  Categories  │                │
                        ├──────────────┤                │
                        │ id (PK)      │────────────────┘
                        │ name         │
                        │ parent_id    │
                        │ created_at   │
                        └──────────────┘

Legend:
PK = Primary Key
FK = Foreign Key
─── = Relationship
```

### 2. Relationship Types

#### One-to-One (1:1)

```sql
-- User has one profile
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE user_profiles (
  id SERIAL PRIMARY KEY,
  user_id INTEGER UNIQUE NOT NULL,  -- UNIQUE ensures 1:1
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  bio TEXT,
  avatar_url VARCHAR(500),
  date_of_birth DATE,
  phone VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Alternative: Include profile data directly in users table
-- Use separate table when profile data is optional or large
```

#### One-to-Many (1:N)

```sql
-- User has many posts
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE posts (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL,  -- No UNIQUE, allows multiple posts
  title VARCHAR(255) NOT NULL,
  content TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
```

#### Many-to-Many (M:N)

```sql
-- Students and courses (many students, many courses)
CREATE TABLE students (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL
);

CREATE TABLE courses (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  code VARCHAR(20) UNIQUE NOT NULL,
  credits INTEGER NOT NULL
);

-- Junction/bridge table
CREATE TABLE enrollments (
  id SERIAL PRIMARY KEY,
  student_id INTEGER NOT NULL,
  course_id INTEGER NOT NULL,
  enrolled_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  grade VARCHAR(2),
  status VARCHAR(20) DEFAULT 'active',

  FOREIGN KEY (student_id) REFERENCES students(id) ON DELETE CASCADE,
  FOREIGN KEY (course_id) REFERENCES courses(id) ON DELETE CASCADE,

  -- Prevent duplicate enrollments
  UNIQUE(student_id, course_id)
);

CREATE INDEX idx_enrollments_student ON enrollments(student_id);
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
```

#### Self-Referencing (Hierarchical)

```sql
-- Categories with parent-child relationship
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  parent_id INTEGER,
  level INTEGER DEFAULT 0,
  path VARCHAR(255),  -- e.g., "1/5/12" for breadcrumb
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL
);

CREATE INDEX idx_categories_parent ON categories(parent_id);

-- Example data:
-- Electronics (id=1, parent_id=NULL)
--   ├── Computers (id=2, parent_id=1)
--   │     ├── Laptops (id=3, parent_id=2)
--   │     └── Desktops (id=4, parent_id=2)
--   └── Phones (id=5, parent_id=1)

-- Employee-Manager relationship
CREATE TABLE employees (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  manager_id INTEGER,
  department VARCHAR(100),

  FOREIGN KEY (manager_id) REFERENCES employees(id) ON DELETE SET NULL
);
```

### 3. Normalization Forms

#### First Normal Form (1NF)

**Rule**: Atomic values, no repeating groups

```sql
-- ❌ Not in 1NF (comma-separated values)
CREATE TABLE orders_bad (
  id INTEGER,
  product_ids VARCHAR(100),  -- "1,2,3"
  quantities VARCHAR(50)     -- "2,1,5"
);

-- ✅ 1NF (atomic values)
CREATE TABLE orders (
  id INTEGER PRIMARY KEY,
  status VARCHAR(20)
);

CREATE TABLE order_items (
  id INTEGER PRIMARY KEY,
  order_id INTEGER,
  product_id INTEGER,
  quantity INTEGER,
  FOREIGN KEY (order_id) REFERENCES orders(id)
);
```

#### Second Normal Form (2NF)

**Rule**: 1NF + No partial dependencies (all non-key attributes fully depend on primary key)

```sql
-- ❌ Not in 2NF (product_name depends only on product_id, not full PK)
CREATE TABLE order_items_bad (
  order_id INTEGER,
  product_id INTEGER,
  product_name VARCHAR(100),  -- Depends only on product_id
  quantity INTEGER,
  PRIMARY KEY (order_id, product_id)
);

-- ✅ 2NF (separate products table)
CREATE TABLE products (
  id INTEGER PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  price DECIMAL(10, 2) NOT NULL
);

CREATE TABLE order_items (
  order_id INTEGER,
  product_id INTEGER,
  quantity INTEGER,
  price_at_purchase DECIMAL(10, 2),  -- Snapshot of price
  PRIMARY KEY (order_id, product_id),
  FOREIGN KEY (product_id) REFERENCES products(id)
);
```

#### Third Normal Form (3NF)

**Rule**: 2NF + No transitive dependencies

```sql
-- ❌ Not in 3NF (city depends on zip_code, not user_id)
CREATE TABLE users_bad (
  id INTEGER PRIMARY KEY,
  name VARCHAR(100),
  zip_code VARCHAR(10),
  city VARCHAR(100),     -- Depends on zip_code
  state VARCHAR(50)      -- Depends on zip_code
);

-- ✅ 3NF (separate zip codes table)
CREATE TABLE zip_codes (
  code VARCHAR(10) PRIMARY KEY,
  city VARCHAR(100) NOT NULL,
  state VARCHAR(50) NOT NULL
);

CREATE TABLE users (
  id INTEGER PRIMARY KEY,
  name VARCHAR(100),
  zip_code VARCHAR(10),
  FOREIGN KEY (zip_code) REFERENCES zip_codes(code)
);
```

### 4. Complete E-commerce Schema

```sql
-- Users and authentication
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  email VARCHAR(255) UNIQUE NOT NULL,
  username VARCHAR(50) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) DEFAULT 'customer',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  CHECK (role IN ('customer', 'admin', 'vendor'))
);

-- User profiles (1:1)
CREATE TABLE profiles (
  id SERIAL PRIMARY KEY,
  user_id UUID UNIQUE NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  phone VARCHAR(20),
  date_of_birth DATE,
  avatar_url VARCHAR(500),

  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Addresses (1:N - user can have multiple addresses)
CREATE TABLE addresses (
  id SERIAL PRIMARY KEY,
  user_id UUID NOT NULL,
  type VARCHAR(20) DEFAULT 'shipping',
  street VARCHAR(255) NOT NULL,
  city VARCHAR(100) NOT NULL,
  state VARCHAR(50) NOT NULL,
  zip_code VARCHAR(10) NOT NULL,
  country VARCHAR(50) NOT NULL,
  is_default BOOLEAN DEFAULT FALSE,

  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  CHECK (type IN ('shipping', 'billing'))
);

CREATE INDEX idx_addresses_user ON addresses(user_id);

-- Categories (hierarchical self-reference)
CREATE TABLE categories (
  id SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  slug VARCHAR(100) UNIQUE NOT NULL,
  parent_id INTEGER,
  description TEXT,
  image_url VARCHAR(500),

  FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL
);

-- Products
CREATE TABLE products (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  slug VARCHAR(255) UNIQUE NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
  compare_at_price DECIMAL(10, 2),
  cost_price DECIMAL(10, 2),
  sku VARCHAR(100) UNIQUE,
  barcode VARCHAR(100),
  stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
  category_id INTEGER,
  vendor_id UUID,
  status VARCHAR(20) DEFAULT 'draft',
  is_featured BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL,
  FOREIGN KEY (vendor_id) REFERENCES users(id) ON DELETE SET NULL,
  CHECK (status IN ('draft', 'active', 'archived'))
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_vendor ON products(vendor_id);
CREATE INDEX idx_products_status ON products(status) WHERE status = 'active';

-- Product images (1:N)
CREATE TABLE product_images (
  id SERIAL PRIMARY KEY,
  product_id INTEGER NOT NULL,
  url VARCHAR(500) NOT NULL,
  alt_text VARCHAR(255),
  position INTEGER DEFAULT 0,

  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

CREATE INDEX idx_product_images_product ON product_images(product_id);

-- Product variants (for size, color, etc.)
CREATE TABLE product_variants (
  id SERIAL PRIMARY KEY,
  product_id INTEGER NOT NULL,
  name VARCHAR(100) NOT NULL,  -- e.g., "Small / Red"
  sku VARCHAR(100) UNIQUE,
  price DECIMAL(10, 2),
  stock INTEGER DEFAULT 0 CHECK (stock >= 0),
  option1_name VARCHAR(50),   -- e.g., "Size"
  option1_value VARCHAR(50),  -- e.g., "Small"
  option2_name VARCHAR(50),   -- e.g., "Color"
  option2_value VARCHAR(50),  -- e.g., "Red"

  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);

CREATE INDEX idx_variants_product ON product_variants(product_id);

-- Tags (M:N with products)
CREATE TABLE tags (
  id SERIAL PRIMARY KEY,
  name VARCHAR(50) UNIQUE NOT NULL,
  slug VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE product_tags (
  product_id INTEGER NOT NULL,
  tag_id INTEGER NOT NULL,

  PRIMARY KEY (product_id, tag_id),
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
  FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE
);

-- Orders
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  order_number VARCHAR(50) UNIQUE NOT NULL,
  user_id UUID NOT NULL,
  status VARCHAR(20) DEFAULT 'pending',
  payment_status VARCHAR(20) DEFAULT 'pending',
  subtotal DECIMAL(10, 2) NOT NULL CHECK (subtotal >= 0),
  tax DECIMAL(10, 2) DEFAULT 0,
  shipping_cost DECIMAL(10, 2) DEFAULT 0,
  total DECIMAL(10, 2) NOT NULL CHECK (total >= 0),
  currency VARCHAR(3) DEFAULT 'USD',
  shipping_address_id INTEGER,
  billing_address_id INTEGER,
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (shipping_address_id) REFERENCES addresses(id),
  FOREIGN KEY (billing_address_id) REFERENCES addresses(id),
  CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled')),
  CHECK (payment_status IN ('pending', 'paid', 'failed', 'refunded'))
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- Order items
CREATE TABLE order_items (
  id SERIAL PRIMARY KEY,
  order_id INTEGER NOT NULL,
  product_id INTEGER NOT NULL,
  variant_id INTEGER,
  quantity INTEGER NOT NULL CHECK (quantity > 0),
  price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
  total DECIMAL(10, 2) NOT NULL CHECK (total >= 0),

  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id),
  FOREIGN KEY (variant_id) REFERENCES product_variants(id)
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);

-- Reviews (1:N - user can review many products)
CREATE TABLE reviews (
  id SERIAL PRIMARY KEY,
  product_id INTEGER NOT NULL,
  user_id UUID NOT NULL,
  order_id INTEGER,  -- Verified purchase
  rating INTEGER NOT NULL CHECK (rating BETWEEN 1 AND 5),
  title VARCHAR(255),
  comment TEXT,
  is_verified BOOLEAN DEFAULT FALSE,
  helpful_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE SET NULL,

  -- User can only review a product once
  UNIQUE(product_id, user_id)
);

CREATE INDEX idx_reviews_product ON reviews(product_id);
CREATE INDEX idx_reviews_user ON reviews(user_id);

-- Wishlist (M:N - users can have many products in wishlist)
CREATE TABLE wishlists (
  user_id UUID NOT NULL,
  product_id INTEGER NOT NULL,
  added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

  PRIMARY KEY (user_id, product_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE
);
```

### 5. Constraints & Data Integrity

```sql
-- Primary key constraint
ALTER TABLE users ADD CONSTRAINT users_pkey PRIMARY KEY (id);

-- Unique constraint
ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);

-- Foreign key constraint with actions
ALTER TABLE posts
ADD CONSTRAINT posts_user_id_fkey
FOREIGN KEY (user_id)
REFERENCES users(id)
ON DELETE CASCADE        -- Delete posts when user is deleted
ON UPDATE CASCADE;       -- Update posts when user id changes

-- Check constraint
ALTER TABLE products
ADD CONSTRAINT products_price_check
CHECK (price >= 0 AND price <= 1000000);

-- Not null constraint
ALTER TABLE users
ALTER COLUMN email SET NOT NULL;

-- Default value
ALTER TABLE users
ALTER COLUMN created_at SET DEFAULT CURRENT_TIMESTAMP;

-- Composite unique constraint
ALTER TABLE enrollments
ADD CONSTRAINT enrollments_student_course_unique
UNIQUE (student_id, course_id);

-- Conditional unique constraint (PostgreSQL)
CREATE UNIQUE INDEX idx_users_username_active
ON users(username)
WHERE deleted_at IS NULL;

-- Multi-column check constraint
ALTER TABLE products
ADD CONSTRAINT products_pricing_check
CHECK (
  compare_at_price IS NULL OR
  compare_at_price >= price
);
```

### 6. Denormalization Strategies

```sql
-- Add computed columns for performance
ALTER TABLE products
ADD COLUMN average_rating DECIMAL(2, 1),
ADD COLUMN review_count INTEGER DEFAULT 0;

-- Update trigger to maintain denormalized data
CREATE OR REPLACE FUNCTION update_product_rating()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE products
  SET
    average_rating = (
      SELECT AVG(rating)::DECIMAL(2,1)
      FROM reviews
      WHERE product_id = NEW.product_id
    ),
    review_count = (
      SELECT COUNT(*)
      FROM reviews
      WHERE product_id = NEW.product_id
    )
  WHERE id = NEW.product_id;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_product_rating_trigger
AFTER INSERT OR UPDATE OR DELETE ON reviews
FOR EACH ROW
EXECUTE FUNCTION update_product_rating();

-- Materialized view for reporting
CREATE MATERIALIZED VIEW product_sales_summary AS
SELECT
  p.id,
  p.name,
  COUNT(DISTINCT oi.order_id) AS times_ordered,
  SUM(oi.quantity) AS total_quantity_sold,
  SUM(oi.total) AS total_revenue
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.id
WHERE o.status = 'delivered'
GROUP BY p.id, p.name;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW CONCURRENTLY product_sales_summary;
```

## Best Practices

### 1. **Naming Conventions**

- Tables: plural, lowercase with underscores (`users`, `order_items`)
- Columns: lowercase with underscores (`first_name`, `created_at`)
- Primary keys: `id` or `table_name_id`
- Foreign keys: `referenced_table_singular_id` (`user_id`, `product_id`)
- Indexes: `idx_table_column` (`idx_users_email`)
- Constraints: `table_column_type` (`users_email_unique`)

### 2. **Use Appropriate Data Types**

```sql
-- String types
VARCHAR(n)    -- Variable length with limit
TEXT          -- Unlimited length
CHAR(n)       -- Fixed length (for codes, etc.)

-- Numeric types
INTEGER       -- Whole numbers
BIGINT        -- Large whole numbers
DECIMAL(p,s)  -- Exact decimals (money)
NUMERIC       -- Same as DECIMAL
FLOAT         -- Approximate decimals

-- Date/Time types
DATE          -- Date only
TIME          -- Time only
TIMESTAMP     -- Date and time
TIMESTAMPTZ   -- Timestamp with timezone (recommended)

-- Boolean
BOOLEAN       -- TRUE/FALSE/NULL

-- JSON
JSON          -- Text-based JSON
JSONB         -- Binary JSON (faster, indexable)

-- UUID
UUID          -- Universally unique identifier
```

### 3. **Indexing Strategy**

```sql
-- Index foreign keys
CREATE INDEX ON order_items(order_id);
CREATE INDEX ON order_items(product_id);

-- Index columns used in WHERE clauses
CREATE INDEX ON users(email);
CREATE INDEX ON products(status) WHERE status = 'active';

-- Composite indexes (order matters)
CREATE INDEX ON orders(user_id, status, created_at DESC);

-- Covering indexes (include columns in index)
CREATE INDEX ON products(category_id) INCLUDE (name, price);
```

### 4. **Handle Soft Deletes**

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;

-- Query only active records
SELECT * FROM users WHERE deleted_at IS NULL;

-- Create partial unique index
CREATE UNIQUE INDEX idx_users_email_active
ON users(email)
WHERE deleted_at IS NULL;
```

### 5. **Audit Trails**

```sql
-- Add audit columns
ALTER TABLE orders
ADD COLUMN created_by UUID REFERENCES users(id),
ADD COLUMN updated_by UUID REFERENCES users(id),
ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
ADD COLUMN updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;

-- Separate audit table
CREATE TABLE audit_log (
  id SERIAL PRIMARY KEY,
  table_name VARCHAR(100) NOT NULL,
  record_id INTEGER NOT NULL,
  action VARCHAR(10) NOT NULL,  -- INSERT, UPDATE, DELETE
  old_data JSONB,
  new_data JSONB,
  changed_by UUID REFERENCES users(id),
  changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 6. **Avoid Common Pitfalls**

- Don't use ENUM types (use lookup tables instead for flexibility)
- Don't store files in database (use file storage + URL)
- Don't store calculated values unless denormalizing intentionally
- Don't use generic EAV (Entity-Attribute-Value) patterns
- Don't create too many indexes (slows writes)

## Key Takeaways

✅ **Start with ER diagrams to visualize relationships**  
✅ **Normalize to 3NF, then denormalize strategically**  
✅ **Use foreign keys to enforce referential integrity**  
✅ **Choose appropriate data types for each column**  
✅ **Index foreign keys and frequently queried columns**  
✅ **Use constraints to enforce business rules at database level**  
✅ **Follow consistent naming conventions**  
✅ **Plan for soft deletes and audit trails**  
✅ **Use junction tables for many-to-many relationships**  
✅ **Document your schema and relationships**

Proper database design is foundational for building scalable, maintainable applications with strong data integrity.
