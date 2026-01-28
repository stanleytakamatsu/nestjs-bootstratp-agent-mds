# Creating a New Domain: Step-by-Step Guide

This guide walks through creating a new domain in the backend, using a "Product" domain as an example.

## Prerequisites

- Familiarity with NestJS and TypeScript
- Understanding of Domain-Driven Design (DDD)
- Knowledge of Prisma ORM
- Access to the codebase

## Step 1: Database Model Definition

First, add the model to your Prisma schema:

```prisma
// prisma/schema.prisma
model Product {
  id          String        @id @default(cuid())
  name        String
  description String?
  price       Decimal       @db.Decimal(10, 2)
  status      ProductStatus @default(ACTIVE)
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  @@map("products")
  @@index([status])
}

enum ProductStatus {
  ACTIVE
  INACTIVE
  DISCONTINUED
}
```

## Step 2: Generate and Apply Migration

```bash
# Generate migration
bun run prisma:migrate --name add-products

# Regenerate Prisma client
bun run prisma:generate
```

## Step 3: Create Domain Structure

Create the domain directory structure:

```bash
mkdir -p src/domains/products/{dto,repositories,services}
mkdir -p src/domains/products/services/__tests__
mkdir -p src/domains/products/repositories/__tests__
touch src/domains/products/{dto,repositories,services}/index.ts
touch src/domains/products/products.module.ts
touch src/domains/products/index.ts
```

Final structure:

```
src/domains/products/
├── dto/
│   ├── product.dto.ts
│   └── index.ts
├── repositories/
│   ├── __tests__/
│   │   └── product.repository.spec.ts
│   ├── product.repository.interface.ts
│   ├── product.repository.ts
│   └── index.ts
├── services/
│   ├── __tests__/
│   │   └── product.service.spec.ts
│   ├── product.service.ts
│   └── index.ts
├── products.module.ts
└── index.ts
```

## Step 4: Implement DTOs

### Base DTOs

```typescript
// src/domains/products/dto/product.dto.ts
import { z } from "zod";

// Base schema
export const productSchema = z.object({
  id: z.string(),
  name: z.string(),
  description: z.string().nullable(),
  price: z.string(), // Decimal as string
  status: z.enum(["ACTIVE", "INACTIVE", "DISCONTINUED"]),
  createdAt: z.date(),
  updatedAt: z.date(),
});

// Create input schema
export const createProductSchema = z.object({
  name: z.string().min(1, "Name is required").max(255, "Name is too long"),
  description: z.string().optional(),
  price: z.string().regex(/^\d+(\.\d{1,2})?$/, "Invalid price format"),
  status: z.enum(["ACTIVE", "INACTIVE", "DISCONTINUED"]).default("ACTIVE"),
});

// Update input schema
export const updateProductSchema = z
  .object({
    name: z.string().min(1).max(255).optional(),
    description: z.string().optional(),
    price: z
      .string()
      .regex(/^\d+(\.\d{1,2})?$/)
      .optional(),
    status: z.enum(["ACTIVE", "INACTIVE", "DISCONTINUED"]).optional(),
  })
  .partial();

// Query schema
export const productQuerySchema = z.object({
  status: z.enum(["ACTIVE", "INACTIVE", "DISCONTINUED"]).optional(),
  name: z.string().optional(),
  page: z.coerce.number().int().min(1).optional(),
  take: z.coerce.number().int().min(1).max(100).optional(),
});

// TypeScript types
export type ProductDto = z.infer<typeof productSchema>;
export type CreateProductDto = z.infer<typeof createProductSchema>;
export type UpdateProductDto = z.infer<typeof updateProductSchema>;
export type ProductQueryDto = z.infer<typeof productQuerySchema>;

// Filter types
export interface ProductFilters {
  name?: string;
  status?: "ACTIVE" | "INACTIVE" | "DISCONTINUED";
}

export interface PaginationDto {
  page?: number;
  take?: number;
}
```

### Barrel Export

```typescript
// src/domains/products/dto/index.ts
export * from "./product.dto";
```

## Step 5: Implement Repository Interface

```typescript
// src/domains/products/repositories/product.repository.interface.ts
import {
  CreateProductDto,
  ProductDto,
  UpdateProductDto,
  ProductFilters,
  PaginationDto,
} from "@/domains/products/dto";

export const IProductRepository = "IProductRepository";

export interface IProductRepository {
  // CRUD operations
  findById(id: string): Promise<ProductDto | null>;
  findAll(
    filters?: ProductFilters,
    pagination?: PaginationDto,
  ): Promise<{ products: ProductDto[]; total: number }>;
  create(data: CreateProductDto): Promise<ProductDto>;
  update(id: string, data: UpdateProductDto): Promise<ProductDto>;
  delete(id: string): Promise<void>;

  // Domain-specific operations
  findByName(name: string): Promise<ProductDto | null>;
}
```

