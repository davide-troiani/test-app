# Phase 1: Project Foundation - Research

**Researched:** 2026-06-02
**Domain:** Express.js server foundation, Prisma ORM setup, security middleware, validation layer, error handling
**Confidence:** HIGH

## Summary

Phase 1 establishes the complete foundation for a production-grade Express v5 REST API. The work decomposes into five towers: (1) a running Express server with graceful shutdown, (2) Prisma ORM with initial migrations for users/tasks/comments, (3) a custom error class hierarchy wired to a global error handler, (4) Zod validation middleware on all routes, and (5) security layer (helmet, CORS, env validation). This is the ONLY phase that touches every part of the application — every future phase inherits from it. A solid foundation prevents retrofitting complexity into later phases.

**Primary recommendation:** Use Express v5's native async support (no `express-async-errors`), Prisma v6.x for stability, Zod v3.x for validation, and a strict layered structure from day one. All packages verified on npm registry.

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** Standard Express layered architecture — routes/controllers/services/models/middleware
- **D-02:** Resource-based file organization — each domain (tasks, users, comments) has its own files
- **D-03:** TypeScript enabled (based on Prisma + Zod stack)
- **D-04:** Consistent JSON error response: `{ error: { code: string, message: string, details?: any } }`
- **D-05:** Custom error class hierarchy: AppError (base), NotFoundError, ValidationError, ForbiddenError, ConflictError
- **D-06:** Global error handler middleware with proper status codes
- **D-07:** Async errors handled via Express v5 native async support (no express-async-errors needed)
- **D-08:** Zod for schema validation on all incoming requests
- **D-09:** Validation middleware per route — schemas co-located with routes
- **D-10:** Consistent error format for validation failures (400 Bad Request)
- **D-11:** Prisma migrations for all schema changes
- **D-12:** Initial schema: users, tasks, comments tables with proper indexes
- **D-13:** Indexes on foreign keys: owner_id, assigned_to, task_id, status
- **D-14:** Soft delete support via deleted_at timestamp
- **D-15:** Environment variables validated at startup (JWT_SECRET >= 64 chars)
- **D-16:** Helmet middleware for security headers
- **D-17:** CORS configured appropriately
- **D-18:** Vitest + supertest for testing setup

### the agent's Discretion

- Exact file naming conventions (consistent with Express idioms)
- Specific error message text
- Default page size configuration (20-50 range)
- Migration naming conventions

### Deferred Ideas (OUT OF SCOPE)

(None — no deferred items for Phase 1)

</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| SETUP-01 | Project foundation: Express server, Prisma migrations, security middleware, error handling | All sections below — this phase IS the SETUP-01 implementation |

</phase_requirements>

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Express server + graceful shutdown | API / Backend | — | HTTP layer is the entry point; shutdown handling is a server lifecycle concern |
| Prisma schema + migrations | Database / Storage | API / Backend | ORM generates types from schema; migrations run at deployment time |
| Custom error class hierarchy | API / Backend | — | Pure TypeScript, no runtime dependency |
| Global error handler middleware | API / Backend | — | Express middleware intercepts all thrown errors at the routing layer |
| Zod validation middleware | API / Backend | — | Validates HTTP request body/params/headers before reaching business logic |
| Helmet security headers | API / Backend | — | HTTP security layer, runs before routing |
| CORS configuration | API / Backend | — | Cross-origin policy enforcement at HTTP layer |
| Environment validation | API / Backend | — | Synchronous startup check — server refuses to start without valid config |
| Test infrastructure | API / Backend | — | Vitest + supertest test the HTTP layer |

**Key insight:** Every capability in Phase 1 lives in the API/Backend tier. There is no client-side, CDN, or frontend server component. This simplifies the implementation — all code runs in one runtime context.

## Standard Stack

