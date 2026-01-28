# NestJS Backend - AI Agents Documentation

> **IMPORTANT**: Before implementing any feature, you MUST read the relevant documentation:
>
> - **Creating/modifying domains**: Read `docs/agents/examples/new-domain.md`
> - **Repository implementation**: Read `docs/agents/patterns/repository.md`
> - **DTO/validation**: Read `docs/agents/patterns/dto.md`
> - **Module structure**: Read `docs/agents/patterns/modules.md`
> - **Testing**: Read `docs/agents/patterns/testing.md`
> - **Architecture questions**: Read `docs/agents/architecture.md`

## Stack

- **Runtime**: Bun 1.3.3+
- **Framework**: NestJS + Fastify
- **Database**: PostgreSQL + Prisma ORM
- **Validation**: Zod
- **Testing**: Bun Test (built-in)
- **Linting**: Oxlint
- **API Docs**: Swagger/OpenAPI

## Commands

```bash
bun install              # Install dependencies
docker-compose up -d     # Start services
bun run dev              # Dev server
bun run test             # Run tests
bun run lint             # Lint code
bun run prisma:generate  # Generate Prisma client
bun run prisma:migrate   # Run migrations
bun run prisma:studio    # View database
```

## Project Structure

```
src/
├── app.module.ts           # Root application module
├── index.ts                # Application entry point
├── controllers/            # API controllers
│   └── root.controller.ts  # Root and health endpoints
├── domains/                # Business domains (DDD)
│   └── {entity}/
│       ├── dto/            # Data transfer objects
│       ├── repositories/   # Data access layer
│       ├── services/       # Business logic
│       └── {entity}.module.ts
├── shared/                 # Shared infrastructure
│   ├── infrastructure/
│   │   └── database/
│   │       └── prisma/
│   │           └── prisma.service.ts
│   └── pipes/
│       └── zod-validation.pipe.ts
prisma/
├── schema.prisma           # Database schema
└── migrations/             # Database migrations
```

## Key Rules

1. **Follow DDD patterns** - each domain is self-contained
2. **Create barrel exports** - always add `index.ts` in module folders
3. **Use Zod schemas** for all input validation
4. **Run `bun run prisma:generate`** after schema changes
5. **Write unit tests** - all services and repositories should have tests
6. **Use absolute imports** - always use `@/` alias instead of relative imports
7. **No unnecessary comments** - only use JSDoc for public APIs and complex functions
8. **Prefer batch operations** - design APIs to handle multiple items in one call
9. **Add Logger to every class** - all services, repositories, and controllers must have a Logger instance

## Logging Rules

**EVERY class (services, repositories, controllers, etc.) MUST have a Logger instance with the class name.**

```typescript
import { Injectable, Logger } from "@nestjs/common";

@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async findById(id: string) {
    this.logger.log(`Finding user by ID: ${id}`);
    // ... implementation
  }

  async create(data: CreateUserDto) {
    this.logger.log(`Creating user with email: ${data.email}`);
    try {
      // ... implementation
      this.logger.log(`User created successfully`);
    } catch (error) {
      this.logger.error(`Failed to create user: ${error.message}`, error.stack);
      throw error;
    }
  }
}
```

### Logger Usage Guidelines

- Use `logger.log()` for general information (method entry, successful operations)
- Use `logger.error()` for errors (include error message and stack trace)
- Use `logger.warn()` for warnings (potential issues, deprecated usage)
- Use `logger.debug()` for debug information (detailed data, development only)
- Use `logger.verbose()` for verbose output (very detailed, rarely used)

### Logger Best Practices

```typescript
// CORRECT - Logger with class name
private readonly logger = new Logger(UserService.name);

// WRONG - Hardcoded string (won't update on class rename)
private readonly logger = new Logger('UserService');

// WRONG - No logger at all
// (missing logger declaration)
```

## Import Rules

**ALWAYS use absolute imports with the `@/` alias**. Never use relative imports like `../` or `../../`.

```typescript
// CORRECT - Use absolute imports
import { MyService } from "@/domains/my-domain/services/my.service";
import { MyDto } from "@/domains/my-domain/dto";
import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";

// WRONG - Never use relative imports
import { MyService } from "../services/my.service";
import { MyDto } from "../../dto";
```

This applies to ALL files including:

- Source files (`*.ts`)
- Test files (`*.spec.ts`)
- Module files (`*.module.ts`)

## Testing

Tests use **Bun Test** (built-in) and follow the pattern `src/**/__tests__/**/*.spec.ts`.

```bash
bun run test                          # Run all tests
bun run test --filter "user"          # Run tests matching pattern
bun run test --coverage               # Run with coverage report
bun run test --watch                  # Run in watch mode
```

### Test Structure

```
src/domains/{domain}/
├── services/__tests__/          # Service unit tests
├── repositories/__tests__/      # Repository unit tests
└── infrastructure/__tests__/    # External API/integration tests
```

### Key Testing Patterns

- **Import from `bun:test`** - Use `import { describe, it, expect, mock, beforeEach } from 'bun:test'`
- **Use `mock()` for mocking** - Bun's built-in mock function
- **No `await` with `.rejects.toThrow()`** - In Bun test, `expect(...).rejects.toThrow()` is synchronous
- **Pass `undefined` to void mocks** - Use `mockFn.mockResolvedValue(undefined)` for void returns
- **Use `as const`** - For literal types like `status: 'ACTIVE' as const`
- **Use helper factories** - Create `createMockEntity()` functions for test data

See `docs/agents/patterns/testing.md` for complete testing guidelines.

## API Docs

- Swagger UI: http://localhost:3001/docs

## Commit Message Guidelines

Write commit messages following this structure:

### Format

```
<type>(<scope>): <subject>

Problem: <brief description of the problem>

Solution: <detailed description of changes>
```

### Rules

1. **Subject line**: Use imperative style, limit to 50 characters
2. **Scope** (optional): Limit to 50 characters
3. **Body**: Use imperative style throughout
4. **Single line summary**: Be descriptive but keep subject to one line

### Types

- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `test`: Adding or updating tests
- `docs`: Documentation changes
- `chore`: Maintenance tasks
- `perf`: Performance improvements

### Example

```
feat(users): add user profile endpoint

Problem: Users could not view or update their profile information.

Solution: Implement GET and PATCH endpoints for user profiles with
proper validation and authorization. Added unit tests for the new
service methods.
```

## Pull Request Guidelines

Write pull request titles and descriptions following this structure:

### Title Format

```
<type>(<scope>): <subject>
```

### Description Format

```
## Problem

<Brief description of the problem being solved>

## Solution

<Detailed description of changes>

## Changes

- <List of main changes>
- <Each change on its own line>

## Testing

- <How the changes were tested>
- <Any specific test scenarios>
```

### Types

- `feat`: New feature
- `fix`: Bug fix
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `test`: Adding or updating tests
- `docs`: Documentation changes
- `chore`: Maintenance tasks
- `perf`: Performance improvements