## Step 6: Implement Repository

```typescript
// src/domains/products/repositories/product.repository.ts
import { Injectable, Logger } from "@nestjs/common";
import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";
import { IProductRepository } from "@/domains/products/repositories/product.repository.interface";
import {
  CreateProductDto,
  ProductDto,
  UpdateProductDto,
  ProductFilters,
  PaginationDto,
} from "@/domains/products/dto";

@Injectable()
export class ProductRepository implements IProductRepository {
  private readonly logger = new Logger(ProductRepository.name);

  constructor(private readonly prismaService: PrismaService) {}

  async findById(id: string): Promise<ProductDto | null> {
    const product = await this.prismaService.product.findUnique({
      where: { id },
    });

    if (!product) {
      return null;
    }

    return {
      ...product,
      price: product.price.toString(),
    };
  }

  async findAll(
    filters?: ProductFilters,
    pagination?: PaginationDto,
  ): Promise<{ products: ProductDto[]; total: number }> {
    const { name, status } = filters || {};
    const { page, take } = pagination || {};
    const skip = page && take ? (page - 1) * take : undefined;

    const where = {
      ...(name && { name: { contains: name, mode: "insensitive" as const } }),
      ...(status && { status }),
    };

    const [products, total] = await Promise.all([
      this.prismaService.product.findMany({
        where,
        skip,
        take,
        orderBy: { createdAt: "desc" },
      }),
      this.prismaService.product.count({ where }),
    ]);

    return {
      products: products.map((product) => ({
        ...product,
        price: product.price.toString(),
      })),
      total,
    };
  }

  async create(data: CreateProductDto): Promise<ProductDto> {
    const product = await this.prismaService.product.create({
      data: {
        ...data,
        price: parseFloat(data.price),
      },
    });

    return {
      ...product,
      price: product.price.toString(),
    };
  }

  async update(id: string, data: UpdateProductDto): Promise<ProductDto> {
    const updateData: Record<string, unknown> = { ...data };
    if (data.price) {
      updateData.price = parseFloat(data.price);
    }

    const product = await this.prismaService.product.update({
      where: { id },
      data: updateData,
    });

    return {
      ...product,
      price: product.price.toString(),
    };
  }

  async delete(id: string): Promise<void> {
    await this.prismaService.product.delete({
      where: { id },
    });
  }

  async findByName(name: string): Promise<ProductDto | null> {
    const product = await this.prismaService.product.findFirst({
      where: { name: { equals: name, mode: "insensitive" } },
    });

    if (!product) {
      return null;
    }

    return {
      ...product,
      price: product.price.toString(),
    };
  }
}
```

### Barrel Export

```typescript
// src/domains/products/repositories/index.ts
export * from "./product.repository.interface";
export * from "./product.repository";
```

## Step 7: Implement Service

```typescript
// src/domains/products/services/product.service.ts
import { Injectable, Inject, Logger, NotFoundException, BadRequestException } from "@nestjs/common";
import { IProductRepository } from "@/domains/products/repositories";
import {
  CreateProductDto,
  ProductDto,
  UpdateProductDto,
  ProductFilters,
  PaginationDto,
} from "@/domains/products/dto";

@Injectable()
export class ProductService {
  private readonly logger = new Logger(ProductService.name);

  constructor(
    @Inject(IProductRepository)
    private readonly productRepository: IProductRepository,
  ) {}

  async findById(id: string): Promise<ProductDto> {
    this.logger.log(`Finding product by ID: ${id}`);
    const product = await this.productRepository.findById(id);
    if (!product) {
      this.logger.warn(`Product not found: ${id}`);
      throw new NotFoundException(`Product with ID ${id} not found`);
    }
    return product;
  }

  async findAll(
    filters?: ProductFilters,
    pagination?: PaginationDto,
  ): Promise<{ products: ProductDto[]; total: number }> {
    return this.productRepository.findAll(filters, pagination);
  }

  async create(data: CreateProductDto): Promise<ProductDto> {
    this.logger.log(`Creating product: ${data.name}`);

    // Business logic validation
    if (parseFloat(data.price) < 0) {
      throw new BadRequestException("Price cannot be negative");
    }

    // Check for duplicate name
    const existingProduct = await this.productRepository.findByName(data.name);
    if (existingProduct) {
      throw new BadRequestException("Product with this name already exists");
    }

    const product = await this.productRepository.create(data);
    this.logger.log(`Product created successfully: ${product.id}`);
    return product;
  }

  async update(id: string, data: UpdateProductDto): Promise<ProductDto> {
    // Check if product exists
    await this.findById(id);

    // Business logic validation
    if (data.price && parseFloat(data.price) < 0) {
      throw new BadRequestException("Price cannot be negative");
    }

    return this.productRepository.update(id, data);
  }

  async delete(id: string): Promise<void> {
    // Check if product exists
    const product = await this.findById(id);

    // Business logic: Can't delete active products
    if (product.status === "ACTIVE") {
      throw new BadRequestException("Cannot delete active products. Please deactivate first.");
    }

    await this.productRepository.delete(id);
  }
}
```

