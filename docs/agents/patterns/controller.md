# Controller & Presentation Layer Patterns

This document covers the Controller and Presentation layer patterns for handling HTTP requests and responses.

## Overview

The Controller layer is responsible for:

- Handling HTTP requests and responses
- Input validation using Zod pipes
- Routing requests to appropriate services
- Transforming data for API responses
- API documentation with Swagger decorators

## File Structure

```
src/controllers/
├── root.controller.ts              # Root and health endpoints
├── index.ts                        # Barrel export
└── {entity}/
    ├── {entity}.controller.ts      # Entity controller
    ├── {entity}.presentation.ts    # Response DTOs and schemas
    └── index.ts                    # Barrel export
```

## Basic Controller Structure

```typescript
// src/controllers/{entity}/{entity}.controller.ts
import {
  Body,
  Controller,
  Delete,
  Get,
  Logger,
  Param,
  Patch,
  Post,
  Query,
} from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { {Entity}Service } from '@/domains/{entity}/services';
import { ZodValidationPipe } from '@/shared/pipes/zod-validation.pipe';
import {
  create{Entity}Schema,
  update{Entity}Schema,
  {entity}QuerySchema,
  Create{Entity}Dto,
  Update{Entity}Dto,
  {Entity}QueryDto,
} from '@/domains/{entity}/dto';

@Controller('{entities}')
@ApiTags('{Entities}')
export class {Entity}Controller {
  private readonly logger = new Logger({Entity}Controller.name);

  constructor(private readonly {entity}Service: {Entity}Service) {}

  @Get()
  @ApiOperation({ summary: 'List all {entities}' })
  @ApiResponse({ status: 200, description: '{Entities} retrieved successfully' })
  async findAll(
    @Query(new ZodValidationPipe({entity}QuerySchema))
    query: {Entity}QueryDto,
  ) {
    this.logger.log('Listing {entities}');
    const { page, take, ...filters } = query;
    const result = await this.{entity}Service.findAll(filters, { page, take });

    return {
      message: '{Entities} retrieved successfully',
      data: result,
    };
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get {entity} by ID' })
  @ApiResponse({ status: 200, description: '{Entity} retrieved successfully' })
  @ApiResponse({ status: 404, description: '{Entity} not found' })
  async findOne(@Param('id') id: string) {
    this.logger.log(`Finding {entity} by ID: ${id}`);
    const {entity} = await this.{entity}Service.findById(id);

    return {
      message: '{Entity} retrieved successfully',
      data: {entity},
    };
  }

  @Post()
  @ApiOperation({ summary: 'Create a new {entity}' })
  @ApiResponse({ status: 201, description: '{Entity} created successfully' })
  @ApiResponse({ status: 400, description: 'Invalid input' })
  async create(
    @Body(new ZodValidationPipe(create{Entity}Schema))
    data: Create{Entity}Dto,
  ) {
    this.logger.log(`Creating {entity}`);
    const {entity} = await this.{entity}Service.create(data);

    return {
      message: '{Entity} created successfully',
      data: {entity},
    };
  }

  @Patch(':id')
  @ApiOperation({ summary: 'Update a {entity}' })
  @ApiResponse({ status: 200, description: '{Entity} updated successfully' })
  @ApiResponse({ status: 404, description: '{Entity} not found' })
  async update(
    @Param('id') id: string,
    @Body(new ZodValidationPipe(update{Entity}Schema))
    data: Update{Entity}Dto,
  ) {
    this.logger.log(`Updating {entity}: ${id}`);
    const {entity} = await this.{entity}Service.update(id, data);

    return {
      message: '{Entity} updated successfully',
      data: {entity},
    };
  }

  @Delete(':id')
  @ApiOperation({ summary: 'Delete a {entity}' })
  @ApiResponse({ status: 200, description: '{Entity} deleted successfully' })
  @ApiResponse({ status: 404, description: '{Entity} not found' })
  async delete(@Param('id') id: string) {
    this.logger.log(`Deleting {entity}: ${id}`);
    await this.{entity}Service.delete(id);

    return {
      message: '{Entity} deleted successfully',
    };
  }
}
```

## Presentation Layer

The presentation layer defines response schemas and DTOs for API documentation.

### Presentation File Structure

