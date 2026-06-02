# Pitfalls Research

**Domain:** Task Management REST API (Node.js + PostgreSQL)
**Researched:** 2026-06-02
**Confidence:** HIGH

---

## Critical Pitfalls

### Pitfall 1: JWT Verification Bypass via `jwt.decode()` Without Verification

**What goes wrong:**
Authentication is bypassed because the codebase uses `jwt.decode()` to extract user context instead of `jwt.verify()`. An attacker can craft a forged JWT with any user ID — the signature is never validated. Every protected endpoint becomes accessible to anyone who can set an HTTP header.

**Why it happens:**
`jwt.decode()` extracts the payload without cryptographic verification. Developers often use `decode()` during development for debugging ("let me just see what's in the token") and accidentally leave it in production. The `decode()` call succeeds even with invalid signatures, making it indistinguishable from `verify()` at runtime unless the return value is checked.

**How to avoid:**
Enforce a single auth middleware that always calls `jwt.verify()`:
```javascript
// middleware/auth.middleware.js — ONE source of truth
const jwt = require('jsonwebtoken');

function authMiddleware(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Missing or invalid Authorization header' });
  }

  const token = authHeader.slice(7);
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET); // ← verify, NOT decode
    req.user = { id: payload.userId, email: payload.email };
    next();
  } catch (err) {
    return res.status(401).json({ error: 'Invalid or expired token' });
  }
}
```
Audit all JWT usage: `rg 'jwt\.decode'` must return zero results in production code. Add a linter rule to flag `jwt.decode`.

**Warning signs:**
- `jwt.decode()` found anywhere in the codebase
- Auth middleware tests only pass with valid tokens but don't verify signature validation
- `jwt.verify` not imported in auth-related files

**Phase to address:**
Phase 2: Authentication — auth middleware must be built with signature verification, not decoding.

---

### Pitfall 2: Missing Authorization Checks on Task Operations (Horizontal Privilege Escalation)

**What goes wrong:**
Users can read, update, or delete tasks belonging to other users. Authenticated users can alter any task in the system because the API checks "is the user logged in?" but not "is the user allowed to access this specific task?"

**Why it happens:**
Authorization is conflated with authentication. The JWT middleware proves identity but doesn't enforce resource-level rules. Controllers call service methods without checking ownership or assignment status. "Authentication middleware exists" is mistakenly treated as "authorization is complete."

**How to avoid:**
Implement authorization checks at the service layer — never trust the controller to enforce it:

```javascript
// task.service.js
async function getTask(taskId, userId) {
  const task = await taskModel.findById(taskId);
  if (!task) throw new NotFoundError('Task not found');

  // Authorization: owner OR assignee can read
  if (task.owner_id !== userId && task.assigned_to !== userId) {
    throw new ForbiddenError('You do not have access to this task');
  }
  return task;
}

async function deleteTask(taskId, userId) {
  const task = await taskModel.findById(taskId);
  if (!task) throw new NotFoundError('Task not found');

  // Authorization: only owner can delete
  if (task.owner_id !== userId) {
    throw new ForbiddenError('Only the task owner can delete this task');
  }
  return taskModel.delete(taskId);
}
```

Define a clear ownership model: task `owner_id` can do everything; `assigned_to` can read and update status; viewers have no API access (pure API, no sharing).

**Warning signs:**
- `taskModel.findById()` called without ownership check
- `req.user.id` used only for setting `created_by`, not for authorization
- Tests pass for "user can delete their own task" but don't test "user cannot delete others' tasks"

**Phase to address:**
Phase 2: Authentication (auth middleware) and Phase 3: Task CRUD (authorization in services).

---

### Pitfall 3: Race Condition on Concurrent Status Updates

**What goes wrong:**
Two concurrent requests try to update the same task's status simultaneously — Request A sets `TODO → IN PROGRESS`, Request B sets `TODO → DONE`. Due to concurrent reads, both see the task as `TODO` and both succeed. The last write wins, but both transitions execute. If business logic expects atomic transition validation (e.g., "can only move to DONE if comments exist"), the race bypasses it entirely.

**Why it happens:**
PostgreSQL's default isolation level is `READ COMMITTED`, which allows non-repeatable reads. A transaction reads the row, another transaction updates it, and the first transaction's subsequent write sees the old value (or worse — both write without seeing each other). No row-level lock is acquired until `UPDATE` executes, and even then, the first transaction's validation happened before the lock was held.

