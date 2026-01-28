# Testing Patterns

This guide covers testing patterns and best practices using **Bun Test** (built-in test runner).

## Important Rules

### 1. Always Use Absolute Imports

**NEVER use relative imports in test files**. Always use the `@/` alias:

```typescript
// CORRECT
import { MyService } from "@/domains/my-domain/services/my.service";
import { IMyRepository } from "@/domains/my-domain/repositories/my.repository.interface";
import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";

// WRONG - Never do this
import { MyService } from "../my.service";
import { IMyRepository } from "../../repositories/my.repository.interface";
```

### 2. Import from `bun:test`

```typescript
import { beforeEach, describe, expect, it, mock, spyOn } from "bun:test";
```

### 3. No `await` with `.rejects.toThrow()`

In Bun test, `expect(...).rejects.toThrow()` is **synchronous** - do NOT use `await`:

```typescript
// CORRECT
expect(service.findById("unknown-id")).rejects.toThrow(NotFoundException);

// WRONG
await expect(service.findById("unknown-id")).rejects.toThrow(NotFoundException);
```

### 4. Pass `undefined` for Void Returns

When mocking functions that return `Promise<void>`, pass `undefined`:

```typescript
// CORRECT
mockRepository.delete.mockResolvedValue(undefined);

// WRONG - TypeScript error
mockRepository.delete.mockResolvedValue();
```

### 5. Use `as const` for Literal Types

When mocking data with enum-like fields, use `as const`:

```typescript
// CORRECT
const createMockEntity = () => ({
  id: "entity-id",
  status: "ACTIVE" as const,
  description: "Description" as string | null,
});

// WRONG - Type will be inferred as `string` instead of `'ACTIVE'`
const createMockEntity = () => ({
  status: "ACTIVE",
});
```

## Test Structure

Tests follow the pattern `src/**/__tests__/**/*.spec.ts`:

```
src/domains/{domain}/
├── services/
│   ├── __tests__/
│   │   └── {entity}.service.spec.ts
│   └── {entity}.service.ts
├── repositories/
│   ├── __tests__/
│   │   └── {entity}.repository.spec.ts
│   └── {entity}.repository.ts
```

## Running Tests

```bash
# Run all tests
bun run test

# Run tests matching a pattern
bun run test --filter "user"

# Run tests for a specific file
bun run test src/domains/users/services/__tests__/user.service.spec.ts

# Run tests in watch mode
bun run test --watch

# Run tests with coverage
bun run test --coverage
```

## Service Testing

Services contain business logic and should be tested with mocked dependencies.

### Basic Service Test Structure

```typescript
import { NotFoundException, BadRequestException } from "@nestjs/common";
import { beforeEach, describe, expect, it, mock } from "bun:test";

import { UserService } from "@/domains/users/services/user.service";
import { IUserRepository } from "@/domains/users/repositories/user.repository.interface";
import { UserDto } from "@/domains/users/dto";

describe("UserService", () => {
  let service: UserService;
  let mockRepository: Record<string, ReturnType<typeof mock>>;

  const mockUserId = "123e4567-e89b-12d3-a456-426614174000";

  // Helper to create mock data - use `as const` for literal types
  const createMockUser = (overrides?: Partial<UserDto>): UserDto => ({
    id: mockUserId,
    email: "test@example.com",
    firstName: "John",
    lastName: "Doe",
    status: "ACTIVE" as const,
    createdAt: new Date(),
    updatedAt: new Date(),
    ...overrides,
  });

  beforeEach(() => {
    // Create mock repository with all methods using mock()
    mockRepository = {
      findById: mock(),
      findAll: mock(),
      findByEmail: mock(),
      create: mock(),
      update: mock(),
      delete: mock(),
    };

    // Instantiate service with mocked dependencies
    service = new UserService(mockRepository as unknown as IUserRepository);
  });

  describe("findById", () => {
    it("should return user when found", async () => {
      const mockUser = createMockUser();
      mockRepository.findById.mockResolvedValue(mockUser);

      const result = await service.findById(mockUserId);

      expect(result).toEqual(mockUser);
      expect(mockRepository.findById).toHaveBeenCalledWith(mockUserId);
    });

    it("should throw NotFoundException when user not found", async () => {
      mockRepository.findById.mockResolvedValue(null);

      // NO await - Bun test handles this synchronously
      expect(service.findById("unknown-id")).rejects.toThrow(NotFoundException);
    });
  });

  describe("create", () => {
    it("should create user successfully", async () => {
      const createData = { email: "new@example.com", firstName: "Jane", lastName: "Doe" };
      const createdUser = createMockUser(createData);

      mockRepository.findByEmail.mockResolvedValue(null);
      mockRepository.create.mockResolvedValue(createdUser);

      const result = await service.create(createData);

      expect(result).toEqual(createdUser);
      expect(mockRepository.create).toHaveBeenCalledWith(createData);
    });

    it("should throw BadRequestException for duplicate email", async () => {
      const createData = { email: "existing@example.com", firstName: "Jane", lastName: "Doe" };
      mockRepository.findByEmail.mockResolvedValue(createMockUser());

      // NO await
      expect(service.create(createData)).rejects.toThrow(BadRequestException);
    });
  });

  describe("delete", () => {
    it("should delete user successfully", async () => {
      const mockUser = createMockUser();
      mockRepository.findById.mockResolvedValue(mockUser);
      // Use undefined for void returns
      mockRepository.delete.mockResolvedValue(undefined);

      await service.delete(mockUserId);

      expect(mockRepository.delete).toHaveBeenCalledWith(mockUserId);
    });

    it("should throw NotFoundException when user not found", async () => {
      mockRepository.findById.mockResolvedValue(null);

      // NO await
      expect(service.delete("unknown-id")).rejects.toThrow(NotFoundException);
    });
  });
});
```

