# REST API & GraphQL Development

## Overview

REST (Representational State Transfer) and GraphQL are two popular API architectural styles. REST is resource-based with fixed endpoints, while GraphQL provides a query language that allows clients to request exactly the data they need. This guide covers building both with Express.js.

## REST vs GraphQL

```typescript
// REST: Multiple endpoints, fixed responses
GET    /api/users          // Get all users
GET    /api/users/1        // Get user by ID
POST   /api/users          // Create user
PUT    /api/users/1        // Update user
DELETE /api/users/1        // Delete user
GET    /api/users/1/posts  // Get user's posts

// GraphQL: Single endpoint, flexible queries
POST /graphql

query {
  user(id: "1") {
    name
    email
    posts {
      title
      createdAt
    }
  }
}
```

## Practical Use Cases

### When to Use REST

- **Simple CRUD operations**
- **Caching is important** (HTTP caching works out of the box)
- **Well-defined resources**
- **Public APIs** (easier to understand and document)

### When to Use GraphQL

- **Complex data requirements**
- **Mobile apps** (reduce over-fetching)
- **Multiple client types** (each can request what they need)
- **Rapid feature development** (no backend changes for new data requirements)

## Building REST APIs with Express

### 1. Complete Product API Example

```typescript
// src/models/product.model.ts
export interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  stock: number;
  imageUrl?: string;
  categories: string[];
  createdById: string;
  createdAt: Date;
  updatedAt: Date;
  isActive: boolean;
}

export interface CreateProductDto {
  name: string;
  description: string;
  price: number;
  stock: number;
  imageUrl?: string;
  categories: string[];
}

export interface UpdateProductDto {
  name?: string;
  description?: string;
  price?: number;
  stock?: number;
  imageUrl?: string;
  categories?: string[];
  isActive?: boolean;
}

export interface FilterProductDto {
  search?: string;
  category?: string;
  minPrice?: number;
  maxPrice?: number;
  page?: number;
  limit?: number;
  sortBy?: string;
  sortOrder?: 'asc' | 'desc';
}
```

```typescript
// src/services/product.service.ts
import { v4 as uuid } from 'uuid';
import prisma from '../config/database';
import { CreateProductDto, UpdateProductDto, FilterProductDto } from '../models/product.model';
import { AppError } from '../utils/AppError';

export class ProductService {
  async create(data: CreateProductDto, userId: string) {
    const product = await prisma.product.create({
      data: {
        ...data,
        id: uuid(),
        createdById: userId,
        isActive: true,
      },
      include: {
        createdBy: {
          select: {
            id: true,
            email: true,
            firstName: true,
            lastName: true,
          },
        },
      },
    });

    return product;
  }

  async findAll(filters: FilterProductDto) {
    const {
      search,
      category,
      minPrice,
      maxPrice,
      page = 1,
      limit = 10,
      sortBy = 'createdAt',
      sortOrder = 'desc',
    } = filters;

    const skip = (page - 1) * limit;

    const where: any = { isActive: true };

    if (search) {
      where.OR = [
        { name: { contains: search, mode: 'insensitive' } },
        { description: { contains: search, mode: 'insensitive' } },
      ];
    }

    if (category) {
      where.categories = { has: category };
    }

    if (minPrice !== undefined || maxPrice !== undefined) {
      where.price = {};
      if (minPrice !== undefined) where.price.gte = minPrice;
      if (maxPrice !== undefined) where.price.lte = maxPrice;
    }

    const [products, total] = await Promise.all([
      prisma.product.findMany({
        where,
        skip,
        take: limit,
        orderBy: { [sortBy]: sortOrder },
        include: {
          createdBy: {
            select: {
              id: true,
              firstName: true,
              lastName: true,
            },
          },
        },
      }),
      prisma.product.count({ where }),
    ]);

    return {
      data: products,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
      },
    };
  }

  async findById(id: string) {
    const product = await prisma.product.findUnique({
      where: { id },
      include: {
        createdBy: {
          select: {
            id: true,
            email: true,
            firstName: true,
            lastName: true,
          },
        },
      },
    });

    if (!product) {
      throw new AppError('Product not found', 404);
    }

    return product;
  }

  async update(id: string, data: UpdateProductDto) {
    const product = await prisma.product.update({
      where: { id },
      data: {
        ...data,
        updatedAt: new Date(),
      },
      include: {
        createdBy: {
          select: {
            id: true,
            firstName: true,
            lastName: true,
          },
        },
      },
    });

    return product;
  }

  async delete(id: string) {
    // Soft delete
    await prisma.product.update({
      where: { id },
      data: { isActive: false },
    });
  }
}

export default new ProductService();
```

