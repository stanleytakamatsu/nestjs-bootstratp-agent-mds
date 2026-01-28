# Repository Pattern Implementation

The Repository Pattern is a core architectural pattern that abstracts data access behind interfaces, enabling clean separation between business logic and data persistence.

## Overview

The Repository Pattern provides several key benefits:

- **Loose Coupling**: Services depend on interfaces, not concrete implementations
- **Testability**: Easy to mock repositories in unit tests
- **Flexibility**: Can switch implementations without changing business logic
- **Clean Architecture**: Clear separation between data access and business logic

## Implementation Structure

```
domains/{entity}/repositories/
├── index.ts                          # Barrel export for clean imports
├── {entity}.repository.interface.ts  # Abstract interface
└── {entity}.repository.ts            # Concrete Prisma implementation
```

## Interface Definition

Always define interfaces first, then implement them:

```typescript
// {entity}.repository.interface.ts
import { Create{Entity}Dto, {Entity}Dto, Update{Entity}Dto } from '@/domains/{entity}/dto';

// Use string token to avoid circular dependency issues
export const I{Entity}Repository = 'I{Entity}Repository';

export interface I{Entity}Repository {
  // Query operations
  findById(id: string): Promise<{Entity}Dto | null>;
  findAll(filters?: {Entity}Filters): Promise<{Entity}Dto[]>;
  count(filters?: {Entity}Filters): Promise<number>;

  // Modification operations
  create(data: Create{Entity}Dto): Promise<{Entity}Dto>;
  update(id: string, data: Update{Entity}Dto): Promise<{Entity}Dto>;
  delete(id: string): Promise<void>;

  // Domain-specific operations
  findByName(name: string): Promise<{Entity}Dto | null>;
}

// Filter types
export interface {Entity}Filters {
  name?: string;
  status?: string;
}

export interface PaginationDto {
  page?: number;
  take?: number;
}
```

## Concrete Implementation

```typescript
// {entity}.repository.ts
import { Injectable, Logger } from '@nestjs/common';
import { PrismaService } from '@/shared/infrastructure/database/prisma/prisma.service';
import {
  I{Entity}Repository,
  {Entity}Filters,
  PaginationDto,
} from '@/domains/{entity}/repositories/{entity}.repository.interface';
import {
  Create{Entity}Dto,
  {Entity}Dto,
  Update{Entity}Dto,
} from '@/domains/{entity}/dto';

@Injectable()
export class {Entity}Repository implements I{Entity}Repository {
  private readonly logger = new Logger({Entity}Repository.name);

  constructor(private readonly prismaService: PrismaService) {}

  async findById(id: string): Promise<{Entity}Dto | null> {
    return this.prismaService.{entity}.findUnique({
      where: { id },
      select: {
        id: true,
        name: true,
        status: true,
        createdAt: true,
        updatedAt: true,
        // Only select needed fields
      },
    });
  }

  async findAll(
    filters?: {Entity}Filters,
    pagination?: PaginationDto,
  ): Promise<{Entity}Dto[]> {
    const { name, status } = filters || {};
    const { page, take } = pagination || {};
    const skip = page && take ? (page - 1) * take : undefined;

    return this.prismaService.{entity}.findMany({
      where: {
        ...(name && { name: { contains: name, mode: 'insensitive' } }),
        ...(status && { status }),
      },
      skip,
      take,
      orderBy: { createdAt: 'desc' },
    });
  }

  async count(filters?: {Entity}Filters): Promise<number> {
    const { name, status } = filters || {};

    return this.prismaService.{entity}.count({
      where: {
        ...(name && { name: { contains: name, mode: 'insensitive' } }),
        ...(status && { status }),
      },
    });
  }

  async create(data: Create{Entity}Dto): Promise<{Entity}Dto> {
    return this.prismaService.{entity}.create({
      data,
    });
  }

  async update(id: string, data: Update{Entity}Dto): Promise<{Entity}Dto> {
    return this.prismaService.{entity}.update({
      where: { id },
      data,
    });
  }

  async delete(id: string): Promise<void> {
    await this.prismaService.{entity}.delete({
      where: { id },
    });
  }

  async findByName(name: string): Promise<{Entity}Dto | null> {
    return this.prismaService.{entity}.findFirst({
      where: { name },
    });
  }
}
```

## Barrel Export

```typescript
// index.ts
export * from "./{entity}.repository.interface";
export * from "./{entity}.repository";
```

## Module Registration

```typescript
// {entity}.module.ts
import { Module } from '@nestjs/common';
import { {Entity}Service } from '@/domains/{entity}/services';
import { I{Entity}Repository, {Entity}Repository } from '@/domains/{entity}/repositories';

@Module({
  providers: [
    {Entity}Service,
    {
      provide: I{Entity}Repository,  // Interface token
      useClass: {Entity}Repository,  // Implementation
    },
  ],
  exports: [{Entity}Service, I{Entity}Repository],
})
export class {Entity}Module {}
```

## Best Practices

### 1. Use Interface Tokens

Always use string tokens for interfaces to avoid circular dependencies:

```typescript
export const IUserRepository = "IUserRepository";
```

### 2. Keep Repositories Focused

Repositories should only handle data access, not business logic:

```typescript
// GOOD - Only data access
async findByEmail(email: string): Promise<User | null> {
  return this.prismaService.user.findUnique({ where: { email } });
}

// BAD - Business logic in repository
async createUserIfNotExists(email: string): Promise<User> {
  let user = await this.findByEmail(email);
  if (!user) {
    user = await this.create({ email });
  }
  return user;
}
```

### 3. Use Selective Queries

Only select the fields you need:

```typescript
async findUserSummary(id: string): Promise<UserSummary | null> {
  return this.prismaService.user.findUnique({
    where: { id },
    select: {
      id: true,
      firstName: true,
      lastName: true,
      email: true,
      // Don't select password or sensitive fields
    },
  });
}
```

### 4. Handle Pagination

```typescript
async findMany(
  filters?: EntityFilters,
  pagination?: PaginationDto,
): Promise<EntityDto[]> {
  const { page, take } = pagination || {};
  const skip = page && take ? (page - 1) * take : undefined;

  return this.prismaService.entity.findMany({
    where: filters,
    skip,
    take,
    orderBy: { createdAt: 'desc' },
  });
}
```

### 5. Batch Operations

```typescript
async updateMany(ids: string[], data: UpdateEntityDto): Promise<number> {
  const result = await this.prismaService.entity.updateMany({
    where: { id: { in: ids } },
    data,
  });
  return result.count;
}

async deleteMany(ids: string[]): Promise<number> {
  const result = await this.prismaService.entity.deleteMany({
    where: { id: { in: ids } },
  });
  return result.count;
}
```

### 6. Soft Deletes

```typescript
async softDelete(id: string): Promise<void> {
  await this.prismaService.entity.update({
    where: { id },
    data: { deletedAt: new Date() },
  });
}

async findAll(filters?: EntityFilters): Promise<EntityDto[]> {
  return this.prismaService.entity.findMany({
    where: {
      ...filters,
      deletedAt: null, // Exclude soft-deleted records
    },
  });
}
```

## Advanced Patterns

### Handling Relations

```typescript
async findWithRelations(id: string): Promise<EntityWithRelations | null> {
  return this.prismaService.entity.findUnique({
    where: { id },
    include: {
      relatedEntity: {
        select: {
          id: true,
          name: true,
        },
      },
      anotherRelation: true,
    },
  });
}
```

### Transactions

Handle transactions at the service level, not in repositories:

```typescript
// In service
async createWithRelated(data: CreateDto): Promise<Entity> {
  return this.prismaService.$transaction(async (tx) => {
    const entity = await tx.entity.create({ data: entityData });
    await tx.relatedEntity.create({ data: { entityId: entity.id, ...relatedData } });
    return entity;
  });
}
```

### Aggregations

```typescript
async countByStatus(): Promise<Record<string, number>> {
  const results = await this.prismaService.entity.groupBy({
    by: ['status'],
    _count: { status: true },
  });

  return results.reduce((acc, result) => {
    acc[result.status] = result._count.status;
    return acc;
  }, {} as Record<string, number>);
}
```

## Testing Repositories

When testing services that use repositories, mock the interface:

```typescript
// In service test
import { mock } from "bun:test";

const mockRepository = {
  findById: mock(),
  create: mock(),
  update: mock(),
  delete: mock(),
};

// Setup mock returns
mockRepository.findById.mockResolvedValue(mockEntity);

// Inject mock into service
const service = new EntityService(mockRepository as unknown as IEntityRepository);
```

## Logger Requirement

**EVERY repository class MUST have a Logger instance with the class name:**

```typescript
import { Injectable, Logger } from "@nestjs/common";

@Injectable()
export class UserRepository implements IUserRepository {
  private readonly logger = new Logger(UserRepository.name);

  constructor(private readonly prismaService: PrismaService) {}

  async findById(id: string): Promise<UserDto | null> {
    this.logger.log(`Finding user by ID: ${id}`);
    const user = await this.prismaService.user.findUnique({
      where: { id },
    });
    if (!user) {
      this.logger.debug(`User not found: ${id}`);
    }
    return user;
  }

  async create(data: CreateUserDto): Promise<UserDto> {
    this.logger.log(`Creating user with email: ${data.email}`);
    try {
      const user = await this.prismaService.user.create({ data });
      this.logger.log(`User created successfully: ${user.id}`);
      return user;
    } catch (error) {
      this.logger.error(`Failed to create user: ${error.message}`, error.stack);
      throw error;
    }
  }
}
```

## Common Mistakes to Avoid

1. **Don't use classes directly**: Always inject interfaces
2. **Don't put business logic in repositories**: Keep them focused on data access
3. **Don't fetch unnecessary data**: Use selective queries
4. **Don't forget to handle null returns**: Always check for null/undefined
5. **Don't hardcode pagination limits**: Accept them as parameters
6. **Don't forget the Logger**: Every repository must have a Logger instance