### Testing Services with Multiple Dependencies

```typescript
import { beforeEach, describe, it, mock } from "bun:test";

import { OrderService } from "@/domains/orders/services/order.service";
import { IOrderRepository } from "@/domains/orders/repositories";
import { IProductRepository } from "@/domains/products/repositories";
import { ICustomerRepository } from "@/domains/customers/repositories";

describe("OrderService", () => {
  let service: OrderService;
  let mockOrderRepository: Record<string, ReturnType<typeof mock>>;
  let mockProductRepository: Record<string, ReturnType<typeof mock>>;
  let mockCustomerRepository: Record<string, ReturnType<typeof mock>>;

  beforeEach(() => {
    mockOrderRepository = {
      findById: mock(),
      create: mock(),
      update: mock(),
    };

    mockProductRepository = {
      findById: mock(),
      updateStock: mock(),
    };

    mockCustomerRepository = {
      findById: mock(),
    };

    service = new OrderService(
      mockOrderRepository as unknown as IOrderRepository,
      mockProductRepository as unknown as IProductRepository,
      mockCustomerRepository as unknown as ICustomerRepository,
    );
  });

  // Tests...
});
```

## Repository Testing

Repositories interact with the database via Prisma. Mock the `PrismaService` to test repository logic.

### Basic Repository Test Structure

```typescript
import { beforeEach, describe, expect, it, mock } from "bun:test";

import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";
import { UserRepository } from "@/domains/users/repositories/user.repository";

describe("UserRepository", () => {
  let repository: UserRepository;
  let mockPrismaService: {
    user: Record<string, ReturnType<typeof mock>>;
  };

  const mockUserId = "123e4567-e89b-12d3-a456-426614174000";

  const createMockUser = (overrides?: Record<string, unknown>) => ({
    id: mockUserId,
    email: "test@example.com",
    firstName: "John",
    lastName: "Doe",
    status: "ACTIVE",
    createdAt: new Date(),
    updatedAt: new Date(),
    ...overrides,
  });

  beforeEach(() => {
    mockPrismaService = {
      user: {
        findUnique: mock(),
        findFirst: mock(),
        findMany: mock(),
        create: mock(),
        update: mock(),
        delete: mock(),
        count: mock(),
      },
    };

    repository = new UserRepository(mockPrismaService as unknown as PrismaService);
  });

  describe("findById", () => {
    it("should find user by id", async () => {
      const mockUser = createMockUser();
      mockPrismaService.user.findUnique.mockResolvedValue(mockUser);

      const result = await repository.findById(mockUserId);

      expect(result).toEqual(mockUser);
      expect(mockPrismaService.user.findUnique).toHaveBeenCalledWith({
        where: { id: mockUserId },
      });
    });

    it("should return null when user not found", async () => {
      mockPrismaService.user.findUnique.mockResolvedValue(null);

      const result = await repository.findById("unknown-id");

      expect(result).toBeNull();
    });
  });

  describe("findAll with filtering", () => {
    it("should apply filters and pagination", async () => {
      const mockUsers = [createMockUser()];
      mockPrismaService.user.findMany.mockResolvedValue(mockUsers);

      const result = await repository.findAll({ status: "ACTIVE" }, { page: 1, take: 10 });

      expect(result).toEqual(mockUsers);
      expect(mockPrismaService.user.findMany).toHaveBeenCalledWith({
        where: { status: "ACTIVE" },
        skip: 0,
        take: 10,
        orderBy: expect.any(Object),
      });
    });
  });

  describe("create", () => {
    it("should create user", async () => {
      const createData = { email: "new@example.com", firstName: "Jane", lastName: "Doe" };
      const createdUser = createMockUser(createData);
      mockPrismaService.user.create.mockResolvedValue(createdUser);

      const result = await repository.create(createData);

      expect(result).toEqual(createdUser);
      expect(mockPrismaService.user.create).toHaveBeenCalledWith({
        data: createData,
      });
    });
  });

  describe("delete", () => {
    it("should delete user by id", async () => {
      mockPrismaService.user.delete.mockResolvedValue({});

      await repository.delete(mockUserId);

      expect(mockPrismaService.user.delete).toHaveBeenCalledWith({
        where: { id: mockUserId },
      });
    });
  });
});
```