**How to avoid:**
Use optimistic locking with a version column, or pessimistic locking with `SELECT ... FOR UPDATE`:

```javascript
// Option A: Optimistic locking (recommended for most cases)
async function updateTaskStatus(taskId, userId, newStatus, expectedVersion) {
  const result = await db('tasks')
    .where({ id: taskId, version: expectedVersion })
    .update({ status: newStatus, version: expectedVersion + 1, updated_at: new Date() });

  if (result === 0) {
    throw new ConflictError('Task was modified by another request. Please refresh and retry.');
  }
  return findById(taskId);
}

// Option B: Pessimistic locking (for critical transitions)
const client = await pool.connect();
try {
  await client.query('BEGIN');
  await client.query('SELECT * FROM tasks WHERE id = $1 FOR UPDATE', [taskId]);
  // Now row is locked — validate transition, then update
  await client.query('UPDATE tasks SET status = $1 WHERE id = $2', [newStatus, taskId]);
  await client.query('COMMIT');
} catch (e) {
  await client.query('ROLLBACK');
  throw e;
} finally {
  client.release();
}
```

For MVP scope: use optimistic locking. Add a `version` INTEGER column to the tasks table with a default of 1 and increment it on every update.

**Warning signs:**
- No `version` or `updated_at` column on tasks table
- Status update tests pass in isolation but fail with parallel requests
- Business rules for status transitions ("can't go DONE without comments") exist but aren't transactional

**Phase to address:**
Phase 4: Task Status Workflow — define the locking strategy before implementing status transitions.

---

### Pitfall 4: N+1 Queries on Task List with Comments/Assignees

**What goes wrong:**
Listing tasks loads each task individually, then for each task, a separate query fetches the assignee's user record and all comments. A list of 50 tasks generates 1 + 50 + 50 = 101 queries. Response times degrade to 2-5 seconds as the task count grows.

**Why it happens:**
Eager loading is not implemented. Models return raw database rows with foreign key IDs (`assigned_to: 42`) but services don't join or preload related data. Each related entity requires a separate round-trip to the database.

**How to avoid:**
Use eager loading in queries — Knex's `withGraphFetched` or raw JOINs:

```javascript
// task.model.js — eager load relations
async findWithRelations(taskIds) {
  return db('tasks')
    .leftJoin('users as owner', 'tasks.owner_id', 'owner.id')
    .leftJoin('users as assignee', 'tasks.assigned_to', 'assignee.id')
    .leftJoin('comments', 'tasks.id', 'comments.task_id')
    .whereIn('tasks.id', taskIds)
    .select(
      'tasks.*',
      'owner.id as owner_id', 'owner.email as owner_email',
      'assignee.id as assignee_id', 'assignee.email as assignee_email',
    )
    // Group by task, aggregate comments
    .groupBy('tasks.id', 'owner.id', 'assignee.id');
}

// Or with Knex Graph (if using objection.js)
async findAll(options) {
  return Task.query()
    .withGraphFetched('[owner, assignee, comments]')
    .orderBy('created_at', 'desc')
    .limit(options.limit)
    .offset(options.offset);
}
```

Create a test that logs the number of queries per endpoint. Set a threshold (e.g., max 5 queries per endpoint) and fail the test if exceeded.

**Warning signs:**
- Response time increases proportionally with task count (linear degradation)
- `findById` calls `findCommentsByTaskId` internally for each returned task
- No JOINs in model query methods — only simple `where` clauses

**Phase to address:**
Phase 3: Task CRUD — implement eager loading in models from the start, not as a later optimization.

---

### Pitfall 5: Missing Pagination on List Endpoints

**What goes wrong:**
`GET /tasks` returns all tasks. With 10,000 tasks in the database, the response is dozens of megabytes of JSON. Clients timeout, browsers hang, and the PostgreSQL connection is held open for many seconds. The API is effectively unusable.

**Why it happens:**
Pagination is treated as a "nice to have" optimization rather than a required constraint. Early implementations return `SELECT * FROM tasks` without LIMIT/OFFSET. Data grows unbounded as users create more tasks.

**How to avoid:**
Enforce pagination on every list endpoint from day one:

