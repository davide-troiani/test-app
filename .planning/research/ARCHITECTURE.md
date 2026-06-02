# Architecture Research

**Domain:** Task Management REST API
**Researched:** 2026-06-02
**Confidence:** HIGH

## Standard Architecture

### System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      API Layer (Express)                     │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   Routes    │  │ Middleware  │  │   Validators │          │
│  │  /tasks     │  │  auth/jwt   │  │  express-    │          │
│  │  /users     │  │  error      │  │  validator   │          │
│  │  /comments  │  │  logging    │  │              │          │
│  └──────┬──────┘  └────┬───────┘  └──────┬───────┘          │
│         │              │                  │                  │
├─────────┴──────────────┴──────────────────┴──────────────────┤
│                    Controller Layer                           │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │  taskController    userController   commentController │    │
│  │  - createTask()    - createUser()   - addComment()   │    │
│  │  - updateTask()    - getProfile()   - listComments() │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                      Service Layer                            │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │  taskService      userService      commentService    │    │
│  │  - business logic - auth logic     - relations       │    │
│  │  - validation     - profiles       - notifications   │    │
│  └─────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                       Model Layer                             │
├─────────────────────────────────────────────────────────────┤
│  ┌───────────┐  ┌───────────┐  ┌───────────┐                │
│  │   Task    │  │   User    │  │  Comment  │                │
│  │  Model    │  │  Model    │  │   Model   │                │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                │
│        │              │              │                       │
├────────┴──────────────┴──────────────┴───────────────────────┤
│                    PostgreSQL (pg)                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  tasks   │  │  users   │  │ comments │  │  org/    │    │
│  │          │  │          │  │          │  │ workspaces│    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| Routes | URL matching, HTTP method dispatch | Express Router instances |
| Middleware | Cross-cutting concerns (auth, errors, logging) | Function chains |
| Controllers | HTTP request/response handling, input extraction | Classes or functions |
| Services | Business logic, orchestration, domain rules | Classes with injected dependencies |
| Models | Data access, schema definition, queries | ORM (Knex/Prisma) or raw SQL |
| Database | Persistent storage, transactions, constraints | PostgreSQL |

## Recommended Project Structure

```
src/
├── app.js                    # Express app setup, middleware wiring
├── server.js                 # Server entry, graceful shutdown
├── config/
│   └── index.js              # Environment config (DB URL, JWT secret)
├── routes/
│   ├── index.js              # Route aggregator
│   ├── tasks.routes.js       # Task endpoints
│   ├── users.routes.js       # User endpoints
│   ├── comments.routes.js    # Comment endpoints
│   └── auth.routes.js        # Auth endpoints (login/register)
├── controllers/
│   ├── task.controller.js
│   ├── user.controller.js
│   ├── comment.controller.js
│   └── auth.controller.js
├── services/
│   ├── task.service.js
│   ├── user.service.js
│   ├── comment.service.js
│   └── auth.service.js
├── models/
│   ├── index.js              # DB connection pool
│   ├── task.model.js
│   ├── user.model.js
│   └── comment.model.js
├── middleware/
│   ├── auth.middleware.js    # JWT verification
│   ├── error.middleware.js   # Global error handler
│   ├── validation.middleware.js
│   └── logger.middleware.js
├── utils/
│   ├── errors.js             # Custom error classes
│   └── helpers.js
└── db/
    └── migrations/           # Knex migration files
```

### Structure Rationale

- **routes/:** Express Router grouping by resource — keeps URL structure explicit and testable
- **controllers/:** Thin HTTP layer — extracts request params, calls services, returns responses
- **services/:** Business logic lives here — reusable, testable without HTTP overhead
- **models/:** Data access only — no business logic, only queries and schema
- **middleware/:** Reusable cross-cutting concerns — auth, errors, logging are composable
- **config/:** Single source of truth for environment variables — avoids scattered process.env

## Architectural Patterns

### Pattern 1: Layered Architecture

**What:** Strict separation between HTTP handling, business logic, and data access.
**When to use:** Default choice for REST APIs. Provides clarity and testability.
**Trade-offs:** More files to maintain, but each layer has a single responsibility.

**Example:**
```javascript
// Controller — handles HTTP only
async function createTask(req, res, next) {
  try {
    const { title, description } = req.body;
    const task = await taskService.create({ title, description, userId: req.user.id });
    res.status(201).json(task);
  } catch (err) {
    next(err);
  }
}

// Service — business logic only
async function createTask(data) {
  if (!data.title.trim()) {
    throw new ValidationError('Title is required');
  }
  return taskModel.create(data);
}

// Model — data access only
async function create(data) {
  const [task] = await db('tasks').insert(data).returning('*');
  return task;
}
```

### Pattern 2: Repository Pattern (via Models)

**What:** Models act as repositories, encapsulating all database queries behind a clean interface.
**When to use:** When you want services to remain decoupled from SQL/ORM syntax.
**Trade-offs:** Adds indirection; worth it when queries grow complex or you need to swap databases.

**Example:**
```javascript
// task.model.js
class TaskModel {
  async findById(id) {
    return db('tasks').where({ id }).first();
  }

  async findByUserId(userId, options = {}) {
    const { limit = 50, offset = 0 } = options;
    return db('tasks')
      .where({ assigned_to: userId })
      .orderBy('created_at', 'desc')
      .limit(limit)
      .offset(offset);
  }
}
```

### Pattern 3: Middleware Chain

**What:** Express middleware composes into chains — each can short-circuit or pass forward.
**When to use:** Auth, validation, error handling, logging — anything that applies to multiple routes.
**Trade-offs:** Order matters; debugging chain composition can be tricky.

