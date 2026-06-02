# Phase 01: project-foundation - Pattern Map

**Mapped:** 2026-06-02
**Files analyzed:** 30
**Analogs found:** 0 / 30

## File Classification

| New/Modified File | Role | Data Flow | Closest Analog | Match Quality |
|-------------------|------|-----------|----------------|---------------|
| `package.json` | config | batch | `.planning/phases/01-project-foundation/01-RESEARCH.md` Standard Stack | research-reference |
| `package-lock.json` | config | batch | None | no-analog |
| `tsconfig.json` | config | transform | `.planning/phases/01-project-foundation/01-RESEARCH.md` Standard Stack | research-reference |
| `vitest.config.ts` | config | batch | `.planning/phases/01-project-foundation/01-RESEARCH.md` Validation Architecture | research-reference |
| `.env.example` | config | request-response | `src/config/index.ts` research example | research-reference |
| `src/app.ts` | config | request-response | `src/app.ts` research example | research-reference |
| `src/server.ts` | config | event-driven | `server.ts` research example | research-reference |
| `src/config/index.ts` | config | transform | `src/config/index.ts` research example | research-reference |
| `src/routes/index.ts` | route | request-response | `src/app.ts` route mounting research example | research-reference |
| `src/routes/health.routes.ts` | route | request-response | `src/routes/health.routes.ts` research example | research-reference |
| `src/controllers/health.controller.ts` | controller | request-response | `src/routes/health.routes.ts` research example | partial |
| `src/models/index.ts` | model | CRUD | `src/models/index.ts` research example | research-reference |
| `src/middleware/error.middleware.ts` | middleware | request-response | `src/middleware/error.middleware.ts` research example | research-reference |
| `src/middleware/validation.middleware.ts` | middleware | request-response | `src/middleware/validation.middleware.ts` research example | research-reference |
| `src/middleware/logger.middleware.ts` | middleware | request-response | `src/app.ts` morgan middleware research example | partial |
| `src/utils/errors.ts` | utility | transform | `src/utils/errors.ts` research example | research-reference |
| `src/schemas/index.ts` | utility | transform | `src/schemas/task.schemas.ts` research example | partial |
| `prisma/schema.prisma` | model | CRUD | `prisma/schema.prisma` research example | research-reference |
| `prisma/migrations/*/migration.sql` | migration | batch | `prisma/schema.prisma` research example | partial |
| `tests/setup.ts` | test | batch | Validation Architecture research map | research-reference |
| `tests/teardown.ts` | test | batch | Validation Architecture research map | research-reference |
| `tests/unit/errors.test.ts` | test | transform | `src/utils/errors.ts` research example | partial |
| `tests/unit/error-handler.test.ts` | test | request-response | `src/middleware/error.middleware.ts` research example | partial |
| `tests/unit/config-validation.test.ts` | test | transform | `src/config/index.ts` research example | partial |
| `tests/unit/validation.test.ts` | test | request-response | `src/middleware/validation.middleware.ts` research example | partial |
| `tests/unit/helmet-headers.test.ts` | test | request-response | `src/app.ts` research example | partial |
| `tests/unit/cors.test.ts` | test | request-response | `src/app.ts` research example | partial |
| `tests/smoke/server.test.ts` | test | event-driven | `server.ts` research example | partial |
| `tests/integration/db-connect.test.ts` | test | CRUD | `src/models/index.ts` research example | partial |
| `tests/integration/migrations.test.ts` | test | batch | `prisma/schema.prisma` research example | partial |

## Pattern Assignments

### `src/app.ts` (config, request-response)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 572-601.

**Imports and middleware order pattern** (lines 572-596):
```typescript
// src/app.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import morgan from 'morgan';
import { errorHandler } from './middleware/error.middleware.js';
import { healthRoutes } from './routes/health.routes.js';

export const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(morgan('combined'));
app.use('/health', healthRoutes);
app.use(errorHandler);
```

**404 response pattern** (lines 598-601):
```typescript
app.use((req, res) => {
  res.status(404).json({ error: { code: 'NOT_FOUND', message: `Route ${req.method} ${req.path} not found` } });
});
```

### `src/server.ts` (config, event-driven)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 503-522.

