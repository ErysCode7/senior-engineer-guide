# MongoDB & NoSQL Patterns

## Overview

MongoDB is a document-oriented NoSQL database that stores data in flexible, JSON-like documents (BSON format). It's designed for scalability, high performance, and developer productivity, making it ideal for applications with evolving schemas and high write loads.

**Key Features:**

- **Document Model**: Flexible schema with nested data
- **Horizontal Scalability**: Sharding for distributed data
- **Replication**: High availability with replica sets
- **Aggregation Pipeline**: Powerful data processing
- **Indexes**: Support for various index types
- **ACID Transactions**: Multi-document transactions

## Practical Use Cases

### 1. **Content Management Systems**

Dynamic content storage

- Blog platforms
- News websites
- E-learning platforms
- Documentation systems

### 2. **Real-time Analytics**

Event tracking and metrics

- User behavior analytics
- Application logs
- IoT sensor data
- Time-series data

### 3. **Product Catalogs**

E-commerce applications

- Product information
- Variant management
- Customer reviews
- Inventory tracking

### 4. **Social Networks**

User-generated content

- User profiles
- Posts and comments
- Activity feeds
- Notifications

### 5. **Mobile Applications**

Flexible data storage

- User preferences
- Offline-first sync
- Dynamic schemas
- Rapid prototyping

## Step-by-Step Implementation

### 1. Install MongoDB

```bash
# macOS
brew tap mongodb/brew
brew install mongodb-community@7.0
brew services start mongodb-community@7.0

# Ubuntu
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
sudo apt update
sudo apt install -y mongodb-org
sudo systemctl start mongod
sudo systemctl enable mongod

# Connect to MongoDB
mongosh
```

### 2. Database & Collection Setup

```javascript
// Connect to database (creates if doesn't exist)
use myapp_db

// Create collection with validation
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "username", "createdAt"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "must be a valid email"
        },
        username: {
          bsonType: "string",
          minLength: 3,
          maxLength: 50,
          description: "must be a string between 3-50 characters"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150,
          description: "must be an integer between 0-150"
        },
        createdAt: {
          bsonType: "date",
          description: "must be a date"
        }
      }
    }
  }
})

// Create indexes
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ username: 1 }, { unique: true })
db.users.createIndex({ createdAt: -1 })
db.users.createIndex({ "profile.city": 1 })
```

### 3. Document Design Patterns

```javascript
// Embedded Document Pattern (1-to-few relationship)
{
  _id: ObjectId("..."),
  email: "john@example.com",
  username: "johndoe",
  profile: {
    firstName: "John",
    lastName: "Doe",
    avatar: "https://...",
    bio: "Software Engineer",
    location: {
      city: "San Francisco",
      country: "USA",
      coordinates: {
        lat: 37.7749,
        lng: -122.4194
      }
    }
  },
  preferences: {
    theme: "dark",
    language: "en",
    notifications: {
      email: true,
      push: false,
      sms: true
    }
  },
  roles: ["user", "premium"],
  createdAt: ISODate("2024-01-01T00:00:00Z"),
  updatedAt: ISODate("2024-01-13T00:00:00Z")
}

// Reference Pattern (1-to-many relationship)
// User document
{
  _id: ObjectId("user123"),
  email: "john@example.com",
  username: "johndoe"
}

// Posts collection (references user)
{
  _id: ObjectId("post456"),
  userId: ObjectId("user123"), // Reference to user
  title: "My First Post",
  content: "Hello World!",
  tags: ["javascript", "mongodb"],
  createdAt: ISODate("2024-01-13T00:00:00Z")
}

// Denormalization Pattern (for read-heavy)
// Post with embedded user info for faster reads
{
  _id: ObjectId("post456"),
  title: "My First Post",
  content: "Hello World!",
  author: {
    id: ObjectId("user123"),
    username: "johndoe",
    avatar: "https://..." // Denormalized data
  },
  comments: [
    {
      id: ObjectId("comment789"),
      userId: ObjectId("user456"),
      username: "janedoe", // Denormalized
      text: "Great post!",
      createdAt: ISODate("2024-01-13T01:00:00Z")
    }
  ],
  stats: {
    views: 1250,
    likes: 45,
    comments: 12
  },
  createdAt: ISODate("2024-01-13T00:00:00Z")
}

// Bucket Pattern (for time-series data)
{
  _id: ObjectId("..."),
  sensorId: "sensor_001",
  date: ISODate("2024-01-13"),
  measurements: [
    { time: ISODate("2024-01-13T00:00:00Z"), temperature: 22.5, humidity: 45 },
    { time: ISODate("2024-01-13T00:05:00Z"), temperature: 22.7, humidity: 46 },
    { time: ISODate("2024-01-13T00:10:00Z"), temperature: 22.6, humidity: 45 }
  ],
  count: 3
}
```

