# Project Research Summary

**Project:** Task Management API
**Domain:** Task Management REST API (Node.js + PostgreSQL)
**Researched:** 2026-06-02
**Confidence:** HIGH

## Executive Summary

This is a traditional REST API built on Express.js and PostgreSQL — a well-understood domain with established patterns and tooling. Experts build these by layering concerns strictly: HTTP handling in controllers, business logic in services, and data access in models. The stack (Express v5, Prisma v6, PostgreSQL 16+, Zod, Vitest) is battle-tested for this exact use case. Authentication is JWT-based and stateless; authorization must be enforced at the service layer, not just at the HTTP layer.

The research surfaces two distinct risk profiles: **security risks** (JWT verification bypass, missing authorization, weak secrets) are the highest-stakes items and must be baked into early phases, while **data layer risks** (missing indexes, N+1 queries, unbounded list endpoints) are architectural requirements that are easy to skip but devastating at scale. The recommended approach is to build in order: project foundation first, then auth, then core features — each phase must address its associated pitfalls before declaring completeness.

## Key Findings

### Recommended Stack

The project uses Express.js v5.x with Prisma v6.x over PostgreSQL 16+ — a mature, well-supported stack with excellent TypeScript support. JWT authentication uses `jsonwebtoken` directly (avoiding Passport.js complexity), with `bcrypt` for password hashing and `Zod` for schema validation. Express v5 natively supports async handlers, which eliminates the need for `express-async-errors` workaround patterns. For testing: Vitest with supertest. The critical version constraint is **Node.js 18+** (required by both Express v5 and Prisma v6). Avoid: Passport.js, Mongoose, Sequelize, and Jest for new projects.

**Core technologies:**
- **Express.js v5.x** — API framework — native async support, massive ecosystem
- **Prisma v6.x** — ORM & migrations — type-safe, PostgreSQL-first, auto connection pooling
- **PostgreSQL 16+** — Database — ACID compliance, JSONB, excellent relational performance
- **Zod v3.x** — Input validation — TypeScript-first schema validation with inference
- **jsonwebtoken v9.x** — JWT auth — stateless, no Passport dependency
- **bcrypt v5.x** — Password hashing — industry standard with built-in salt (cost factor 12)
- **Vitest** — Testing — fast, Vite-native, Jest-compatible, preferred over Jest for new projects

### Expected Features

The MVP scope is tight: user registration/login, task CRUD with soft delete, task status workflow (TODO → IN PROGRESS → DONE), comments on tasks, task assignment, and pagination on all list endpoints. Every feature beyond these adds validation risk before product-market fit is established.

**Must have (table stakes):**
- User registration & login — JWT-based, stateless
- Task CRUD (create, read, update, delete) — with soft delete via `deleted_at`
- Task status workflow — TODO, IN PROGRESS, DONE with transition validation
- Comments on tasks — basic collaboration
- Task assignment — assign to users; filter "my tasks"
- Pagination — cursor or offset-based, 20-50 per page default

**Should have (competitive advantage):**
- Task priorities — HIGH/MEDIUM/LOW enum, low cost, significant UX impact
- Due dates — simple date field, enables overdue queries
- Task filtering & sorting — filter by status, assignee, priority; sort by created, updated, due
- Activity log — audit trail of status changes, assignments, updates

**Defer (v2+):**
- Full-text search — PostgreSQL tsvector when users have enough tasks to search
- Subtasks — hierarchical work breakdown, limit depth to 2-3 levels
- Task tags/labels — many-to-many, add when status alone isn't enough
- Bulk operations — batch efficiency, defer until manual one-by-one becomes painful

### Architecture Approach

Standard Express layered architecture: routes → controllers → services → models → PostgreSQL. Middleware handles cross-cutting concerns (auth, errors, logging) with a strict chain order. Services contain all business logic and are the correct layer for authorization checks. Models are data access only — no business logic, no direct `db` imports from controllers. Project structure follows resource-based grouping: `routes/`, `controllers/`, `services/`, `models/`, `middleware/` each contain files per domain entity (tasks, users, comments). Soft delete is implemented via `deleted_at` timestamps. Optimistic locking via a `version` column handles concurrent status updates.

**Major components:**
1. **Routes** — Express Router per resource, URL matching and HTTP method dispatch
2. **Controllers** — Thin HTTP layer, extracts request data, calls services, returns responses
3. **Services** — Business logic, authorization enforcement, domain validation (no HTTP concerns)
4. **Models** — Data access only, all DB queries encapsulated behind clean interfaces
5. **Middleware** — Auth (JWT verify, not decode), error handling, validation, logging

### Critical Pitfalls

