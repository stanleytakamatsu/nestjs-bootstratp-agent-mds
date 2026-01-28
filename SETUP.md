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
| Prisma     | 7.3.x   | ORM with type-safe queries           |
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
  "@prisma/adapter-pg": "7.3.0",
  "@prisma/client": "7.3.0",
  "class-transformer": "^0.5.1",
  "class-validator": "^0.14.3",
  "dotenv": "^17.2.3",
  "nestjs-zod": "^5.0.1",
  "pg": "^8.16.0",
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
  "@types/pg": "^8.15.4",
  "bun-types": "^1.3.6",
  "oxlint": "^1.2.0",
  "prisma": "7.3.0",
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
    "start:prod": "bun dist/main",
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
    "prisma:migrate:deploy": "bun prisma migrate deploy",
    "prisma:migrate:status": "bun prisma migrate status",
    "prisma:migrate:reset": "bun prisma migrate reset",
    "prisma:studio": "bun prisma studio",
    "db:push": "bun prisma db push",
    "db:seed": "bun prisma db seed",
    "docker:up": "docker compose up -d",
    "docker:up:db": "docker compose up -d db",
    "docker:up:app": "docker compose up -d app",
    "docker:down": "docker compose down",
    "docker:clean": "docker compose down -v",
    "docker:logs": "docker compose logs -f",
    "docker:logs:app": "docker compose logs -f app",
    "docker:logs:db": "docker compose logs -f db",
    "docker:exec:app": "docker compose exec app sh",
    "vercel:build": "bun run prisma:generate && bun run build",
    "vercel:migrate": "bun run prisma:migrate:deploy"
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
    "@prisma/adapter-pg": "7.3.0",
    "@prisma/client": "7.3.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.3",
    "dotenv": "^17.2.3",
    "nestjs-zod": "^5.0.1",
    "pg": "^8.16.0",
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
    "@types/pg": "^8.15.4",
    "bun-types": "^1.3.6",
    "oxlint": "^1.2.0",
    "prisma": "7.3.0",
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

### 4. Trust Prisma Postinstall Scripts

Prisma requires postinstall scripts to run. Trust the prisma package:

```bash
bun pm trust prisma
bun pm trust @prisma/client
bun pm trust @prisma/adapter-pg
```

Then reinstall to run the postinstall scripts:

```bash
bun install
```

### 5. Create tsconfig.json

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

### 6. Create nest-cli.json

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

### 7. Create oxlint.json

```json
{
  "rules": {
    "no-unused-vars": "warn",
    "no-console": "off"
  }
}
```

### 8. Create docker-compose.yml

This configuration is for **local development only**. It includes both the PostgreSQL database and the application service with hot-reload support.

```yaml
services:
  # PostgreSQL Database
  db:
    image: postgres:18.1-alpine
    container_name: my-backend-db
    restart: unless-stopped
    ports:
      - "6433:5432"
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
    networks:
      - my-backend-network

  # Application Service (Local Development)
  app:
    image: oven/bun:1.3.3-alpine
    container_name: my-backend-app
    working_dir: /app
    restart: unless-stopped
    ports:
      - "3001:3001"
    environment:
      NODE_ENV: development
      PORT: 3001
      APP_NAME: My Backend
      DATABASE_URL: postgresql://postgres:postgres@db:5432/my-backend
      DIRECT_URL: postgresql://postgres:postgres@db:5432/my-backend
    volumes:
      - .:/app # Mount entire project
      - /app/node_modules # Exclude node_modules (use container's)
      - bun_cache:/root/.bun/install/cache # Cache bun packages
    depends_on:
      db:
        condition: service_healthy
    networks:
      - my-backend-network
    command: sh -c "bun install && bun run prisma:generate && bun run dev"

volumes:
  postgres_data:
  bun_cache:

networks:
  my-backend-network:
    driver: bridge
```

**Key Features:**

- **Volume Mapping**: Your local code is mounted to `/app`, enabling hot-reload
- **node_modules Exclusion**: Container uses its own node_modules to avoid OS compatibility issues
- **Bun Cache**: Shared cache volume speeds up subsequent `bun install` runs
- **Auto Setup**: Runs `bun install`, `prisma:generate`, and `dev` on startup

**Note:** When running the app in Docker, the DATABASE_URL uses `db:5432` (internal Docker network). When running locally outside Docker, use `localhost:6433`.

