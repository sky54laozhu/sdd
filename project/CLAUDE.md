# SpecTask — Project Constitution

## Project Identity / 项目身份

**Project**: SpecTask
**Description**: A spec-aware CLI task manager that demonstrates SDD methodology, linking tasks to their origin specifications for full traceability.
**Domain**: Developer Tools
**Stage**: Greenfield

---

## Tech Stack / 技术栈

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | TypeScript | 5.x |
| Runtime | Node.js | >= 18 LTS |
| Module System | ESM | — |
| CLI Framework | Commander.js | 12.x |
| Storage | better-sqlite3 | 11.x |
| Date Utilities | date-fns | 3.x |
| Testing | Vitest | 2.x |
| Output | chalk | 5.x |
| Package Manager | pnpm | latest |

---

## Architecture Conventions / 架构约定

### Pattern / 架构模式

Layered architecture with strict unidirectional dependency flow.

### Layering / 分层

```
CLI Layer (Commander.js commands)
    ↓ calls
Service Layer (business logic, validation, orchestration)
    ↓ calls
Repository Layer (data access interface, abstract over storage)
    ↓ calls
Storage Layer (better-sqlite3 implementation)
```

### Module Boundaries / 模块边界

- CLI layer MUST NOT import from Repository or Storage layers directly
- Service layer MUST NOT import from CLI layer
- Repository layer defines interfaces; Storage layer implements them
- Cross-cutting concerns (logging, Result type) live in utils/
- Dependency injection via factory functions, NOT class-based DI containers

### Directory Structure / 目录结构

```
src/
├── cli/               # Command definitions (one file per command group)
├── services/          # Business logic (one file per domain concept)
├── repositories/      # Data access interfaces + implementations
├── storage/           # SQLite setup, migrations, connection management
├── types/             # TypeScript interfaces, enums, type aliases
├── hooks/             # Plugin hook system
└── utils/             # Shared utilities (Result, validators, formatters)
```

---

## Code Standards / 编码标准

### TypeScript Configuration

- `strict: true` — non-negotiable
- `noUncheckedIndexedAccess: true`
- `exactOptionalPropertyTypes: true`
- Target: ES2022
- Module: ESNext with NodeNext module resolution

### Naming / 命名

- Files: `kebab-case.ts` (e.g., `task-service.ts`)
- Functions: `camelCase` (e.g., `createTask`, `findTaskById`)
- Types/Interfaces: `PascalCase` (e.g., `Task`, `CreateTaskInput`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `MAX_TITLE_LENGTH`, `DEFAULT_PRIORITY`)
- Boolean variables: `is`/`has`/`should`/`can` prefix (e.g., `isOverdue`, `hasSpecRef`)

### Style / 风格

- Max file length: 400 lines (hard limit)
- Max function length: 40 lines
- Max nesting depth: 3 levels (use early returns)
- Immutability: ALWAYS prefer immutable patterns; never mutate function arguments
- No class-based OOP; use plain objects + functions + closures
- Named exports only; no default exports
- No barrel files (index.ts re-exports)

### Error Handling / 错误处理

Use the Result pattern — NEVER throw exceptions in service or repository layers:

```typescript
type Result<T, E = string> =
  | { success: true; data: T }
  | { success: false; error: E };
```

- CLI layer is the ONLY place where process.exit() may be called
- Service functions return Result<T>
- Repository functions return Result<T>
- Use descriptive error messages that include context (e.g., task ID, field name)

### Forbidden Patterns / 禁止模式

- No `any` type (use `unknown` and narrow)
- No `console.log` in service/repository layers (use structured return values)
- No synchronous file I/O outside of better-sqlite3 operations
- No mutation of function parameters
- No nested ternary expressions
- No `eval()` or `Function()` constructor

---

## Testing Requirements / 测试要求

### Coverage Target

- Minimum: 80% line coverage
- Service layer: 90%+
- Repository layer: 85%+

### Strategy

- Unit tests: All service functions, utility functions, validators
- Integration tests: Repository layer with real SQLite (in-memory)
- Test files: Co-located with source (e.g., `task-service.ts` → `task-service.test.ts`)

### Commands

```bash
pnpm test              # Run all tests
pnpm test:coverage     # Run with coverage report
pnpm test:watch        # Watch mode during development
```

### TDD Workflow

Mandatory for all feature implementation:
1. Write failing test (RED)
2. Write minimal implementation to pass (GREEN)
3. Refactor while tests stay green (REFACTOR)
4. Verify coverage meets threshold

---

## Security Boundaries / 安全边界

### Sensitive Areas

- `src/services/auth-service.ts` — password hashing, session management
- `src/storage/` — database file access
- `src/hooks/` — dynamic code loading from user-defined hooks

### Forbidden Operations

- NEVER store passwords in plaintext; always use bcrypt/scrypt hash
- NEVER use `eval()` or `new Function()` to load hook code
- NEVER trust CLI input without validation (use zod or manual validation)
- NEVER hardcode secrets, file paths, or credentials
- NEVER commit database files or session files to version control

### Input Validation Rules

- All CLI arguments validated before reaching service layer
- Title: non-empty string, max 200 characters
- Priority: must be one of 'low' | 'medium' | 'high' | 'critical'
- Status: must be one of 'pending' | 'in-progress' | 'completed' | 'archived'
- Due date: valid ISO 8601 date, must not be in the past (for creation)
- Tags: comma-separated alphanumeric strings, max 10 tags, max 30 chars each

---

## SDD Workflow / SDD 工作流

### Phases

```
Constitution → Specify → Plan → Tasks → Implement → Verify
```

### Workflow Rules

1. **No implementation without a spec** — Every feature must have a spec file in `specs/` before coding begins
2. **No code without a plan** — Implementation plan must exist in `plans/` before task decomposition
3. **Tests before implementation** — Task ordering always puts test creation before production code
4. **Verify against spec** — Every completed feature must trace back to spec requirements
5. **Phase gate review** — No advancing to next phase without reviewing current phase output

### File Locations

```
specs/          — Feature specifications (EARS format)
plans/          — Implementation plans
src/            — Implementation code
```

---

## Additional Instructions / 附加指令

- Prefer composition over inheritance (no classes)
- All dates stored and compared in UTC
- CLI output uses chalk for color; always provide --no-color flag
- Database file location: `~/.spectask/data.db`
- Session file location: `~/.spectask/session.json`
- Hook files location: `~/.spectask/hooks/`
- Use nanoid (21 chars) for all entity IDs
- All list commands support --json flag for machine-readable output