1. **JWT verification bypass** — Using `jwt.decode()` instead of `jwt.verify()` allows forged tokens. Always use `jwt.verify()` with `JWT_SECRET` env var validated at startup (≥ 64 chars). Audit for `jwt.decode` in production code.
2. **Missing authorization checks** — Confusing authentication (is user logged in?) with authorization (can user access this task?). Enforce ownership/assignment checks in service layer on every operation.
3. **Race condition on status updates** — Concurrent requests can both succeed. Use optimistic locking with a `version` column (`WHERE id = $1 AND version = $expected`). Add `version` INTEGER column to tasks table.
4. **N+1 queries on list endpoints** — Eager-load assignee and comment relations using JOINs from the start. Create a query-count test: list tasks generates ≤ 3 queries regardless of count.
5. **Missing indexes on foreign keys** — `owner_id`, `assigned_to`, `task_id` columns need indexes. Add in migrations alongside table definitions. Verify with `EXPLAIN ANALYZE` showing Index Scan, not Seq Scan.
6. **Missing pagination on list endpoints** — `GET /tasks` without LIMIT returns unbounded results. Enforce `DEFAULT_PAGE_SIZE = 20`, `MAX_PAGE_SIZE = 100` from day one.

## Implications for Roadmap

Based on research, the following phase structure surfaces naturally from feature dependencies, architecture layers, and pitfall mappings.

### Phase 1: Project Foundation
**Rationale:** All downstream phases depend on a working Express server, PostgreSQL connection, Zod validation, and a standardized error format. This phase must be completed before anything else runs, and its pitfalls (indexes, error format, validation, config validation) affect every other phase.

**Delivers:**
- Express v5 server with graceful startup/shutdown
- Prisma schema with migrations (users, tasks, comments tables)
- Indexes on `owner_id`, `assigned_to`, `task_id`, `status`
- Global error handler with consistent `{ error: { code, message } }` shape
- Custom error class hierarchy (AppError, NotFoundError, ValidationError, ForbiddenError, ConflictError)
- Zod validation middleware for all incoming requests
- Config validation (fail fast if `JWT_SECRET` is missing or < 64 chars)
- Security headers via `helmet()`, CORS configuration
- Vitest + supertest test setup with query count thresholds
- Soft delete support (`deleted_at` column, `softDelete()` model method)

**Addresses:** Pitfalls #5 (indexes), #7 (error format), #10 (request validation), #9 (JWT secret), #8 (soft delete)

---

### Phase 2: Authentication
**Rationale:** JWT auth is a prerequisite for all task operations — users must be identifiable before any authorization can be enforced. This phase builds the auth service and registers JWT verification as the single source of truth.

**Delivers:**
- User registration endpoint (bcrypt cost factor 12, never return `password_hash`)
- Login endpoint (returns JWT with `userId` and `email` payload)
- Auth middleware using `jwt.verify()` (NOT `decode`)
- `req.user = { id, email }` injection into request context
- Rate limiting on auth endpoints (5 attempts/minute)
- `GET /users/me` endpoint (excludes password fields)
- Token expiration handling (distinguish `TokenExpiredError` from `JsonWebTokenError`)

**Addresses:** Pitfalls #1 (JWT verification), #9 (secret storage), #10 (input validation on auth endpoints), password hash exposure

---

### Phase 3: Task CRUD
**Rationale:** Core functionality. All task operations require auth (Phase 2) and foundation (Phase 1). This phase implements the full task lifecycle with authorization, pagination, eager loading, and soft delete — all pitfalls from the data layer must be addressed here.

**Delivers:**
- Task create, read, update, delete endpoints
- Authorization checks in service layer (owner can do everything, assignee can read/update status)
- Pagination on all list endpoints (page/limit params, total count, totalPages)
- Eager loading of assignee and comment counts on list queries (N+1 prevention)
- Soft delete (returns 404 for deleted tasks)
- Filter by status, assignee, owner on task list
- Optimistic locking via `version` column (handles concurrent updates)
- Task filtering and sorting by status, created_at, updated_at

**Addresses:** Pitfalls #2 (authorization), #4 (N+1), #5 (pagination), #8 (soft delete), #3 (race condition — version column ready)

---

### Phase 4: Task Status Workflow + Comments
**Rationale:** Task status transitions and comments both enhance the core task entity. They require the task CRUD foundation (Phase 3) and share the same authorization model. Implementing both together avoids duplicated integration work.

**Delivers:**
- Status transition validation (TODO → IN PROGRESS → DONE; enforce via optimistic locking)
- Concurrent status update handling (409 Conflict on version mismatch)
- Comment CRUD (create, list, delete)
- Activity log for key events (status changes, assignments)
- Assignee reassignment with validation (user must exist)

**Addresses:** Pitfalls #3 (race condition on status), #2 (authorization on comments), activity audit requirement

---

