# Module Development Patterns

This document outlines the patterns and best practices for creating and managing NestJS modules.

## Module Structure

### Basic Module Template

```
domains/{entity}/
├── dto/                    # Data Transfer Objects
│   ├── {entity}.dto.ts
│   └── index.ts
├── repositories/           # Data Access Layer
│   ├── {entity}.repository.interface.ts
│   ├── {entity}.repository.ts
│   └── index.ts
├── services/              # Business Logic Layer
│   ├── {entity}.service.ts
│   ├── __tests__/
│   │   └── {entity}.service.spec.ts
│   └── index.ts
├── {entity}.module.ts     # Module Definition
└── index.ts              # Barrel Export
```

## Module Definition

### Basic Module Setup

```typescript
// {entity}.module.ts
import { Module } from '@nestjs/common';
import { I{Entity}Repository } from '@/domains/{entity}/repositories/{entity}.repository.interface';
import { {Entity}Repository } from '@/domains/{entity}/repositories/{entity}.repository';
import { {Entity}Service } from '@/domains/{entity}/services/{entity}.service';

@Module({
  imports: [
    // Import required modules
  ],
  providers: [
    // Service
    {Entity}Service,

    // Repository mapping
    {
      provide: I{Entity}Repository,  // Interface token
      useClass: {Entity}Repository,  // Concrete implementation
    },
  ],
  exports: [
    // Export what other modules need
    {Entity}Service,
    I{Entity}Repository,
  ],
})
export class {Entity}Module {}
```

### Advanced Module Configuration

```typescript
// {entity}.module.ts
import { Module, forwardRef } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { I{Entity}Repository } from '@/domains/{entity}/repositories';
import { {Entity}Repository } from '@/domains/{entity}/repositories';
import { {Entity}Service } from '@/domains/{entity}/services';

@Module({
  imports: [
    // Configuration for module-specific settings
    ConfigModule,

    // Use forwardRef for circular dependencies
    forwardRef(() => OtherModule),
  ],
  providers: [
    // Core services
    {Entity}Service,

    // Repository implementations
    {
      provide: I{Entity}Repository,
      useClass: {Entity}Repository,
    },

    // Additional providers with factory
    {
      provide: '{ENTITY}_CONFIG',
      useFactory: () => ({
        timeout: 5000,
        retries: 3,
      }),
    },
  ],
  exports: [
    // Public exports
    {Entity}Service,
    I{Entity}Repository,
  ],
})
export class {Entity}Module {}
```

## Dependency Injection Patterns

### Service Dependencies

```typescript
// services/{entity}.service.ts
import { Injectable, Inject, Logger } from '@nestjs/common';
import { I{Entity}Repository } from '@/domains/{entity}/repositories';

@Injectable()
export class {Entity}Service {
  private readonly logger = new Logger({Entity}Service.name);

  constructor(
    // Inject repositories by interface token
    @Inject(I{Entity}Repository)
    private readonly {entity}Repository: I{Entity}Repository,
  ) {}

  async findById(id: string) {
    this.logger.log(`Finding {entity} by ID: ${id}`);
    return this.{entity}Repository.findById(id);
  }
}
```

**IMPORTANT**: Every service, repository, and controller MUST have a Logger instance:

```typescript
private readonly logger = new Logger(ClassName.name);
```

### Factory Provider

```typescript
// In module definition
{
  provide: I{Entity}Service,
  useFactory: (
    repository: I{Entity}Repository,
    configService: ConfigService,
  ) => {
    return new {Entity}Service(
      repository,
      configService.get('{ENTITY}_CONFIG'),
    );
  },
  inject: [I{Entity}Repository, ConfigService],
}
```

## Inter-Module Communication

### Shared Modules

```typescript
// shared.module.ts
@Module({
  providers: [PrismaService, LoggerService],
  exports: [PrismaService, LoggerService],
})
export class SharedModule {}
```

### Module Imports

```typescript
// dependent.module.ts
@Module({
  imports: [
    // Import module to use its exports
    {Entity}Module,
    SharedModule,
  ],
})
export class DependentModule {}
```

### Circular Dependencies

Use `forwardRef()` to resolve circular dependencies:

```typescript
// module-a.ts
@Module({
  imports: [forwardRef(() => ModuleB)],
  providers: [ServiceA],
  exports: [ServiceA],
})
export class ModuleA {}

// module-b.ts
@Module({
  imports: [forwardRef(() => ModuleA)],
  providers: [ServiceB],
  exports: [ServiceB],
})
export class ModuleB {}
```

## Module Lifecycle Hooks

### OnModuleInit

```typescript
@Injectable()
export class {Entity}Service implements OnModuleInit {
  async onModuleInit() {
    // Initialize connections
    // Load initial data
    // Setup event listeners
  }
}
```

### OnApplicationBootstrap

```typescript
@Injectable()
export class {Entity}Service implements OnApplicationBootstrap {
  async onApplicationBootstrap() {
    // Run after all modules are initialized
    // Start background jobs
  }
}
```

### OnModuleDestroy

```typescript
@Injectable()
export class {Entity}Service implements OnModuleDestroy {
  async onModuleDestroy() {
    // Cleanup resources
    // Close connections
  }
}
```

## Best Practices

### 1. Keep Modules Focused

- Each module should have a single responsibility
- Don't put unrelated functionality in the same module

### 2. Use Interface-Based DI

- Always depend on interfaces, not concrete classes
- This makes testing easier and reduces coupling

```typescript
// Define interface token as string constant
export const IUserRepository = 'IUserRepository';

// Use in service
@Inject(IUserRepository)
private readonly userRepository: IUserRepository,
```

### 3. Export Only What's Needed

- Minimize module exports to maintain encapsulation
- Don't expose implementation details

### 4. Handle Circular Dependencies

- Use `forwardRef()` only when necessary
- Consider refactoring to avoid circular dependencies

### 5. Use Barrel Exports

```typescript
// index.ts
export * from "./{entity}.module";
export * from "./services";
export * from "./repositories";
export * from "./dto";
```

## Common Module Patterns

### Configuration Module

```typescript
// config.module.ts
@Module({
  imports: [ConfigModule.forRoot()],
  providers: [
    {
      provide: "APP_CONFIG",
      useFactory: (configService: ConfigService) => ({
        database: {
          url: configService.get("DATABASE_URL"),
        },
        port: configService.get("PORT"),
      }),
      inject: [ConfigService],
    },
  ],
  exports: ["APP_CONFIG"],
})
export class AppConfigModule {}
```

### Feature Flag Module

```typescript
// feature-flags.module.ts
@Module({
  providers: [
    {
      provide: "FEATURE_FLAGS",
      useFactory: (configService: ConfigService) => ({
        newFeature: configService.get("NEW_FEATURE_ENABLED") === "true",
        betaAccess: configService.get("BETA_ACCESS") === "true",
      }),
      inject: [ConfigService],
    },
  ],
  exports: ["FEATURE_FLAGS"],
})
export class FeatureFlagsModule {}
```

## Troubleshooting

### Common Issues

1. **Circular Dependency Error**
   - Use `forwardRef()` for imports
   - Consider moving shared code to a separate module

2. **Provider Not Found**
   - Check if the provider is exported from its module
   - Verify the module is imported
   - Check for typos in provider tokens

3. **Module Not Found**
   - Verify import paths are correct
   - Check barrel exports (index.ts files)
   - Ensure modules are in the right location

### Debugging Tips

1. Check module graph by logging imports
2. Use console.log in module constructors:
   ```typescript
   constructor() {
     console.log('Initializing {Entity}Module');
   }
   ```
