# Prisma & Database Infrastructure Patterns

This document covers the Prisma ORM setup and shared database infrastructure patterns.

## Overview

The database layer uses:

- **Prisma ORM 7.x** - Type-safe database client with driver adapters
- **PostgreSQL** - Relational database
- **PrismaService** - NestJS injectable wrapper with pg adapter

## File Structure

```
src/shared/infrastructure/database/
├── prisma/
│   ├── prisma.service.ts    # PrismaService class
│   └── index.ts             # Barrel export
├── index.ts                 # Barrel export
prisma/
├── schema.prisma            # Database schema definition
├── prisma.config.ts         # Prisma configuration (Prisma 7+)
├── migrations/              # Database migrations
└── seed.ts                  # Database seeding (optional)
```

## Prisma 7 Configuration

### prisma.config.ts

Prisma 7 requires a configuration file for driver adapters:

```typescript
// prisma/prisma.config.ts
import path from "node:path";
import type { PrismaConfig } from "prisma";

export default {
  earlyAccess: true,
  schema: path.join("prisma", "schema.prisma"),
  migrate: {
    // Use DIRECT_URL for migrations (bypasses connection poolers in serverless)
    url: process.env.DIRECT_URL,
  },
} satisfies PrismaConfig;
```

### Environment Variables

```env
# DATABASE_URL: Used by the application (supports connection pooling)
DATABASE_URL=postgresql://user:pass@localhost:6433/mydb

# DIRECT_URL: Direct connection for migrations (bypasses poolers)
# Required for serverless deployments (Neon, Supabase, etc.)
DIRECT_URL=postgresql://user:pass@localhost:6433/mydb
```

**Why two URLs?**

In serverless environments, database connections often go through a connection pooler (e.g., PgBouncer). While poolers are efficient for application queries, they don't support the transaction-based operations that Prisma migrations require.

| Environment | DATABASE_URL                | DIRECT_URL                 |
| ----------- | --------------------------- | -------------------------- |
| Local Dev   | Direct connection           | Same as DATABASE_URL       |
| Neon        | Pooled (`-pooler` hostname) | Direct (without `-pooler`) |
| Supabase    | Pooler (port 6543)          | Direct (port 5432)         |

### Required Dependencies

```json
{
  "dependencies": {
    "@prisma/adapter-pg": "7.3.0",
    "@prisma/client": "7.3.0",
    "pg": "^8.16.0"
  },
  "devDependencies": {
    "@types/pg": "^8.15.4",
    "prisma": "7.3.0"
  }
}
```

**IMPORTANT**: Keep all Prisma packages (`prisma`, `@prisma/client`, `@prisma/adapter-pg`) at the same version.

### Trust Prisma Postinstall Scripts

Prisma requires postinstall scripts. Trust them with bun:

```bash
bun pm trust prisma
bun pm trust @prisma/client
bun pm trust @prisma/adapter-pg
bun install
```

## PrismaService

The `PrismaService` is a shared infrastructure class that wraps the Prisma client using the driver adapter pattern (required for Prisma 7).

### Implementation

