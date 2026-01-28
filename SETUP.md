# NestJS Backend - Initial Setup Guide

This guide provides complete instructions for setting up a new NestJS backend project with Bun, Fastify, PostgreSQL, Prisma, and Zod.

---

## IMPORTANT: Before Starting Setup

**AI AGENT INSTRUCTION**: Before executing any setup steps, you MUST ask the user for the project name. This name will be used throughout the setup for:

- Package name in `package.json`
- Docker container names
- Database name
- Application name in `.env`
- Directory name

**Ask the user:**

> "What is the name of your project? (e.g., `my-awesome-api`, `user-service`, `ecommerce-backend`)"

Once the user provides the project name, replace all occurrences of `my-backend` in this guide with the provided project name.

**Naming conventions:**

- Use kebab-case for the project name (e.g., `my-project-name`)
- The project name will be used for: package name, docker containers, database name
- APP_NAME in `.env` can use a human-readable format (e.g., "My Project Name")

---

## Prerequisites

- **Bun** 1.3.3+ - [Install Bun](https://bun.sh/docs/installation)
- **Docker** & **Docker Compose** - For running PostgreSQL
- **Git** - For version control

## Tech Stack Overview

| Technology | Version | Purpose                              |
| ---------- | ------- | ------------------------------------ |
| Bun        | 1.3.3+  | JavaScript runtime & package manager |
| NestJS     | 11.x    | Backend framework                    |
| Fastify    | 11.x    | HTTP server (faster than Express)    |
| PostgreSQL | 18.x    | Relational database                  |
| Prisma     | 7.x     | ORM with type-safe queries           |
| Zod        | 4.x     | Schema validation                    |
| TypeScript | 5.x     | Type safety                          |
| Oxlint     | 1.x     | Fast linter                          |
| Swagger    | 11.x    | API documentation                    |

## Dependencies

### Production Dependencies

```json
{
  "@fastify/static": "^8.3.0",
  "@fastify/swagger": "^9.6.1",
  "@fastify/swagger-ui": "^5.2.3",
  "@nestjs/cli": "^11.0.14",
  "@nestjs/common": "^11.1.9",
  "@nestjs/config": "^4.0.2",
  "@nestjs/core": "^11.1.9",
  "@nestjs/platform-fastify": "^11.1.9",
  "@nestjs/swagger": "^11.2.3",
  "@prisma/adapter-pg": "7.2.0",
  "@prisma/client": "^7.2.0",
  "class-transformer": "^0.5.1",
  "class-validator": "^0.14.3",
  "dotenv": "^17.2.3",
  "nestjs-zod": "^5.0.1",
  "reflect-metadata": "^0.2.2",
  "rxjs": "^7.8.2",
  "zod": "^4.2.1"
}
```

### Development Dependencies

```json
{
  "@nestjs/schematics": "^11.0.9",
  "@nestjs/testing": "^11.1.9",
  "@swc/cli": "^0.7.9",
  "@swc/core": "^1.15.5",
  "@types/node": "^25.0.3",
  "bun-types": "^1.3.6",
  "oxlint": "^1.2.0",
  "prisma": "7.2.0",
  "ts-node": "^10.9.2",
  "tsconfig-paths": "^4.2.0",
  "typescript": "^5.9.3"
}
```

## Step-by-Step Setup

### 1. Create Project Directory

```bash
mkdir my-backend && cd my-backend
bun init -y
```

### 2. Create package.json

```json
{
  "name": "my-backend",
  "version": "1.0.0",
  "scripts": {
    "start": "bun nest start",
    "dev": "bun nest start --watch",
    "debug": "bun nest start --debug --watch",
    "start:prod": "bun dist/index",
    "build": "bun nest build",
    "format": "bunx oxfmt src test",
    "format:check": "bunx oxfmt --check src test",
    "lint": "oxlint -c oxlint.json --fix",
    "lint:check": "oxlint -c oxlint.json",
    "test": "bun test",
    "test:watch": "bun test --watch",
    "test:cov": "bun test --coverage",
    "prisma:generate": "bun prisma generate",
    "prisma:migrate": "bun prisma migrate dev",
    "prisma:studio": "bun prisma studio",
    "db:push": "bun prisma db push",
    "db:reset": "bun prisma migrate reset",
    "docker:stop": "docker-compose down",
    "docker:clean": "docker-compose down -v"
  },
  "dependencies": {
    "@fastify/static": "^8.3.0",
    "@fastify/swagger": "^9.6.1",
    "@fastify/swagger-ui": "^5.2.3",
    "@nestjs/cli": "^11.0.14",
    "@nestjs/common": "^11.1.9",
    "@nestjs/config": "^4.0.2",
    "@nestjs/core": "^11.1.9",
    "@nestjs/platform-fastify": "^11.1.9",
    "@nestjs/swagger": "^11.2.3",
    "@prisma/adapter-pg": "7.2.0",
    "@prisma/client": "^7.2.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.3",
    "dotenv": "^17.2.3",
    "nestjs-zod": "^5.0.1",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.2",
    "zod": "^4.2.1"
  },
  "devDependencies": {
    "@nestjs/schematics": "^11.0.9",
    "@nestjs/testing": "^11.1.9",
    "@swc/cli": "^0.7.9",
    "@swc/core": "^1.15.5",
    "@types/node": "^25.0.3",
    "bun-types": "^1.3.6",
    "oxlint": "^1.2.0",
    "prisma": "7.2.0",
    "ts-node": "^10.9.2",
    "tsconfig-paths": "^4.2.0",
    "typescript": "^5.9.3"
  },
  "packageManager": "bun@1.3.3"
}
```

### 3. Install Dependencies

```bash
bun install
```

### 4. Create tsconfig.json

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2022",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "noImplicitAny": true,
    "strictBindCallApply": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "paths": {
      "@/*": ["src/*"]
    },
    "types": ["bun-types", "node"]
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 5. Create nest-cli.json

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true,
    "builder": "swc",
    "typeCheck": true
  }
}
```

### 6. Create oxlint.json

```json
{
  "rules": {
    "no-unused-vars": "warn",
    "no-console": "off"
  }
}
```

### 7. Create docker-compose.yml

```yaml
services:
  db:
    image: postgres:18.1-alpine
    container_name: my-backend-db
    restart: unless-stopped
    ports:
      - "6432:5432"
    environment:
      POSTGRES_DB: my-backend
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  postgres_data:
```

### 8. Create .env.example

```env
# Application
NODE_ENV=development
PORT=3001
APP_NAME=My Backend

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:6432/my-backend
```

### 9. Create .env (copy from .env.example)

```bash
cp .env.example .env
```

### 10. Create Folder Structure

```bash
mkdir -p src/controllers
mkdir -p src/domains
mkdir -p src/shared/infrastructure/database/prisma
mkdir -p src/shared/pipes
mkdir -p prisma
```

### 11. Create Prisma Schema

**prisma/schema.prisma**

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["driverAdapters"]
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Add your models here
// Example:
// model User {
//   id        String   @id @default(cuid())
//   email     String   @unique
//   name      String?
//   createdAt DateTime @default(now())
//   updatedAt DateTime @updatedAt
//
//   @@map("users")
// }
```