```typescript
// src/controllers/{entity}/{entity}.presentation.ts
import { z } from 'zod';
import { createZodDto } from 'nestjs-zod';

// Response wrapper schema
const ApiResponseSchema = <T extends z.ZodTypeAny>(dataSchema: T) =>
  z.object({
    message: z.string(),
    data: dataSchema.optional(),
  });

// Entity public schema (what the API returns)
export const {Entity}PublicSchema = z.object({
  id: z.string(),
  name: z.string(),
  status: z.enum(['ACTIVE', 'INACTIVE']),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

// Single item response
export const {Entity}ResponseSchema = ApiResponseSchema({Entity}PublicSchema);
export class {Entity}ResponseDto extends createZodDto({Entity}ResponseSchema) {}

// List response with pagination
export const {Entity}ListDataSchema = z.object({
  {entities}: z.array({Entity}PublicSchema),
  total: z.number(),
  page: z.number().optional(),
  totalPages: z.number().optional(),
});

export const {Entity}ListResponseSchema = ApiResponseSchema({Entity}ListDataSchema);
export class {Entity}ListResponseDto extends createZodDto({Entity}ListResponseSchema) {}

// Types
export type {Entity}Public = z.infer<typeof {Entity}PublicSchema>;
export type {Entity}Response = z.infer<typeof {Entity}ResponseSchema>;
export type {Entity}ListResponse = z.infer<typeof {Entity}ListResponseSchema>;
```

## Response Format

### Standard Response Structure

All API responses follow a consistent format:

```typescript
// Success response with data
{
  "message": "Operation completed successfully",
  "data": { /* result */ }
}

// Success response without data (e.g., delete)
{
  "message": "Entity deleted successfully"
}

// List response with pagination
{
  "message": "Items retrieved successfully",
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "totalPages": 10
  }
}
```

### Response Helper

```typescript
// src/shared/presentation/response.dto.ts
import { z } from "zod";

export const ApiResponse = <T extends z.ZodTypeAny>(dataSchema: T) =>
  z.object({
    message: z.string(),
    data: dataSchema.optional(),
  });

export interface ApiResponseType<T> {
  message: string;
  data?: T;
}
```

## Swagger Documentation

### Basic Decorators

```typescript
import { ApiTags, ApiOperation, ApiResponse, ApiParam, ApiQuery, ApiBody } from "@nestjs/swagger";

@Controller("users")
@ApiTags("Users") // Groups endpoints in Swagger UI
export class UsersController {
  @Get(":id")
  @ApiOperation({ summary: "Get user by ID" })
  @ApiParam({ name: "id", description: "User ID", type: String })
  @ApiResponse({ status: 200, description: "User found" })
  @ApiResponse({ status: 404, description: "User not found" })
  async findOne(@Param("id") id: string) {
    // ...
  }

  @Get()
  @ApiOperation({ summary: "List users" })
  @ApiQuery({ name: "status", required: false, enum: ["ACTIVE", "INACTIVE"] })
  @ApiQuery({ name: "page", required: false, type: Number })
  @ApiQuery({ name: "take", required: false, type: Number })
  async findAll(@Query() query: UserQueryDto) {
    // ...
  }

  @Post()
  @ApiOperation({ summary: "Create user" })
  @ApiBody({ type: CreateUserDto })
  @ApiResponse({ status: 201, description: "User created" })
  @ApiResponse({ status: 400, description: "Invalid input" })
  async create(@Body() data: CreateUserDto) {
    // ...
  }
}
```

### Response Schema Documentation

```typescript
@Get(':id')
@ApiOperation({ summary: 'Get user by ID' })
@ApiResponse({
  status: 200,
  description: 'User retrieved successfully',
  schema: {
    type: 'object',
    properties: {
      message: { type: 'string', example: 'User retrieved successfully' },
      data: {
        type: 'object',
        properties: {
          id: { type: 'string', example: 'cuid123' },
          email: { type: 'string', example: 'user@example.com' },
          name: { type: 'string', example: 'John Doe' },
          status: { type: 'string', enum: ['ACTIVE', 'INACTIVE'] },
          createdAt: { type: 'string', format: 'date-time' },
        },
      },
    },
  },
})
async findOne(@Param('id') id: string) {
  // ...
}
```

## Input Validation

### Using ZodValidationPipe

```typescript
import { ZodValidationPipe } from '@/shared/pipes/zod-validation.pipe';
import { createUserSchema, CreateUserDto } from '@/domains/users/dto';

@Post()
async create(
  @Body(new ZodValidationPipe(createUserSchema))
  data: CreateUserDto,
) {
  return this.userService.create(data);
}

@Get()
async findAll(
  @Query(new ZodValidationPipe(userQuerySchema))
  query: UserQueryDto,
) {
  return this.userService.findAll(query);
}
```

### Path Parameters

```typescript
@Get(':id')
async findOne(@Param('id') id: string) {
  // id is always a string from URL
  return this.userService.findById(id);
}

@Get(':userId/posts/:postId')
async findUserPost(
  @Param('userId') userId: string,
  @Param('postId') postId: string,
) {
  return this.postService.findByUserAndId(userId, postId);
}
```

## Error Handling

Controllers should let services throw exceptions. NestJS handles them automatically:

```typescript
// In service (throws exception)
async findById(id: string): Promise<UserDto> {
  const user = await this.userRepository.findById(id);
  if (!user) {
    throw new NotFoundException(`User with ID ${id} not found`);
  }
  return user;
}

// In controller (no try-catch needed)
@Get(':id')
async findOne(@Param('id') id: string) {
  // NotFoundException from service will be handled by NestJS
  const user = await this.userService.findById(id);
  return {
    message: 'User retrieved successfully',
    data: user,
  };
}
```

### Custom Error Responses

For custom error handling, use exception filters:

```typescript
// src/shared/filters/http-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException, Logger } from "@nestjs/common";
import { FastifyReply } from "fastify";

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<FastifyReply>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    this.logger.error(`HTTP Exception: ${JSON.stringify(exceptionResponse)}`);

    response.status(status).send({
      message:
        typeof exceptionResponse === "string"
          ? exceptionResponse
          : (exceptionResponse as any).message || "An error occurred",
      error: exception.name,
      statusCode: status,
    });
  }
}
```

## Route Naming Conventions

| Operation       | HTTP Method | Route                      | Example               |
| --------------- | ----------- | -------------------------- | --------------------- |
| List all        | GET         | `/{entities}`              | `/users`              |
| Get one         | GET         | `/{entities}/:id`          | `/users/:id`          |
| Create          | POST        | `/{entities}`              | `/users`              |
| Update          | PATCH       | `/{entities}/:id`          | `/users/:id`          |
| Replace         | PUT         | `/{entities}/:id`          | `/users/:id`          |
| Delete          | DELETE      | `/{entities}/:id`          | `/users/:id`          |
| Custom action   | POST        | `/{entities}/:id/{action}` | `/users/:id/activate` |
| Nested resource | GET         | `/{entities}/:id/{nested}` | `/users/:id/posts`    |

## Barrel Export

```typescript
// src/controllers/{entity}/index.ts
export * from "./{entity}.controller";
export * from "./{entity}.presentation";
```

```typescript
// src/controllers/index.ts
export * from "./root.controller";
export * from "./{entity}";
```

## Registering Controllers

```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { RootController } from "@/controllers/root.controller";
import { UsersController } from "@/controllers/users/users.controller";
import { UsersModule } from "@/domains/users/users.module";

@Module({
  imports: [UsersModule],
  controllers: [RootController, UsersController],
})
export class AppModule {}
```

## Best Practices

### 1. Always Include Logger

```typescript
export class UsersController {
  private readonly logger = new Logger(UsersController.name);
  // ...
}
```

### 2. Use Consistent Response Format

```typescript
// Always return { message, data }
return {
  message: "Operation successful",
  data: result,
};
```

### 3. Validate All Inputs

```typescript
// Body
@Body(new ZodValidationPipe(schema)) data: Dto

// Query
@Query(new ZodValidationPipe(querySchema)) query: QueryDto
```

### 4. Keep Controllers Thin

Controllers should only:

- Validate input
- Call service methods
- Format responses

Business logic belongs in services.

### 5. Use Descriptive Route Names

```typescript
// Good
@Controller('users')
@Controller('user-profiles')
@Controller('order-items')

// Bad
@Controller('usr')
@Controller('data')
```

### 6. Document All Endpoints

Every endpoint should have:

- `@ApiOperation` with summary
- `@ApiResponse` for success and error cases

## Common Patterns

### Pagination Response

```typescript
@Get()
async findAll(@Query() query: QueryDto) {
  const { page = 1, take = 10, ...filters } = query;
  const { items, total } = await this.service.findAll(filters, { page, take });
  const totalPages = Math.ceil(total / take);

  return {
    message: 'Items retrieved successfully',
    data: {
      items,
      total,
      page,
      totalPages,
    },
  };
}
```

### Bulk Operations

```typescript
@Post('bulk')
@ApiOperation({ summary: 'Create multiple items' })
async createBulk(
  @Body(new ZodValidationPipe(bulkCreateSchema))
  data: BulkCreateDto,
) {
  const items = await this.service.createMany(data.items);

  return {
    message: `${items.length} items created successfully`,
    data: { items, count: items.length },
  };
}

@Delete('bulk')
@ApiOperation({ summary: 'Delete multiple items' })
async deleteBulk(
  @Body(new ZodValidationPipe(bulkDeleteSchema))
  data: BulkDeleteDto,
) {
  const count = await this.service.deleteMany(data.ids);

  return {
    message: `${count} items deleted successfully`,
    data: { count },
  };
}
```

### File Upload

```typescript
import { FastifyRequest } from 'fastify';

@Post('upload')
@ApiOperation({ summary: 'Upload file' })
async upload(@Req() request: FastifyRequest) {
  const file = await request.file();

  if (!file) {
    throw new BadRequestException('No file provided');
  }

  const result = await this.service.processFile(file);

  return {
    message: 'File uploaded successfully',
    data: result,
  };
}
```
