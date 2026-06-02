# Phase 1: Project Foundation - Context

**Gathered:** 2026-06-02
**Status:** Ready for planning

<domain>
## Phase Boundary

Express server with Prisma migrations, security middleware, validation, error handling, and environment configuration. This foundation must be solid before any API endpoints are written.

</domain>

<decisions>
## Implementation Decisions

### Project Structure
- **D-01:** Standard Express layered architecture — routes/controllers/services/models/middleware
- **D-02:** Resource-based file organization — each domain (tasks, users, comments) has its own files
- **D-03:** TypeScript enabled (based on Prisma + Zod stack)

### Error Handling
- **D-04:** Consistent JSON error response: `{ error: { code: string, message: string, details?: any } }`
- **D-05:** Custom error class hierarchy: AppError (base), NotFoundError, ValidationError, ForbiddenError, ConflictError
- **D-06:** Global error handler middleware with proper status codes
- **D-07:** Async errors handled via Express v5 native async support (no express-async-errors needed)

### Validation Middleware
- **D-08:** Zod for schema validation on all incoming requests
- **D-09:** Validation middleware per route — schemas co-located with routes
- **D-10:** Consistent error format for validation failures (400 Bad Request)

### Database Migrations
- **D-11:** Prisma migrations for all schema changes
- **D-12:** Initial schema: users, tasks, comments tables with proper indexes
- **D-13:** Indexes on foreign keys: owner_id, assigned_to, task_id, status
- **D-14:** Soft delete support via deleted_at timestamp

### Configuration & Security
- **D-15:** Environment variables validated at startup (JWT_SECRET >= 64 chars)
- **D-16:** Helmet middleware for security headers
- **D-17:** CORS configured appropriately
- **D-18:** Vitest + supertest for testing setup

### the agent's Discretion
- Exact file naming conventions (consistent with Express idioms)
- Specific error message text
- Default page size configuration (20-50 range)
- Migration naming conventions

</decisions>

<specifics>
## Specific Ideas

(No specific user references for Phase 1 — foundation phase, standard patterns)

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

No external specs — requirements are fully captured in decisions above.

From ROADMAP.md:
- `.planning/ROADMAP.md` §Phase 1 — Success criteria and requirements

From Research:
- `.planning/research/SUMMARY.md` §Phase 1 — Pitfall mappings (indexes, error format, validation, JWT secret, soft delete)
- `.planning/research/STACK.md` — Tech versions and recommendations
- `.planning/research/ARCHITECTURE.md` — Project structure and layered architecture patterns

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
(None yet — this is the first phase)

### Established Patterns
- Layered architecture: routes → controllers → services → models → database
- Middleware chain: auth → errors → logging → validation

### Integration Points
- Future phases will import from this foundation
- Auth middleware (Phase 2) will extend this middleware structure
- Services (Phase 3+) will use models from this phase

</code_context>

<deferred>
## Deferred Ideas

(None — discussion stayed within phase scope)

</deferred>

---

*Phase: 01-project-foundation*
*Context gathered: 2026-06-02*
*Auto-mode: All decisions defaulted to recommended options*