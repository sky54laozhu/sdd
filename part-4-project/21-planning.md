# Chapter 21: 架构与实现计划 (Architecture and Implementation Plan)

> Plan 是连接"想要什么"（Spec）和"具体做什么"（Tasks）的桥梁——它将需求中描述的行为翻译为可执行的技术方案，包括架构决策、数据模型、文件结构和实施阶段。

---

## 本章目标

基于已完成的 Constitution（Chapter 19）和 Specifications（Chapter 20），撰写 SpecTask 的第一版实施计划。这份计划必须**覆盖 spec 中的每一条需求**，同时为 Claude Code 提供足够的技术细节来生成高质量代码。

**产出物**：`project/plans/v1-plan.md`

---

## Plan 撰写的输入与约束

开始撰写 plan 之前，明确输入条件：

| 输入 | 来源 | 作用 |
|------|------|------|
| CLAUDE.md | Chapter 19 | 技术栈约束、架构模式、编码标准 |
| SPEC-001 | Chapter 20 | Task CRUD 的功能需求（12 条 REQ） |
| SPEC-002 | Chapter 20 | Auth 的功能需求（12 条 REQ） |

**约束回顾**（来自 Constitution）：
- 四层架构：CLI → Service → Repository → Storage
- Result pattern（不使用 throw）
- Factory functions（不使用 class-based DI）
- Strict TypeScript + ESM
- 80%+ test coverage
- better-sqlite3 作为存储

---

## 架构决策过程

Plan 的第一步是做出架构决策（Architecture Decisions）。每个决策使用精简的 ADR（Architecture Decision Record）格式记录。

### 决策 1: CLI Framework 选型

**为什么需要决策**：Node.js 生态有多个 CLI framework（Commander.js、yargs、oclif、clipanion）。Constitution 已指定 Commander.js，但 plan 需要记录这个选择的理由。

**思考过程**：
- Commander.js：最成熟，TypeScript 支持好，轻量级
- yargs：功能丰富但 API 较复杂
- oclif：企业级但过重（for 一个 500-800 行项目）
- clipanion：TypeScript-first 但社区较小

结论：Commander.js 在成熟度和轻量级之间取得最佳平衡。

### 决策 2: 存储方案

**为什么需要决策**：虽然 Constitution 指定了 better-sqlite3，plan 需要决定 migration strategy 和 schema design approach。

**思考过程**：
- 简单 `CREATE TABLE IF NOT EXISTS`（适合 v1，无需 migration 框架）
- 正式 migration 框架（drizzle-kit、knex migrations）——对 v1 过重
- 手动版本号 migration——中间方案

结论：v1 使用 `CREATE TABLE IF NOT EXISTS`，当需要 schema 变更时再引入版本化 migration。

### 决策 3: Password Hashing

**为什么需要决策**：Spec 要求 bcrypt cost 12，但 bcrypt 有 native binding 和 pure-JS 两种实现。

**思考过程**：
- `bcrypt`（native）：更快但需要 node-gyp 编译环境
- `bcryptjs`（pure JS）：跨平台无编译依赖，但慢约 3x
- 对于 CLI 工具，每次 login 只调用一次 hash verify，3x 的差异（~100ms vs ~300ms）可接受

结论：使用 `bcryptjs` 避免 native binding 带来的安装问题。

### 决策 4: ID Generation

**为什么需要决策**：Spec 要求 21-char nanoid，但需要确认是否使用 nanoid 库或自定义实现。

**思考过程**：
- `nanoid` 库：成熟、URL-safe、可配置长度
- `crypto.randomUUID()`：Node.js 内置，但格式为 UUID（36 chars with dashes）
- 自定义实现：无必要

结论：使用 `nanoid` 库，默认 21 字符。

---

## 系统设计

### 分层架构图

