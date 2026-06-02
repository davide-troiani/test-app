# Task Management API

## What This Is

A RESTful API for task management built in Node.js with PostgreSQL. Provides authentication, task CRUD operations, comments, and task assignment. Designed for developers building task management integrations or lightweight project tools.

## Core Value

A clean, reliable task management API with minimal feature set — create, track, and assign tasks with persistent state.

## Requirements

### Validated

(None yet — validate through task PRs)

### Active

- [ ] User authentication (signup, login, logout)
- [ ] Task CRUD (create, read, update, delete)
- [ ] Task statuses: TODO, IN PROGRESS, DONE
- [ ] Task comments (add, read, delete)
- [ ] Task assignment (assign to users)
- [ ] User can view assigned tasks

### Out of Scope

- Task search/filtering — basic listing only
- Task hierarchy (subtasks) — flat task structure
- Task priorities/weights — status-based only
- Webhooks/notifications — no event push
- Rate limiting — future consideration
- Admin dashboard — pure API, no UI

## Context

- Greenfield project, no existing codebase
- PostgreSQL for relational data (tasks, users, comments)
- Node.js runtime (Express.js assumed)
- RESTful API design
- Session or JWT-based authentication

## Constraints

- **Tech Stack**: Node.js + PostgreSQL — explicit choice
- **API Style**: RESTful — JSON request/response
- **Auth**: User authentication required — no anonymous access
- **Database**: PostgreSQL — relational structure for tasks and users

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| RESTful API | Standard, widely understood, easy to integrate | — Pending |
| PostgreSQL | Relational data (tasks, users, assignments) benefits from SQL | — Pending |
| JWT-based auth | Stateless, scalable, common pattern | — Pending |

---

*Last updated: 2026-06-02 after initialization*

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gtd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gtd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state