```typescript
// src/shared/infrastructure/database/prisma/prisma.service.ts
import { Injectable, Logger, OnModuleInit, OnModuleDestroy } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";
import { Pool } from "pg";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(PrismaService.name);
  private pool: Pool;

  constructor() {
    const pool = new Pool({
      connectionString: process.env.DATABASE_URL,
    });

    const adapter = new PrismaPg(pool);

    super({
      adapter,
      log: [
        { emit: "event", level: "query" },
        { emit: "event", level: "error" },
        { emit: "event", level: "warn" },
      ],
    });

    this.pool = pool;
  }

  async onModuleInit() {
    this.logger.log("Connecting to database...");
    await this.$connect();
    this.logger.log("Database connected successfully");

    // Optional: Log queries in development
    if (process.env.NODE_ENV === "development") {
      // @ts-ignore
      this.$on("query", (e: any) => {
        this.logger.debug(`Query: ${e.query}`);
        this.logger.debug(`Duration: ${e.duration}ms`);
      });
    }
  }

  async onModuleDestroy() {
    this.logger.log("Disconnecting from database...");
    await this.$disconnect();
    await this.pool.end();
    this.logger.log("Database disconnected");
  }

  /**
   * Clean database for testing purposes
   * WARNING: Only use in test environment
   */
  async cleanDatabase() {
    if (process.env.NODE_ENV !== "test") {
      throw new Error("cleanDatabase can only be used in test environment");
    }

    const models = Reflect.ownKeys(this).filter(
      (key) => typeof key === "string" && !key.startsWith("_") && !key.startsWith("$"),
    );

    return Promise.all(
      models.map((modelKey) => {
        // @ts-ignore
        return this[modelKey]?.deleteMany?.();
      }),
    );
  }
}
```

### Barrel Exports

```typescript
// src/shared/infrastructure/database/prisma/index.ts
export * from "./prisma.service";
```

```typescript
// src/shared/infrastructure/database/index.ts
export * from "./prisma";
```

```typescript
// src/shared/infrastructure/index.ts
export * from "./database";
```

```typescript
// src/shared/index.ts
export * from "./infrastructure";
export * from "./pipes";
```

## Prisma Schema

### Basic Schema Structure (Prisma 7)

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
}

// Example model
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  firstName String   @map("first_name")
  lastName  String   @map("last_name")
  status    UserStatus @default(ACTIVE)
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  // Relations
  posts     Post[]

  @@map("users")
  @@index([status])
  @@index([email])
}

model Post {
  id        String   @id @default(cuid())
  title     String
  content   String?
  published Boolean  @default(false)
  authorId  String   @map("author_id")
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  // Relations
  author    User     @relation(fields: [authorId], references: [id], onDelete: Cascade)

  @@map("posts")
  @@index([authorId])
}

enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
}
```

**IMPORTANT Prisma 7 Changes:**

- The `url` property is no longer used in the datasource block
- The `previewFeatures = ["driverAdapters"]` is no longer needed (now stable)
- Database connection is configured via the adapter in PrismaService

### Schema Conventions

1. **Model naming**: PascalCase singular (e.g., `User`, `Post`)
2. **Table naming**: Use `@@map("table_name")` with snake_case plural
3. **Field naming**: camelCase in Prisma, snake_case in DB with `@map()`
4. **IDs**: Use `cuid()` for string IDs
5. **Timestamps**: Always include `createdAt` and `updatedAt`
6. **Indexes**: Add `@@index` for frequently queried fields
7. **Relations**: Define both sides of the relationship

## Using PrismaService in Repositories

```typescript
// src/domains/users/repositories/user.repository.ts
import { Injectable, Logger } from "@nestjs/common";
import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";
import { IUserRepository } from "@/domains/users/repositories/user.repository.interface";
import { CreateUserDto, UserDto, UpdateUserDto } from "@/domains/users/dto";

@Injectable()
export class UserRepository implements IUserRepository {
  private readonly logger = new Logger(UserRepository.name);

  constructor(private readonly prisma: PrismaService) {}

  async findById(id: string): Promise<UserDto | null> {
    this.logger.log(`Finding user by ID: ${id}`);
    return this.prisma.user.findUnique({
      where: { id },
    });
  }

  async findByEmail(email: string): Promise<UserDto | null> {
    this.logger.log(`Finding user by email: ${email}`);
    return this.prisma.user.findUnique({
      where: { email },
    });
  }

  async findAll(): Promise<UserDto[]> {
    this.logger.log("Finding all users");
    return this.prisma.user.findMany({
      orderBy: { createdAt: "desc" },
    });
  }