### 12. Create PrismaService

**src/shared/infrastructure/database/prisma/prisma.service.ts**

```typescript
import { Injectable, Logger, OnModuleInit, OnModuleDestroy } from "@nestjs/common";
import { PrismaClient } from "@prisma/client";

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(PrismaService.name);

  async onModuleInit() {
    this.logger.log("Connecting to database...");
    await this.$connect();
    this.logger.log("Database connected successfully");
  }

  async onModuleDestroy() {
    this.logger.log("Disconnecting from database...");
    await this.$disconnect();
    this.logger.log("Database disconnected");
  }
}
```

**src/shared/infrastructure/database/prisma/index.ts**

```typescript
export * from "./prisma.service";
```

**src/shared/infrastructure/database/index.ts**

```typescript
export * from "./prisma";
```

**src/shared/infrastructure/index.ts**

```typescript
export * from "./database";
```

### 13. Create ZodValidationPipe

**src/shared/pipes/zod-validation.pipe.ts**

```typescript
import { PipeTransform, Injectable, ArgumentMetadata, BadRequestException } from "@nestjs/common";
import { ZodSchema, ZodError } from "zod";

@Injectable()
export class ZodValidationPipe implements PipeTransform {
  constructor(private schema: ZodSchema) {}

  transform(value: unknown, metadata: ArgumentMetadata) {
    try {
      return this.schema.parse(value);
    } catch (error) {
      if (error instanceof ZodError) {
        const messages = error.errors.map((err) => `${err.path.join(".")}: ${err.message}`);
        throw new BadRequestException({
          message: "Validation failed",
          errors: messages,
        });
      }
      throw error;
    }
  }
}
```

**src/shared/pipes/index.ts**

```typescript
export * from "./zod-validation.pipe";
```

**src/shared/index.ts**

```typescript
export * from "./infrastructure";
export * from "./pipes";
```

### 14. Create Root Controller

**src/controllers/root.controller.ts**

```typescript
import { Controller, Get, Logger } from "@nestjs/common";
import { ApiTags, ApiOperation, ApiResponse } from "@nestjs/swagger";
import { ConfigService } from "@nestjs/config";

@Controller()
@ApiTags("Root")
export class RootController {
  private readonly logger = new Logger(RootController.name);

  constructor(private readonly configService: ConfigService) {}

  @Get()
  @ApiOperation({ summary: "Get application name" })
  @ApiResponse({
    status: 200,
    description: "Returns the application name",
    schema: {
      type: "object",
      properties: {
        name: { type: "string" },
      },
    },
  })
  getAppName(): { name: string } {
    this.logger.log("Fetching application name");
    const appName = this.configService.get<string>("APP_NAME", "My Backend");
    return { name: appName };
  }

  @Get("healthz")
  @ApiOperation({ summary: "Health check endpoint" })
  @ApiResponse({
    status: 200,
    description: "Returns health status",
    schema: {
      type: "object",
      properties: {
        status: { type: "string", example: "ok" },
        message: { type: "string", example: "success" },
      },
    },
  })
  healthCheck(): { status: string; message: string } {
    this.logger.log("Health check requested");
    return {
      status: "ok",
      message: "success",
    };
  }
}
```

