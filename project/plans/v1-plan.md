# SpecTask v1 Implementation Plan

## Metadata / 元数据

| Field | Value |
|-------|-------|
| **Plan ID** | PLAN-001 |
| **Spec Reference** | SPEC-001, SPEC-002 |
| **Author** | SDD Tutorial |
| **Created** | 2024-12-01 |
| **Status** | Approved |

---

## Architecture Decisions / 架构决策

### AD-01: CLI Framework

- **Context**: Need a CLI argument parsing framework for SpecTask
- **Options Considered**:
  1. Commander.js — Mature, lightweight, great TS support / Minimal built-in validation
  2. yargs — Feature-rich / Complex API, heavier
  3. oclif — Enterprise-grade / Overkill for 500-800 line project
- **Decision**: Commander.js
- **Rationale**: Best balance of maturity, TypeScript support, and simplicity for a bounded-scope project
- **Consequences**: Manual input validation needed (Commander doesn't validate argument values)

### AD-02: Storage and Migration Strategy

- **Context**: Need to decide schema management approach for SQLite
- **Options Considered**:
  1. CREATE TABLE IF NOT EXISTS (inline) — Simple, no extra tooling / No rollback support
  2. drizzle-kit migrations — Full migration framework / Heavy dependency for v1
  3. Manual version tracking — Middle ground / Custom code to maintain
- **Decision**: CREATE TABLE IF NOT EXISTS (inline schema in database.ts)
- **Rationale**: v1 schema is stable; migration framework adds complexity without benefit until schema evolves
- **Consequences**: If schema changes post-v1, will need to introduce migration strategy

### AD-03: Password Hashing

- **Context**: Need to hash passwords per SPEC-002 REQ-U02 (bcrypt, cost 12)
- **Options Considered**:
  1. bcrypt (native) — Fast / Requires node-gyp, platform-specific build issues
  2. bcryptjs (pure JS) — Cross-platform, zero native deps / ~3x slower
  3. Node.js crypto.scrypt — Built-in / Not bcrypt (spec explicitly requires bcrypt)
- **Decision**: bcryptjs
- **Rationale**: Cross-platform compatibility is more important than speed for a CLI tool that hashes once per login
- **Consequences**: Login takes ~200-300ms for hash verification (acceptable for CLI)

### AD-04: ID Generation

- **Context**: SPEC-001 REQ-U03 requires 21-character unique IDs
- **Options Considered**:
  1. nanoid — Battle-tested, URL-safe, configurable / External dependency
  2. crypto.randomUUID() — Built-in / UUID format (36 chars with dashes)
  3. Custom implementation — No dependency / Reinventing the wheel
- **Decision**: nanoid
- **Rationale**: Exact match for spec requirement (21 chars), well-maintained, tiny footprint
- **Consequences**: One additional dependency (nanoid)

---

## System Design / 系统设计

### Component Diagram

```
CLI Layer (Commander.js)
├── auth-commands.ts      → auth-service
├── task-commands.ts      → task-service
├── notify-command.ts     → notification-service
└── index.ts              → program entry

Service Layer (Business Logic)
├── auth-service.ts       → user-repository, session file I/O
├── task-service.ts       → task-repository, validators
└── notification-service.ts → task-repository, date-utils

Repository Layer (Data Access)
├── user-repository.ts    → database
└── task-repository.ts    → database

Storage Layer (SQLite)
├── database.ts           → better-sqlite3
└── paths.ts              → ~/.spectask/* constants
```

### Component Responsibilities

| Component | Responsibility | Interface |
|-----------|---------------|-----------|
| CLI Layer | Parse args, format output, handle exit codes | Commander.js commands |
| auth-service | Register, login, logout, session check | Factory function returning { register, login, logout, getCurrentUser } |
| task-service | CRUD validation, state transitions, orchestration | Factory function returning { create, list, show, update, delete } |
| notification-service | Due date analysis, reminder generation | Factory function returning { checkDueDates } |
| user-repository | User persistence (find, create) | Factory function returning { findByUsername, create } |
| task-repository | Task persistence (CRUD + filtered queries) | Factory function returning { create, findById, findByUserId, update, delete } |
| database | Connection, initialization, schema | Factory function returning { db instance } |

### Data Flow: Create Task

```
User input: spectask add "Write tests" --priority high
    │
    ▼
CLI Layer: parse args → { title: "Write tests", priority: "high" }
    │
    ▼
Service Layer: validate input → generate ID → build task object
    │
    ▼
Repository Layer: serialize tags to JSON → execute INSERT
    │
    ▼
Storage Layer: better-sqlite3 db.prepare().run()
    │
    ▼
Result flows back up: Result<Task> → format output → terminal
```

---

## Data Model / 数据模型

### Type Definitions

```typescript
// src/types/result.ts
type Result<T, E = string> =
  | { readonly success: true; readonly data: T }
  | { readonly success: false; readonly error: E };

// src/types/task.ts
type TaskStatus = 'pending' | 'in-progress' | 'completed' | 'archived';
type TaskPriority = 'low' | 'medium' | 'high' | 'critical';

interface Task {
  readonly id: string;
  readonly userId: string;
  readonly title: string;
  readonly description: string | null;
  readonly status: TaskStatus;
  readonly priority: TaskPriority;
  readonly tags: readonly string[];
  readonly dueDate: string | null;
  readonly specRef: string | null;
  readonly createdAt: string;
  readonly updatedAt: string;
}

interface CreateTaskInput {
  readonly title: string;
  readonly description?: string;
  readonly priority?: TaskPriority;
  readonly tags?: readonly string[];
  readonly dueDate?: string;
  readonly specRef?: string;
}

interface UpdateTaskInput {
  readonly title?: string;
  readonly description?: string;
  readonly status?: TaskStatus;
  readonly priority?: TaskPriority;
  readonly tags?: readonly string[];
  readonly dueDate?: string | null;
  readonly specRef?: string | null;
}

interface TaskFilters {
  readonly status?: TaskStatus;
  readonly priority?: TaskPriority;
  readonly tag?: string;
  readonly specRef?: string;
}

// src/types/user.ts
interface User {
  readonly id: string;
  readonly username: string;
  readonly passwordHash: string;
  readonly createdAt: string;
}

interface Session {
  readonly userId: string;
  readonly username: string;
}
```

### Database Schema

```sql
-- Users table
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY,
  username TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_users_username ON users(username);

-- Tasks table
CREATE TABLE IF NOT EXISTS tasks (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  status TEXT NOT NULL DEFAULT 'pending',
  priority TEXT NOT NULL DEFAULT 'medium',
  tags TEXT NOT NULL DEFAULT '[]',
  due_date TEXT,
  spec_ref TEXT,
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX IF NOT EXISTS idx_tasks_user_id ON tasks(user_id);
CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status);
CREATE INDEX IF NOT EXISTS idx_tasks_priority ON tasks(priority);
CREATE INDEX IF NOT EXISTS idx_tasks_due_date ON tasks(due_date);
```

---

## File Structure Plan / 文件结构计划

### New Files

```
src/
├── cli/
│   ├── index.ts              — Program entry, command registration
│   ├── auth-commands.ts      — register, login, logout, whoami commands
│   ├── task-commands.ts      — add, list, show, update, delete commands
│   └── notify-command.ts     — notify command
├── services/
│   ├── auth-service.ts       — Auth business logic
│   ├── auth-service.test.ts  — Unit tests
│   ├── task-service.ts       — Task CRUD business logic
│   ├── task-service.test.ts  — Unit tests
│   └── notification-service.ts — Due date checking
├── repositories/
│   ├── user-repository.ts    — User data access
│   ├── user-repository.test.ts — Integration tests
│   ├── task-repository.ts    — Task data access
│   └── task-repository.test.ts — Integration tests
├── storage/
│   ├── database.ts           — SQLite init and schema
│   ├── database.test.ts      — Schema creation tests
│   └── paths.ts              — Path constants
├── types/
│   ├── task.ts               — Task types and enums
│   ├── user.ts               — User and Session types
│   └── result.ts             — Result type
├── hooks/
│   ├── hook-runner.ts        — Hook discovery and execution
│   └── hook-types.ts         — Hook event type definitions
└── utils/
    ├── validators.ts         — Input validation
    ├── validators.test.ts    — Validator tests
    ├── formatters.ts         — Terminal output formatting
    └── date-utils.ts         — Date helpers
```

---

## Dependency Graph / 依赖图

```
types/result.ts ─────────────────┐
types/task.ts ─────────────┐     │
types/user.ts ──────┐      │     │
                    │      │     │
                    ▼      ▼     ▼
storage/database.ts ──────────────────┐
storage/paths.ts ─────────────────┐   │
                                  │   │
                                  ▼   ▼
repositories/user-repository.ts ────────┐
repositories/task-repository.ts ────┐   │
                                    │   │
                                    ▼   ▼
services/auth-service.ts ─────────────────┐
services/task-service.ts ──────────┐      │
services/notification-service.ts ──┤      │
                                   │      │
                                   ▼      ▼
cli/auth-commands.ts ─────────────────────┐
cli/task-commands.ts ──────────────────┐  │
cli/notify-command.ts ─────────────┐   │  │
                                   │   │  │
                                   ▼   ▼  ▼
                              cli/index.ts
```

### Critical Path

```
types → storage → repositories → services → cli
```

---

## Risk Assessment / 风险评估

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|-----------|
| 1 | better-sqlite3 native build failure on user's platform | Medium | High | Document prerequisites; provide clear error message; consider sql.js fallback |
| 2 | bcryptjs ESM compatibility issues | Low | Medium | Validate in Phase 2 tests; fallback to dynamic import if needed |
| 3 | File permission errors on ~/.spectask/ | Low | Medium | Check and create directory with clear error on first run |
| 4 | Hook system security (arbitrary code execution) | Medium | Medium | Document that hooks run in same process; recommend trusted hooks only |
| 5 | Large tag arrays causing SQLite JSON parsing overhead | Low | Low | Enforce max 10 tags per spec; test with boundary data |

---

## Implementation Phases / 实施阶段

### Phase 1: Foundation (基础搭建)

**Goal**: Establish project skeleton, type system, and storage layer

- [ ] Initialize project: package.json, tsconfig.json, vitest.config.ts, .gitignore
- [ ] Define all types: result.ts, task.ts, user.ts
- [ ] Implement storage: database.ts (init, schema), paths.ts
- [ ] Write foundation tests: type assertions, database init

**Verification**: `pnpm build` succeeds, `pnpm test` passes, database file creates correctly

### Phase 2: Auth (用户认证)

**Goal**: Implement user registration, login, and session management

- [ ] Implement user-repository.ts (findByUsername, create)
- [ ] Implement auth-service.ts (register, login, logout, getCurrentUser)
- [ ] Implement auth-commands.ts (CLI commands)
- [ ] Write unit tests for auth-service
- [ ] Write integration tests for user-repository

**Verification**: All SPEC-002 acceptance criteria pass

### Phase 3: Task CRUD (核心功能)

**Goal**: Implement full task create, read, update, delete

- [ ] Implement task-repository.ts (create, findById, findByUserId, update, delete)
- [ ] Implement validators.ts (title, priority, status, tags, dueDate)
- [ ] Implement task-service.ts (createTask, listTasks, showTask, updateTask, deleteTask)
- [ ] Implement task-commands.ts (CLI commands)
- [ ] Write unit tests for task-service, validators
- [ ] Write integration tests for task-repository

**Verification**: All SPEC-001 acceptance criteria pass

### Phase 4: Enhancements (增强功能)

**Goal**: Add spec linking, notifications, hooks, and polished output

- [ ] Implement spec linking (--spec flag, specRef validation)
- [ ] Implement notification-service.ts and notify-command.ts
- [ ] Implement hook-runner.ts and hook-types.ts
- [ ] Implement formatters.ts (chalk-based colored output)
- [ ] Implement date-utils.ts
- [ ] Final test coverage verification (>= 80%)

**Verification**: All features work end-to-end, coverage meets threshold

---

## Requirement Traceability Matrix

| Requirement | Phase | Component |
|-------------|-------|-----------|
| SPEC-001 REQ-U01~U06 | Phase 1 + 3 | types/, storage/, task-repository |
| SPEC-001 REQ-E01~E08 | Phase 3 | task-service, task-commands |
| SPEC-001 REQ-S01~S03 | Phase 3 | task-service (state validation) |
| SPEC-001 REQ-O01~O03 | Phase 3 + 4 | task-commands, task-service |
| SPEC-001 REQ-X01~X06 | Phase 3 | validators, task-service |
| SPEC-002 REQ-U01~U03 | Phase 2 | types/, storage/, user-repository |
| SPEC-002 REQ-E01~E04 | Phase 2 | auth-service, auth-commands |
| SPEC-002 REQ-S01~S02 | Phase 2 | auth-service (session check) |
| SPEC-002 REQ-X01~X05 | Phase 2 | auth-service, validators |

---

## Performance Considerations / 性能考虑

- **Expected load**: Single user, max 10,000 tasks
- **Response time target**: < 100ms for all CLI operations
- **SQLite optimization**: WAL mode for better read performance; prepared statements for repeated queries
- **Startup time**: Lazy-load heavy modules (chalk, date-fns) only when needed