```typescript
// src/controllers/product.controller.ts
import { Request, Response } from 'express';
import productService from '../services/product.service';
import { asyncHandler } from '../utils/asyncHandler';

export class ProductController {
  create = asyncHandler(async (req: Request, res: Response) => {
    const product = await productService.create(req.body, req.user!.id);
    
    res.status(201).json({
      success: true,
      data: product,
    });
  });

  findAll = asyncHandler(async (req: Request, res: Response) => {
    const filters = {
      search: req.query.search as string,
      category: req.query.category as string,
      minPrice: req.query.minPrice ? parseFloat(req.query.minPrice as string) : undefined,
      maxPrice: req.query.maxPrice ? parseFloat(req.query.maxPrice as string) : undefined,
      page: req.query.page ? parseInt(req.query.page as string) : 1,
      limit: req.query.limit ? parseInt(req.query.limit as string) : 10,
      sortBy: req.query.sortBy as string,
      sortOrder: req.query.sortOrder as 'asc' | 'desc',
    };

    const result = await productService.findAll(filters);

    res.json({
      success: true,
      ...result,
    });
  });

  findOne = asyncHandler(async (req: Request, res: Response) => {
    const product = await productService.findById(req.params.id);

    res.json({
      success: true,
      data: product,
    });
  });

  update = asyncHandler(async (req: Request, res: Response) => {
    const product = await productService.update(req.params.id, req.body);

    res.json({
      success: true,
      data: product,
    });
  });

  delete = asyncHandler(async (req: Request, res: Response) => {
    await productService.delete(req.params.id);

    res.status(204).send();
  });
}

export default new ProductController();
```

```typescript
// src/routes/product.routes.ts
import { Router } from 'express';
import productController from '../controllers/product.controller';
import { authenticate } from '../middleware/auth.middleware';
import { authorize } from '../middleware/roles.middleware';
import { validateCreateProduct, validateUpdateProduct } from '../middleware/validation.middleware';

const router = Router();

// Public routes
router.get('/', productController.findAll);
router.get('/:id', productController.findOne);

// Protected routes
router.use(authenticate);
router.post('/', validateCreateProduct, productController.create);
router.patch('/:id', validateUpdateProduct, productController.update);
router.delete('/:id', authorize('admin'), productController.delete);

export default router;
```

```typescript
// src/middleware/validation.middleware.ts (additional validators)
import { body, query } from 'express-validator';

export const validateCreateProduct = [
  body('name').trim().notEmpty().withMessage('Product name is required'),
  body('description').trim().notEmpty().withMessage('Description is required'),
  body('price').isFloat({ min: 0 }).withMessage('Price must be a positive number'),
  body('stock').isInt({ min: 0 }).withMessage('Stock must be a non-negative integer'),
  body('imageUrl').optional().isURL().withMessage('Invalid image URL'),
  body('categories').isArray().withMessage('Categories must be an array'),
  validationHandler,
];

export const validateUpdateProduct = [
  body('name').optional().trim().notEmpty().withMessage('Product name cannot be empty'),
  body('price').optional().isFloat({ min: 0 }).withMessage('Price must be positive'),
  body('stock').optional().isInt({ min: 0 }).withMessage('Stock must be non-negative'),
  validationHandler,
];

export const validateProductQuery = [
  query('page').optional().isInt({ min: 1 }).withMessage('Page must be >= 1'),
  query('limit').optional().isInt({ min: 1, max: 100 }).withMessage('Limit must be 1-100'),
  query('minPrice').optional().isFloat({ min: 0 }),
  query('maxPrice').optional().isFloat({ min: 0 }),
  validationHandler,
];
```

