# REST API & GraphQL Development

## Overview

REST (Representational State Transfer) and GraphQL are two popular API architectural styles. REST is resource-based with fixed endpoints, while GraphQL provides a query language that allows clients to request exactly the data they need.

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

## Step-by-Step Implementation

### 1. Building a REST API with NestJS

```typescript
// products/entities/product.entity.ts
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  ManyToOne,
  CreateDateColumn,
} from "typeorm";
import { User } from "../../users/entities/user.entity";

@Entity("products")
export class Product {
  @PrimaryGeneratedColumn("uuid")
  id: string;

  @Column()
  name: string;

  @Column("text")
  description: string;

  @Column("decimal", { precision: 10, scale: 2 })
  price: number;

  @Column({ default: 0 })
  stock: number;

  @Column({ nullable: true })
  imageUrl: string;

  @Column("simple-array")
  categories: string[];

  @ManyToOne(() => User, { eager: true })
  createdBy: User;

  @CreateDateColumn()
  createdAt: Date;

  @Column({ default: true })
  isActive: boolean;
}

// products/dto/create-product.dto.ts
import {
  IsString,
  IsNumber,
  IsArray,
  IsOptional,
  IsUrl,
  Min,
} from "class-validator";
import { ApiProperty } from "@nestjs/swagger";
import { Type } from "class-transformer";

export class CreateProductDto {
  @ApiProperty()
  @IsString()
  name: string;

  @ApiProperty()
  @IsString()
  description: string;

  @ApiProperty()
  @IsNumber()
  @Min(0)
  @Type(() => Number)
  price: number;

  @ApiProperty()
  @IsNumber()
  @Min(0)
  @Type(() => Number)
  stock: number;

  @ApiProperty({ required: false })
  @IsOptional()
  @IsUrl()
  imageUrl?: string;

  @ApiProperty()
  @IsArray()
  @IsString({ each: true })
  categories: string[];
}

// products/dto/update-product.dto.ts
import { PartialType } from "@nestjs/swagger";
import { CreateProductDto } from "./create-product.dto";

export class UpdateProductDto extends PartialType(CreateProductDto) {}

// products/dto/filter-product.dto.ts
import { IsOptional, IsString, IsNumber, Min, Max } from "class-validator";
import { Type } from "class-transformer";
import { ApiPropertyOptional } from "@nestjs/swagger";

export class FilterProductDto {
  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  search?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  category?: string;

  @ApiPropertyOptional()
  @IsOptional()
  @IsNumber()
  @Min(0)
  @Type(() => Number)
  minPrice?: number;

  @ApiPropertyOptional()
  @IsOptional()
  @IsNumber()
  @Type(() => Number)
  maxPrice?: number;

  @ApiPropertyOptional({ default: 1 })
  @IsOptional()
  @IsNumber()
  @Min(1)
  @Type(() => Number)
  page?: number = 1;

  @ApiPropertyOptional({ default: 20 })
  @IsOptional()
  @IsNumber()
  @Min(1)
  @Max(100)
  @Type(() => Number)
  limit?: number = 20;

  @ApiPropertyOptional({ enum: ["price", "name", "createdAt"] })
  @IsOptional()
  @IsString()
  sortBy?: "price" | "name" | "createdAt" = "createdAt";

  @ApiPropertyOptional({ enum: ["ASC", "DESC"] })
  @IsOptional()
  @IsString()
  order?: "ASC" | "DESC" = "DESC";
}

// products/products.service.ts
import { Injectable, NotFoundException } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository, Like, Between } from "typeorm";
import { Product } from "./entities/product.entity";
import { CreateProductDto } from "./dto/create-product.dto";
import { UpdateProductDto } from "./dto/update-product.dto";
import { FilterProductDto } from "./dto/filter-product.dto";

@Injectable()
export class ProductsService {
  constructor(
    @InjectRepository(Product)
    private readonly productsRepository: Repository<Product>
  ) {}

  async create(
    createProductDto: CreateProductDto,
    userId: string
  ): Promise<Product> {
    const product = this.productsRepository.create({
      ...createProductDto,
      createdBy: { id: userId } as any,
    });

    return this.productsRepository.save(product);
  }

  async findAll(filterDto: FilterProductDto) {
    const { search, category, minPrice, maxPrice, page, limit, sortBy, order } =
      filterDto;

    const query = this.productsRepository
      .createQueryBuilder("product")
      .leftJoinAndSelect("product.createdBy", "user");

    // Apply filters
    if (search) {
      query.andWhere(
        "(product.name LIKE :search OR product.description LIKE :search)",
        { search: `%${search}%` }
      );
    }

    if (category) {
      query.andWhere("product.categories LIKE :category", {
        category: `%${category}%`,
      });
    }

    if (minPrice !== undefined) {
      query.andWhere("product.price >= :minPrice", { minPrice });
    }

    if (maxPrice !== undefined) {
      query.andWhere("product.price <= :maxPrice", { maxPrice });
    }

    // Apply sorting
    query.orderBy(`product.${sortBy}`, order);

    // Apply pagination
    const skip = (page - 1) * limit;
    query.skip(skip).take(limit);

    const [data, total] = await query.getManyAndCount();

    return {
      data,
      meta: {
        total,
        page,
        limit,
        totalPages: Math.ceil(total / limit),
      },
    };
  }

  async findOne(id: string): Promise<Product> {
    const product = await this.productsRepository.findOne({
      where: { id },
      relations: ["createdBy"],
    });

    if (!product) {
      throw new NotFoundException(`Product with ID ${id} not found`);
    }

    return product;
  }

  async update(
    id: string,
    updateProductDto: UpdateProductDto
  ): Promise<Product> {
    const product = await this.findOne(id);
    Object.assign(product, updateProductDto);
    return this.productsRepository.save(product);
  }

  async remove(id: string): Promise<void> {
    const product = await this.findOne(id);
    await this.productsRepository.remove(product);
  }
}

// products/products.controller.ts
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  Query,
  UseGuards,
  HttpCode,
  HttpStatus,
} from "@nestjs/common";
import {
  ApiTags,
  ApiOperation,
  ApiBearerAuth,
  ApiResponse,
} from "@nestjs/swagger";
import { ProductsService } from "./products.service";
import { CreateProductDto } from "./dto/create-product.dto";
import { UpdateProductDto } from "./dto/update-product.dto";
import { FilterProductDto } from "./dto/filter-product.dto";
import { JwtAuthGuard } from "../auth/guards/jwt-auth.guard";
import { Public } from "../auth/decorators/public.decorator";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@ApiTags("products")
@Controller("products")
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  @Post()
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()
  @ApiOperation({ summary: "Create a new product" })
  @ApiResponse({ status: 201, description: "Product created successfully" })
  create(
    @Body() createProductDto: CreateProductDto,
    @CurrentUser("id") userId: string
  ) {
    return this.productsService.create(createProductDto, userId);
  }

  @Get()
  @Public()
  @ApiOperation({ summary: "Get all products with filters" })
  @ApiResponse({ status: 200, description: "Products retrieved successfully" })
  findAll(@Query() filterDto: FilterProductDto) {
    return this.productsService.findAll(filterDto);
  }

  @Get(":id")
  @Public()
  @ApiOperation({ summary: "Get product by ID" })
  @ApiResponse({ status: 200, description: "Product found" })
  @ApiResponse({ status: 404, description: "Product not found" })
  findOne(@Param("id") id: string) {
    return this.productsService.findOne(id);
  }

  @Patch(":id")
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()
  @ApiOperation({ summary: "Update product" })
  update(@Param("id") id: string, @Body() updateProductDto: UpdateProductDto) {
    return this.productsService.update(id, updateProductDto);
  }

  @Delete(":id")
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: "Delete product" })
  remove(@Param("id") id: string) {
    return this.productsService.remove(id);
  }
}
```