### 4. CRUD Operations

```javascript
// Insert documents
db.users.insertOne({
  email: "john@example.com",
  username: "johndoe",
  profile: { firstName: "John", lastName: "Doe" },
  roles: ["user"],
  createdAt: new Date(),
});

// Insert multiple
db.users.insertMany([
  { email: "user1@example.com", username: "user1", createdAt: new Date() },
  { email: "user2@example.com", username: "user2", createdAt: new Date() },
]);

// Find documents
db.users.find({ email: "john@example.com" });
db.users.findOne({ username: "johndoe" });

// Query with projection (select specific fields)
db.users.find({ roles: "admin" }, { email: 1, username: 1, _id: 0 });

// Query nested fields
db.users.find({ "profile.city": "San Francisco" });

// Query arrays
db.users.find({ roles: { $in: ["admin", "moderator"] } });

// Update document
db.users.updateOne(
  { email: "john@example.com" },
  {
    $set: { "profile.bio": "Senior Engineer" },
    $currentDate: { updatedAt: true },
  }
);

// Update multiple documents
db.users.updateMany({ roles: "user" }, { $addToSet: { roles: "verified" } });

// Upsert (update or insert)
db.users.updateOne(
  { email: "new@example.com" },
  {
    $set: { username: "newuser", profile: {} },
    $setOnInsert: { createdAt: new Date() },
  },
  { upsert: true }
);

// Delete documents
db.users.deleteOne({ email: "john@example.com" });
db.users.deleteMany({ "profile.inactive": true });
```

### 5. Aggregation Pipeline

```javascript
// Basic aggregation - user statistics
db.users.aggregate([
  {
    $match: { roles: "premium" },
  },
  {
    $group: {
      _id: "$profile.location.country",
      count: { $sum: 1 },
      avgAge: { $avg: "$age" },
    },
  },
  {
    $sort: { count: -1 },
  },
  {
    $limit: 10,
  },
]);

// Join collections with $lookup
db.posts.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "author",
    },
  },
  {
    $unwind: "$author",
  },
  {
    $project: {
      title: 1,
      content: 1,
      "author.username": 1,
      "author.email": 1,
      createdAt: 1,
    },
  },
]);

// Complex aggregation - top products by revenue
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2024-02-01"),
      },
    },
  },
  {
    $unwind: "$items",
  },
  {
    $group: {
      _id: "$items.productId",
      totalRevenue: {
        $sum: { $multiply: ["$items.quantity", "$items.price"] },
      },
      totalQuantity: { $sum: "$items.quantity" },
      orderCount: { $sum: 1 },
    },
  },
  {
    $lookup: {
      from: "products",
      localField: "_id",
      foreignField: "_id",
      as: "product",
    },
  },
  {
    $unwind: "$product",
  },
  {
    $project: {
      productName: "$product.name",
      totalRevenue: 1,
      totalQuantity: 1,
      orderCount: 1,
      avgOrderValue: {
        $divide: ["$totalRevenue", "$orderCount"],
      },
    },
  },
  {
    $sort: { totalRevenue: -1 },
  },
  {
    $limit: 20,
  },
]);

// Time-series aggregation
db.measurements.aggregate([
  {
    $match: {
      sensorId: "sensor_001",
      timestamp: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2024-01-02"),
      },
    },
  },
  {
    $group: {
      _id: {
        $dateToString: {
          format: "%Y-%m-%d %H:00",
          date: "$timestamp",
        },
      },
      avgTemperature: { $avg: "$temperature" },
      minTemperature: { $min: "$temperature" },
      maxTemperature: { $max: "$temperature" },
      count: { $sum: 1 },
    },
  },
  {
    $sort: { _id: 1 },
  },
]);

// Faceted search (multiple aggregations)
db.products.aggregate([
  {
    $facet: {
      categories: [
        { $group: { _id: "$category", count: { $sum: 1 } } },
        { $sort: { count: -1 } },
      ],
      priceRanges: [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 50, 100, 200, 500, 1000],
            default: "1000+",
            output: { count: { $sum: 1 } },
          },
        },
      ],
      topRated: [
        { $sort: { rating: -1 } },
        { $limit: 5 },
        { $project: { name: 1, rating: 1, price: 1 } },
      ],
    },
  },
]);
```