### Barrel Export

```typescript
// src/domains/products/services/index.ts
export * from "./product.service";
```

## Step 8: Create Module

```typescript
// src/domains/products/products.module.ts
import { Module } from "@nestjs/common";
import { ProductService } from "@/domains/products/services";
import { IProductRepository, ProductRepository } from "@/domains/products/repositories";

@Module({
  providers: [
    ProductService,
    {
      provide: IProductRepository,
      useClass: ProductRepository,
    },
  ],
  exports: [ProductService, IProductRepository],
})
export class ProductsModule {}
```

### Domain Barrel Export

```typescript
// src/domains/products/index.ts
export * from "./dto";
export * from "./repositories";
export * from "./services";
export * from "./products.module";
```

## Step 9: Create Controller

```typescript
// src/controllers/products/products.controller.ts
import { Body, Controller, Delete, Get, Param, Post, Patch, Query, Logger } from "@nestjs/common";
import { ApiTags, ApiOperation, ApiResponse } from "@nestjs/swagger";
import { ProductService } from "@/domains/products/services";
import { ZodValidationPipe } from "@/shared/pipes/zod-validation.pipe";
import {
  createProductSchema,
  updateProductSchema,
  productQuerySchema,
  CreateProductDto,
  UpdateProductDto,
  ProductQueryDto,
} from "@/domains/products/dto";

@Controller("products")
@ApiTags("Products")
export class ProductsController {
  private readonly logger = new Logger(ProductsController.name);

  constructor(private readonly productService: ProductService) {}

  @Get()
  @ApiOperation({ summary: "List all products" })
  @ApiResponse({ status: 200, description: "Products retrieved successfully" })
  async findAll(
    @Query(new ZodValidationPipe(productQuerySchema))
    query: ProductQueryDto,
  ) {
    const { page, take, ...filters } = query;
    const result = await this.productService.findAll(filters, { page, take });

    return {
      message: "Products retrieved successfully",
      data: result,
    };
  }

  @Get(":id")
  @ApiOperation({ summary: "Get product by ID" })
  @ApiResponse({ status: 200, description: "Product retrieved successfully" })
  @ApiResponse({ status: 404, description: "Product not found" })
  async findOne(@Param("id") id: string) {
    const product = await this.productService.findById(id);
    return {
      message: "Product retrieved successfully",
      data: product,
    };
  }

  @Post()
  @ApiOperation({ summary: "Create a new product" })
  @ApiResponse({ status: 201, description: "Product created successfully" })
  @ApiResponse({ status: 400, description: "Invalid input" })
  async create(
    @Body(new ZodValidationPipe(createProductSchema))
    data: CreateProductDto,
  ) {
    const product = await this.productService.create(data);
    return {
      message: "Product created successfully",
      data: product,
    };
  }

  @Patch(":id")
  @ApiOperation({ summary: "Update a product" })
  @ApiResponse({ status: 200, description: "Product updated successfully" })
  @ApiResponse({ status: 404, description: "Product not found" })
  async update(
    @Param("id") id: string,
    @Body(new ZodValidationPipe(updateProductSchema))
    data: UpdateProductDto,
  ) {
    const product = await this.productService.update(id, data);
    return {
      message: "Product updated successfully",
      data: product,
    };
  }

  @Delete(":id")
  @ApiOperation({ summary: "Delete a product" })
  @ApiResponse({ status: 200, description: "Product deleted successfully" })
  @ApiResponse({ status: 404, description: "Product not found" })
  @ApiResponse({ status: 400, description: "Cannot delete active product" })
  async delete(@Param("id") id: string) {
    await this.productService.delete(id);
    return {
      message: "Product deleted successfully",
    };
  }
}
```

### Controller Barrel Export

```typescript
// src/controllers/products/index.ts
export * from "./products.controller";
```

## Step 10: Update Root Module

Add the new module to the app module:

```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { RootController } from "@/controllers/root.controller";
import { ProductsController } from "@/controllers/products/products.controller";
import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";
import { ProductsModule } from "@/domains/products/products.module";

@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true }), ProductsModule],
  controllers: [RootController, ProductsController],
  providers: [PrismaService],
  exports: [PrismaService],
})
export class AppModule {}
```

## Step 11: Create Tests

### Service Test

