# AI Agents Documentation

This directory contains comprehensive documentation for AI agents working with this NestJS backend codebase.

## Documentation Structure

```
docs/agents/
├── README.md              # This file - documentation overview
├── architecture.md        # Architecture patterns and DDD structure
├── patterns/
│   ├── modules.md        # NestJS module development patterns
│   ├── repository.md     # Repository pattern implementation
│   ├── dto.md            # DTO and Zod validation patterns
│   ├── controller.md     # Controller and presentation layer
│   ├── prisma.md         # Prisma and database infrastructure
│   └── testing.md        # Testing patterns with Bun Test
└── examples/
    └── new-domain.md     # Step-by-step guide for creating new domains
```

## Quick Reference

### Before Implementing Any Feature

1. **Read the relevant pattern documentation** based on your task
2. **Follow DDD principles** - keep domains self-contained
3. **Use absolute imports** - always use `@/` alias
4. **Write tests** - all business logic should be tested

### Common Tasks

| Task                       | Documentation            |
| -------------------------- | ------------------------ |
| Create a new domain/module | `examples/new-domain.md` |
| Implement data access      | `patterns/repository.md` |
| Add input validation       | `patterns/dto.md`        |
| Create API endpoints       | `patterns/controller.md` |
| Database and Prisma        | `patterns/prisma.md`     |
| Structure a module         | `patterns/modules.md`    |
| Write tests                | `patterns/testing.md`    |
| Understand architecture    | `architecture.md`        |

### Key Principles

1. **Domain-Driven Design (DDD)**: Organize code by business domains
2. **Interface-based DI**: Depend on interfaces, not implementations
3. **Zod Validation**: Use Zod schemas for all input validation
4. **Barrel Exports**: Always create `index.ts` files for clean imports
5. **Bun Test**: Use Bun's built-in test runner with its specific patterns

### Code Quality Rules

- **Absolute imports only** - Never use relative imports (`../`, `../../`)
- **No unnecessary comments** - Code should be self-documenting
- **Minimal public API** - Export only what's needed
- **Consistent naming** - Follow established conventions

## Getting Started

If you're new to this codebase:

1. Start with `architecture.md` to understand the overall structure
2. Read `patterns/modules.md` to understand how modules are organized
3. Review `examples/new-domain.md` for a complete walkthrough
4. Reference other pattern docs as needed for specific implementations
