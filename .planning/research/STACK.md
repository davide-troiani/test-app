# Stack Research

**Domain:** Task Management REST API
**Researched:** 2026-06-02
**Confidence:** HIGH

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| **Express.js** | ^5.x | API Framework | Industry standard with massive ecosystem, excellent middleware support, and well-documented patterns. Express v5 supports async handlers natively, which aligns with modern PostgreSQL patterns. |
| **Prisma** | ^6.x | ORM & Migration | Type-safe database access with automatic migrations. PostgreSQL-first with excellent connection pooling support. Generates type-safe clients from schema, reducing runtime errors. |
| **PostgreSQL** | 16+ | Database | Chosen explicitly. ACID compliance, JSONB support for flexibility, excellent performance for relational data like tasks/users. |
| **bcrypt** | ^5.x | Password Hashing | Industry standard for password hashing with built-in salt. NACL/recommended settings for cost factor. |
| **jsonwebtoken** | ^9.x | JWT Auth | Lightweight, standard library for JWT. No Passport dependency for stateless auth. |
| **Zod** | ^3.x | Input Validation | TypeScript-first schema validation with excellent inference. Better DX than Joi for TS projects. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| **helmet** | ^8.x | Security Headers | Set HTTP security headers (CSP, X-Frame, etc.). Use on every production API. |
| **cors** | ^2.x | CORS | Configure Cross-Origin Resource Sharing. Required for browser clients. |
| **morgan** | ^1.x | HTTP Logging | Request logging middleware. Essential for debugging and audit trails. |
| **express-async-errors** | ^3.x | Async Error Handling | Automatically catches async errors and passes to error handler. Eliminates try/catch in every handler. |
| **zod-to-json-schema** | — | Schema Documentation | Convert Zod schemas to JSON Schema for OpenAPI docs. |
| **swagger-ui-express** | ^5.x | API Documentation | Serve Swagger/OpenAPI documentation. |
| **dotenv** | ^16.x | Environment Config | Load environment variables from .env files in development. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| **Vitest** | Testing Framework | Vite-native, fast, Jest-compatible. Preferred over Jest for new projects. |
| **supertest** | API Testing | Test Express HTTP endpoints. Integrates with Vitest. |
| **tsx** | TypeScript Execution | Run TypeScript files without compilation. Faster for scripts/migrations. |
| **eslint** + **typescript-eslint** | Linting | Enforce code quality and catch errors early. |
| **prettier** | Formatting | Consistent code style across team. |
| **nodemon** | Development Server | Auto-restart on file changes. Use with `--watch` flag. |
| **pg-format** | SQL Formatting | Format raw SQL queries for readability. |

## Installation