```typescript
// src/domains/products/services/__tests__/product.service.spec.ts
import { BadRequestException, NotFoundException } from "@nestjs/common";
import { beforeEach, describe, expect, it, mock } from "bun:test";

import { ProductService } from "@/domains/products/services/product.service";
import { IProductRepository } from "@/domains/products/repositories";
import { ProductDto } from "@/domains/products/dto";

describe("ProductService", () => {
  let service: ProductService;
  let mockRepository: Record<string, ReturnType<typeof mock>>;

  const mockProductId = "123e4567-e89b-12d3-a456-426614174000";

  const createMockProduct = (overrides?: Partial<ProductDto>): ProductDto => ({
    id: mockProductId,
    name: "Test Product",
    description: "Test Description",
    price: "99.99",
    status: "ACTIVE" as const,
    createdAt: new Date(),
    updatedAt: new Date(),
    ...overrides,
  });

  beforeEach(() => {
    mockRepository = {
      findById: mock(),
      findAll: mock(),
      findByName: mock(),
      create: mock(),
      update: mock(),
      delete: mock(),
    };

    service = new ProductService(mockRepository as unknown as IProductRepository);
  });

  describe("findById", () => {
    it("should return product when found", async () => {
      const mockProduct = createMockProduct();
      mockRepository.findById.mockResolvedValue(mockProduct);

      const result = await service.findById(mockProductId);

      expect(result).toEqual(mockProduct);
      expect(mockRepository.findById).toHaveBeenCalledWith(mockProductId);
    });

    it("should throw NotFoundException when product not found", () => {
      mockRepository.findById.mockResolvedValue(null);

      expect(service.findById("unknown-id")).rejects.toThrow(NotFoundException);
    });
  });

  describe("create", () => {
    it("should create product successfully", async () => {
      const createData = { name: "New Product", price: "49.99" };
      const createdProduct = createMockProduct(createData);

      mockRepository.findByName.mockResolvedValue(null);
      mockRepository.create.mockResolvedValue(createdProduct);

      const result = await service.create(createData);

      expect(result).toEqual(createdProduct);
    });

    it("should throw BadRequestException for negative price", () => {
      const createData = { name: "Test", price: "-10" };

      expect(service.create(createData)).rejects.toThrow("Price cannot be negative");
    });

    it("should throw BadRequestException for duplicate name", () => {
      const createData = { name: "Existing Product", price: "99.99" };
      mockRepository.findByName.mockResolvedValue(createMockProduct());

      expect(service.create(createData)).rejects.toThrow("Product with this name already exists");
    });
  });

  describe("delete", () => {
    it("should delete inactive product successfully", async () => {
      const inactiveProduct = createMockProduct({ status: "INACTIVE" as const });
      mockRepository.findById.mockResolvedValue(inactiveProduct);
      mockRepository.delete.mockResolvedValue(undefined);

      await service.delete(mockProductId);

      expect(mockRepository.delete).toHaveBeenCalledWith(mockProductId);
    });

    it("should throw BadRequestException when deleting active product", () => {
      const activeProduct = createMockProduct({ status: "ACTIVE" as const });
      mockRepository.findById.mockResolvedValue(activeProduct);

      expect(service.delete(mockProductId)).rejects.toThrow("Cannot delete active products");
    });
  });
});
```

### Running Tests

```bash
# Run all product domain tests
bun run test --filter "product"

# Run only service tests
bun run test --filter "product.service"

# Run with coverage
bun run test --coverage
```

## Summary Checklist

- [ ] Add Prisma model to schema
- [ ] Run migration and generate client
- [ ] Create domain folder structure
- [ ] Implement DTOs with Zod schemas
- [ ] Create repository interface
- [ ] Implement repository
- [ ] Implement service with business logic
- [ ] Create module
- [ ] Create controller
- [ ] Update root module
- [ ] Write unit tests
- [ ] Test endpoints manually

## Common Pitfalls to Avoid

1. **Forgetting barrel exports** - Create index.ts files for clean imports
2. **Using relative imports** - Always use `@/` alias
3. **Not checking null returns** - Repository methods can return null
4. **Missing input validation** - Always validate incoming data with Zod
5. **Business logic in repositories** - Keep repositories focused on data access
6. **Forgetting to update app.module.ts** - Register new modules and controllers
7. **Forgetting Logger** - Every class (service, repository, controller) must have a Logger instance

## Logger Requirement

**IMPORTANT**: Every class must have a Logger instance with the class name:

```typescript
import { Injectable, Logger } from "@nestjs/common";

@Injectable()
export class MyService {
  private readonly logger = new Logger(MyService.name);

  // Use logger in methods
  async doSomething() {
    this.logger.log("Doing something");
  }
}
```

This applies to:

- Services
- Repositories
- Controllers
- Any other injectable class