### 2. Building a GraphQL API with NestJS

```bash
# Install dependencies
npm install @nestjs/graphql @nestjs/apollo @apollo/server graphql
```

```typescript
// app.module.ts
import { ApolloDriver, ApolloDriverConfig } from "@nestjs/apollo";
import { GraphQLModule } from "@nestjs/graphql";
import { join } from "path";

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), "src/schema.gql"),
      sortSchema: true,
      playground: true,
      context: ({ req }) => ({ req }),
    }),
    // ... other modules
  ],
})
export class AppModule {}

// products/models/product.model.ts
import { ObjectType, Field, ID, Float } from "@nestjs/graphql";
import { User } from "../../users/models/user.model";

@ObjectType()
export class Product {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;

  @Field()
  description: string;

  @Field(() => Float)
  price: number;

  @Field()
  stock: number;

  @Field({ nullable: true })
  imageUrl?: string;

  @Field(() => [String])
  categories: string[];

  @Field(() => User)
  createdBy: User;

  @Field()
  createdAt: Date;

  @Field()
  isActive: boolean;
}

// products/dto/create-product.input.ts
import { InputType, Field, Float } from "@nestjs/graphql";
import { IsString, IsNumber, IsArray, Min } from "class-validator";

@InputType()
export class CreateProductInput {
  @Field()
  @IsString()
  name: string;

  @Field()
  @IsString()
  description: string;

  @Field(() => Float)
  @IsNumber()
  @Min(0)
  price: number;

  @Field()
  @IsNumber()
  @Min(0)
  stock: number;

  @Field({ nullable: true })
  imageUrl?: string;

  @Field(() => [String])
  @IsArray()
  categories: string[];
}

// products/dto/filter-product.input.ts
import { InputType, Field, Int } from "@nestjs/graphql";

@InputType()
export class FilterProductInput {
  @Field({ nullable: true })
  search?: string;

  @Field({ nullable: true })
  category?: string;

  @Field(() => Int, { nullable: true, defaultValue: 1 })
  page?: number;

  @Field(() => Int, { nullable: true, defaultValue: 20 })
  limit?: number;
}

// products/products.resolver.ts
import {
  Resolver,
  Query,
  Mutation,
  Args,
  ID,
  ResolveField,
  Parent,
} from "@nestjs/graphql";
import { UseGuards } from "@nestjs/common";
import { ProductsService } from "./products.service";
import { Product } from "./models/product.model";
import { CreateProductInput } from "./dto/create-product.input";
import { UpdateProductInput } from "./dto/update-product.input";
import { FilterProductInput } from "./dto/filter-product.input";
import { GqlAuthGuard } from "../auth/guards/gql-auth.guard";
import { CurrentUser } from "../auth/decorators/current-user.decorator";

@Resolver(() => Product)
export class ProductsResolver {
  constructor(private readonly productsService: ProductsService) {}

  @Mutation(() => Product)
  @UseGuards(GqlAuthGuard)
  createProduct(
    @Args("input") input: CreateProductInput,
    @CurrentUser("id") userId: string
  ) {
    return this.productsService.create(input, userId);
  }

  @Query(() => [Product], { name: "products" })
  findAll(@Args("filter", { nullable: true }) filter?: FilterProductInput) {
    return this.productsService.findAll(filter || {});
  }

  @Query(() => Product, { name: "product" })
  findOne(@Args("id", { type: () => ID }) id: string) {
    return this.productsService.findOne(id);
  }

  @Mutation(() => Product)
  @UseGuards(GqlAuthGuard)
  updateProduct(
    @Args("id", { type: () => ID }) id: string,
    @Args("input") input: UpdateProductInput
  ) {
    return this.productsService.update(id, input);
  }

  @Mutation(() => Boolean)
  @UseGuards(GqlAuthGuard)
  async removeProduct(@Args("id", { type: () => ID }) id: string) {
    await this.productsService.remove(id);
    return true;
  }

  // Field resolver example
  @ResolveField()
  async similarProducts(@Parent() product: Product) {
    // Logic to find similar products
    return this.productsService.findSimilar(product.id, product.categories);
  }
}

// GraphQL queries (client-side)
/*
# Create product
mutation CreateProduct($input: CreateProductInput!) {
  createProduct(input: $input) {
    id
    name
    price
  }
}

# Get products with filters
query GetProducts($filter: FilterProductInput) {
  products(filter: $filter) {
    id
    name
    price
    stock
    categories
    createdBy {
      id
      name
    }
  }
}

# Get single product
query GetProduct($id: ID!) {
  product(id: $id) {
    id
    name
    description
    price
    stock
    imageUrl
    categories
    createdBy {
      id
      firstName
      lastName
    }
    similarProducts {
      id
      name
      price
    }
  }
}

# Update product
mutation UpdateProduct($id: ID!, $input: UpdateProductInput!) {
  updateProduct(id: $id, input: $input) {
    id
    name
    price
  }
}

# Delete product
mutation DeleteProduct($id: ID!) {
  removeProduct(id: $id)
}
*/
```