```javascript
// Constant defined once — shared across all list endpoints
const DEFAULT_PAGE_SIZE = 20;
const MAX_PAGE_SIZE = 100;

// In task.controller.js
async function listTasks(req, res, next) {
  const page = Math.max(1, parseInt(req.query.page) || 1);
  const limit = Math.min(MAX_PAGE_SIZE, parseInt(req.query.limit) || DEFAULT_PAGE_SIZE);
  const offset = (page - 1) * limit;

  const { tasks, total } = await taskService.list({ limit, offset });

  res.json({
    data: tasks,
    pagination: {
      page,
      limit,
      total,
      totalPages: Math.ceil(total / limit),
    },
  });
}
```

Consider cursor-based pagination for large datasets — offset pagination breaks with concurrent inserts (page jumps). For MVP: offset pagination is acceptable but should be documented as a limitation.

**Warning signs:**
- `SELECT * FROM tasks` without `.limit()` in any model method
- No `count` query alongside data queries
- API response structure has no `pagination` key

**Phase to address:**
Phase 3: Task CRUD — pagination is table stakes for any list endpoint. Do not ship a list endpoint without it.

---

### Pitfall 6: Missing Indexes on Foreign Keys

**What goes wrong:**
Every query filtering by `assigned_to`, `owner_id`, or `task_id` (for comments) does a sequential scan on a growing table. A `tasks` table with 100k rows and an unindexed `assigned_to` column takes 500ms-2s per query. The API becomes slow under load despite sufficient compute.

**Why it happens:**
Developers create the schema with foreign key columns but forget to add indexes. PostgreSQL enforces referential integrity with constraints but doesn't auto-create indexes. Foreign key lookups in WHERE clauses (`WHERE assigned_to = $1`) are common and must be indexed.

**How to avoid:**
Add indexes in migrations alongside the foreign key definition:

```javascript
// db/migrations/001_create_tables.js
exports.up = async function(knex) {
  await knex.schema.createTable('tasks', (table) => {
    table.uuid('id').primary();
    table.string('title').notNullable();
    table.string('status').defaultTo('TODO');
    table.uuid('owner_id').notNullable().references('id').inTable('users');
    table.uuid('assigned_to').references('id').inTable('users'); // nullable for unassigned
    table.integer('version').defaultTo(1);
    table.timestamps(true, true);
  });

  // Indexes — separate from constraint definition
  await knex.schema.alterTable('tasks', (table) => {
    table.index('owner_id');
    table.index('assigned_to');
    table.index('status');
    table.index(['owner_id', 'status']); // composite for common filter combo
  });

  await knex.schema.createTable('comments', (table) => {
    table.uuid('id').primary();
    table.uuid('task_id').notNullable().references('id').inTable('tasks');
    table.uuid('user_id').notNullable().references('id').inTable('users');
    table.text('content').notNullable();
    table.timestamps(true, true);
  });

  await knex.schema.alterTable('comments', (table) => {
    table.index('task_id'); // foreign key — always index
    table.index(['task_id', 'created_at']); // comments on task, ordered by time
  });
};
```

Add these migrations immediately. Check existing schema with `SELECT * FROM pg_indexes WHERE tablename = 'tasks'`.

**Warning signs:**
- `EXPLAIN ANALYZE` shows `Seq Scan` on tasks/comments tables for filtered queries
- Response time degrades as data grows (not just at extreme scale)
- No index definition in migration files

**Phase to address:**
Phase 1: Project Setup — migrations must include indexes. Verify with `EXPLAIN ANALYZE` before considering any query "optimized."

---

### Pitfall 7: Inconsistent Error Response Format

**What goes wrong:**
Some errors return `{ "error": "Not found" }`, others return `{ "message": "Task not found" }`, others return `{ "errors": ["Not found"] }`, and HTTP status codes vary — a 404 for "task not found," a 400 for "user not found," a 200 with error body for validation failures. Clients can't reliably handle errors across different endpoints.

**Why it happens:**
Error handling is implemented ad-hoc in each controller. Each developer chooses a format based on immediate needs. No shared error schema or global error handler is established early.

**How to avoid:**
Define a single error response schema and enforce it globally:

```javascript
// utils/errors.js — unified error class hierarchy
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code; // machine-readable: TASK_NOT_FOUND, VALIDATION_FAILED
  }
}

class NotFoundError extends AppError {
  constructor(resource = 'Resource') {
    super(`${resource} not found`, 404, `${resource.toUpperCase()}_NOT_FOUND`);
  }
}

class ValidationError extends AppError {
  constructor(message, errors = []) {
    super(message, 400, 'VALIDATION_FAILED');
    this.errors = errors; // field-level validation details
  }
}

class ForbiddenError extends AppError {
  constructor() {
    super('You do not have permission to perform this action', 403, 'FORBIDDEN');
  }
}

class ConflictError extends AppError {
  constructor(message = 'Conflict') {
    super(message, 409, 'CONFLICT');
  }
}
```

```javascript
// middleware/error.middleware.js — global handler
function errorHandler(err, req, res, next) {
  if (err instanceof AppError) {
    const response = {
      error: {
        code: err.code,
        message: err.message,
      },
    };
    if (err.errors) response.error.details = err.errors;
    return res.status(err.statusCode).json(response);
  }

  // Unexpected errors — don't leak internals
  console.error('Unhandled error:', err);
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
    },
  });
}
```

Standardize on: `{ error: { code: "...", message: "...", details?: [...] } }` for all responses.

**Warning signs:**
- Different error shapes across endpoints (use Postman/curl to test every endpoint with invalid input)
- Some errors include stack traces or database error messages in responses
- No shared error utility — each controller has its own error handling

**Phase to address:**
Phase 1: Project Setup — define error classes and global handler before any endpoint is implemented. Retrofitting is painful.

---

### Pitfall 8: Soft Delete Not Implemented, Hard Delete Removes Evidence

**What goes wrong:**
Users delete tasks and comments, which removes them from the database entirely. Assigned tasks disappear from assignee's views without explanation. If another user has the task ID bookmarked, they get a 404. No audit trail exists for what was deleted. In collaborative environments, permanent deletion breaks referential integrity and destroys historical context.

**Why it happens:**
"Delete" is implemented as `DELETE FROM tasks WHERE id = $1`. The MVP scope doesn't explicitly address this, and hard delete is simpler. The downstream cost (broken references, no recovery, no audit) isn't visible until users start relying on the system.

**How to avoid:**
Implement soft delete with a `deleted_at` timestamp:

```javascript
// tasks table migration
table.timestamp('deleted_at').nullable(); // null = active, timestamp = deleted

// task.model.js
async softDelete(taskId) {
  return db('tasks')
    .where({ id: taskId })
    .update({ deleted_at: new Date() }); // NOT .del()
}

async findById(id) {
  // Always filter out soft-deleted
  return db('tasks')
    .where({ id })
    .whereNull('deleted_at')
    .first();
}
```

For MVP: soft delete is low cost (add column, change two methods). Implement it early — converting hard delete to soft delete in production requires a data migration.

**Warning signs:**
- `.del()` or `DELETE FROM` in model methods
- No `deleted_at` column in tasks or comments tables
- Deleted tasks cause 404 responses rather than being filtered out

**Phase to address:**
Phase 3: Task CRUD — implement soft delete from the start. Add a separate `permanentDelete` method for administrative cleanup only.

---

### Pitfall 9: JWT Secret Stored as Plain String in Code or Weak Secret

**What goes wrong:**
The JWT signing secret is hardcoded as a string (`const secret = 'my-super-secret'`), committed to git, and deployed. Anyone with access to the code can forge any user's token. Even if it's an environment variable, if the secret is too short or predictable, brute-force attacks can compromise all tokens.

**Why it happens:**
During development, convenience wins: hardcoded short strings are easy to test. The security implications aren't apparent until production. Weak random generation (e.g., `Math.random()` or short strings) makes the secret guessable.

**How to avoid:**
- Generate secrets with `crypto.randomBytes(64).toString('hex')` — store in `.env`, never in code
- Validate JWT secret presence at startup: throw an error if `process.env.JWT_SECRET` is undefined in production
- Enforce minimum length (64 characters for HS256)
- Rotate secrets on incident response — invalidate all existing tokens

```javascript
// config/index.js
const secret = process.env.JWT_SECRET;
if (!secret) throw new Error('JWT_SECRET environment variable is required');
if (secret.length < 64) throw new Error('JWT_SECRET must be at least 64 characters');
module.exports = { jwtSecret: secret };
```