### 9. Create .env.example

```env
# Application
NODE_ENV=development
PORT=3001
APP_NAME=My Backend

# Database (Local Development - running app outside Docker)
# Use localhost:6433 when running the app locally with `bun run dev`
DATABASE_URL=postgresql://postgres:postgres@localhost:6433/my-backend
DIRECT_URL=postgresql://postgres:postgres@localhost:6433/my-backend

# Database (Docker - running app inside Docker)
# Use db:5432 when running the app inside Docker with `docker-compose up`
# Uncomment the lines below and comment out the lines above if running fully in Docker
# DATABASE_URL=postgresql://postgres:postgres@db:5432/my-backend
# DIRECT_URL=postgresql://postgres:postgres@db:5432/my-backend
```

**Note:** The docker-compose.yml already sets the correct DATABASE_URL for the containerized app via environment variables. The .env file values are used for local development only.

**Database URL Configuration:**

| Environment     | DATABASE_URL                 | DIRECT_URL                 |
| --------------- | ---------------------------- | -------------------------- |
| Local Dev       | `localhost:6433`             | Same as DATABASE_URL       |
| Docker (app)    | `db:5432` (internal network) | Same as DATABASE_URL       |
| Neon            | Pooled (`-pooler` hostname)  | Direct (without `-pooler`) |
| Supabase        | Pooler (port 6543)           | Direct (port 5432)         |
| Vercel Postgres | Pooled connection            | Direct connection          |
| Railway         | Direct connection            | Same as DATABASE_URL       |

### 10. Create .env (copy from .env.example)

```bash
cp .env.example .env
```

### 11. Create Folder Structure

```bash
mkdir -p src/controllers
mkdir -p src/domains
mkdir -p src/shared/infrastructure/database/prisma
mkdir -p src/shared/pipes
mkdir -p prisma
```

### 12. Create Prisma Schema

For detailed Prisma patterns and conventions, see `docs/agents/patterns/prisma.md`.

**prisma/schema.prisma**

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
}

// Add your models here
// Example:
// model User {
//   id        String   @id @default(cuid())
//   email     String   @unique
//   firstName String   @map("first_name")
//   lastName  String   @map("last_name")
//   status    UserStatus @default(ACTIVE)
//   createdAt DateTime @default(now()) @map("created_at")
//   updatedAt DateTime @updatedAt @map("updated_at")
//
//   @@map("users")
//   @@index([status])
// }
//
// enum UserStatus {
//   ACTIVE
//   INACTIVE
// }
```

**Schema Conventions:**

- Model names: PascalCase singular (e.g., `User`, `Post`)
- Table names: Use `@@map("table_name")` with snake_case plural
- Field names: camelCase in Prisma, snake_case in DB with `@map()`
- IDs: Use `cuid()` for string IDs
- Timestamps: Always include `createdAt` and `updatedAt`
- Indexes: Add `@@index` for frequently queried fields

**IMPORTANT Prisma 7 Note**: The `url` property is no longer used in the datasource block. Database connection is configured via `prisma.config.ts`.

### 13. Create Prisma Config

**prisma/prisma.config.ts**

```typescript
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

**Why DIRECT_URL?**

In serverless environments (Vercel, Neon, Supabase), database connections often go through a connection pooler (e.g., PgBouncer). While poolers are great for application queries, they don't support the transaction-based operations that Prisma migrations require.

- `DATABASE_URL`: Used by your application at runtime (can use pooled connections)
- `DIRECT_URL`: Used by Prisma CLI for migrations (must bypass the pooler)

### 14. Create PrismaService

The `PrismaService` is a shared infrastructure class that wraps the Prisma client using the driver adapter pattern for Prisma 7. For detailed patterns and advanced usage, see `docs/agents/patterns/prisma.md`.

**src/shared/infrastructure/database/prisma/prisma.service.ts**

```typescript
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
  }

  async onModuleDestroy() {
    this.logger.log("Disconnecting from database...");
    await this.$disconnect();
    await this.pool.end();
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

### 15. Create ZodValidationPipe

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
        const messages = error.issues.map((issue) => `${issue.path.join(".")}: ${issue.message}`);
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

### 16. Create Root Controller

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

### 17. Create App Module

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

### 18. Create Application Entry Point

**src/main.ts**

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

### 19. Create .gitignore

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

# Coverage
coverage/

# Bun
bun.lockb
```

