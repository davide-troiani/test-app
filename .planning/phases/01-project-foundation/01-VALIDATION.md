---
phase: 01
slug: project-foundation
status: draft
nyquist_compliant: true
wave_0_complete: false
created: 2026-06-02
---

# Phase 01 - Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Vitest v4 + supertest v7 |
| **Config file** | `vitest.config.ts` |
| **Quick run command** | `rtk npx vitest run` |
| **Full suite command** | `rtk npx vitest run --reporter=verbose` |
| **Estimated runtime** | ~30 seconds |

---

## Sampling Rate

- **After every task commit:** Run `rtk npx vitest run`
- **After every plan wave:** Run `rtk npx vitest run --reporter=verbose`
- **Before `$gtd-verify-work`:** Full suite must be green
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 01-01-01 | 01 | 0 | SETUP-01a | T-01-04 | Server starts with validated config | smoke | `rtk npx vitest run tests/smoke/server.test.ts` | No - W0 | pending |
| 01-01-02 | 01 | 0 | SETUP-01b | T-01-04 | Server closes cleanly on SIGTERM | smoke | `rtk npx vitest run tests/smoke/shutdown.test.ts` | No - W0 | pending |
| 01-01-03 | 01 | 0 | SETUP-01c | T-01-05 | Prisma connects only with valid `DATABASE_URL` | integration | `rtk npx vitest run tests/integration/db-connect.test.ts` | No - W0 | pending |
| 01-01-04 | 01 | 0 | SETUP-01d | T-01-05 | Prisma migrations apply before DB-dependent tests | integration | `rtk npx vitest run tests/integration/migrations.test.ts` | No - W0 | pending |
| 01-01-05 | 01 | 0 | SETUP-01e | T-01-02 | Helmet emits security headers | unit | `rtk npx vitest run tests/unit/helmet-headers.test.ts` | No - W0 | pending |
| 01-01-06 | 01 | 0 | SETUP-01f | T-01-03 | Production CORS rejects unlisted origins | unit | `rtk npx vitest run tests/unit/cors.test.ts` | No - W0 | pending |
| 01-01-07 | 01 | 0 | SETUP-01g | T-01-06 | Zod failures use the canonical JSON error envelope | unit | `rtk npx vitest run tests/unit/validation.test.ts` | No - W0 | pending |
| 01-01-08 | 01 | 0 | SETUP-01h | T-01-04 | Startup fails when `JWT_SECRET` is shorter than 64 characters | unit | `rtk npx vitest run tests/unit/config-validation.test.ts` | No - W0 | pending |
| 01-01-09 | 01 | 0 | SETUP-01i | T-01-01 | App errors preserve public code/message/details only | unit | `rtk npx vitest run tests/unit/errors.test.ts` | No - W0 | pending |
| 01-01-10 | 01 | 0 | SETUP-01j | T-01-01 | Production unknown errors return `INTERNAL_ERROR` without stack text | unit | `rtk npx vitest run tests/unit/error-handler.test.ts` | No - W0 | pending |

*Status: pending / green / red / flaky*

---

## Wave 0 Requirements

- [ ] `vitest.config.ts` - Vitest configuration with TypeScript support and test match patterns
- [ ] `tests/setup.ts` - Shared test environment setup
- [ ] `tests/teardown.ts` - Prisma/test resource teardown
- [ ] `tests/unit/errors.test.ts` - SETUP-01i custom error classes
- [ ] `tests/unit/error-handler.test.ts` - SETUP-01j global error handler JSON shape
- [ ] `tests/unit/config-validation.test.ts` - SETUP-01h startup config validation
- [ ] `tests/unit/validation.test.ts` - SETUP-01g Zod validation error format
- [ ] `tests/unit/helmet-headers.test.ts` - SETUP-01e security headers
- [ ] `tests/unit/cors.test.ts` - SETUP-01f CORS middleware behavior
- [ ] `tests/smoke/server.test.ts` - SETUP-01a server starts
- [ ] `tests/smoke/shutdown.test.ts` - SETUP-01b graceful shutdown
- [ ] `tests/integration/db-connect.test.ts` - SETUP-01c Prisma connection
- [ ] `tests/integration/migrations.test.ts` - SETUP-01d migration success

---

## Manual-Only Verifications

All phase behaviors have automated verification. Manual setup may still be required to provide a reachable PostgreSQL `DATABASE_URL` for integration tests.

---

## Validation Sign-Off

- [x] All tasks have automated verify commands or Wave 0 dependencies
- [x] Sampling continuity: no 3 consecutive tasks without automated verify
- [x] Wave 0 covers all missing references
- [x] No watch-mode flags
- [x] Feedback latency < 30s
- [x] `nyquist_compliant: true` set in frontmatter

**Approval:** approved 2026-06-02