**Warning signs:**
- `JWT_SECRET` not in `.env.example` (developers don't know it exists)
- Secret in code (`const secret = '...'`) instead of env var
- Secret logged in error messages or stack traces
- No startup validation

**Phase to address:**
Phase 1: Project Setup — config validation must fail fast if secrets are missing or weak.

---

### Pitfall 10: Missing Request Validation — Garbage In, Unexpected Errors Out

**What goes wrong:**
Requests with missing required fields (`POST /tasks` without `title`) cause database errors (NOT NULL constraint violation) that bubble up as 500 Internal Server Error. Invalid `status` values (`"INPROGRESS"` instead of `"IN PROGRESS"`) cause type errors. The API responds with raw database error messages that leak schema details to clients.

**Why it happens:**
Input validation is skipped for speed. Controllers pass `req.body` directly to model methods without checking field presence, types, or allowed values. The database rejects invalid data, but the error handling is minimal.

**How to avoid:**
Validate all input at the controller or middleware layer before any database call:

```javascript
// middleware/validation.middleware.js
const Joi = require('joi');

const createTaskSchema = Joi.object({
  title: Joi.string().min(1).max(255).required(),
  description: Joi.string().max(5000).optional(),
  status: Joi.string().valid('TODO', 'IN PROGRESS', 'DONE').optional().default('TODO'),
  assigned_to: Joi.uuid().optional().allow(null),
});

function validate(schema) {
  return (req, res, next) => {
    const { error, value } = schema.validate(req.body, { abortEarly: false });
    if (error) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_FAILED',
          message: 'Invalid request data',
          details: error.details.map(d => ({ field: d.path.join('.'), message: d.message })),
        },
      });
    }
    req.validatedBody = value;
    next();
  };
}

// In routes
router.post('/tasks', authMiddleware, validate(createTaskSchema), taskController.create);
```

Never trust `req.body` — always validate. Use `express-validator` or `Joi` (or `zod`). Return 400 with field-level details, never 500 with a raw database error.

**Warning signs:**
- Controller catches `error` and returns `500` with `err.message` — leaks DB internals
- No validation middleware on any route
- Tests use invalid data and expect 500 instead of 400

**Phase to address:**
Phase 1: Project Setup — validation middleware is a prerequisite. Phase 2: Authentication (input validation on signup/login).

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| Skip indexes during initial build — "add them later" | Faster initial migration | Queries degrade silently as data grows; adding indexes on large tables locks them for minutes | Never — index creation is cheap upfront, expensive mid-production |
| Hardcode JWT secret for testing | No env setup needed | Production leaks secret if test code is committed | Only in isolated unit tests, never in integration or e2e |
| Return raw database rows as API response | Less code, faster | API shape couples to schema; any schema change breaks clients | Never in production — always transform in service layer |
| `SELECT *` everywhere | Convenient, works fast | Returns too much data, includes sensitive fields (password hashes), no control over shape | Only for admin/internal endpoints with explicit field listing |
| Skip `deleted_at` — implement hard delete | Simpler, fewer queries | No recovery, breaks references, no audit trail | MVP only if "permanent deletion is a feature" is explicitly required |
| Ignore async error handling in Express | Fewer try/catch blocks | Unhandled promise rejections crash the server | Never — Express requires explicit async error propagation |

---

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| PostgreSQL connection | Connecting in every request — connection overhead | Use a connection pool (`pg.Pool`) initialized once at startup, configure `max` connections (default 10, increase based on load) |
| PostgreSQL SSL | Disabling SSL in production for "simplicity" | Always use SSL in production (`ssl: { rejectUnauthorized: true }`); use `pg.Pool` TLS config |
| JWT verification | Catching all errors and returning 401 without distinguishing token-expired from token-invalid | Catch `TokenExpiredError` separately — log it, return 401 with `code: 'TOKEN_EXPIRED'` for client-side refresh handling |
| Environment config | Using `process.env` scattered across files | Single `config/index.js` that loads all env vars and validates types at startup |
| Process signals | Ignoring SIGTERM — connections don't close gracefully on deploy | Register `process.on('SIGTERM', gracefulShutdown)` that closes DB pool before exiting |

---

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| N+1 queries on list endpoints | Response time = O(n) where n = task count | Always use JOINs or eager loading; add query count test | Breaks at ~100 tasks per list request |
| No pagination on list endpoints | Response size grows unbounded; timeouts | Enforce LIMIT/OFFSET (or cursor) on every list endpoint | Breaks at ~1,000 tasks in database |
| Missing foreign key indexes | Slow filtered queries, high CPU on DB | Add indexes on `owner_id`, `assigned_to`, `task_id` in migrations | Breaks at ~10,000 rows per table |
| Synchronous DB connection at startup | Server hangs if DB is slow or unreachable | Async initialization with timeout; health check endpoint | Breaks immediately on cold start |
| No connection pool sizing | Exhausts DB connections under concurrent load | Configure `pg.Pool` max based on `max_connections` from PostgreSQL config | Breaks at ~50-100 concurrent users |
| Missing `EXPLAIN ANALYZE` review | Performance problems discovered in production | Review query plans for every new model method before merging | Breaks when data volume exceeds dev/test data |

---

## Security Mistakes

Domain-specific security issues beyond general web security.

| Mistake | Risk | Prevention |
|---------|------|------------|
| Returning `password_hash` in API responses | Password hashes exposed to clients — enables offline cracking if DB is leaked | Explicitly exclude `password_hash` in all SELECT statements; never `SELECT *` |
| Using `Math.random()` for token generation | Predictable tokens — session hijacking | Use `crypto.randomBytes()` for all token generation |
| No rate limiting on auth endpoints | Brute force attacks on login/signup | Implement rate limiting with Redis (or in-memory for MVP) on `/auth/*` endpoints — 5 attempts per minute |
| Storing JWT secret in version control | Anyone with repo access can forge tokens | Secret only in environment variables; `.env.example` documents the variable name without the value |
| Missing CORS configuration | Unexpected cross-origin requests succeed | Explicitly set `cors({ origin: allowedOrigins })` — never `cors()` with defaults in production |
| Not validating UUID format | Invalid UUIDs cause database errors and potential SQL injection (if using string concatenation) | Validate UUID format with regex or Joi before DB query |
| Missing security headers | XSS, clickjacking, MIME-type sniffing attacks | Use `helmet()` middleware — sets CSP, X-Frame-Options, X-Content-Type-Options headers |

---

## UX Pitfalls

Common user experience mistakes in this domain (for API consumers).

| Pitfall | User Impact | Better Approach |
|---------|-------------|-----------------|
| Vague error messages ("An error occurred") | Client can't determine what went wrong or how to fix it | Return machine-readable error codes + human-readable messages: `{ code: 'TASK_TITLE_REQUIRED', message: 'Title is required' }` |
| Inconsistent field names in responses | Client must handle multiple response shapes for same entity type | Standardize: use `created_at` (snake_case), never mix `createdAt` and `created_at` in the same entity |
| No filtering on task list | Clients must fetch all tasks and filter client-side | Support `?status=TODO&assigned_to=uuid` query params from day one |
| 404 for deleted resources vs 410 Gone | Clients can't distinguish "never existed" from "was deleted" | Use 404 for non-existent; 410 for soft-deleted (optional — implement if clients need it) |
| No idempotency key on create endpoints | Retried POST requests create duplicate tasks | Accept `Idempotency-Key` header; return same response for duplicate keys (optional for MVP) |
| Missing `Content-Type: application/json` handling | API works but browser/dev tools show confusing behavior | Always return `application/json`; reject non-JSON requests with 415 Unsupported Media Type |

---

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Auth endpoints:** JWT is verified (NOT decoded) — search for `jwt.decode` and ensure zero results
- [ ] **Authorization:** Task ownership is checked on every operation — test accessing another user's task returns 403, not the data
- [ ] **Status updates:** Concurrent updates use optimistic locking — test two simultaneous status changes, one should fail with 409
- [ ] **List endpoints:** Pagination is enforced — test with `limit=1000` returns at most `MAX_PAGE_SIZE` items
- [ ] **Database indexes:** `EXPLAIN ANALYZE` on `WHERE assigned_to = $1` shows `Index Scan`, not `Seq Scan`
- [ ] **Error responses:** All errors return consistent `{ error: { code, message } }` shape — test each endpoint with invalid input
- [ ] **Input validation:** Invalid requests return 400, not 500 — test `POST /tasks` with `{}` and `POST /tasks` with `{ title: "" }`
- [ ] **Password handling:** `password_hash` column is excluded from all SELECT statements — never returned in API responses
- [ ] **Soft delete:** Deleted tasks are filtered out of queries — test: delete a task, then `GET /tasks/:id` returns 404
- [ ] **JWT secret:** Config throws on startup if secret is missing or < 64 chars — test by unsetting `JWT_SECRET` env var
- [ ] **Connection pool:** DB pool is initialized once, released on shutdown — test graceful shutdown, no leaked connections
- [ ] **Security headers:** `helmet()` is applied — test response headers include `Content-Security-Policy`, `X-Frame-Options`

---

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| JWT verification bypass in production | HIGH | Immediate: audit all auth middleware calls — replace `decode` with `verify`. Force logout all users (invalidate all tokens via secret rotation). Review access logs for anomalous requests. |
| Missing foreign key indexes | MEDIUM | Run `CREATE INDEX CONCURRENTLY` on live tables (non-blocking). Monitor with `EXPLAIN ANALYZE` before/after. Add to migration pipeline so future deploys include it. |
| Hard delete without audit trail | HIGH | Soft delete cannot be retrofitted without historical data — it's lost. Add `deleted_at` now for future deletes. Accept that past deletions cannot be recovered. |
| No pagination deployed | LOW | Add LIMIT/OFFSET to model queries — minimal code change. Add `page`/`limit` query params. Old clients without pagination params get first page; test with large datasets to verify. |
| Inconsistent error format in production | MEDIUM | Introduce error middleware incrementally — start with new endpoints, then migrate existing ones. Don't break old error shapes until all clients are updated. |
| Race condition on status updates | MEDIUM | Add `version` column with `WHERE version = $expected` optimistic lock. Existing rows need migration to set `version = 1`. Test concurrent updates with load testing tool (k6, Artillery). |

---

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| JWT verification bypass | Phase 2: Authentication | Audit `rg 'jwt\.decode'` returns zero; test with forged token returns 401 |
| Missing authorization checks | Phase 2 (auth) + Phase 3 (CRUD) | Test: user A cannot GET/PUT/DELETE user B's tasks — returns 403 |
| Race condition on status updates | Phase 4: Task Status Workflow | Concurrent status update test: one returns 409 Conflict |
| N+1 queries | Phase 3: Task CRUD | Query count test: list tasks generates ≤ 3 queries regardless of count |
| Missing pagination | Phase 3: Task CRUD | Test: `GET /tasks` without params returns max 20 items; includes pagination metadata |
| Missing foreign key indexes | Phase 1: Project Setup | `EXPLAIN ANALYZE` on foreign key queries uses Index Scan |
| Inconsistent error format | Phase 1: Project Setup | All endpoints return `{ error: { code, message } }` — no variation |
| Soft delete missing | Phase 3: Task CRUD | Deleted tasks return 404; `deleted_at` column exists and is populated |
| JWT secret in code | Phase 1: Project Setup | Startup validation fails if secret is hardcoded or too short |
| Missing request validation | Phase 1: Project Setup | Invalid input on every endpoint returns 400, never 500 |
| Password hash in responses | Phase 2: Authentication | `GET /users/me` response has no `password_hash` or `password` field |
| Missing security headers | Phase 1: Project Setup | Response headers include CSP, X-Frame-Options, X-Content-Type-Options |

---

## Sources

- Auth0 `node-jsonwebtoken` documentation — jwt.verify vs jwt.decode — https://github.com/auth0/node-jsonwebtoken
- PostgreSQL documentation — transaction isolation, row-level locking — https://www.postgresql.org/docs/current/sql-set-transaction.html
- node-postgres documentation — connection pooling, error handling — https://node-postgres.com/
- Knex.js documentation — transactions, eager loading — https://knexjs.org/
- Express.js production best practices — error handling, security — https://expressjs.com/en/advanced/best-practice-production.html
- OWASP API Security Top 10 — authorization, authentication, rate limiting — https://owasp.org/API-Security/
- Common REST API pitfalls — post-mortems from API development communities

---

*Pitfalls research for: Task Management REST API (Node.js + PostgreSQL)*
*Researched: 2026-06-02*