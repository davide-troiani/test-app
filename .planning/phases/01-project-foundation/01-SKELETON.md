# Walking Skeleton - Task Management API

**Phase:** 1
**Generated:** 2026-06-02

## Capability Proven End-to-End

A developer can start the Express API, apply the Prisma migration, call GET /health for a real database read, and call POST /health/probe for a real HTTP-triggered database write.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Framework | Express v5 with TypeScript | Matches the locked stack and uses native async error propagation without express-async-errors. |
| Data layer | PostgreSQL with Prisma Client and Prisma Migrate | Provides typed database access, migration history, and the users/tasks/comments schema required by later slices. |
| Auth | JWT_SECRET is validated in Phase 1; JWT routes and middleware are implemented in Phase 2 | Phase 1 must validate secret safety, while registration/login behavior belongs to the authentication slice. |
| Validation | Zod route middleware with canonical error envelope | Enforces D-08 through D-10 and gives later routes a shared validation contract. |
| Error handling | AppError hierarchy plus global Express error handler | Enforces consistent `{ error: { code, message, details? } }` responses and prevents production stack disclosure. |
| Security middleware | helmet(), allowlist CORS, and body-safe morgan logging | Covers the Phase 1 ASVS L1 security controls before business endpoints are added. |
| Deployment target | Local full-stack run command: `rtk npm run db:deploy && rtk npm run db:generate && rtk npm run dev` | The project is an API service with local PostgreSQL as the first executable environment. |
| Directory layout | Layered Express layout under src/routes, src/controllers, src/services, src/models, src/middleware, src/config, src/utils, and src/schemas | Implements D-01 and D-02 while keeping resource-specific files clear for future phases. |

## Stack Touched in Phase 1

- [x] Project scaffold (Node package, TypeScript, Vitest, dev/build/test scripts)
- [x] Routing - /health and /health/probe
- [x] Database - Prisma read through GET /health and write through POST /health/probe
- [x] API interaction - supertest and curl-compatible HTTP endpoints
- [x] Deployment - documented local full-stack run command

## Out of Scope (Deferred to Later Slices)

- User registration, login, JWT issuance, and protected route middleware
- Password hashing and bcrypt integration
- Task CRUD and owner/assignee authorization
- Status transitions, optimistic concurrency, and comments
- Task priorities, due dates, search, filtering, tags, attachments, realtime updates, and admin UI

## Subsequent Slice Plan

Each later phase adds one vertical slice on top of this skeleton without altering its architectural decisions:

- Phase 2: Users can register, login, and access protected endpoints via JWT.
- Phase 3: Users can create, read, update, delete, and assign tasks.
- Phase 4: Users can transition task status and collaborate with comments.
