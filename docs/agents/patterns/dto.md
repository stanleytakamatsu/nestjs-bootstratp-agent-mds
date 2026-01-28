# DTO Implementation Patterns

This document covers Data Transfer Object (DTO) patterns using Zod for validation.

## Overview

DTOs serve multiple purposes:

- **Type Safety**: Ensure data conforms to expected shapes
- **Validation**: Validate incoming data before processing
- **Transformation**: Convert data between different layers
- **Documentation**: Serve as API contracts

## File Structure

```
domains/{entity}/dto/
├── {entity}.dto.ts        # Main entity DTOs
├── create-{entity}.dto.ts # Creation-specific DTOs (optional)
└── index.ts               # Barrel exports
```

## Core DTO Pattern

### Basic Schema Definition

```typescript
// {entity}.dto.ts
import { z } from 'zod';

// 1. Define the base schema
export const {entity}Schema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(255),
  description: z.string().optional(),
  status: z.enum(['ACTIVE', 'INACTIVE']),
  createdAt: z.date(),
  updatedAt: z.date(),
});

// 2. Create specialized schemas for different use cases
export const {entity}WithRelationsSchema = {entity}Schema.extend({
  relatedItems: z.array(z.object({
    id: z.string(),
    name: z.string(),
  })).optional(),
});

// 3. Create input validation schemas
export const create{Entity}Schema = z.object({
  name: z.string().min(1, 'Name is required').max(255, 'Name is too long'),
  description: z.string().optional(),
  status: z.enum(['ACTIVE', 'INACTIVE']).default('ACTIVE'),
});

export const update{Entity}Schema = z.object({
  name: z.string().min(1).max(255).optional(),
  description: z.string().optional(),
  status: z.enum(['ACTIVE', 'INACTIVE']).optional(),
}).partial();

// 4. Export TypeScript types
export type {Entity}Dto = z.infer<typeof {entity}Schema>;
export type {Entity}WithRelationsDto = z.infer<typeof {entity}WithRelationsSchema>;
export type Create{Entity}Dto = z.infer<typeof create{Entity}Schema>;
export type Update{Entity}Dto = z.infer<typeof update{Entity}Schema>;

// 5. Filter types
export interface {Entity}Filters {
  name?: string;
  status?: 'ACTIVE' | 'INACTIVE';
}
```

### Barrel Export

```typescript
// index.ts
export * from "./{entity}.dto";
```

## Input Validation Schemas

### Create Input

```typescript
export const createUserSchema = z.object({
  // Required fields with validation messages
  email: z.string().min(1, "Email is required").email("Invalid email format"),

  firstName: z.string().min(1, "First name is required").max(100, "First name is too long"),

  lastName: z.string().min(1, "Last name is required").max(100, "Last name is too long"),

  // Optional fields
  phone: z.string().optional(),

  // Fields with defaults
  role: z.enum(["USER", "ADMIN"]).default("USER"),
});

export type CreateUserDto = z.infer<typeof createUserSchema>;
```

### Update Input

```typescript
export const updateUserSchema = z
  .object({
    firstName: z.string().min(1).max(100).optional(),
    lastName: z.string().min(1).max(100).optional(),
    phone: z.string().optional(),
    role: z.enum(["USER", "ADMIN"]).optional(),
  })
  .partial(); // Makes all fields optional

export type UpdateUserDto = z.infer<typeof updateUserSchema>;
```

### Query Parameters

Use `z.coerce` for automatic type conversion from query strings:

```typescript
export const userQuerySchema = z.object({
  // Filter fields
  status: z.enum(["ACTIVE", "INACTIVE"]).optional(),
  search: z.string().optional(),
  role: z.enum(["USER", "ADMIN"]).optional(),

  // Pagination - use coerce for type conversion from string
  page: z.coerce.number().int().min(1).optional(),
  take: z.coerce.number().int().min(1).max(100).optional(),

  // Sorting
  sortBy: z.enum(["name", "createdAt", "updatedAt"]).optional(),
  sortOrder: z.enum(["asc", "desc"]).optional(),
});

export type UserQueryDto = z.infer<typeof userQuerySchema>;
```

## Advanced Validation Patterns

### Conditional Validation

```typescript
export const createAccountSchema = z
  .object({
    type: z.enum(["INDIVIDUAL", "BUSINESS"]),
    name: z.string().min(1),
    companyName: z.string().optional(),
    taxId: z.string().optional(),
  })
  .superRefine((data, ctx) => {
    if (data.type === "BUSINESS") {
      if (!data.companyName) {
        ctx.addIssue({
          code: z.ZodIssueCode.custom,
          message: "Company name is required for business accounts",
          path: ["companyName"],
        });
      }
      if (!data.taxId) {
        ctx.addIssue({
          code: z.ZodIssueCode.custom,
          message: "Tax ID is required for business accounts",
          path: ["taxId"],
        });
      }
    }
  });
```

### Array Validation

```typescript
export const bulkCreateSchema = z.object({
  items: z
    .array(
      z.object({
        name: z.string().min(1),
        value: z.number(),
      }),
    )
    .min(1, "At least one item is required")
    .max(50, "Too many items (max 50)"),
});

export type BulkCreateDto = z.infer<typeof bulkCreateSchema>;
```

### Nested Object Validation