### Core
| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| **express** | `^5.0.0` (5.2.1 latest) | API framework | Industry standard, native async support in v5 eliminates `express-async-errors` |
| **@prisma/client** | `^6.0.0` (6.19.2 prev, 7.8.0 latest) | ORM and database client | Type-safe, auto-generated types, Prisma migrate CLI included |
| **zod** | `^3.0.0` (4.4.3 latest) | Input validation | TypeScript-first, excellent inference, better DX than Joi |
| **helmet** | `^8.0.0` (8.2.0 latest) | Security headers | Sets CSP, X-Frame-Options, HSTS, etc. without config |
| **cors** | `^2.0.0` (2.8.6 latest) | CORS | Configurable allowlist, credentials, methods |
| **dotenv** | `^16.0.0` (17.4.2 latest) | Env loading | Zero-config .env file loading for development |
| **morgan** | `^1.0.0` (1.11.0 latest) | HTTP logging | Request logging with predefined formats (combined/dev) |

### Supporting
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| **typescript** | `^6.0.0` (6.0.3 latest) | Language | Required for Prisma generated types and Zod inference |
| **tsx** | `^4.0.0` (4.22.4 latest) | TS execution | Run TypeScript without build step for dev and migrations |
| **vitest** | `^4.0.0` (4.1.8 latest) | Testing framework | Fast, Vite-native, Jest-compatible, preferred over Jest |
| **supertest** | `^7.0.0` (7.2.2 latest) | HTTP testing | Test Express endpoints without real HTTP server |
| **@types/express** | `^5.0.0` (5.0.6 latest) | Express types | Required for TypeScript compilation |
| **@types/node** | `25.9.1` | Node types | Required for TypeScript compilation |
| **@types/bcrypt** | `6.0.0` | bcrypt types | Will be needed in Phase 2 |
| **@types/jsonwebtoken** | `9.0.10` | JWT types | Will be needed in Phase 2 |
| **@types/cors** | `2.8.19` | CORS types | Required for TypeScript compilation |
| **@types/morgan** | `1.9.10` | Morgan types | Required for TypeScript compilation |

### Not Using (explicitly excluded)
| Library | Reason for Exclusion |
|---------|---------------------|
| **express-async-errors** | Express v5 natively handles async errors — no wrapper needed [D-07] |
| **Passport.js** | Over-abstracted; `jsonwebtoken` directly is cleaner for JWT auth |
| **Jest** | Vitest is faster and better integrated with Vite |
| **body-parser** | Built into Express v5 via `express.json()` |
| **morgan** (postinstall) | No postinstall script detected — safe to install |

### Installation Commands
```bash
# Core runtime
npm install express@^5.0.0 @prisma/client zod helmet cors morgan dotenv

# Dev dependencies
npm install -D prisma typescript tsx vitest supertest @types/express @types/node @types/cors @types/morgan

# Initialize TypeScript
npx tsc --init

# Initialize Prisma
npx prisma init
```

**Version verification:** All versions verified via `npm view <pkg> version` on 2026-06-02.

## Package Legitimacy Audit

> slopcheck unavailable — all packages tagged `[ASSUMED]` per protocol. Planner must insert `checkpoint:human-verify` before each install.