### 2. Advanced REST Patterns

```typescript
// src/routes/advanced-patterns.ts
import { Router } from 'express';
import { asyncHandler } from '../utils/asyncHandler';

const router = Router();

// Nested resources
router.get('/users/:userId/posts', asyncHandler(async (req, res) => {
  const posts = await prisma.post.findMany({
    where: { authorId: req.params.userId },
  });
  res.json({ success: true, data: posts });
}));

// Bulk operations
router.post('/products/bulk', asyncHandler(async (req, res) => {
  const products = await prisma.product.createMany({
    data: req.body.products,
  });
  res.json({ success: true, count: products.count });
}));

// Partial updates (PATCH)
router.patch('/products/:id', asyncHandler(async (req, res) => {
  const product = await prisma.product.update({
    where: { id: req.params.id },
    data: req.body,
  });
  res.json({ success: true, data: product });
}));

// Filtering and sorting
router.get('/products', asyncHandler(async (req, res) => {
  const { sort, filter, page = 1, limit = 10 } = req.query;
  
  const products = await prisma.product.findMany({
    where: filter ? JSON.parse(filter as string) : {},
    orderBy: sort ? JSON.parse(sort as string) : { createdAt: 'desc' },
    skip: (Number(page) - 1) * Number(limit),
    take: Number(limit),
  });
  
  res.json({ success: true, data: products });
}));

export default router;
```

## Building GraphQL APIs with Express

### 1. Setup Apollo Server with Express

```bash
npm install apollo-server-express graphql
npm install @graphql-tools/schema
```

```typescript
// src/graphql/server.ts
import { ApolloServer } from 'apollo-server-express';
import { makeExecutableSchema } from '@graphql-tools/schema';
import express from 'express';
import { typeDefs } from './schema';
import { resolvers } from './resolvers';
import { GraphQLContext } from './types';

const schema = makeExecutableSchema({
  typeDefs,
  resolvers,
});

export const createApolloServer = () => {
  return new ApolloServer({
    schema,
    context: ({ req }): GraphQLContext => {
      const token = req.headers.authorization?.replace('Bearer ', '') || '';
      // Verify token and get user
      return {
        req,
        user: null, // decoded user from token
      };
    },
    formatError: (error) => {
      console.error('GraphQL Error:', error);
      return error;
    },
  });
};

// Integrate with Express
export const setupGraphQL = async (app: express.Application) => {
  const server = createApolloServer();
  await server.start();
  
  server.applyMiddleware({
    app,
    path: '/graphql',
  });

  console.log(\`🚀 GraphQL server ready at /graphql\`);
};
```

### 2. GraphQL Schema Definition

```typescript
// src/graphql/schema.ts
import { gql } from 'apollo-server-express';

export const typeDefs = gql\`
  type User {
    id: ID!
    email: String!
    firstName: String!
    lastName: String!
    products: [Product!]!
    createdAt: String!
  }

  type Product {
    id: ID!
    name: String!
    description: String!
    price: Float!
    stock: Int!
    imageUrl: String
    categories: [String!]!
    createdBy: User!
    createdAt: String!
    updatedAt: String!
  }

  type ProductConnection {
    edges: [ProductEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }

  type ProductEdge {
    node: Product!
    cursor: String!
  }

  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }

  input CreateProductInput {
    name: String!
    description: String!
    price: Float!
    stock: Int!
    imageUrl: String
    categories: [String!]!
  }

  input UpdateProductInput {
    name: String
    description: String
    price: Float
    stock: Int
    imageUrl: String
    categories: [String!]
  }

  input ProductFilter {
    search: String
    category: String
    minPrice: Float
    maxPrice: Float
  }

  type Query {
    me: User!
    user(id: ID!): User
    users(page: Int, limit: Int): [User!]!
    
    product(id: ID!): Product
    products(
      filter: ProductFilter
      first: Int
      after: String
    ): ProductConnection!
  }

  type Mutation {
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    deleteProduct(id: ID!): Boolean!
  }

  type Subscription {
    productAdded: Product!
    productUpdated(id: ID!): Product!
  }
\`;
```