**src/controllers/index.ts**

```typescript
export * from "./root.controller";
```

### 15. Create App Module

**src/app.module.ts**

```typescript
import { Module } from "@nestjs/common";
import { ConfigModule } from "@nestjs/config";
import { RootController } from "@/controllers/root.controller";
import { PrismaService } from "@/shared/infrastructure/database/prisma/prisma.service";

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ".env",
    }),
  ],
  controllers: [RootController],
  providers: [PrismaService],
  exports: [PrismaService],
})
export class AppModule {}
```

### 16. Create Application Entry Point

**src/index.ts**

```typescript
import { NestFactory } from "@nestjs/core";
import { FastifyAdapter, NestFastifyApplication } from "@nestjs/platform-fastify";
import { SwaggerModule, DocumentBuilder } from "@nestjs/swagger";
import { ConfigService } from "@nestjs/config";
import { AppModule } from "@/app.module";

async function bootstrap() {
  const app = await NestFactory.create<NestFastifyApplication>(
    AppModule,
    new FastifyAdapter({ logger: true }),
  );

  const configService = app.get(ConfigService);
  const port = configService.get<number>("PORT", 3001);
  const appName = configService.get<string>("APP_NAME", "My Backend");

  // Swagger documentation
  const config = new DocumentBuilder()
    .setTitle(appName)
    .setDescription("API Documentation")
    .setVersion("1.0")
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup("docs", app, document);

  // Enable CORS
  app.enableCors();

  await app.listen(port, "0.0.0.0");
  console.log(`Application is running on: http://localhost:${port}`);
  console.log(`Swagger docs available at: http://localhost:${port}/docs`);
}

bootstrap();
```

### 17. Create .gitignore

```gitignore
# Dependencies
node_modules/

# Build output
dist/

# Environment files
.env
.env.local
.env.*.local

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
logs/
*.log
npm-debug.log*

# Prisma
prisma/migrations/**/migration_lock.toml

# Coverage
coverage/

# Bun
bun.lockb
```

## Running the Application

### 1. Start PostgreSQL

```bash
docker-compose up -d
```

### 2. Generate Prisma Client

```bash
bun run prisma:generate
```

### 3. Start Development Server

```bash
bun run dev
```

## Verification

After starting the application, verify the setup:

### 1. Root Endpoint

```bash
curl http://localhost:3001/
```

**Expected Response:**

```json
{
  "name": "My Backend"
}
```

### 2. Health Check Endpoint

```bash
curl http://localhost:3001/healthz
```

**Expected Response:**

```json
{
  "status": "ok",
  "message": "success"
}
```

### 3. Swagger Documentation

Open http://localhost:3001/docs in your browser to see the API documentation.

## Logger Requirement

**IMPORTANT**: Every class (service, repository, controller) MUST have a Logger instance with the class name.

```typescript
import { Injectable, Logger } from "@nestjs/common";

@Injectable()
export class MyService {
  private readonly logger = new Logger(MyService.name);

  async doSomething() {
    this.logger.log("Doing something");
    try {
      // ... implementation
      this.logger.log("Operation completed successfully");
    } catch (error) {
      this.logger.error(`Operation failed: ${error.message}`, error.stack);
      throw error;
    }
  }
}
```

### Logger Levels

- `logger.log()` - General information
- `logger.error()` - Errors (include stack trace)
- `logger.warn()` - Warnings
- `logger.debug()` - Debug information
- `logger.verbose()` - Verbose output

## Next Steps

1. **Create your first domain** - See `docs/agents/examples/new-domain.md`
2. **Add authentication** - Implement guards and decorators
3. **Add database models** - Define Prisma schema and run migrations
4. **Write tests** - See `docs/agents/patterns/testing.md`

## Troubleshooting

### Database Connection Issues

1. Ensure Docker is running: `docker ps`
2. Check PostgreSQL container: `docker-compose logs db`
3. Verify DATABASE_URL in `.env` matches docker-compose settings

### Port Already in Use

Change the PORT in `.env` or stop the conflicting process:

```bash
lsof -i :3001
kill -9 <PID>
```

### Prisma Client Not Found

Run Prisma generate after schema changes:

```bash
bun run prisma:generate
```

### TypeScript Path Aliases Not Working

Ensure `tsconfig.json` has the correct paths configuration and restart your IDE.

---

## Post-Setup Cleanup

**AI AGENT INSTRUCTION**: After successfully completing the setup and verifying that:

1. The application starts without errors (`bun run dev`)
2. The root endpoint (`/`) returns the correct application name
3. The health check endpoint (`/healthz`) returns `{ status: "ok", message: "success" }`

You MUST delete this `SETUP.md` file as it is no longer needed:

```bash
rm SETUP.md
```

This file is only meant to guide the initial project setup. Once the project is bootstrapped and working, it should be removed to keep the repository clean.

**Confirm with the user before deleting:**

> "The setup is complete and verified. Should I delete the SETUP.md file now that it's no longer needed?"