  async create(data: CreateUserDto): Promise<UserDto> {
    this.logger.log(`Creating user: ${data.email}`);
    return this.prisma.user.create({
      data,
    });
  }

  async update(id: string, data: UpdateUserDto): Promise<UserDto> {
    this.logger.log(`Updating user: ${id}`);
    return this.prisma.user.update({
      where: { id },
      data,
    });
  }

  async delete(id: string): Promise<void> {
    this.logger.log(`Deleting user: ${id}`);
    await this.prisma.user.delete({
      where: { id },
    });
  }
}
```

## Transactions

### Basic Transaction

```typescript
async createUserWithProfile(data: CreateUserWithProfileDto): Promise<User> {
  return this.prisma.$transaction(async (tx) => {
    const user = await tx.user.create({
      data: {
        email: data.email,
        firstName: data.firstName,
        lastName: data.lastName,
      },
    });

    await tx.profile.create({
      data: {
        userId: user.id,
        bio: data.bio,
        avatar: data.avatar,
      },
    });

    return user;
  });
}
```

### Transaction with Isolation Level

```typescript
async transferBalance(fromId: string, toId: string, amount: number): Promise<void> {
  await this.prisma.$transaction(
    async (tx) => {
      const from = await tx.account.update({
        where: { id: fromId },
        data: { balance: { decrement: amount } },
      });

      if (from.balance < 0) {
        throw new Error('Insufficient balance');
      }

      await tx.account.update({
        where: { id: toId },
        data: { balance: { increment: amount } },
      });
    },
    {
      isolationLevel: 'Serializable',
      maxWait: 5000,
      timeout: 10000,
    },
  );
}
```

## Query Patterns

### Filtering

```typescript
async findAll(filters?: UserFilters): Promise<UserDto[]> {
  const { status, search } = filters || {};

  return this.prisma.user.findMany({
    where: {
      ...(status && { status }),
      ...(search && {
        OR: [
          { firstName: { contains: search, mode: 'insensitive' } },
          { lastName: { contains: search, mode: 'insensitive' } },
          { email: { contains: search, mode: 'insensitive' } },
        ],
      }),
    },
  });
}
```

### Pagination

```typescript
async findAllPaginated(
  filters?: UserFilters,
  pagination?: PaginationDto,
): Promise<{ users: UserDto[]; total: number }> {
  const { page = 1, take = 10 } = pagination || {};
  const skip = (page - 1) * take;

  const [users, total] = await Promise.all([
    this.prisma.user.findMany({
      where: filters,
      skip,
      take,
      orderBy: { createdAt: 'desc' },
    }),
    this.prisma.user.count({ where: filters }),
  ]);

  return { users, total };
}
```

### Include Relations

```typescript
async findWithPosts(id: string): Promise<UserWithPosts | null> {
  return this.prisma.user.findUnique({
    where: { id },
    include: {
      posts: {
        where: { published: true },
        orderBy: { createdAt: 'desc' },
        take: 10,
      },
    },
  });
}
```

### Select Specific Fields

```typescript
async findUserSummary(id: string): Promise<UserSummary | null> {
  return this.prisma.user.findUnique({
    where: { id },
    select: {
      id: true,
      firstName: true,
      lastName: true,
      email: true,
      // Exclude sensitive fields
    },
  });
}
```

### Aggregations

```typescript
async countByStatus(): Promise<Record<string, number>> {
  const results = await this.prisma.user.groupBy({
    by: ['status'],
    _count: { status: true },
  });

  return results.reduce((acc, result) => {
    acc[result.status] = result._count.status;
    return acc;
  }, {} as Record<string, number>);
}
```

## Soft Deletes

### Schema with Soft Delete

```prisma
model User {
  id        String    @id @default(cuid())
  email     String    @unique
  deletedAt DateTime? @map("deleted_at")
  // ... other fields

  @@map("users")
  @@index([deletedAt])
}
```

### Soft Delete Implementation

```typescript
async softDelete(id: string): Promise<void> {
  this.logger.log(`Soft deleting user: ${id}`);
  await this.prisma.user.update({
    where: { id },
    data: { deletedAt: new Date() },
  });
}