```bash
# Core
npm install express@^5.0.0 prisma @prisma/client bcrypt jsonwebtoken zod helmet cors morgan express-async-errors dotenv zod-to-json-schema swagger-ui-express

# Dev dependencies
npm install -D vitest @vitest/ui supertest @types/express @types/node @types/bcrypt @types/jsonwebtoken @types/cors @types/morgan @types/swagger-ui-express typescript tsx nodemon eslint prettier typescript-eslint eslint-config-prettier

# Initialize Prisma
npx prisma init
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Express.js | Fastify | Choose Fastify when: raw performance is critical, built-in JSON Schema validation is desired, or team prefers opinionated structure. Fastify benchmarks show 2-3x faster throughput, but Express ecosystem is larger and more battle-tested for typical CRUD APIs. |
| Prisma | Knex.js | Choose Knex when: you need fine-grained control over SQL, prefer query builder pattern over ORM, or need to mix raw SQL with query builder. Knex is better for complex reporting queries or when team knows SQL well. |
| Prisma | raw `pg` | Choose `pg` directly when: maximum performance is required, minimal abstraction is preferred, or you're building a high-throughput service. Sacrifices type safety and migration tooling. |
| Zod | Joi | Choose Joi when: working on plain JavaScript projects or needing broader ecosystem integration. Zod has better TypeScript inference and is more actively maintained. |
| JWT (stateless) | Session-based auth | Choose sessions when: you need to revoke tokens instantly, store sensitive data in session, or are building SSR with cookies. JWT is simpler for stateless REST APIs. |
| Vitest | Jest | Choose Jest when: existing test suite uses Jest, enterprise tooling requires Jest, or team is familiar with Jest conventions. Vitest is faster and has better Vite integration. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| **Passport.js** | Over-abstracted, opinionated, heavy middleware chain. For JWT auth, just use `jsonwebtoken` directly. Passport adds complexity without value for token-based auth. | `jsonwebtoken` directly with custom middleware |
| **Mongoose** | MongoDB ODM. Not applicable for PostgreSQL project. | Prisma (for PostgreSQL) |
| **Sequelize** | Legacy ORM with outdated patterns, poor TypeScript support. Prisma and Drizzle are superior choices for new projects. | Prisma or Knex |
| **Jest** (new projects) | Slow startup, heavy configuration, Vite integration issues. Vitest is faster and modern. | Vitest |
| **body-parser** (explicit) | Built into Express v5. No need for separate package. | Express `express.json()` middleware |
| **crypto** (for passwords) | Use bcrypt or argon2. Raw crypto for hashing is error-prone and slow by default. | `bcrypt` |
| **mongoose** | MongoDB ODM, incompatible with PostgreSQL choice. | N/A |

## Stack Patterns by Variant

**If TypeScript is required (recommended):**
- Use `tsx` for development execution
- Use Prisma's generated types throughout
- Use Zod with TypeScript inference (`z.infer<typeof schema>`)
- Use `typescript-eslint` for type-aware linting

**If maximum performance is critical:**
- Consider Fastify instead of Express
- Use raw `pg` with connection pooling
- Implement response compression (`compression` middleware)
- Consider query result caching

**If rapid prototyping is priority:**
- Use Prisma with SQLite first (easier setup), migrate to PostgreSQL later
- Skip OpenAPI docs initially, add via `swagger-ui-express` later
- Use `concurrently` to run dev server and tests together

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| Express ^5.0 | Node.js 18+ | Express v5 requires Node 18+. Async route handlers now work natively. |
| Prisma ^6.x | Node.js 18+ | Requires Node 18+. PostgreSQL 12+ recommended. |
| Zod ^3.x | Node.js 14+ (TS 4+) | TypeScript-first. v4 has breaking changes around `z.input/output`. |
| Vitest ^2.x | Node.js 18+ | Vite 6 peer dependency. Works with TypeScript out of the box. |
| bcrypt ^5.x | Node.js 14+ | Native addon. Cost factor 12 is recommended for production. |
| jsonwebtoken ^9.x | Node.js 14+ | ESM and CJS compatible. HS256 is fine for most use cases. |
| helmet ^8.x | Node.js 16+ | Sets security headers. No special configuration needed for basic use. |

## PostgreSQL-Specific Notes

### Connection Pooling

Prisma handles connection pooling automatically, but for production:
- Set `connection_limit` in connection string (typically 5-10 per instance)
- Use PgBouncer for multi-instance deployments
- Direct connections bypass pooling for long-running jobs

### Schema Design Patterns

For task management, use these patterns:
- `createdAt`/`updatedAt` timestamps on all tables (use `now()` default)
- UUID primary keys for distributed systems, SERIAL for single-instance
- `userId` foreign key on tasks for ownership checks
- Enum type for task status (TODO, IN_PROGRESS, DONE)

### Recommended Indexes

```sql
-- Tasks table
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_assignee ON tasks(assignee_id);

-- Comments table
CREATE INDEX idx_comments_task_id ON comments(task_id);
```

## Sources

- [Context7: /fastify/fastify](https://context7.com/fastify/fastify) — Fastify plugin architecture, route handling, TypeScript support
- [Context7: /prisma/web](https://context7.com/prisma/web) — Prisma migrations, PostgreSQL connection pooling, schema design
- [Context7: /colinhacks/zod](https://context7.com/colinhacks/zod) — Zod schema validation, TypeScript inference, email/UUID validators
- [Context7: /websites/vitest_dev](https://context7.com/websites/vitest_dev) — Vitest API testing patterns, test context
- [Express.js Official Docs](https://expressjs.com/en/5x/api) — Express v5 routing, middleware, error handling
- [Fastify Official](https://www.fastify.io/) — Fastify performance benchmarks, plugin ecosystem, TypeScript support

---
*Stack research for: Task Management API*
*Researched: 2026-06-02*