**Lifecycle pattern**:
```typescript
// server.ts - async init + graceful shutdown
import { app } from './app.js';
import { prisma } from './models/index.js';

const server = app.listen(config.PORT, () => {
  console.log(`Server running on port ${config.PORT}`);
});

const shutdown = async (signal: string) => {
  console.log(`${signal} received, shutting down...`);
  server.close();
  await prisma.$disconnect();
  process.exit(0);
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

### `src/config/index.ts` and `.env.example` (config, transform)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 326-368.

**Environment validation pattern**:
```typescript
import 'dotenv/config';

interface Config {
  DATABASE_URL: string;
  JWT_SECRET: string;
  PORT: number;
  NODE_ENV: string;
}

function validate(): Config {
  const errors: string[] = [];

  const dbUrl = process.env.DATABASE_URL;
  if (!dbUrl) errors.push('DATABASE_URL is required');

  const jwtSecret = process.env.JWT_SECRET;
  if (!jwtSecret) {
    errors.push('JWT_SECRET is required');
  } else if (jwtSecret.length < 64) {
    errors.push('JWT_SECRET must be at least 64 characters');
  }

  const port = parseInt(process.env.PORT ?? '3000', 10);
  if (isNaN(port) || port < 1 || port > 65535) {
    errors.push('PORT must be a valid port number (1-65535)');
  }

  if (errors.length > 0) {
    throw new Error(`Configuration errors:\n${errors.join('\n')}`);
  }

  return { DATABASE_URL: dbUrl!, JWT_SECRET: jwtSecret!, PORT: port, NODE_ENV: process.env.NODE_ENV ?? 'development' };
}

export const config = validate();
```

### `src/routes/health.routes.ts` and `src/controllers/health.controller.ts` (route/controller, request-response)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 607-622.

**Route + health check pattern**:
```typescript
// src/routes/health.routes.ts
import { Router } from 'express';
import { prisma } from '../models/index.js';

const router = Router();

router.get('/', async (req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`;
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  } catch (err) {
    res.status(503).json({ status: 'error', message: 'Database unavailable' });
  }
});

export { router as healthRoutes };
```

### `src/models/index.ts` (model, CRUD)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 628-633.

**Prisma client pattern**:
```typescript
// src/models/index.ts
import { PrismaClient } from '@prisma/client';

export const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
});
```

### `prisma/schema.prisma` and `prisma/migrations/*/migration.sql` (model/migration, CRUD/batch)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 413-477.

**Schema, FK index, and soft delete pattern**:
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Task {
  id          String   @id @default(uuid())
  title       String
  description String?
  status      String   @default("TODO")
  ownerId     String
  assigneeId  String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  deletedAt   DateTime?

  owner       User     @relation("TaskOwner", fields: [ownerId], references: [id])
  assignee    User?    @relation("TaskAssignee", fields: [assigneeId], references: [id])
  comments    Comment[]

  @@index([ownerId])
  @@index([assigneeId])
  @@index([status])
  @@index([deletedAt])
}
```

Also copy the `User` and `Comment` model structure from lines 425-439 and 462-477, including `deletedAt` and indexes.

### `src/utils/errors.ts` (utility, transform)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 242-284.

**Error hierarchy pattern**:
```typescript
export class AppError extends Error {
  constructor(
    public readonly code: string,
    public readonly message: string,
    public readonly statusCode: number = 500,
    public readonly details?: any
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }

  toJSON() {
    return { error: { code: this.code, message: this.message, details: this.details } };
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super('VALIDATION_ERROR', message, 400, details);
  }
}
```

Also include `NotFoundError`, `ForbiddenError`, and `ConflictError` from lines 262-284.

### `src/middleware/error.middleware.ts` (middleware, request-response)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 288-313.

**Global error handler pattern**:
```typescript
import type { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/errors.js';

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  console.error(`[ERROR] ${req.method} ${req.path}:`, err.message);

  if (err instanceof AppError) {
    res.status(err.statusCode).json(err.toJSON());
    return;
  }

  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: process.env.NODE_ENV === 'production' ? 'Internal server error' : err.message,
    },
  });
}
```

### `src/middleware/validation.middleware.ts` (middleware, request-response)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 376-402.

**Zod validation pattern**:
```typescript
import type { Request, Response, NextFunction } from 'express';
import { ZodSchema } from 'zod';