### 3. GraphQL Resolvers

```typescript
// src/graphql/resolvers.ts
import { GraphQLError } from 'graphql';
import productService from '../services/product.service';
import userService from '../services/user.service';
import { GraphQLContext } from './types';

export const resolvers = {
  Query: {
    me: async (_: any, __: any, context: GraphQLContext) => {
      if (!context.user) {
        throw new GraphQLError('Not authenticated', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }
      return userService.findById(context.user.id);
    },

    user: async (_: any, { id }: { id: string }) => {
      return userService.findById(id);
    },

    users: async (_: any, { page = 1, limit = 10 }: { page: number; limit: number }) => {
      const result = await userService.findAll(page, limit);
      return result.data;
    },

    product: async (_: any, { id }: { id: string }) => {
      return productService.findById(id);
    },

    products: async (
      _: any,
      {
        filter,
        first = 10,
        after,
      }: {
        filter?: any;
        first: number;
        after?: string;
      }
    ) => {
      const page = after ? parseInt(Buffer.from(after, 'base64').toString()) + 1 : 1;
      
      const result = await productService.findAll({
        ...filter,
        page,
        limit: first,
      });

      const edges = result.data.map((product, index) => ({
        node: product,
        cursor: Buffer.from(\`\${page - 1 + index}\`).toString('base64'),
      }));

      return {
        edges,
        pageInfo: {
          hasNextPage: result.pagination.page < result.pagination.totalPages,
          hasPreviousPage: result.pagination.page > 1,
          startCursor: edges[0]?.cursor,
          endCursor: edges[edges.length - 1]?.cursor,
        },
        totalCount: result.pagination.total,
      };
    },
  },

  Mutation: {
    createProduct: async (
      _: any,
      { input }: { input: any },
      context: GraphQLContext
    ) => {
      if (!context.user) {
        throw new GraphQLError('Not authenticated', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }

      return productService.create(input, context.user.id);
    },

    updateProduct: async (
      _: any,
      { id, input }: { id: string; input: any },
      context: GraphQLContext
    ) => {
      if (!context.user) {
        throw new GraphQLError('Not authenticated', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }

      return productService.update(id, input);
    },

    deleteProduct: async (
      _: any,
      { id }: { id: string },
      context: GraphQLContext
    ) => {
      if (!context.user) {
        throw new GraphQLError('Not authenticated', {
          extensions: { code: 'UNAUTHENTICATED' },
        });
      }

      await productService.delete(id);
      return true;
    },
  },

  // Field resolvers
  User: {
    products: async (parent: any) => {
      return prisma.product.findMany({
        where: { createdById: parent.id },
      });
    },
  },

  Product: {
    createdBy: async (parent: any) => {
      return userService.findById(parent.createdById);
    },
  },
};
```

### 4. DataLoader for N+1 Prevention

```bash
npm install dataloader
```

```typescript
// src/graphql/dataloaders.ts
import DataLoader from 'dataloader';
import prisma from '../config/database';

export const createLoaders = () => {
  const userLoader = new DataLoader(async (userIds: readonly string[]) => {
    const users = await prisma.user.findMany({
      where: { id: { in: [...userIds] } },
    });

    const userMap = new Map(users.map((user) => [user.id, user]));
    return userIds.map((id) => userMap.get(id) || null);
  });

  const productsByUserLoader = new DataLoader(async (userIds: readonly string[]) => {
    const products = await prisma.product.findMany({
      where: { createdById: { in: [...userIds] } },
    });

    const productsByUser = new Map<string, any[]>();
    products.forEach((product) => {
      const existing = productsByUser.get(product.createdById) || [];
      productsByUser.set(product.createdById, [...existing, product]);
    });

    return userIds.map((id) => productsByUser.get(id) || []);
  });

  return {
    userLoader,
    productsByUserLoader,
  };
};

// Update context
export interface GraphQLContext {
  req: any;
  user: any;
  loaders: ReturnType<typeof createLoaders>;
}
```