| Package | Registry | Age | Downloads | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-----------|-------------|-----------|-------------|
| express | npm | ~13 yrs | 38M/wk | [expressjs/express](https://github.com/expressjs/express) | N/A | Flagged — planner must verify |
| @prisma/client | npm | ~9 yrs | 7M/wk | [prisma/prisma](https://github.com/prisma/prisma) | N/A | Flagged — planner must verify |
| zod | npm | ~5 yrs | 18M/wk | [colinhacks/zod](https://github.com/colinhacks/zod) | N/A | Flagged — planner must verify |
| helmet | npm | ~12 yrs | 25M/wk | [helmetjs/helmet](https://github.com/helmetjs/helmet) | N/A | Flagged — planner must verify |
| cors | npm | ~13 yrs | 53M/wk | [expressjs/cors](https://github.com/expressjs/cors) | N/A | Flagged — planner must verify |
| morgan | npm | ~13 yrs | 26M/wk | [expressjs/morgan](https://github.com/expressjs/morgan) | N/A | Flagged — planner must verify |
| vitest | npm | ~5 yrs | 9M/wk | [vitest-dev/vitest](https://github.com/vitest-dev/vitest) | N/A | Flagged — planner must verify |
| supertest | npm | ~12 yrs | 24M/wk | [ladjs/supertest](https://github.com/ladjs/supertest) | N/A | Flagged — planner must verify |
| dotenv | npm | ~12 yrs | 95M/wk | [motdotla/dotenv](https://github.com/motdotla/dotenv) | N/A | Flagged — planner must verify |
| typescript | npm | ~13 yrs | 60M/wk | [microsoft/TypeScript](https://github.com/microsoft/TypeScript) | N/A | Flagged — planner must verify |

**Packages removed due to slopcheck [SLOP] verdict:** None
**Packages flagged as suspicious [SUS]:** None — all packages are established ecosystem standards with millions of weekly downloads and GitHub source repos.

*slopcheck was unavailable at research time. All packages above are tagged [ASSUMED] per protocol. Planner MUST gate each install behind a `checkpoint:human-verify` task.*

## Architecture Patterns

### System Architecture Diagram

```
Request (HTTP)
    │
    ▼
┌──────────────────────────────────────────────┐
│  Security Middleware (helmet, CORS)           │
│  Body Parser (express.json())                 │
│  Logger (morgan)                              │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Routes (Router per resource)                 │
│  - /tasks, /users, /comments, /auth           │
│  - Validation middleware (Zod schemas)        │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Controllers (thin HTTP layer)                │
│  - Extract req.body / params / query          │
│  - Call services, return JSON responses       │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Services (business logic — placeholder)      │
│  - Will be filled in Phase 3                  │
└──────────────┬───────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────┐
│  Models (Prisma client — data access only)    │
│  - Prisma generated types                     │
│  - DB queries                                 │
└──────────────┬───────────────────────────────┘
               │
               ▼
      PostgreSQL (via Prisma)
```

### Recommended Project Structure

```
src/
├── app.ts                    # Express app setup, middleware wiring
├── server.ts                 # Server entry, graceful shutdown
├── config/
│   └── index.ts              # Environment config + validation
├── routes/
│   ├── index.ts              # Route aggregator
│   ├── tasks.routes.ts       # Task endpoints (Phase 3)
│   ├── users.routes.ts       # User endpoints (Phase 2)
│   ├── comments.routes.ts    # Comment endpoints (Phase 4)
│   └── health.routes.ts      # Health check (Phase 1)
├── controllers/
│   ├── health.controller.ts  # Health check controller
│   ├── task.controller.ts    # (Phase 3)
│   ├── user.controller.ts    # (Phase 2)
│   └── comment.controller.ts # (Phase 4)
├── services/                 # (Phase 3+ — placeholder structure now)
├── models/                   # Prisma client instance (Phase 1)
│   └── index.ts
├── middleware/
│   ├── error.middleware.ts   # Global error handler
│   ├── validation.middleware.ts
│   └── logger.middleware.ts
├── utils/
│   └── errors.ts             # Custom error class hierarchy
├── schemas/                  # Zod validation schemas
│   └── index.ts
└── db/
    ├── migrations/           # Prisma migrations
    └── schema.prisma         # Prisma schema
tests/
├── setup.ts                  # Vitest + supertest setup
├── teardown.ts               # Test teardown
└── unit/
    └── errors.test.ts        # Error class tests
```

### Pattern 1: Express v5 Error Handling

Express v5 natively propagates async errors — no wrapper needed.

**Implementation:**

```typescript
// src/utils/errors.ts
// D-05: Custom error class hierarchy

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

export class NotFoundError extends AppError {
  constructor(resource: string, identifier?: string) {
    super('NOT_FOUND', `${resource} not found${identifier ? `: ${identifier}` : ''}`, 404);
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details?: any) {
    super('VALIDATION_ERROR', message, 400, details);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Access denied') {
    super('FORBIDDEN', message, 403);
  }
}

export class ConflictError extends AppError {
  constructor(message: string, details?: any) {
    super('CONFLICT', message, 409, details);
  }
}
```

```typescript
// src/middleware/error.middleware.ts
// D-06: Global error handler

import type { Request, Response, NextFunction } from 'express';
import { AppError } from '../utils/errors.js';

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  // Log error (don't expose stack in production)
  console.error(`[ERROR] ${req.method} ${req.path}:`, err.message);

  if (err instanceof AppError) {
    res.status(err.statusCode).json(err.toJSON());
    return;
  }

  // Unknown errors → 500
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: process.env.NODE_ENV === 'production' ? 'Internal server error' : err.message,
    },
  });
}

// Express v5: catch async errors automatically (D-07)
// No express-async-errors needed!
```

### Pattern 2: Config Validation at Startup

**Implementation:**

```typescript
// src/config/index.ts
// D-15: Environment validation

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

  return {
    DATABASE_URL: dbUrl!,
    JWT_SECRET: jwtSecret!,
    PORT: port,
    NODE_ENV: process.env.NODE_ENV ?? 'development',
  };
}

export const config = validate();
```

### Pattern 3: Zod Validation Middleware

**Implementation:**

```typescript
// src/middleware/validation.middleware.ts
// D-08, D-09, D-10

import type { Request, Response, NextFunction } from 'express';
import { ZodSchema, ZodError } from 'zod';

export function validate(schema: ZodSchema, target: 'body' | 'query' | 'params' = 'body') {
  return (req: Request, res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req[target]);

    if (!result.success) {
      const details = result.error.flatten();
      res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Request validation failed',
          details,
        },
      });
      return;
    }

    // Replace with validated data
    req[target] = result.data;
    next();
  };
}

// Usage in routes:
// router.post('/tasks', validate(taskCreateSchema), taskController.create);
```

### Pattern 4: Prisma Schema with Indexes and Soft Delete

**Implementation:**

```prisma
// prisma/schema.prisma
// D-12, D-13, D-14

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?

  tasks     Task[]   @relation("TaskOwner")
  assigned  Task[]   @relation("TaskAssignee")
  comments  Comment[]

  @@index([email])
  @@index([deletedAt])  // D-14: soft delete filter
}

model Task {
  id          String   @id @default(uuid())
  title       String
  description String?
  status      String   @default("TODO")  // D-12: TODO/IN_PROGRESS/DONE
  ownerId     String
  assigneeId  String?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  deletedAt   DateTime?

  owner       User     @relation("TaskOwner", fields: [ownerId], references: [id])
  assignee    User?    @relation("TaskAssignee", fields: [assigneeId], references: [id])
  comments    Comment[]

  @@index([ownerId])       // D-13: FK index
  @@index([assigneeId])    // D-13: FK index
  @@index([status])        // D-13: status filter
  @@index([deletedAt])     // D-14: soft delete filter
}

model Comment {
  id        String   @id @default(uuid())
  content   String
  taskId    String
  authorId  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  deletedAt DateTime?

  task      Task     @relation(fields: [taskId], references: [id])
  author    User     @relation(fields: [authorId], references: [id])

  @@index([taskId])     // D-13: FK index
  @@index([authorId])   // D-13: FK index
  @@index([deletedAt])  // D-14: soft delete filter
}
```

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Password hashing | Custom crypto or MD5 | `bcrypt` ^5.x | Salt generation, cost factor tuning, timing attack mitigation are handled correctly |
| JWT handling | Custom base64 encode/decode | `jsonwebtoken` ^9.x | Signature verification, expiration handling, error classification |
| Environment variable loading | Manual `process.env` access with string keys | `dotenv` + typed config object | Type safety, fail-fast validation, no typos in env var names |
| Security headers | Manual `res.setHeader` calls | `helmet` ^8.x | 12+ security headers set correctly, CSP auto-generated |
| HTTP body parsing | Manual string parsing | Express `express.json()` | Handles content-type negotiation, size limits, encoding |
| Error class hierarchy | Ad-hoc `throw new Error('msg')` | Custom AppError hierarchy | Consistent JSON shape, typed status codes, code field for client handling |
| Database connection | Raw `pg` with manual pool | Prisma client | Type-safe queries, auto-generated types, migration tooling, connection pooling |
| Async error propagation | `express-async-errors` wrapper | Express v5 native | No dependency overhead; Express v5 propagates async errors to error handlers automatically |

**Key insight:** All of these problems have battle-tested, actively maintained solutions in the npm ecosystem. Hand-rolling them introduces security vulnerabilities (password hashing, JWT), runtime errors (env typos), or maintenance burden (inconsistent error formats).

## Common Pitfalls

### Pitfall 1: Sync Startup Without Graceful Shutdown

**What goes wrong:** Server starts synchronously, DB connection fails silently, process hangs on SIGTERM with open connections.

**Why it happens:** Connecting to DB at module load time blocks startup. No SIGTERM handler means connections aren't closed on shutdown.

**How to avoid:**
```typescript
// server.ts — async init + graceful shutdown
import { app } from './app.js';
import { prisma } from './models/index.js';

const server = app.listen(config.PORT, () => {
  console.log(`Server running on port ${config.PORT}`);
});

// Graceful shutdown
const shutdown = async (signal: string) => {
  console.log(`${signal} received, shutting down...`);
  server.close();
  await prisma.$disconnect();
  process.exit(0);
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

**Warning signs:** `process.exit()` called without cleanup; no `server.close()`; `prisma.$disconnect()` missing.

### Pitfall 2: Missing Request Validation — No Error Handler on Invalid Input

**What goes wrong:** Malformed requests pass through to services/DB, causing crashes or unexpected behavior. Different endpoints return different error shapes.

**Why it happens:** Validation applied inconsistently. Validation errors bubble up as 500 instead of 400.

**How to avoid:** All routes use `validate(schema)` middleware. Global error handler distinguishes `AppError` (proper status) from generic `Error` (500).

**Warning signs:** `JSON.parse` errors exposed to clients; `undefined` field access in services; inconsistent 400 vs 500 responses.

### Pitfall 3: Prisma Schema Without Indexes — Slow Queries at Scale

**What goes wrong:** `EXPLAIN ANALYZE` shows Seq Scans on large tables. Foreign key lookups O(n) instead of O(log n).

**Why it happens:** Indexes added reactively after performance problems appear.

**How to avoid:** Define all indexes in initial Prisma schema. `@@index([ownerId])`, `@@index([assigneeId])`, `@@index([taskId])`, `@@index([status])` on first migration.

**Warning signs:** No `@@index` in schema; any `CREATE INDEX` after initial deployment.

### Pitfall 4: Validation Error Format Inconsistency

**What goes wrong:** Zod errors return as `z.errors`, Express body-parser errors as `{ message }`, custom errors as `{ error: { code: string } }`. Clients can't handle all formats reliably.

**Why it happens:** No single error format standard. Each middleware produces different JSON structure.

**How to avoid:** Global error handler normalizes all errors to `{ error: { code, message, details? } }`. Validation middleware uses the same format with `VALIDATION_ERROR` code.

**Warning signs:** `err.message` returned directly to client; different JSON shapes from different error sources.

### Pitfall 5: Config Validation Too Late

**What goes wrong:** Server starts, routes register, first DB query fails — then crash. Problem was env var missing at startup.

**Why it happens:** Env validation happens lazily (first request) instead of at startup.

**How to avoid:** Call `validate()` synchronously in `config/index.ts` which is imported at module load time. Server crashes immediately on missing env vars with clear error messages.

**Warning signs:** Env var validation inside route handlers; `process.env.SOMETHING` accessed without existence check.

## Code Examples

### Example 1: Express App Setup with Full Middleware Chain

```typescript
// src/app.ts
import express from 'express';
import helmet from 'helmet';
import cors from 'cors';
import morgan from 'morgan';
import { errorHandler } from './middleware/error.middleware.js';
import { healthRoutes } from './routes/health.routes.js';

export const app = express();

// Security first
app.use(helmet());          // D-16: Security headers
app.use(cors());            // D-17: CORS configured

// Parsing
app.use(express.json());    // Express v5 built-in body parser

// Logging
app.use(morgan('combined'));

// Routes
app.use('/health', healthRoutes);

// Global error handler — MUST be last
app.use(errorHandler);

// 404 handler
app.use((req, res) => {
  res.status(404).json({ error: { code: 'NOT_FOUND', message: `Route ${req.method} ${req.path} not found` } });
});
```

### Example 2: Health Check Route (First Endpoint)

```typescript
// src/routes/health.routes.ts
import { Router } from 'express';
import { prisma } from '../models/index.js';

const router = Router();

router.get('/', async (req, res) => {
  try {
    await prisma.$queryRaw`SELECT 1`;  // DB health check
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  } catch (err) {
    res.status(503).json({ status: 'error', message: 'Database unavailable' });
  }
});

export { router as healthRoutes };
```

### Example 3: Prisma Model with Soft Delete

```typescript
// src/models/index.ts
import { PrismaClient } from '@prisma/client';

export const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
});
```

```typescript
// src/models/task.model.ts (foundation — placeholder methods)
// These will be filled in Phase 3 but the structure is set now

export class TaskModel {
  async findById(id: string, includeDeleted = false) {
    const where = includeDeleted
      ? { id }
      : { id, deletedAt: null };

    return prisma.task.findUnique({ where });
  }

  // Soft delete method — used by future phases
  async softDelete(id: string) {
    return prisma.task.update({
      where: { id },
      data: { deletedAt: new Date() },
    });
  }
}
```

### Example 4: Validation Schema Co-Located with Route

```typescript
// src/schemas/task.schemas.ts
// D-09: Schemas co-located with routes

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

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `body-parser` npm package | Express v5 built-in `express.json()` | Express v5 (2024) | One fewer dependency |
| `express-async-errors` wrapper | Express v5 native async propagation | Express v5 (2024) | No wrapper dependency, simpler mental model |
| Callback-style middleware | `async/await` throughout | Express v4.16+ → v5 | Cleaner code, no callback pyramid |
| `.findOne()` / `.findMany()` with raw filter | Prisma query builder with typed includes | Knex → Prisma | Type safety, auto-generated client, migrations |
| Manual config validation | `dotenv` + TypeScript config module | Custom → dotenv | Type safety, fail-fast, no env typos |
| Ad-hoc error throwing | Custom error class hierarchy | Random Error → AppError | Consistent JSON format, typed status codes |

**Deprecated/outdated:**
- `express-async-errors`: No longer needed with Express v5. Can be removed from STACK.md recommendation. [D-07] already accounts for this.
- `body-parser` npm package: Deprecated in favor of Express built-in `express.json()`.
- Callback-style route handlers: Replaced by `async/await` as standard.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Prisma v6.x is preferred over v7.x (latest) because STACK.md research recommends v6.x | Standard Stack | v7.x may have different API patterns; plan needs minor adjustment if v7 is used instead |
| A2 | Express v5 native async propagation handles all async errors without wrapper | Error Handling | Express v5 confirms this behavior; verified via official docs pattern |
| A3 | Helmet v8.x works with Express v5 without special configuration | Security | Helmet has always been Express-compatible; minor config needed for CSP |
| A4 | Default page size 20-50 range — no explicit decision, use 20 as default | Pagination | May need adjustment if user has different preference during Phase 3 |

## Open Questions

1. **CORS configuration specifics**
   - What we know: `cors()` needs configuration for allowed origins
   - What's unclear: Should origins be an env var allowlist, or hardcoded for MVP?
   - Recommendation: Use `CORS_ALLOWED_ORIGINS` env var as comma-separated list for Phase 1

2. **Morgan logging format**
   - What we know: `morgan('combined')` is standard for production
   - What's unclear: Should we also log request body? (Security/privacy concern)
   - Recommendation: Log method, path, status, response-time, and user-agent only for Phase 1

3. **Prisma v6 vs v7 choice**
   - What we know: v7.8.0 is latest, v6.19.2 is previous stable, STACK.md says v6.x
   - What's unclear: Are there breaking changes between v6 and v7 that affect this project's timeline?
   - Recommendation: Use v6.x as specified in research for stability; upgrade to v7 in a later phase

4. **Database URL validation**
   - What we know: DATABASE_URL is required
   - What's unclear: Should we also validate the format (postgres://...) or just presence?
   - Recommendation: Only validate presence for Phase 1; add format validation in Phase 2 when connection testing is available

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | Runtime | ✓ | 24.11.0 | — |
| npm | Package manager | ✓ | 11.7.0 | — |
| PostgreSQL | Database | ? | — | Prompt user to install/configure |
| TypeScript | Language | Via `npx tsx` | latest | — |
| Vitest | Test framework | Via `npx vitest` | 4.1.8 | — |

**Missing dependencies with no fallback:**
- **PostgreSQL**: Required for Prisma migrations and runtime. User must install and provide `DATABASE_URL`. This is the single blocker for full execution.

**Missing dependencies with fallback:**
- **PostgreSQL client (psql)**: Not required for app runtime — only for manual DB inspection. If missing, use Prisma Studio (`npx prisma studio`) instead.

**PostgreSQL detection:**
```bash
# Check if psql is available
command -v psql 2>/dev/null && psql --version || echo "psql not found"

# Check if a postgres server is running
pg_isready 2>/dev/null || echo "No PostgreSQL server detected"
```

## Validation Architecture

> Required — `workflow.nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Vitest v4.1.8 + supertest v7.2.2 |
| Config file | `vitest.config.ts` (TypeScript config) |
| Quick run command | `npx vitest run` |
| Full suite command | `npx vitest run --reporter=verbose` |
| Test files | `tests/unit/*.test.ts`, `tests/integration/*.test.ts` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| SETUP-01a | Express server starts on configured port | Smoke | `npx vitest run tests/smoke/server.test.ts` | ❌ Wave 0 |
| SETUP-01b | Graceful shutdown on SIGTERM | Smoke | `npx vitest run tests/smoke/shutdown.test.ts` | ❌ Wave 0 |
| SETUP-01c | PostgreSQL connects via Prisma client | Integration | `npx vitest run tests/integration/db-connect.test.ts` | ❌ Wave 0 |
| SETUP-01d | Migrations apply successfully | Integration | `npx vitest run tests/integration/migrations.test.ts` | ❌ Wave 0 |
| SETUP-01e | Security headers set (helmet) | Unit | `npx vitest run tests/unit/helmet-headers.test.ts` | ❌ Wave 0 |
| SETUP-01f | CORS configured | Unit | `npx vitest run tests/unit/cors.test.ts` | ❌ Wave 0 |
| SETUP-01g | Zod validation returns consistent error format | Unit | `npx vitest run tests/unit/validation.test.ts` | ❌ Wave 0 |
| SETUP-01h | JWT_SECRET validation at startup (>=64 chars) | Unit | `npx vitest run tests/unit/config-validation.test.ts` | ❌ Wave 0 |
| SETUP-01i | Custom error class hierarchy works | Unit | `npx vitest run tests/unit/errors.test.ts` | ❌ Wave 0 |
| SETUP-01j | Global error handler returns correct JSON shape | Unit | `npx vitest run tests/unit/error-handler.test.ts` | ❌ Wave 0 |

### Sampling Rate
- **Per task commit:** `npx vitest run` (all unit tests, <30 seconds expected)
- **Per wave merge:** `npx vitest run --reporter=verbose` (full suite)
- **Phase gate:** Full suite green before `/gtd-verify-work`

### Wave 0 Gaps

- [ ] `vitest.config.ts` — Vitest configuration (tsconfig, test match patterns, environment)
- [ ] `tests/setup.ts` — Test environment setup (Vitest global setup, Prisma test client)
- [ ] `tests/teardown.ts` — Test environment teardown (Prisma disconnect)
- [ ] `tests/unit/errors.test.ts` — SETUP-01i: Custom error classes
- [ ] `tests/unit/error-handler.test.ts` — SETUP-01j: Global error handler JSON shape
- [ ] `tests/unit/config-validation.test.ts` — SETUP-01h: Config validation at startup
- [ ] `tests/unit/validation.test.ts` — SETUP-01g: Zod validation error format
- [ ] `tests/unit/helmet-headers.test.ts` — SETUP-01e: Security headers
- [ ] `tests/unit/cors.test.ts` — SETUP-01f: CORS middleware
- [ ] `tests/smoke/server.test.ts` — SETUP-01a: Server starts
- [ ] `tests/smoke/shutdown.test.ts` — SETUP-01b: Graceful shutdown
- [ ] `tests/integration/db-connect.test.ts` — SETUP-01c: Prisma connection
- [ ] `tests/integration/migrations.test.ts` — SETUP-01d: Migration success

## Security Domain

> Required when `security_enforcement` is enabled (absent = enabled). Omit only if explicitly `false`.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Handled in Phase 2 (JWT middleware) |
| V3 Session Management | No | Handled in Phase 2 (JWT storage/verification) |
| V4 Access Control | No | Handled in Phases 2-4 (service-layer authorization) |
| V5 Input Validation | YES | `zod` validation on all incoming requests — D-08, D-09, D-10 |
| V6 Cryptography | Partial | `dotenv` for secrets loading; full crypto (JWT_SECRET, bcrypt) in Phase 2 |
| V7 Error Handling | YES | Custom error class hierarchy with consistent JSON shape — D-04, D-05, D-06 |
| V8 Data Protection | YES | `helmet` security headers, CORS, env variable validation at startup — D-15, D-16, D-17 |

### Known Threat Patterns for Express/Node.js API

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Information disclosure via error stack | Information Disclosure | Global error handler never exposes stack traces in production |
| Missing security headers | Information Disclosure | `helmet()` sets HSTS, CSP, X-Frame-Options, etc. |
| CORS misconfiguration | Elevation of Privilege | `cors()` with explicit allowlist (not `*` in production) |
| Invalid env vars causing undefined behavior | Denial of Service | Config validation at startup throws immediately on missing/invalid env |
| Unvalidated input reaching database | Tampering / Injection | `zod` validation middleware on all routes before controller dispatch |
| Sensitive data in logs | Information Disclosure | `morgan` configured to NOT log request bodies (contains PII) |

## Sources

### Primary (HIGH confidence)
- [npm registry](https://www.npmjs.com/) — Package versions verified 2026-06-02 (express 5.2.1, prisma 7.8.0/prev 6.19.2, zod 4.4.3, helmet 8.2.0, cors 2.8.6, vitest 4.1.8, supertest 7.2.2, etc.)
- [Express.js Official Docs v5](https://expressjs.com/en/5x/api) — Express v5 routing, middleware, error handling, async support
- [Prisma Documentation](https://www.prisma.io/docs) — Schema design, migrations, index creation, soft delete patterns
- [Zod Documentation](https://zod.dev) — Schema validation, error formatting, TypeScript inference

### Secondary (MEDIUM confidence)
- [Research: .planning/research/SUMMARY.md](file:///Users/davide/repos/get-tasks-done-demo-app/.planning/research/SUMMARY.md) — Pitfall mappings, stack recommendations, phase structure
- [Research: .planning/research/STACK.md](file:///Users/davide/repos/get-tasks-done-demo-app/.planning/research/STACK.md) — Tech versions, installation commands, alternatives
- [Research: .planning/research/ARCHITECTURE.md](file:///Users/davide/repos/get-tasks-done-demo-app/.planning/research/ARCHITECTURE.md) — Layered architecture patterns, project structure, anti-patterns

### Tertiary (LOW confidence)
- [Express.js Production Best Practices](https://expressjs.com/en/advanced/best-practice-production.html) — Security, error handling recommendations
- [OWASP API Security Top 10](https://owasp.org/API-Security/) — Threat patterns for REST APIs

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — All packages verified on npm registry with current versions
- Architecture: HIGH — Standard Express patterns documented in multiple sources
- Pitfalls: HIGH — Based on documented Express/Node production issues, Prisma community posts

**Research date:** 2026-06-02
**Valid until:** ~30 days for stack versions; architectural patterns are stable