**Example:**
```javascript
// app.js
app.use(helmet());           // Security headers first
app.use(cors());             // Then CORS
app.use(express.json());     // Body parsing
app.use(loggerMiddleware);   // Request logging
app.use('/api', authMiddleware);  // JWT check for /api routes
app.use('/api', routes);     // Routes
app.use(errorMiddleware);    // Global error handler last
```

## Data Flow

### Request Flow (Create Task)

```
POST /api/tasks { title: "Fix bug", description: "..." }
     │
     ▼
┌─────────────────────────────┐
│  Middleware Chain           │
│  1. helmet()                │
│  2. cors()                  │
│  3. express.json()          │
│  4. authMiddleware (JWT)    │
└────────────┬────────────────┘
             │ ✓ Valid JWT → req.user set
             ▼
┌─────────────────────────────┐
│  Router                     │
│  POST /tasks →              │
│    taskController.createTask│
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  Controller                 │
│  - extract req.body         │
│  - call taskService.create()│
│  - return 201 with JSON     │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  Service                    │
│  - validate input           │
│  - enrich with userId       │
│  - call taskModel.create()  │
│  - return clean entity      │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  Model (pg/Knex)            │
│  - INSERT INTO tasks ...    │
│  - PostgreSQL executes      │
│  - Return inserted row      │
└────────────┬────────────────┘
             │
             ▼
      Response: 201 Created
      { id: 1, title: "Fix bug", ... }
```

### Key Data Flows

1. **Create flow:** Request → Middleware (auth) → Controller → Service (validate) → Model (INSERT) → Response
2. **Read flow:** Request → Middleware (auth) → Controller → Service (authorization check) → Model (SELECT) → Response
3. **Update flow:** Request → Middleware (auth) → Controller → Service (verify ownership) → Model (UPDATE) → Response
4. **Delete flow:** Request → Middleware (auth) → Controller → Service (soft-delete or hard-delete) → Model (DELETE) → Response

### Authorization Flow

```
Request with JWT
     │
     ▼
authMiddleware.extractToken()  → parse "Authorization: Bearer <token>"
     │
     ▼
jwt.verify(token, secret)      → returns { userId, email }
     │
     ▼
req.user = { id, email }       → injects into request context
     │
     ▼
Controller/Service checks      → "Does req.user.id match task.owner_id?"
```

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0-1k users | Monolith is fine. Single PostgreSQL instance handles everything. Index on `assigned_to`, `owner_id`. |
| 1k-100k users | Add connection pooling (pg-pool). Add Redis cache for hot queries (task lists, user profiles). Consider read replicas for reporting queries. |
| 100k+ users | Split into services if domain grows. Queue with Bull for heavy operations (bulk assignment, notifications). Consider partitioning `tasks` table by `workspace_id`. |

### Scaling Priorities

1. **First bottleneck — DB queries:** Add indexes on foreign keys (`assigned_to`, `owner_id`, `project_id`). Use EXPLAIN ANALYZE to find slow queries. N+1 query problems appear early — eager-load relations in models.
2. **Second bottleneck — Connection exhaustion:** PostgreSQL has connection limits (~100 default). Use a pool (pg-pool, Prisma connection pool). If hitting ceiling, move to connection pooler like PgBouncer before scaling out.

## Anti-Patterns

### Anti-Pattern 1: Fat Controller

**What people do:** Put business logic directly in controllers — validation, database calls, response formatting all in one function.
**Why it's wrong:** Logic is tied to HTTP; cannot reuse in CLI, background jobs, or tests without mocking HTTP. Becomes unmaintainable as logic grows.
**Do this instead:** Keep controllers thin — extract to services. Controllers only handle HTTP serialization/deserialization.

### Anti-Pattern 2: Leaky Abstraction

**What people do:** Calling `db.query()` or raw SQL scattered across controllers and services.
**Why it's wrong:** Changing database (PostgreSQL → MySQL) requires editing every file. No isolation for testing — must use real DB.
**Do this instead:** Hide all DB access behind models. Services and controllers never import `db` directly — only models.

### Anti-Pattern 3: Synchronous Startup

**What people do:** Connecting to DB synchronously at module load time, or making startup async without graceful shutdown.
**Why it's wrong:** Server hangs if DB is slow to connect. No cleanup on SIGTERM → stuck connections, broken deployments.
**Do this instead:** Async server init with explicit `await db.connect()`, register `process.on('SIGTERM', gracefulShutdown)`.

### Anti-Pattern 4: Mixed Responsibility Models

**What people do:** Models that do validation AND data access AND business logic.
**Why it's wrong:** Violates single responsibility; creates tight coupling between schema and domain rules.
**Do this instead:** Models = data access only. Services = business logic only.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| PostgreSQL | pg library with connection pool | Connection string in config, TLS for production |
| JWT | jsonwebtoken library | Verify in middleware, payload contains userId |
| Redis (future) | ioredis | Use for session cache, rate limiting, hot data cache |
| Email (future) | nodemailer / SendGrid | Queue emails via Bull, don't send synchronously |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| Controller ↔ Service | Direct function call | Service injected or imported — no HTTP involved |
| Service ↔ Model | Direct function call | Models return raw entities, services shape domain objects |
| Middleware ↔ Controller | Express request object | Middleware sets `req.user`, `req.context`; controller reads |
| Route ↔ Controller | Express routing | Routes map URL + method → controller method |

## Sources

- Express.js Documentation — https://expressjs.com/
- PostgreSQL Documentation — https://www.postgresql.org/docs/
- Node.js Production Best Practices — https://expressjs.com/en/advanced/best-practice-production.html
- REST API Design — https://restfulapi.net/
- Knex.js (query builder pattern reference) — https://knexjs.org/

---

*Architecture research for: Task Management REST API*
*Researched: 2026-06-02*