### 6. Indexes & Performance

```javascript
// Single field index
db.users.createIndex({ email: 1 });

// Compound index
db.users.createIndex({ username: 1, createdAt: -1 });

// Multikey index (for arrays)
db.users.createIndex({ roles: 1 });

// Text index for full-text search
db.products.createIndex({ name: "text", description: "text" });

// Query with text search
db.products
  .find(
    {
      $text: { $search: "laptop gaming" },
    },
    {
      score: { $meta: "textScore" },
    }
  )
  .sort({ score: { $meta: "textScore" } });

// Geospatial index
db.locations.createIndex({ coordinates: "2dsphere" });

// Find nearby locations
db.locations.find({
  coordinates: {
    $near: {
      $geometry: {
        type: "Point",
        coordinates: [-122.4194, 37.7749],
      },
      $maxDistance: 5000, // 5km
    },
  },
});

// Partial index (conditional)
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { deletedAt: { $exists: false } } }
);

// TTL index (auto-delete old documents)
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 86400 } // 24 hours
);

// Check index usage
db.users.explain("executionStats").find({ email: "john@example.com" });

// List indexes
db.users.getIndexes();

// Drop index
db.users.dropIndex("email_1");
```

### 7. MongoDB with Mongoose (Node.js/Express)

```bash
npm install mongoose
```

```typescript
// src/models/user.model.ts
import mongoose, { Document, Schema } from "mongoose";

export interface IUser extends Document {
  email: string;
  username: string;
  passwordHash: string;
  profile: {
    firstName?: string;
    lastName?: string;
    avatar?: string;
    bio?: string;
    location?: {
      city?: string;
      country?: string;
    };
  };
  roles: string[];
  preferences: Record<string, any>;
  lastLoginAt?: Date;
  deletedAt?: Date;
  createdAt: Date;
  updatedAt: Date;
  isAdmin(): boolean;
}

const UserSchema = new Schema<IUser>(
  {
    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
    },
    username: {
      type: String,
      required: true,
      unique: true,
      minlength: 3,
      maxlength: 50,
    },
    passwordHash: {
      type: String,
      required: true,
    },
    profile: {
      type: Object,
      default: {},
    },
    roles: {
      type: [String],
      default: ["user"],
    },
    preferences: {
      type: Object,
      default: {},
    },
    lastLoginAt: Date,
    deletedAt: Date,
  },
  {
    timestamps: true,
  }
);

// Add indexes
UserSchema.index({ email: 1 });
UserSchema.index({ username: 1 });
UserSchema.index({ createdAt: -1 });
UserSchema.index({ roles: 1 });

// Add virtual fields
UserSchema.virtual("fullName").get(function () {
  return `${this.profile.firstName || ""} ${
    this.profile.lastName || ""
  }`.trim();
});

// Add instance methods
UserSchema.methods.isAdmin = function () {
  return this.roles.includes("admin");
};

// Add static methods
UserSchema.statics.findByEmail = function (email: string) {
  return this.findOne({ email: email.toLowerCase() });
};

export const User = mongoose.model<IUser>("User", UserSchema);
```

```typescript
// src/services/user.service.ts
import { Model, Types } from "mongoose";
import { User, IUser } from "../models/user.model";

export class UserService {
  private userModel: Model<IUser> = User;

  async create(createUserDto: any): Promise<IUser> {
    const user = new this.userModel(createUserDto);
    return user.save();
  }

  async findAll(page: number = 1, limit: number = 10): Promise<IUser[]> {
    return this.userModel
      .find({ deletedAt: { $exists: false } })
      .sort({ createdAt: -1 })
      .skip((page - 1) * limit)
      .limit(limit)
      .select("-passwordHash")
      .exec();
  }

  async findById(id: string): Promise<User | null> {
    return this.userModel.findById(id).select("-passwordHash").exec();
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.userModel.findOne({ email: email.toLowerCase() }).exec();
  }

  async update(id: string, updateDto: any): Promise<User | null> {
    return this.userModel
      .findByIdAndUpdate(
        id,
        { $set: updateDto },
        { new: true, runValidators: true }
      )
      .select("-passwordHash")
      .exec();
  }

  async softDelete(id: string): Promise<void> {
    await this.userModel
      .findByIdAndUpdate(id, { deletedAt: new Date() })
      .exec();
  }

  async search(query: string): Promise<User[]> {
    return this.userModel
      .find({
        $or: [
          { email: new RegExp(query, "i") },
          { username: new RegExp(query, "i") },
        ],
        deletedAt: { $exists: false },
      })
      .select("-passwordHash")
      .limit(20)
      .exec();
  }

  async getUserStatistics() {
    return this.userModel.aggregate([
      {
        $facet: {
          total: [{ $count: "count" }],
          byRole: [
            { $unwind: "$roles" },
            { $group: { _id: "$roles", count: { $sum: 1 } } },
          ],
          newUsers30d: [
            {
              $match: {
                createdAt: {
                  $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000),
                },
              },
            },
            { $count: "count" },
          ],
        },
      },
    ]);
  }
}
```