## Running the Application

### Option A: Local Development (Recommended for Development)

#### 1. Start PostgreSQL Only

```bash
docker compose up -d db
```

#### 2. Generate Prisma Client

```bash
bun run prisma:generate
```

#### 3. Start Development Server

```bash
bun run dev
```

### Option B: Full Docker Development

Run everything in Docker with volume mapping for hot-reload.

#### 1. Start All Services

```bash
docker compose up
```

Or run in detached mode (background):

```bash
docker compose up -d
```

This will:

- Start PostgreSQL
- Start the app container with your code mounted
- Auto-run `bun install`, `prisma:generate`, and `bun run dev`
- **Hot-reload enabled**: Changes to your local files are automatically detected

#### 2. Run Migrations (first time or after schema changes)

```bash
docker compose exec app bun run prisma:migrate
```

#### 3. View Logs

```bash
docker compose logs -f
```

#### 4. Access App Container Shell

```bash
docker compose exec app sh
```

### Docker Commands Reference

| Command                      | Description                                     |
| ---------------------------- | ----------------------------------------------- |
| `docker compose up`          | Start all services with logs (foreground)       |
| `docker compose up -d`       | Start all services in background                |
| `docker compose up -d db`    | Start only PostgreSQL                           |
| `docker compose up -d app`   | Start only the application                      |
| `docker compose down`        | Stop all services                               |
| `docker compose down -v`     | Stop and remove volumes (WARNING: deletes data) |
| `docker compose logs -f`     | Follow logs for all services                    |
| `docker compose logs -f app` | Follow logs for app only                        |
| `docker compose logs -f db`  | Follow logs for database only                   |
| `docker compose exec app sh` | Open shell inside app container                 |

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

## Serverless Deployment (Vercel, Neon, etc.)

When deploying to serverless platforms, migrations require special handling because connection poolers don't support migration transactions.

### Environment Variables for Serverless

```env
# Production example with Neon
DATABASE_URL=postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/mydb?sslmode=require
DIRECT_URL=postgresql://user:pass@ep-xxx.region.aws.neon.tech/mydb?sslmode=require
```

### Running Migrations in CI/CD

**Option 1: Vercel Build Command**

In your Vercel project settings, set the build command to:

```bash
bun run vercel:build
```

And add a separate migration step (e.g., in GitHub Actions):

```yaml
- name: Run migrations
  run: bun run vercel:migrate
  env:
    DIRECT_URL: ${{ secrets.DIRECT_URL }}
```

**Option 2: Vercel CLI with Migration**

```bash
# Deploy with migrations
bun run prisma:migrate:deploy && vercel --prod
```

### Package.json Scripts Reference

| Script                  | Purpose                                     |
| ----------------------- | ------------------------------------------- |
| `prisma:migrate`        | Create new migration (development)          |
| `prisma:migrate:deploy` | Apply pending migrations (production)       |
| `prisma:migrate:status` | Check migration status                      |
| `prisma:migrate:reset`  | Reset database and reapply all migrations   |
| `vercel:build`          | Generate Prisma client + build (for Vercel) |
| `vercel:migrate`        | Deploy migrations (for CI/CD)               |

### Checking Migration Status

Before deploying, check if there are pending migrations:

```bash
bun run prisma:migrate:status
```

## Next Steps

1. **Create your first domain** - See `docs/agents/examples/new-domain.md`
2. **Add authentication** - Implement guards and decorators
3. **Add database models** - Define Prisma schema and run migrations
4. **Write tests** - See `docs/agents/patterns/testing.md`

## Troubleshooting

### Database Connection Issues

1. Ensure Docker is running: `docker ps`
2. Check PostgreSQL container: `docker-compose logs db`
3. Verify DATABASE_URL in `.env` matches docker-compose settings (port 6433)

### Port Already in Use

Change the PORT in `.env` or stop the conflicting process:

```bash
lsof -i :3001
kill -9 <PID>
```

For PostgreSQL port conflicts (6433), check for other containers:

```bash
docker ps | grep 6433
```

### Prisma Client Not Found

Run Prisma generate after schema changes:

```bash
bun run prisma:generate
```

### Prisma Postinstall Scripts Not Running

If you see warnings about postinstall scripts being blocked:

```bash
bun pm trust prisma
bun pm trust @prisma/client
bun pm trust @prisma/adapter-pg
bun install
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