```
┌─────────────────────────────────────────────────────────┐
│                     CLI Layer                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │ auth.ts  │ │ task.ts  │ │ notify.ts│ │ hooks.ts  │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────┬─────┘  │
│       │             │            │              │         │
├───────┼─────────────┼────────────┼──────────────┼────────┤
│       ▼             ▼            ▼              ▼        │
│                   Service Layer                           │
│  ┌────────────┐ ┌────────────┐ ┌─────────────────────┐  │
│  │auth-service│ │task-service│ │notification-service │  │
│  └─────┬──────┘ └─────┬──────┘ └──────────┬──────────┘  │
│        │               │                    │            │
├────────┼───────────────┼────────────────────┼────────────┤
│        ▼               ▼                    ▼            │
│                 Repository Layer                          │
│  ┌────────────┐ ┌────────────┐                           │
│  │ user-repo  │ │ task-repo  │                           │
│  └─────┬──────┘ └─────┬──────┘                           │
│        │               │                                 │
├────────┼───────────────┼─────────────────────────────────┤
│        ▼               ▼                                 │
│                  Storage Layer                            │
│  ┌─────────────────────────────────────────────────┐     │
│  │              database.ts (SQLite)                │     │
│  │         ~/.spectask/data.db                      │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 各层职责

| Layer | 职责 | 约束 |
|-------|------|------|
| CLI | 解析命令行参数，格式化输出，调用 Service | 唯一允许 process.exit() 的层 |
| Service | 业务逻辑、输入验证、编排多个 Repository | 返回 Result<T>，不允许 throw |
| Repository | 数据访问抽象，定义接口 | 返回 Result<T>，隔离 SQL 细节 |
| Storage | SQLite 连接管理、表创建、低级操作 | 封装 better-sqlite3 API |

### 数据流示例：创建任务

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

## 数据模型

### Type Definitions

```typescript
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
  readonly dueDate: string | null;      // ISO 8601
  readonly specRef: string | null;      // file path
  readonly createdAt: string;           // ISO 8601
  readonly updatedAt: string;           // ISO 8601
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

// src/types/result.ts
type Result<T, E = string> =
  | { readonly success: true; readonly data: T }
  | { readonly success: false; readonly error: E };
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
  tags TEXT NOT NULL DEFAULT '[]',       -- JSON array
  due_date TEXT,                          -- ISO 8601
  spec_ref TEXT,                          -- file path
  created_at TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE INDEX IF NOT EXISTS idx_tasks_user_id ON tasks(user_id);
CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status);
CREATE INDEX IF NOT EXISTS idx_tasks_priority ON tasks(priority);
CREATE INDEX IF NOT EXISTS idx_tasks_due_date ON tasks(due_date);
```

**设计说明**：
- `tags` 存储为 JSON 字符串（SQLite 无原生数组类型），Repository 层负责序列化/反序列化
- `status` 和 `priority` 使用 TEXT 而非 INTEGER enum——可读性优先
- `due_date` 和 `spec_ref` 可为 NULL（Optional requirements）
- Indexes 覆盖常见查询模式（按 status/priority 过滤、按 due_date 排序）

---

## 文件结构计划

```
src/
├── cli/
│   ├── index.ts              — CLI 入口，注册所有命令
│   ├── auth-commands.ts      — register, login, logout, whoami
│   ├── task-commands.ts      — add, list, show, update, delete
│   └── notify-command.ts     — notify (due date checks)
├── services/
│   ├── auth-service.ts       — registration, login, session management
│   ├── task-service.ts       — CRUD business logic, validation
│   └── notification-service.ts — due date checking logic
├── repositories/
│   ├── user-repository.ts    — User data access (interface + implementation)
│   └── task-repository.ts    — Task data access (interface + implementation)
├── storage/
│   ├── database.ts           — SQLite connection, initialization, schema
│   └── paths.ts              — File path constants (~/.spectask/*)
├── types/
│   ├── task.ts               — Task, CreateTaskInput, UpdateTaskInput
│   ├── user.ts               — User, Session
│   └── result.ts             — Result<T, E> type definition
├── hooks/
│   ├── hook-runner.ts        — Discover and execute lifecycle hooks
│   └── hook-types.ts         — Hook event definitions
└── utils/
    ├── validators.ts         — Input validation functions
    ├── formatters.ts         — Terminal output formatting
    └── date-utils.ts         — Date comparison and formatting helpers
```

**命名规则验证**（对照 Constitution）：
- 文件名：kebab-case （符合）
- 每个文件单一职责（符合）
- 预估每文件 40-100 行，总量约 600-800 行（符合 Constitution "max 400 lines per file"）

---

## 依赖图

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

**关键路径**：`types → storage → repositories → services → cli`

这意味着实现顺序是 bottom-up：先建地基（types、storage），再建上层（services、cli）。

---

## 风险评估

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|-----------|
| 1 | better-sqlite3 native binding 安装失败（特定平台） | Medium | High | 文档中提供 troubleshooting 步骤；考虑 fallback 到 sql.js |
| 2 | bcryptjs 在 ESM 环境下的兼容性 | Low | Medium | 测试中验证 ESM import；必要时使用 Node.js crypto.scrypt 替代 |
| 3 | SQLite 并发写入冲突（虽然单用户） | Low | Low | better-sqlite3 是同步的，单进程无并发问题 |
| 4 | 文件系统权限（~/.spectask/ 不可写） | Low | Medium | 首次运行时检查并给出清晰错误提示 |
| 5 | 大量任务时 SQLite 查询性能 | Low | Medium | 添加索引覆盖常见查询；Spec 已约束 10000 任务上限 |

---

## 实施阶段

### Phase 1: Foundation（基础搭建）

**目标**：建立项目骨架、类型系统和存储层

**交付物**：
- [ ] 项目初始化：package.json, tsconfig.json, vitest.config.ts
- [ ] Type definitions: result.ts, task.ts, user.ts
- [ ] Storage layer: database.ts (SQLite 初始化、schema 创建), paths.ts
- [ ] 基础 unit tests: types 和 storage 层

**验证方式**：
- `pnpm build` 编译成功
- `pnpm test` 全部通过
- SQLite 数据库文件正确创建

**覆盖 Spec Requirements**：REQ-U01 ~ REQ-U06（data model 约束）

### Phase 2: Auth（用户认证）

**目标**：实现用户注册、登录、session 管理

**交付物**：
- [ ] user-repository.ts: findByUsername, create
- [ ] auth-service.ts: register, login, logout, getCurrentUser
- [ ] auth-commands.ts: register, login, logout, whoami CLI 命令
- [ ] Unit tests for auth-service
- [ ] Integration tests for user-repository

**验证方式**：
- 可以成功 register → login → whoami → logout
- 密码哈希存储正确
- Session 文件正确创建/删除
- 所有 SPEC-002 Acceptance Criteria 通过

**覆盖 Spec Requirements**：SPEC-002 全部（REQ-U01 ~ REQ-X05）

### Phase 3: Task CRUD（核心功能）

**目标**：实现任务的完整增删改查

**交付物**：
- [ ] task-repository.ts: create, findById, findAll (with filters), update, delete
- [ ] task-service.ts: createTask, listTasks, showTask, updateTask, deleteTask
- [ ] task-commands.ts: add, list, show, update, delete CLI 命令
- [ ] validators.ts: title, priority, status, tags, dueDate 验证
- [ ] Unit tests for task-service, validators
- [ ] Integration tests for task-repository

**验证方式**：
- 完整 CRUD 流程可运行
- 所有过滤和排序正常工作
- 状态转换规则正确执行
- 所有 SPEC-001 Acceptance Criteria 通过

**覆盖 Spec Requirements**：SPEC-001 全部（REQ-U01 ~ REQ-X06）

### Phase 4: Spec Linking + Notifications + Hooks（增强功能）

**目标**：实现 spec 关联、到期提醒和插件系统

**交付物**：
- [ ] Spec linking: --spec flag 支持，specRef 验证和查询
- [ ] notification-service.ts: 检查 due date，生成提醒消息
- [ ] notify-command.ts: notify CLI 命令
- [ ] hook-runner.ts: 发现和执行 lifecycle hooks
- [ ] hook-types.ts: 事件类型定义
- [ ] formatters.ts: 彩色 terminal 输出
- [ ] date-utils.ts: 日期比较和格式化

**验证方式**：
- `spectask add --spec <path>` 正确关联
- `spectask notify` 显示正确的提醒
- Hook 文件被正确发现和执行
- 覆盖率 >= 80%

---

## Requirement Traceability Matrix

确认 plan 覆盖了 spec 中的所有需求：

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

**Gap check**: 每条 REQ 都有对应的 Phase 和 Component。无遗漏。

---

## 完整 Plan 文件

以下是 `project/plans/v1-plan.md` 的完整内容。基于 [plan template](../templates/plan-template.md) 定制：

```markdown
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
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY,
  username TEXT NOT NULL UNIQUE,
  password_hash TEXT NOT NULL,
  created_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_users_username ON users(username);

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
│   ├── task-service.ts       — Task CRUD business logic
│   ├── task-service.test.ts  — Unit tests
│   ├── auth-service.test.ts  — Unit tests
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

## Performance Considerations / 性能考虑

- **Expected load**: Single user, max 10,000 tasks
- **Response time target**: < 100ms for all CLI operations
- **SQLite optimization**: WAL mode for better read performance; prepared statements for repeated queries
- **Startup time**: Lazy-load heavy modules (chalk, date-fns) only when needed
```

---

## Phase Gate Review

在 plan 定稿前，进行一次 phase gate review——检查 plan 是否完整覆盖了所有 spec requirements：

### Completeness Check

**问题 1**: 每条 SPEC-001 和 SPEC-002 的 REQ 是否都有对应的 Phase 和 Component？

答案：是的。Traceability matrix 确认了完整覆盖。

**问题 2**: Plan 是否与 Constitution 的约束一致？

检查项：
- 四层架构？ 是的（CLI → Service → Repository → Storage）
- Result pattern？ 是的（所有 Service 和 Repository 返回 Result<T>）
- Factory functions？ 是的（Component Responsibilities 表中注明）
- Strict TypeScript？ 是的（type definitions 使用 readonly、union types）
- 80%+ coverage？ 是的（Phase 4 最终验证）
- kebab-case 文件名？ 是的（File Structure 全部符合）

**问题 3**: 实现顺序是否考虑了依赖关系？

答案：是的。Phase 1（types, storage）→ Phase 2（auth，因为 Task CRUD 依赖 userId）→ Phase 3（task CRUD）→ Phase 4（增强功能依赖前三个 phase）。

**结论**: Plan is complete and ready for task decomposition.

---

## 从 Plan 到 Tasks

Plan 完成后，下一步是 **task decomposition（任务分解）**——将每个 Phase 的交付物拆解为 Claude Code 可以逐个执行的原子任务。这属于 SDD 五阶段中的第四阶段（Tasks），超出本章范围，但我们已经为它打下了坚实基础。

关键转换规则：
- Plan 中每个 `[ ]` checkbox 变成一个 task
- 每个 task 遵循 TDD workflow：先写 test，再写 implementation
- Task 的执行顺序遵循 dependency graph（自底向上）

---

## Key Takeaways (要点回顾)

1. **Plan 的核心作用是将 Spec 的 "What" 翻译为 "How"**——它回答架构选型、数据模型、文件组织和实施顺序等技术问题
2. **Architecture Decisions 用精简 ADR 格式记录**——Context → Options → Decision → Rationale → Consequences，让后来者理解为什么这样选
3. **Dependency graph 决定了实现顺序**——bottom-up（types → storage → repository → service → cli）确保每一步都有稳固的下层支撑
4. **Requirement Traceability Matrix 是 plan 完整性的终极验证**——每条 REQ 必须有对应的 Phase 和 Component，无遗漏
5. **Phase Gate Review 是推进到下一阶段前的必经检查**——确认 plan 与 constitution 一致、覆盖所有 spec requirements、依赖顺序合理

---

## Next (下一章)

[Chapter 22: 任务分解与 TDD 实现 →](./22-implementation.md)