```typescript
// Update resolver to use DataLoader
Product: {
  createdBy: async (parent: any, _: any, context: GraphQLContext) => {
    return context.loaders.userLoader.load(parent.createdById);
  },
},

User: {
  products: async (parent: any, _: any, context: GraphQLContext) => {
    return context.loaders.productsByUserLoader.load(parent.id);
  },
},
```

### 5. GraphQL Subscriptions

```bash
npm install graphql-subscriptions
```

```typescript
// src/graphql/pubsub.ts
import { PubSub } from 'graphql-subscriptions';

export const pubsub = new PubSub();

export const EVENTS = {
  PRODUCT_ADDED: 'PRODUCT_ADDED',
  PRODUCT_UPDATED: 'PRODUCT_UPDATED',
  PRODUCT_DELETED: 'PRODUCT_DELETED',
};
```

```typescript
// Update resolvers with subscriptions
import { pubsub, EVENTS } from './pubsub';

export const resolvers = {
  // ... existing resolvers

  Mutation: {
    createProduct: async (_: any, { input }: any, context: GraphQLContext) => {
      if (!context.user) {
        throw new GraphQLError('Not authenticated');
      }

      const product = await productService.create(input, context.user.id);
      
      // Publish event
      pubsub.publish(EVENTS.PRODUCT_ADDED, {
        productAdded: product,
      });

      return product;
    },
  },

  Subscription: {
    productAdded: {
      subscribe: () => pubsub.asyncIterator([EVENTS.PRODUCT_ADDED]),
    },
    productUpdated: {
      subscribe: (_: any, { id }: { id: string }) => 
        pubsub.asyncIterator([`${EVENTS.PRODUCT_UPDATED}_${id}`]),
    },
  },
};
```

## Error Handling

### REST Error Handling

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
      error: {
        message: err.message,
        code: err.statusCode,
      },
    });
  }

  // Prisma errors
  if (err.name === 'PrismaClientKnownRequestError') {
    return res.status(400).json({
      success: false,
      error: {
        message: 'Database error',
        details: err.message,
      },
    });
  }

  console.error('Unexpected error:', err);

  res.status(500).json({
    success: false,
    error: {
      message: 'Internal server error',
    },
  });
};
```

### GraphQL Error Handling

```typescript
// src/graphql/errors.ts
import { GraphQLError } from 'graphql';

export class AuthenticationError extends GraphQLError {
  constructor(message: string = 'Not authenticated') {
    super(message, {
      extensions: {
        code: 'UNAUTHENTICATED',
        http: { status: 401 },
      },
    });
  }
}

export class ForbiddenError extends GraphQLError {
  constructor(message: string = 'Forbidden') {
    super(message, {
      extensions: {
        code: 'FORBIDDEN',
        http: { status: 403 },
      },
    });
  }
}