### 8. Transactions

```typescript
// Multi-document transaction
import { Connection } from "mongoose";

export class OrderService {
  constructor(
    @InjectConnection() private connection: Connection,
    @InjectModel(Order.name) private orderModel: Model<OrderDocument>,
    @InjectModel(Product.name) private productModel: Model<ProductDocument>
  ) {}

  async createOrder(userId: string, items: any[]) {
    const session = await this.connection.startSession();
    session.startTransaction();

    try {
      // Create order
      const order = await this.orderModel.create(
        [
          {
            userId,
            items,
            status: "pending",
            total: items.reduce(
              (sum, item) => sum + item.price * item.quantity,
              0
            ),
          },
        ],
        { session }
      );

      // Update product stock
      for (const item of items) {
        const result = await this.productModel.updateOne(
          {
            _id: item.productId,
            stock: { $gte: item.quantity },
          },
          {
            $inc: { stock: -item.quantity },
          },
          { session }
        );

        if (result.modifiedCount === 0) {
          throw new Error(`Insufficient stock for product ${item.productId}`);
        }
      }

      await session.commitTransaction();
      return order[0];
    } catch (error) {
      await session.abortTransaction();
      throw error;
    } finally {
      session.endSession();
    }
  }
}
```

## Best Practices

### 1. **Schema Design**

- Embed data for 1-to-few relationships
- Reference data for 1-to-many relationships
- Denormalize for read-heavy workloads
- Consider document size limits (16MB)
- Design for your query patterns

### 2. **Indexing Strategy**

```javascript
// Index cardinality: high cardinality first
db.users.createIndex({ country: 1, city: 1, zipCode: 1 });

// Cover queries with indexes
db.users.find({ email: "..." }, { username: 1, _id: 0 });
// Create: db.users.createIndex({ email: 1, username: 1 })

// Avoid indexing everything
// Only index fields used in queries, sorts, and joins
```

### 3. **Query Optimization**

```javascript
// Use projection to limit fields
db.users.find({}, { username: 1, email: 1 });

// Limit results
db.users.find().limit(100);

// Use explain to analyze queries
db.users.find({ email: "..." }).explain("executionStats");
```

### 4. **Connection Pooling**

```typescript
MongooseModule.forRoot(process.env.MONGODB_URI, {
  maxPoolSize: 10,
  minPoolSize: 5,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
});
```

### 5. **Replication & High Availability**

```bash
# Replica set configuration
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 1, arbiterOnly: true }
  ]
})
```

### 6. **Backup & Restore**

```bash
# Backup database
mongodump --uri="mongodb://localhost:27017/myapp_db" --out=/backup

# Restore database
mongorestore --uri="mongodb://localhost:27017/myapp_db" /backup/myapp_db

# Backup specific collection
mongodump --uri="mongodb://localhost:27017/myapp_db" --collection=users

# Compressed backup
mongodump --uri="mongodb://localhost:27017/myapp_db" --gzip --out=/backup
```

### 7. **Security**

```javascript
// Enable authentication
use admin
db.createUser({
  user: "admin",
  pwd: "secure_password",
  roles: ["userAdminAnyDatabase", "dbAdminAnyDatabase"]
})

// Create application user
use myapp_db
db.createUser({
  user: "myapp_user",
  pwd: "app_password",
  roles: [{ role: "readWrite", db: "myapp_db" }]
})
```

## Key Takeaways

✅ **MongoDB excels at flexible, document-based data models**  
✅ **Design schema based on query patterns**  
✅ **Use embedding for related data accessed together**  
✅ **Aggregation pipeline enables complex data transformations**  
✅ **Index strategically for query performance**  
✅ **Replica sets provide high availability**  
✅ **Transactions ensure multi-document consistency**  
✅ **Mongoose provides schema validation and middleware**  
✅ **Monitor query performance with explain()**  
✅ **Regular backups are essential for data protection**

MongoDB is ideal for applications requiring flexible schemas, horizontal scalability, and high write throughput.