async findAll(): Promise<UserDto[]> {
  return this.prisma.user.findMany({
    where: { deletedAt: null }, // Exclude soft-deleted
    orderBy: { createdAt: 'desc' },
  });
}

async restore(id: string): Promise<void> {
  this.logger.log(`Restoring user: ${id}`);
  await this.prisma.user.update({
    where: { id },
    data: { deletedAt: null },
  });
}
```

## Migration Commands

```bash
# Create a new migration (development)
bun run prisma:migrate --name add_users_table

# Apply migrations (development)
bun run prisma:migrate

# Apply pending migrations (production/CI)
bun run prisma:migrate:deploy

# Check migration status
bun run prisma:migrate:status

# Reset database (WARNING: deletes all data)
bun run prisma:migrate:reset

# Generate Prisma client after schema changes
bun run prisma:generate

# Open Prisma Studio (database GUI)
bun run prisma:studio

# Push schema changes without migration (development only)
bun run db:push
```

### Serverless Deployment Migrations

When deploying to serverless platforms (Neon, Supabase), use the DIRECT_URL for migrations:

```bash
# In CI/CD pipeline
bun run prisma:migrate:deploy
```

The `prisma.config.ts` automatically uses `DIRECT_URL` for migrations, bypassing connection poolers.

## Module Registration

### Shared Module (Recommended)

```typescript
// src/shared/shared.module.ts
import { Global, Module } from "@nestjs/common";
import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class SharedModule {}
```

### App Module

```typescript
// src/app.module.ts
import { Module } from "@nestjs/common";
import { SharedModule } from "@/shared/shared.module";

@Module({
  imports: [
    SharedModule, // PrismaService available globally
    // ... other modules
  ],
})
export class AppModule {}
```

## Best Practices

### 1. Always Use Logger

```typescript
private readonly logger = new Logger(UserRepository.name);
```

### 2. Use Selective Queries

```typescript
// Good - only select needed fields
const user = await this.prisma.user.findUnique({
  where: { id },
  select: { id: true, email: true, firstName: true },
});

// Avoid - fetches all fields
const user = await this.prisma.user.findUnique({
  where: { id },
});
```

### 3. Add Indexes for Queried Fields

```prisma
@@index([status])
@@index([email])
@@index([createdAt])
```

### 4. Use Transactions for Multiple Operations

```typescript
await this.prisma.$transaction([
  this.prisma.user.update({ ... }),
  this.prisma.profile.update({ ... }),
]);
```

### 5. Handle Prisma Errors

```typescript
import { PrismaClientKnownRequestError } from '@prisma/client/runtime/library';

async create(data: CreateUserDto): Promise<UserDto> {
  try {
    return await this.prisma.user.create({ data });
  } catch (error) {
    if (error instanceof PrismaClientKnownRequestError) {
      if (error.code === 'P2002') {
        throw new ConflictException('Email already exists');
      }
    }
    throw error;
  }
}
```

### 6. Use Enums for Status Fields

```prisma
enum UserStatus {
  ACTIVE
  INACTIVE
  SUSPENDED
}

model User {
  status UserStatus @default(ACTIVE)
}
```

### 7. Close Pool on Shutdown

When using the driver adapter pattern, ensure the pool is properly closed:

```typescript
async onModuleDestroy() {
  await this.$disconnect();
  await this.pool.end(); // Important: close the pg pool
}
```

## Common Prisma Error Codes

| Code  | Description                   |
| ----- | ----------------------------- |
| P2002 | Unique constraint violation   |
| P2003 | Foreign key constraint failed |
| P2025 | Record not found              |
| P2014 | Required relation violation   |
| P2016 | Query interpretation error    |