```typescript
export const createOrderSchema = z.object({
  customerId: z.string().uuid(),

  address: z.object({
    street: z.string().min(1),
    city: z.string().min(1),
    state: z.string().min(2).max(2),
    zipCode: z.string().regex(/^\d{5}(-\d{4})?$/, "Invalid ZIP code"),
  }),

  items: z
    .array(
      z.object({
        productId: z.string().uuid(),
        quantity: z.number().int().min(1),
        price: z.string().regex(/^\d+(\.\d{1,2})?$/, "Invalid price format"),
      }),
    )
    .min(1),
});
```

### Date Range Validation

```typescript
export const dateRangeSchema = z
  .object({
    startDate: z.string().datetime("Invalid date format"),
    endDate: z.string().datetime("Invalid date format"),
  })
  .superRefine((data, ctx) => {
    if (new Date(data.endDate) <= new Date(data.startDate)) {
      ctx.addIssue({
        code: z.ZodIssueCode.custom,
        message: "End date must be after start date",
        path: ["endDate"],
      });
    }
  });
```

### Password Validation

```typescript
export const passwordSchema = z
  .string()
  .min(8, "Password must be at least 8 characters")
  .regex(
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
    "Password must contain uppercase, lowercase, and number",
  );

export const createUserWithPasswordSchema = z
  .object({
    email: z.string().email(),
    password: passwordSchema,
    confirmPassword: z.string(),
  })
  .refine((data) => data.password === data.confirmPassword, {
    message: "Passwords do not match",
    path: ["confirmPassword"],
  });
```

## Controller Integration

### Using ZodValidationPipe

```typescript
import { Controller, Post, Body, Get, Query, Patch, Param } from "@nestjs/common";
import { ZodValidationPipe } from "@/shared/pipes/zod-validation.pipe";
import { createUserSchema, updateUserSchema, userQuerySchema } from "@/domains/users/dto";

@Controller("users")
export class UsersController {
  // POST with body validation
  @Post()
  async create(
    @Body(new ZodValidationPipe(createUserSchema))
    data: CreateUserDto,
  ) {
    return this.userService.create(data);
  }

  // GET with query validation
  @Get()
  async findAll(
    @Query(new ZodValidationPipe(userQuerySchema))
    query: UserQueryDto,
  ) {
    return this.userService.findAll(query);
  }

  // PATCH with body validation
  @Patch(":id")
  async update(
    @Param("id") id: string,
    @Body(new ZodValidationPipe(updateUserSchema))
    data: UpdateUserDto,
  ) {
    return this.userService.update(id, data);
  }
}
```

### Response Transformation

```typescript
@Get(':id')
async findOne(@Param('id') id: string) {
  const user = await this.userService.findById(id);

  return {
    message: 'User retrieved successfully',
    data: {
      ...user,
      // Add computed fields if needed
      fullName: `${user.firstName} ${user.lastName}`,
    },
  };
}
```

## Common Validation Patterns

### Email Validation

```typescript
email: z.string().email("Invalid email format");
```

### UUID Validation

```typescript
id: z.string().uuid("Invalid ID format");
```

### URL Validation

```typescript
website: z.string().url("Invalid URL format").optional();
```

### Phone Number

```typescript
phone: z.string()
  .regex(/^\+?[1-9]\d{1,14}$/, "Invalid phone number")
  .optional();
```

### Decimal/Money

```typescript
price: z.string().regex(/^\d+(\.\d{1,2})?$/, "Invalid price format");
```

### Enum Validation

```typescript
status: z.enum(["ACTIVE", "INACTIVE", "PENDING"]);
```

### Boolean with Default

```typescript
isActive: z.boolean().default(true);
```

### Nullable vs Optional

```typescript
// Optional - field can be omitted
description: z.string().optional(); // string | undefined

// Nullable - field can be null
description: z.string().nullable(); // string | null

// Both
description: z.string().optional().nullable(); // string | null | undefined
```

## Best Practices

### 1. Always Include Validation Messages

```typescript
// GOOD
name: z.string().min(1, "Name is required");

// BAD - Generic error
name: z.string().min(1);
```

### 2. Use z.coerce for Query Parameters

```typescript
// GOOD - Converts string to number
page: z.coerce.number().int().min(1);

// BAD - Requires manual conversion
page: z.number().int().min(1);
```

### 3. Set Reasonable Limits

```typescript
// GOOD
take: z.coerce.number().int().min(1).max(100);

// BAD - No limits
take: z.number();
```

### 4. Use Partial for Updates

```typescript
// GOOD - All fields optional
export const updateSchema = baseSchema.partial();

// BAD - Manual optionals for each field
export const updateSchema = z.object({
  field1: z.string().optional(),
  field2: z.number().optional(),
});
```

### 5. Separate Create and Update Schemas

```typescript
// Create - required fields
export const createSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

// Update - all optional
export const updateSchema = createSchema.partial();
```

## Testing DTOs

```typescript
import { describe, it, expect } from "bun:test";
import { createUserSchema } from "@/domains/users/dto";

describe("User DTOs", () => {
  describe("createUserSchema", () => {
    it("should validate correct data", () => {
      const validData = {
        email: "test@example.com",
        firstName: "John",
        lastName: "Doe",
      };

      const result = createUserSchema.safeParse(validData);
      expect(result.success).toBe(true);
    });

    it("should reject invalid email", () => {
      const invalidData = {
        email: "invalid-email",
        firstName: "John",
        lastName: "Doe",
      };

      const result = createUserSchema.safeParse(invalidData);
      expect(result.success).toBe(false);
    });

    it("should require firstName", () => {
      const invalidData = {
        email: "test@example.com",
        lastName: "Doe",
      };

      const result = createUserSchema.safeParse(invalidData);
      expect(result.success).toBe(false);
    });
  });
});
```
