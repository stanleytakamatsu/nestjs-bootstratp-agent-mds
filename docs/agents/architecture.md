# Architecture Patterns

This document outlines the key architectural patterns used in this NestJS backend project.

## Domain-Driven Design (DDD)

The application follows DDD principles with clear separation of concerns:

1. **Domain Isolation**: Each domain is self-contained
2. **Repository Pattern**: Abstract data access behind interfaces
3. **Service Layer**: Business logic encapsulated in services
4. **DTO Pattern**: Data transfer objects for input/output
5. **Dependency Injection**: Loose coupling between components

### Core Technologies

- **Runtime**: Bun 1.3.3+ - High-performance JavaScript runtime and package manager
- **Framework**: NestJS - Progressive Node.js framework with dependency injection
- **Server**: Fastify - High-performance HTTP server
- **Database**: PostgreSQL - Open-source relational database
- **ORM**: Prisma - Next-generation ORM with type safety
- **Validation**: Zod - TypeScript-first schema validation
- **Documentation**: Swagger/OpenAPI - Interactive API documentation

## Project Structure

The project follows a Domain-Driven Design (DDD) architecture:

```
src/
├── app.module.ts           # Root application module
├── index.ts                # Application entry point
├── controllers/            # API controllers organized by feature
│   ├── root.controller.ts  # Root controller (healthz, app info)
│   └── {entity}/           # Entity-specific controllers
│       └── {entity}.controller.ts
├── domains/                # Business domains
│   └── {entity}/
│       ├── dto/            # Data transfer objects
│       │   ├── {entity}.dto.ts
│       │   └── index.ts
│       ├── repositories/   # Data access layer
│       │   ├── {entity}.repository.interface.ts
│       │   ├── {entity}.repository.ts
│       │   └── index.ts
│       ├── services/       # Business logic
│       │   ├── {entity}.service.ts
│       │   ├── __tests__/
│       │   │   └── {entity}.service.spec.ts
│       │   └── index.ts
│       ├── {entity}.module.ts
│       └── index.ts
├── shared/                 # Shared infrastructure
│   ├── infrastructure/
│   │   └── database/
│   │       └── prisma/
│   │           └── prisma.service.ts
│   └── pipes/
│       └── zod-validation.pipe.ts
prisma/
├── schema.prisma           # Database schema definition
└── migrations/             # Database migrations
```

## Module Architecture

NestJS modules organize domains with clear dependency management:

### Module Structure

```typescript
@Module({
  imports: [
    /* Module dependencies */
  ],
  providers: [
    EntityService, // Business logic
    {
      provide: IEntityRepository, // Interface token
      useClass: EntityRepository, // Implementation
    },
  ],
  exports: [EntityService, IEntityRepository], // Public exports
})
export class EntityModule {}
```

### Key Features

- **Interface-based DI**: Services depend on interfaces, not implementations
- **Provider Registration**: Explicit mapping of interfaces to implementations
- **Dependency Resolution**: NestJS handles dependency injection automatically
- **Module Isolation**: Each domain encapsulates its own functionality

## Barrel Exports (index.ts)

**CRITICAL**: Every module folder MUST have an `index.ts` file that exports all its contents.

```typescript
// Instead of:
import { UserDto } from "@/domains/users/dto/user.dto";
import { IUserRepository } from "@/domains/users/repositories/user.repository.interface";
import { UserService } from "@/domains/users/services/user.service";

// Use barrel exports:
import { UserDto } from "@/domains/users/dto";
import { IUserRepository } from "@/domains/users/repositories";
import { UserService } from "@/domains/users/services";
```

**Apply to ALL module folders**:

- `domains/{entity}/dto/index.ts` - Export all DTOs
- `domains/{entity}/repositories/index.ts` - Export all repository interfaces and implementations
- `domains/{entity}/services/index.ts` - Export all services
- `domains/{entity}/index.ts` - Export all domain contents

**Example dto/index.ts**:

```typescript
export * from "./user.dto";
export * from "./create-user.dto";
```

**Example services/index.ts**:

```typescript
export * from "./user.service";
```

**Example domain index.ts** (at `domains/{entity}/index.ts`):

```typescript
export * from "./dto";
export * from "./repositories";
export * from "./services";
export * from "./{entity}.module";
```

## Root Module Configuration

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    SharedModule, // Global utilities
    UsersModule, // Domain modules
    ProductsModule,
  ],
  controllers: [RootController],
  providers: [PrismaService],
})
export class AppModule {}
```

## Domain Layer Responsibilities

### Controllers (Presentation Layer)

- Handle HTTP requests/responses
- Input validation using Zod pipes
- Route to appropriate services
- Transform responses

### Services (Business Logic Layer)

- Implement business rules
- Orchestrate repository calls
- Handle transactions
- Throw business exceptions

### Repositories (Data Access Layer)

- Abstract database operations
- Implement Prisma queries
- Handle data transformations
- Support filtering and pagination

### DTOs (Data Transfer Objects)

- Define input/output shapes
- Implement Zod validation
- Provide TypeScript types

## Key Architectural Decisions

1. **Stateless Services**: No state between requests
2. **Interface-based DI**: Loose coupling via dependency injection
3. **Soft Deletes**: Use `deletedAt` field instead of hard deletes
4. **UUID Identifiers**: String-based UUIDs for all entities
5. **Decimal Prices**: No floating-point for monetary values
6. **Audit Fields**: `createdAt`, `updatedAt` on all entities
7. **Logger in Every Class**: All classes must have a Logger instance

## Logging Standards

**EVERY class MUST have a Logger instance initialized with the class name.**

```typescript
import { Injectable, Logger } from "@nestjs/common";

@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async findById(id: string): Promise<UserDto> {
    this.logger.log(`Finding user by ID: ${id}`);
    const user = await this.userRepository.findById(id);
    if (!user) {
      this.logger.warn(`User not found: ${id}`);
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }
}
```

### Logger Levels

| Level     | Usage                   | Example                                   |
| --------- | ----------------------- | ----------------------------------------- |
| `log`     | General information     | Method entry, successful operations       |
| `error`   | Errors with stack trace | `this.logger.error(message, error.stack)` |
| `warn`    | Potential issues        | Missing optional data, deprecated usage   |
| `debug`   | Development details     | Request payloads, intermediate values     |
| `verbose` | Very detailed output    | Rarely used, for deep debugging           |

### Logger Pattern

```typescript
// Always use ClassName.name for the logger context
private readonly logger = new Logger(ClassName.name);
```

This ensures:

- Consistent log formatting with class context
- Automatic updates when class is renamed
- Easy filtering of logs by class/module

## Schema Patterns

The data model follows these key patterns:

- **Multi-tenancy** through organization-based data isolation (optional)
- **Audit fields** (createdAt, updatedAt) for all entities
- **Proper indexing** for optimal query performance
- **Cascading deletes** for maintaining referential integrity
- **Enum-based status management** for consistent state tracking

## Error Handling

Use NestJS built-in exceptions:

```typescript
import {
  NotFoundException,
  BadRequestException,
  ConflictException,
  UnauthorizedException,
} from '@nestjs/common';

// In service
async findById(id: string): Promise<Entity> {
  const entity = await this.repository.findById(id);
  if (!entity) {
    throw new NotFoundException(`Entity with ID ${id} not found`);
  }
  return entity;
}
```

## Response Format

Consistent response structure:

```typescript
// Success response
return {
  message: 'Operation completed successfully',
  data: { /* result */ },
};

// List response with pagination
return {
  message: 'Items retrieved successfully',
  data: {
    items: [...],
    total: 100,
    page: 1,
    totalPages: 10,
  },
};
```
