# NestJS Bootstrap Documentation

A comprehensive documentation package for bootstrapping NestJS backend projects with modern best practices.

## What's Included

This repository contains AI agent documentation and setup guides for creating production-ready NestJS applications following Domain-Driven Design (DDD) patterns.

### Tech Stack

- **Runtime**: Bun 1.3.3+
- **Framework**: NestJS + Fastify
- **Database**: PostgreSQL + Prisma ORM
- **Validation**: Zod
- **Testing**: Bun Test
- **Linting**: Oxlint
- **API Docs**: Swagger/OpenAPI

## Getting Started

### For AI Agents (Claude, Gemini, etc.)

Read the corresponding instruction file:

- `CLAUDE.md` - Instructions for Claude AI
- `GEMINI.md` - Instructions for Gemini AI

### For Developers

1. Copy this documentation to your new project
2. Follow `SETUP.md` for initial project setup
3. Reference `AGENTS.md` for coding standards

## Documentation Structure

```
├── README.md                      # This file
├── CLAUDE.md                      # Claude AI instructions
├── GEMINI.md                      # Gemini AI instructions
├── AGENTS.md                      # Main coding standards
├── SETUP.md                       # Initial project setup guide
└── docs/
    └── agents/
        ├── README.md              # Documentation overview
        ├── architecture.md        # Architecture and DDD patterns
        ├── patterns/
        │   ├── modules.md         # NestJS module development
        │   ├── repository.md      # Repository pattern
        │   ├── dto.md             # DTO and Zod validation
        │   └── testing.md         # Testing with Bun Test
        └── examples/
            └── new-domain.md      # Step-by-step domain creation
```

## Key Features

- **Domain-Driven Design**: Organized by business domains with clear separation of concerns
- **Type Safety**: Strict TypeScript with Zod validation
- **Interface-based DI**: Loose coupling via dependency injection
- **Comprehensive Testing**: Patterns for Bun Test with proper mocking
- **Logging Standards**: Every class includes a Logger instance
- **Absolute Imports**: Clean imports using `@/` alias

## Usage

### Creating a New Project

1. Have an AI agent read `SETUP.md`
2. Provide your project name when asked
3. Follow the step-by-step setup guide
4. Delete `SETUP.md` after successful setup

### Adding New Features

1. Read `AGENTS.md` for coding standards
2. Follow `docs/agents/examples/new-domain.md` for creating new domains
3. Reference pattern docs as needed

## License

MIT
