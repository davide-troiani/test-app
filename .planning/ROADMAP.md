# Roadmap: Task Management API

## Overview

A RESTful task management API built with Node.js and PostgreSQL. The journey spans from project foundation through authentication, then core task operations with assignment, finishing with status workflow and collaboration features.

## Phases

**Project Mode:** MVP (each phase delivers an end-to-end user capability)

- [ ] **Phase 1: Project Foundation** - Express server, Prisma migrations, security middleware, error handling
- [ ] **Phase 2: Authentication** - JWT-based user registration, login, and auth middleware
- [ ] **Phase 3: Task CRUD + Assignment** - Full task lifecycle with user assignment
- [ ] **Phase 4: Status Workflow + Comments** - Status transitions, concurrency handling, and task comments

## Phase Details

### Phase 1: Project Foundation
**Goal**: Express server with Prisma migrations, security middleware, and error handling foundation
**Mode**: mvp
**Depends on**: Nothing (first phase)
**Requirements**: SETUP-01
**Success Criteria** (what must be TRUE):
  1. Express server starts and handles graceful shutdown
  2. PostgreSQL database connects via Prisma with working migrations
  3. Security headers (helmet) and CORS are configured
  4. Zod validates incoming requests with consistent error responses
  5. Environment variables are validated at startup (including JWT_SECRET ≥ 64 chars)
**Plans**: 5 plans
  - `01-01-PLAN.md` — Package legitimacy gate and TypeScript/Vitest scaffold
  - `01-02-PLAN.md` — Complete Wave 0 validation contracts for all Phase 1 test files
  - `01-03-PLAN.md` — Prisma schema, blocking migration application, and health HTTP DB read/write slice
  - `01-04-PLAN.md` — Startup config validation, server PORT wiring, Helmet, CORS allowlist, and safe request logging
  - `01-05-PLAN.md` — Canonical error hierarchy, global error handler wiring, Zod validation, and full verification

### Phase 2: Authentication
**Goal**: Users can register, login, and access protected endpoints via JWT
**Mode**: mvp
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05
**Success Criteria** (what must be TRUE):
  1. User can sign up with email and password (password hashed with bcrypt)
  2. User receives JWT token after successful login
  3. Authenticated requests persist user identity via valid JWT token
  4. User can log out (client-side token removal)
  5. All non-auth API endpoints reject requests without valid JWT
**Plans**: TBD

### Phase 3: Task CRUD + Assignment
**Goal**: Users can create, read, update, delete tasks and assign them to other users
**Mode**: mvp
**Depends on**: Phase 2
**Requirements**: TASK-01, TASK-02, TASK-03, TASK-04, TASK-05, TASK-08, ASGN-01, ASGN-02, ASGN-03, ASGN-04
**Success Criteria** (what must be TRUE):
  1. User can create a task with title and description
  2. User can read a single task by ID
  3. User can list all tasks with pagination
  4. User can update their own task's title and description
  5. User can delete their own task
  6. User can assign a task to another user
  7. User can unassign a task
  8. User can reassign a task to a different user
  9. User can view tasks assigned to them
**Plans**: TBD

### Phase 4: Status Workflow + Comments
**Goal**: Users can transition task status and add comments to tasks
**Mode**: mvp
**Depends on**: Phase 3
**Requirements**: TASK-06, TASK-07, CMNT-01, CMNT-02, CMNT-03, CMNT-04
**Success Criteria** (what must be TRUE):
  1. User can update task status (TODO → IN PROGRESS → DONE)
  2. Concurrent status updates are handled atomically (version column, 409 on conflict)
  3. User can add a comment to a task
  4. User can view all comments on a task
  5. User can delete their own comments
  6. Comments are ordered by creation time consistently
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Project Foundation | 0/? | Not started | - |
| 2. Authentication | 0/? | Not started | - |
| 3. Task CRUD + Assignment | 0/? | Not started | - |
| 4. Status Workflow + Comments | 0/? | Not started | - |

## Coverage

**v1 Requirements: 21**
- Authentication: 5 (AUTH-01 to AUTH-05)
- Tasks: 8 (TASK-01 to TASK-08)
- Task Assignment: 4 (ASGN-01 to ASGN-04)
- Comments: 4 (CMNT-01 to CMNT-04)

**Phase Mapping:**
| Phase | Requirements |
|-------|--------------|
| 1 | SETUP-01 |
| 2 | AUTH-01, AUTH-02, AUTH-03, AUTH-04, AUTH-05 |
| 3 | TASK-01, TASK-02, TASK-03, TASK-04, TASK-05, TASK-08, ASGN-01, ASGN-02, ASGN-03, ASGN-04 |
| 4 | TASK-06, TASK-07, CMNT-01, CMNT-02, CMNT-03, CMNT-04 |

**Status:** ✓ All 21 v1 requirements mapped to phases