### 3. GraphQL Subscriptions (Real-time)

```typescript
// products/products.resolver.ts
import { Subscription } from "@nestjs/graphql";
import { PubSub } from "graphql-subscriptions";

const pubSub = new PubSub();

@Resolver(() => Product)
export class ProductsResolver {
  // ... other resolvers

  @Mutation(() => Product)
  async createProduct(@Args("input") input: CreateProductInput) {
    const product = await this.productsService.create(input);

    // Publish event
    pubSub.publish("productAdded", { productAdded: product });

    return product;
  }

  @Subscription(() => Product, {
    name: "productAdded",
  })
  subscribeToProductAdded() {
    return pubSub.asyncIterator("productAdded");
  }

  @Subscription(() => Product, {
    name: "productUpdated",
    filter: (payload, variables) => {
      // Only send updates for specific categories
      return payload.productUpdated.categories.includes(variables.category);
    },
  })
  subscribeToProductUpdated(
    @Args("category", { nullable: true }) category?: string
  ) {
    return pubSub.asyncIterator("productUpdated");
  }
}

// Client-side subscription
/*
subscription OnProductAdded {
  productAdded {
    id
    name
    price
  }
}

subscription OnProductUpdated($category: String) {
  productUpdated(category: $category) {
    id
    name
    price
  }
}
*/
```