export class NotFoundError extends GraphQLError {
  constructor(resource: string) {
    super(\`\${resource} not found\`, {
      extensions: {
        code: 'NOT_FOUND',
        http: { status: 404 },
      },
    });
  }
}
```

## Testing

### Testing REST APIs

```typescript
// src/__tests__/product.routes.test.ts
import request from 'supertest';
import app from '../server';

describe('Product API', () => {
  let authToken: string;
  let productId: string;

  beforeAll(async () => {
    const res = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    authToken = res.body.data.accessToken;
  });

  describe('POST /api/v1/products', () => {
    it('should create a new product', async () => {
      const productData = {
        name: 'Test Product',
        description: 'Test description',
        price: 99.99,
        stock: 10,
        categories: ['electronics'],
      };

      const res = await request(app)
        .post('/api/v1/products')
        .set('Authorization', \`Bearer \${authToken}\`)
        .send(productData);

      expect(res.status).toBe(201);
      expect(res.body.success).toBe(true);
      expect(res.body.data.name).toBe(productData.name);
      productId = res.body.data.id;
    });
  });

  describe('GET /api/v1/products', () => {
    it('should return products with filters', async () => {
      const res = await request(app)
        .get('/api/v1/products')
        .query({ category: 'electronics', minPrice: 50 });

      expect(res.status).toBe(200);
      expect(res.body.success).toBe(true);
      expect(Array.isArray(res.body.data)).toBe(true);
    });
  });
});
```

### Testing GraphQL APIs

```typescript
// src/__tests__/graphql.test.ts
import request from 'supertest';
import app from '../server';

describe('GraphQL API', () => {
  let authToken: string;

  beforeAll(async () => {
    const res = await request(app)
      .post('/api/v1/auth/login')
      .send({ email: 'test@example.com', password: 'password123' });
    authToken = res.body.data.accessToken;
  });

  describe('Query: products', () => {
    it('should fetch products', async () => {
      const query = \`
        query {
          products(first: 10) {
            edges {
              node {
                id
                name
                price
              }
            }
            totalCount
          }
        }
      \`;

      const res = await request(app)
        .post('/graphql')
        .set('Authorization', \`Bearer \${authToken}\`)
        .send({ query });

      expect(res.status).toBe(200);
      expect(res.body.data.products).toBeDefined();
      expect(res.body.errors).toBeUndefined();
    });
  });

  describe('Mutation: createProduct', () => {
    it('should create a product', async () => {
      const mutation = \`
        mutation CreateProduct($input: CreateProductInput!) {
          createProduct(input: $input) {
            id
            name
            price
          }
        }
      \`;

      const variables = {
        input: {
          name: 'GraphQL Product',
          description: 'Created via GraphQL',
          price: 149.99,
          stock: 5,
          categories: ['test'],
        },
      };

      const res = await request(app)
        .post('/graphql')
        .set('Authorization', \`Bearer \${authToken}\`)
        .send({ query: mutation, variables });

      expect(res.status).toBe(200);
      expect(res.body.data.createProduct).toBeDefined();
      expect(res.body.data.createProduct.name).toBe('GraphQL Product');
    });
  });
});
```

## Best Practices

### REST Best Practices

1. **Use proper HTTP methods**: GET, POST, PUT/PATCH, DELETE
2. **Return appropriate status codes**: 200, 201, 204, 400, 404, 500
3. **Version your API**: `/api/v1/`, `/api/v2/`
4. **Use query parameters for filtering**: `?search=term&page=1`
5. **Implement pagination**: Cursor or offset-based
6. **Use HATEOAS**: Include links to related resources
7. **Rate limiting**: Protect against abuse
8. **Caching headers**: ETag, Cache-Control

### GraphQL Best Practices

1. **Use DataLoader**: Prevent N+1 query problems
2. **Implement depth limiting**: Prevent deeply nested queries
3. **Add query complexity analysis**: Limit expensive queries
4. **Use cursor-based pagination**: More reliable than offset
5. **Type your schema carefully**: Clear, consistent naming
6. **Add descriptions**: Document your schema
7. **Implement proper error handling**: Use error extensions
8. **Monitor query performance**: Track slow queries

## Key Takeaways

1. **REST** is simple, cacheable, and well-understood
2. **GraphQL** offers flexibility and reduces over-fetching
3. **Express** works great with both approaches
4. **Apollo Server** seamlessly integrates with Express
5. **DataLoader** solves N+1 query problems
6. **Proper validation** is crucial for both
7. **Error handling** patterns differ between REST and GraphQL
8. **Testing** is essential for API reliability

Choose REST for simple, cacheable APIs. Choose GraphQL for complex data requirements and multiple client types.