### Phase 5: Filters, Sorting & Priorities
**Rationale:** By this point, the core loop is functional. This phase adds the first differentiators (priorities, due dates, multi-field filtering) that make the API competitive with Linear/Asana. These are table-stakes for a usable task API but not core to the MVP.

**Delivers:**
- Task priority enum (HIGH, MEDIUM, LOW)
- Due dates with overdue query support
- Multi-filter API (status, assignee, priority, due date range)
- Sort by created_at, updated_at, due_date, priority
- Full-text search via PostgreSQL tsvector (optional — low priority)

**Addresses:** P2 features from FEATURES.md; enables "find work fast" UX

---

### Phase Ordering Rationale

1. **Foundation before features** — error handling, validation, and indexes must exist before any endpoint is written. Trying to retrofit them later is expensive and error-prone.
2. **Auth before resource operations** — every task, comment, and user operation requires a verified user context. Auth is the gatekeeper.
3. **CRUD before workflows** — status transitions and comments build on the task entity. Implementing them after CRUD avoids re-architecting the task model.
4. **Differentiators last** — priorities, due dates, and advanced filtering add value but aren't needed to validate the core concept. Ship the MVP first.

---

### Research Flags

**Phases likely needing deeper research during planning:**
- **Phase 4 (Status Workflow):** Concurrent update handling with optimistic locking needs verification with load testing (k6 or Artillery). The transition validation rules (e.g., "can only move to DONE if comments exist") need explicit decision from product before implementing.
- **Phase 5 (Filters & Priorities):** Full-text search implementation details — whether PostgreSQL tsvector is sufficient or if a dedicated search layer is needed. Depends on expected data scale.

**Phases with standard patterns (skip research-phase):**
- **Phase 1 (Foundation):** Express server setup, Prisma migrations, Zod validation, error handling — all well-documented, no novel patterns.
- **Phase 2 (Authentication):** JWT auth with bcrypt is a solved pattern. Only deviation is ensuring `verify` over `decode`.
- **Phase 3 (Task CRUD):** Standard REST patterns with layered architecture. Repository pattern via Prisma is well-documented.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Based on Context7 library research, official docs (Express, Prisma, Zod, Vitest), version compatibility verified across all components |
| Features | HIGH | Based on Linear/Asana/Trello API analysis, domain-specific research, clear MVP definition with P1/P2/P3 prioritization |
| Architecture | HIGH | Express layered architecture is a well-established pattern; Prisma replaces Knex but the service/model separation holds; project structure recommendations are standard |
| Pitfalls | HIGH | Based on OWASP API Security Top 10, node-postgres documentation, Express.js production best practices, and community post-mortems |

**Overall confidence:** HIGH

### Gaps to Address

- **Status transition rules:** The research describes optional business rules ("can only move to DONE if comments exist") but doesn't define whether these are required for MVP. Needs product decision before Phase 4 planning.
- **Cursor vs. offset pagination:** The research recommends offset pagination for MVP but notes it breaks with concurrent inserts. If large datasets are expected early, cursor-based pagination should be chosen — but this needs explicit decision.
- **Activity log implementation details:** The activity log is identified as P2 but its schema (separate table vs. JSONB column) and event types need definition during Phase 5 planning.

## Sources

### Primary (HIGH confidence)
- [Context7: /prisma/web](https://context7.com/prisma/web) — Prisma migrations, PostgreSQL connection pooling, schema design
- [Context7: /colinhacks/zod](https://context7.com/colinhacks/zod) — Zod schema validation, TypeScript inference
- [Context7: /websites/vitest_dev](https://context7.com/websites/vitest_dev) — Vitest API testing patterns
- [Express.js Official Docs](https://expressjs.com/en/5x/api) — Express v5 routing, middleware, error handling
- [Auth0 node-jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) — `jwt.verify` vs `jwt.decode` distinction
- [PostgreSQL Documentation](https://www.postgresql.org/docs/) — Transaction isolation, row-level locking, indexing
- [Express.js Production Best Practices](https://expressjs.com/en/advanced/best-practice-production.html) — Security, error handling
- [OWASP API Security Top 10](https://owasp.org/API-Security/) — Authorization, authentication, rate limiting

### Secondary (MEDIUM confidence)
- [Linear API documentation](https://linear.app/docs/api) — Modern task API patterns
- [Asana API reference](https://developers.asana.com/docs) — Enterprise collaboration features
- [Trello API](https://developer.atlassian.com/cloud/trello/rest/) — Board-based workflow model
- [Knex.js documentation](https://knexjs.org/) — Query builder patterns, eager loading reference

### Tertiary (LOW confidence)
- Competition feature analysis — based on public API documentation, not internal knowledge

---

*Research completed: 2026-06-02*
*Ready for roadmap: yes*