### 4. DataLoader for N+1 Problem

```typescript
// common/dataloader/dataloader.service.ts
import DataLoader from "dataloader";
import { Injectable, Scope } from "@nestjs/common";
import { UsersService } from "../../users/users.service";

@Injectable({ scope: Scope.REQUEST })
export class DataloaderService {
  constructor(private readonly usersService: UsersService) {}

  // User loader
  readonly userLoader = new DataLoader<string, User>(async (userIds) => {
    const users = await this.usersService.findByIds(userIds as string[]);

    // Map users to match the order of userIds
    const userMap = new Map(users.map((user) => [user.id, user]));
    return userIds.map((id) => userMap.get(id)!);
  });
}

// products/products.resolver.ts
@Resolver(() => Product)
export class ProductsResolver {
  constructor(
    private readonly productsService: ProductsService,
    private readonly dataloaderService: DataloaderService
  ) {}

  @ResolveField(() => User)
  async createdBy(@Parent() product: Product) {
    // Use DataLoader instead of direct query
    return this.dataloaderService.userLoader.load(product.createdBy.id);
  }
}
```

### 5. REST API Rate Limiting

```typescript
// Install throttler
// npm install @nestjs/throttler

// app.module.ts
import { ThrottlerModule } from "@nestjs/throttler";

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        ttl: 60000, // 60 seconds
        limit: 10, // 10 requests per ttl
      },
    ]),
  ],
})
export class AppModule {}

// main.ts
import { ThrottlerGuard } from "@nestjs/throttler";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Apply globally
  app.useGlobalGuards(new ThrottlerGuard());

  await app.listen(3000);
}

// Custom rate limit on specific endpoint
import { Throttle } from "@nestjs/throttler";

@Controller("products")
export class ProductsController {
  @Throttle({ default: { limit: 3, ttl: 60000 } }) // 3 requests per minute
  @Post()
  create(@Body() createProductDto: CreateProductDto) {
    return this.productsService.create(createProductDto);
  }
}
```

## Best Practices

### REST API Best Practices

```typescript
// 1. Use proper HTTP methods and status codes
@Post()       // 201 Created
@Get()        // 200 OK
@Patch()      // 200 OK
@Delete()     // 204 No Content

// 2. Version your API
app.setGlobalPrefix('api/v1');

// 3. Implement pagination
interface PaginatedResponse<T> {
  data: T[];
  meta: {
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  };
}

// 4. Use filtering, sorting, and searching
@Get()
findAll(
  @Query('search') search?: string,
  @Query('sort') sort?: string,
  @Query('page') page?: number,
) {}

// 5. Return consistent error responses
{
  "statusCode": 404,
  "message": "Product not found",
  "error": "Not Found",
  "timestamp": "2024-01-13T10:00:00.000Z",
  "path": "/api/v1/products/123"
}
```

### GraphQL Best Practices

```typescript
// 1. Use Input types for mutations
@InputType()
class CreateProductInput { }

// 2. Implement field-level authorization
@ResolveField()
@UseGuards(GqlAuthGuard)
async sensitiveData(@Parent() product: Product) {}

// 3. Use DataLoader to avoid N+1 queries
// (shown in example above)

// 4. Implement pagination for lists
@ObjectType()
class PaginatedProducts {
  @Field(() => [Product])
  items: Product[];

  @Field(() => Int)
  total: number;

  @Field()
  hasMore: boolean;
}

// 5. Add descriptions to schema
@Field({ description: 'The product price in USD' })
price: number;

// 6. Handle errors gracefully
try {
  return await this.productsService.findOne(id);
} catch (error) {
  throw new GraphQLError('Product not found', {
    extensions: { code: 'PRODUCT_NOT_FOUND' },
  });
}
```

## Key Takeaways

1. **REST**: Simple, cacheable, well-suited for CRUD operations
2. **GraphQL**: Flexible, efficient, great for complex data requirements
3. **Choose based on needs**: REST for simplicity, GraphQL for flexibility
4. **Implement proper validation**: Use DTOs/Input types with class-validator
5. **Handle errors consistently**: Provide meaningful error messages
6. **Optimize queries**: Use DataLoader for GraphQL, eager loading for REST
7. **Document your API**: Swagger for REST, introspection for GraphQL
8. **Rate limiting**: Protect your API from abuse

Both REST and GraphQL have their place. REST excels at simple, cacheable operations, while GraphQL shines with complex, nested data requirements.