## Common Testing Patterns

### Testing Validation Errors

```typescript
// NO await with rejects.toThrow() in Bun test
it("should throw BadRequestException for invalid input", () => {
  expect(service.create({ name: "" })).rejects.toThrow(BadRequestException);
  expect(service.create({ name: "" })).rejects.toThrow("Name is required");
});
```

### Testing Multiple Mock Return Values

```typescript
it("should handle sequential calls", async () => {
  mockRepository.findById
    .mockResolvedValueOnce(existingEntity) // First call
    .mockResolvedValueOnce(null); // Second call

  // First call returns entity
  const result1 = await service.findById("id-1");
  expect(result1).toEqual(existingEntity);

  // Second call returns null
  const result2 = await service.findById("id-2");
  expect(result2).toBeNull();
});
```

### Testing with expect.objectContaining

```typescript
it("should create with correct data", async () => {
  await service.create(data);

  expect(mockRepository.create).toHaveBeenCalledWith(
    expect.objectContaining({
      name: data.name,
      status: expect.any(String),
    }),
  );
});
```

### Testing Dates

```typescript
it("should set timestamps correctly", async () => {
  await repository.softDelete("entity-id");

  expect(mockPrismaService.entity.update).toHaveBeenCalledWith({
    where: { id: "entity-id" },
    data: { deletedAt: expect.any(Date) },
  });
});
```

### Testing Error Messages

```typescript
// NO await in Bun test
it("should throw with specific message", () => {
  expect(service.delete("id")).rejects.toThrow(new NotFoundException("Entity not found"));
});
```

### Spying on Methods

```typescript
import { spyOn } from "bun:test";

it("should call internal method", async () => {
  const validateSpy = spyOn(service, "validate").mockReturnValue(true);

  await service.create(data);

  expect(validateSpy).toHaveBeenCalledWith(data);
});
```

## Test File Template

Use this template when creating new test files:

```typescript
import { describe, it, expect, beforeEach, mock } from "bun:test";
import { NotFoundException } from "@nestjs/common";

import { MyService } from "@/domains/my-domain/services/my.service";
import { IMyRepository } from "@/domains/my-domain/repositories/my.repository.interface";
import { EntityDto } from "@/domains/my-domain/dto";

describe("MyService", () => {
  let service: MyService;
  let mockRepository: Record<string, ReturnType<typeof mock>>;

  const mockEntityId = "123e4567-e89b-12d3-a456-426614174000";

  // Helper functions for creating mock data
  // IMPORTANT: Include ALL required properties from the DTO/schema
  const createMockEntity = (overrides?: Partial<EntityDto>): EntityDto => ({
    id: mockEntityId,
    name: "Test Entity",
    status: "ACTIVE" as const, // Use 'as const' for literal types
    createdAt: new Date(),
    updatedAt: new Date(),
    ...overrides,
  });

  beforeEach(() => {
    // Reset and create fresh mocks using mock()
    mockRepository = {
      findById: mock(),
      create: mock(),
      update: mock(),
      delete: mock(),
    };

    service = new MyService(mockRepository as unknown as IMyRepository);
  });

  describe("methodName", () => {
    it("should handle success case", async () => {
      // Arrange
      const mockData = createMockEntity();
      mockRepository.findById.mockResolvedValue(mockData);

      // Act
      const result = await service.methodName(mockEntityId);

      // Assert
      expect(result).toEqual(mockData);
      expect(mockRepository.findById).toHaveBeenCalledWith(mockEntityId);
    });

    it("should handle error case", () => {
      // Arrange
      mockRepository.findById.mockResolvedValue(null);

      // Act & Assert - NO await with rejects.toThrow() in Bun test
      expect(service.methodName("unknown")).rejects.toThrow(NotFoundException);
    });

    it("should handle void return", async () => {
      // For void returns, pass undefined to mockResolvedValue
      mockRepository.delete.mockResolvedValue(undefined);

      await service.delete(mockEntityId);

      expect(mockRepository.delete).toHaveBeenCalledWith(mockEntityId);
    });
  });
});
```

## Best Practices Summary

1. **Use descriptive test names** - Test names should describe the expected behavior
2. **One assertion per test** - Keep tests focused on a single behavior
3. **Use helper functions** - Create reusable mock data factories with ALL required properties
4. **Use `as const` for literal types** - Ensures type safety for enum-like fields
5. **Use `mock()` from bun:test** - Never use `jest.fn()`, always use `mock()`
6. **No `await` with `rejects.toThrow()`** - Bun test handles this synchronously
7. **Pass `undefined` for void returns** - Use `mockResolvedValue(undefined)`
8. **Always use `@/` imports** - Never use relative imports in test files
9. **Test edge cases** - Include tests for null, empty, and boundary values
10. **Test error handling** - Verify exceptions are thrown with correct messages
11. **Keep tests fast** - Avoid unnecessary async operations
12. **Use meaningful mock IDs** - Use UUIDs that are easy to track in tests