export function validate(schema: ZodSchema, target: 'body' | 'query' | 'params' = 'body') {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req[target]);

    if (!result.success) {
      const details = result.error.flatten();
      res.status(400).json({
        error: { code: 'VALIDATION_ERROR', message: 'Request validation failed', details },
      });
      return;
    }

    req[target] = result.data;
    next();
  };
}
```

### `src/middleware/logger.middleware.ts` (middleware, request-response)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 576 and 589-590.

**Logging pattern**:
```typescript
import morgan from 'morgan';

app.use(morgan('combined'));
```

Planner note: if split into a middleware file, export a configured `morgan('combined')` middleware and keep request bodies out of logs.

### `src/schemas/index.ts` (utility, transform)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 662-678 as the future route-schema style.

**Zod schema pattern**:
```typescript
import { z } from 'zod';

export const taskCreateSchema = z.object({
  title: z.string().min(1).max(255),
  description: z.string().max(5000).optional(),
  status: z.enum(['TODO', 'IN_PROGRESS', 'DONE']).default('TODO'),
});

export const taskUpdateSchema = z.object({
  title: z.string().min(1).max(255).optional(),
  description: z.string().max(5000).optional().nullable(),
}).refine(data => Object.keys(data).length > 0, {
  message: 'At least one field must be provided',
});
```

### Test files (test, batch/request-response/CRUD)

**Analog:** No existing source analog. Use `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 760-800.

**Test framework pattern**:
```text
Framework: Vitest v4.1.8 + supertest v7.2.2
Config file: vitest.config.ts
Quick run command: rtk npx vitest run
Test files: tests/unit/*.test.ts, tests/integration/*.test.ts
```

**Required test coverage map**:
```text
tests/unit/errors.test.ts              -> custom error classes
tests/unit/error-handler.test.ts       -> global error handler JSON shape
tests/unit/config-validation.test.ts   -> startup config validation
tests/unit/validation.test.ts          -> Zod validation error format
tests/unit/helmet-headers.test.ts      -> helmet security headers
tests/unit/cors.test.ts                -> CORS middleware
tests/smoke/server.test.ts             -> server starts
tests/smoke/shutdown.test.ts           -> graceful shutdown
tests/integration/db-connect.test.ts   -> Prisma connection
tests/integration/migrations.test.ts   -> migrations apply
```

## Shared Patterns

### Import Style
**Source:** `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 572-578, 607-609, 628-629
**Apply to:** All TypeScript source files
```typescript
import express from 'express';
import { errorHandler } from './middleware/error.middleware.js';
import { healthRoutes } from './routes/health.routes.js';
import { prisma } from '../models/index.js';
```

Use ESM-style `.js` suffixes for relative TypeScript imports that will compile to JavaScript.

### Response Shape
**Source:** `.planning/phases/01-project-foundation/01-CONTEXT.md` D-04 and `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 257-259, 388-394, 599-600
**Apply to:** Errors, validation failures, 404s
```typescript
{ error: { code: string, message: string, details?: any } }
```

### Error Handling
**Source:** `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 235-313
**Apply to:** Controllers, routes, services, middleware
```typescript
if (err instanceof AppError) {
  res.status(err.statusCode).json(err.toJSON());
  return;
}
```

Express v5 native async error support means no `express-async-errors` dependency.

### Validation
**Source:** `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 371-403
**Apply to:** All routes that accept body, params, or query input
```typescript
router.post('/tasks', validate(taskCreateSchema), taskController.create);
```

### Security Middleware
**Source:** `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 582-590 and 832-841
**Apply to:** `src/app.ts`, security tests
```typescript
app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(morgan('combined'));
```

### Database and Shutdown
**Source:** `.planning/phases/01-project-foundation/01-RESEARCH.md` lines 628-633 and 513-522
**Apply to:** `src/models/index.ts`, `src/server.ts`, integration tests
```typescript
export const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
});

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

## No Analog Found

No application source files exist yet. The repository currently contains GTD planning scaffolding, research artifacts, and workflow tooling, but no `src/`, `prisma/`, or `tests/` implementation tree.

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| All files listed in File Classification | mixed | mixed | Phase 1 is the first source-code phase; only research examples exist |

## Metadata

**Analog search scope:** `src`, `prisma`, `tests`, project root, `.planning/research`, `.planning/phases/01-project-foundation`
**Files scanned:** Planning artifacts plus project root listing; no application source files found
**Pattern extraction date:** 2026-06-02
**Project instructions applied:** `AGENTS.md` and referenced `/Users/davide/.codex/RTK.md`
