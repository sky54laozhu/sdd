# Chapter 19: 撰写项目宪法 (Writing the Project Constitution)

> Constitution 不是文档——它是 Claude Code 的操作系统内核。写好它，后续所有 phase 的产出质量都会提升；写错它，整个项目就在沙滩上建高楼。

---

## 本章目标

本章将从零撰写 SpecTask 项目的 `CLAUDE.md`（项目宪法），并解释每一节的设计意图。完成后你将得到一份可以直接用于真实开发的 constitution 文件。

**产出物**：`project/CLAUDE.md`

---

## Constitution 的设计原则

在动笔之前，先回顾三条核心设计原则（详见 [Chapter 6](../part-2-claude-code-mastery/06-claude-md-constitution.md)）：

1. **Constraining over Permissive**（约束优于许可）：明确说"不许做什么"比列举"可以做什么"更有效
2. **Permanent over Transient**（永久优于临时）：Constitution 应只包含项目生命周期内不变的规则
3. **Actionable over Aspirational**（可执行优于愿景性）：每条规则都应可被直接验证（testable）

带着这三条原则，我们逐节撰写。

---

## 逐节撰写过程

### Section 1: Project Identity

**思考过程**：Claude Code 需要在 10 秒内理解"我在为什么项目工作"。这一节就是项目的电梯演讲（elevator pitch）。

关键决策：
- 明确说明这是一个 **demonstration project**——这影响 Claude 的决策（比如不需要考虑生产级 scaling）
- 标注 **development stage = greenfield**——Claude 会选择更积极的重构策略而非保守的兼容性策略
- 声明 **domain = developer tools**——让 Claude 知道用户是开发者，可以使用技术术语

### Section 2: Tech Stack

**思考过程**：这是 Claude Code 做技术决策的第一参考。如果这里写了 "Vitest"，Claude 就不会在测试时引入 Jest。

关键决策：
- 锁定 TypeScript 5.x（不是 "latest"）——避免 Claude 引入尚不稳定的特性
- 指定 ESM modules——避免 Claude 写出 CommonJS 代码
- 明确 pnpm 作为 package manager——统一所有命令调用

### Section 3: Architecture Conventions

**思考过程**：架构分层是 Claude Code 放置新文件时的导航图。没有这一节，Claude 会把所有逻辑塞进一个文件。

关键决策：
- 四层架构：CLI → Service → Repository → Storage
- **Dependency injection via factory functions**（工厂函数注入）而非 class-based DI container
- 明确禁止跨层调用：CLI 不能直接访问 Storage

### Section 4: Code Standards

**思考过程**：这是最常被 Claude 参照的一节。每一条都必须是"可以用 grep 验证"的硬性规则。

关键决策：
- `strict: true`——不容协商
- Result pattern 替代 throw——这会根本性地改变 Claude 生成的错误处理代码
- 禁止 class-based OOP——函数式风格更适合小型 CLI 工具

### Section 5: Testing

**思考过程**：TDD 是 SDD 的核心实践之一。如果 constitution 不强制 "test first"，Claude 会默认先写实现。

关键决策：
- 80%+ coverage 不是建议，是硬性要求
- Test files co-located（与源文件同目录）而非集中在 `tests/` 目录
- 明确 TDD 工作流步骤

### Section 6: Security Boundaries

**思考过程**：即使是本地 CLI 工具，安全边界也不可缺少。密码处理、输入验证是最低要求。

关键决策：
- 禁止 `eval()`——防止 plugin hook 中的代码注入
- 所有 CLI input 必须验证——Commander.js 不会自动做这件事
- 密码使用单向哈希存储

### Section 7: SDD Workflow

**思考过程**：这一节是"元规则"——它定义了如何使用 SDD 来开发这个项目本身。

关键决策：
- 五阶段必须顺序执行，phase gate review 不可跳过
- 每个 feature 必须有 spec 文件才能开始编码
- 验证环节必须对照 spec requirements 逐条检查

---

## 完整 Constitution 文件

以下是 `project/CLAUDE.md` 的完整内容：

```markdown
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
```

---

## 逐节深度解析

### Project Identity 为什么不写更多？

注意我们只用了三行描述项目。这是刻意的。Constitution 中的项目描述不是 README——它的读者是 Claude Code，不是人类开发者。Claude 需要的是：

1. 项目名称（用于生成一致的命名）
2. 项目类型（影响架构选择）
3. 开发阶段（影响重构策略）

把详细的功能描述放在 specs/ 里，而不是 Constitution 里。

### 为什么 Result Pattern 如此重要？

传统 JavaScript/TypeScript 项目依赖 throw/catch 进行错误处理。但在 SDD 语境下，这有一个致命问题：**throw 是隐式的控制流转移**。当 Claude Code 生成代码时，如果它不知道某个函数可能 throw，就可能遗漏 error handling。

Result pattern 把错误变成显式的返回值类型：

```typescript
// 传统方式——Claude 可能忘记 try/catch
function findTask(id: string): Task {
  const task = db.get(id);
  if (!task) throw new Error(`Task ${id} not found`);
  return task;
}

// Result pattern——类型系统强制 Claude 处理错误
function findTask(id: string): Result<Task> {
  const task = db.get(id);
  if (!task) return { success: false, error: `Task ${id} not found` };
  return { success: true, data: task };
}
```

当 Claude Code 看到返回类型是 `Result<Task>` 时，TypeScript 的类型收窄（narrowing）机制会**强制**它在使用 `data` 之前检查 `success` 字段。这就是 "making invalid states unrepresentable" 的实践。

### Factory Functions vs Class-Based DI

我们选择 factory functions 做依赖注入：

```typescript
// Factory function pattern
function createTaskService(deps: { taskRepo: TaskRepository }) {
  return {
    createTask: (input: CreateTaskInput): Result<Task> => {
      // 使用 deps.taskRepo...
    },
    findById: (id: string): Result<Task> => {
      // 使用 deps.taskRepo...
    },
  };
}
```

这比 class-based DI 更适合小型 CLI 项目的原因：
- 无需 DI container 库（减少依赖）
- 返回值是 plain object（序列化友好、测试友好）
- 闭包天然提供 encapsulation（不需要 private 关键字）
- Claude Code 生成这种代码时错误率更低（结构更简单）

---

## Anti-Patterns: 宪法中不该出现什么

### 1. 实现细节（Implementation Details）

```markdown
<!-- 错误示范 -->
## Database Schema
CREATE TABLE tasks (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  ...
);
```

**问题**：Schema 会随着开发演进。把它放在 Constitution 意味着每次改 schema 都要改 Constitution。Schema 属于 plans/ 或 src/storage/。

### 2. 临时性决策（Transient Decisions）

```markdown
<!-- 错误示范 -->
## Current Sprint Goals
- Implement user registration by Dec 15
- Fix bug #42 before release
```

**问题**：Sprint goals 有时效性，Constitution 是永久性文件。把时间相关的信息放在 task tracking 系统中。

### 3. 过于宽泛的愿景（Vague Aspirations）

```markdown
<!-- 错误示范 -->
## Philosophy
- Write clean code
- Follow best practices
- Make it scalable
```

**问题**："Clean" 和 "best practices" 对 Claude Code 没有操作意义。它需要具体的规则，比如 "max function length: 40 lines" 或 "use Result pattern for errors"。

### 4. 重复 .eslintrc / tsconfig 的内容

```markdown
<!-- 错误示范 -->
## ESLint Rules
- no-unused-vars: error
- semi: ['error', 'always']
- ...
```

**问题**：Claude Code 会读取项目中的 `.eslintrc` 和 `tsconfig.json`。在 Constitution 中重复这些配置既冗余又容易产生不一致。Constitution 应只声明**原则**（比如 "strict TypeScript"），具体规则由配置文件承载。

---

## 验证你的 Constitution

写完 Constitution 后，用以下问题自查：

| 检查项 | 标准 |
|--------|------|
| 每条规则是否可验证？ | 能写出一个 test 或 grep 来检查违规 |
| 是否有重复？ | 与 tsconfig.json / .eslintrc 无冗余 |
| 是否太长？ | Constitution 一般在 100-200 行之间 |
| 是否包含临时信息？ | 无日期、sprint、feature flags |
| 新人 Claude session 能否据此开始工作？ | 无需额外口头解释 |

---

## Claude Code 如何使用 Constitution

当你启动一个新的 Claude Code session 时：

1. Claude 自动读取项目根目录的 `CLAUDE.md`
2. 内容被注入为 system context，优先级高于 Claude 的默认行为
3. 后续所有代码生成、重构、审查都以 Constitution 为约束
4. 如果你的指令与 Constitution 冲突，Claude 会提醒你（而非沉默违规）

这就是为什么 Constitution 的质量直接决定了项目的代码质量——它是 Claude Code 的"规则引擎"。

---

## Key Takeaways (要点回顾)

1. **Constitution 是约束性文件，不是描述性文件**——每条规则都应可验证（verifiable），而非仅仅是美好愿望
2. **Result pattern 是 SDD 项目中错误处理的首选**——它把错误从隐式控制流变成显式类型，让 Claude 不可能遗漏错误处理
3. **Factory functions 优于 class-based DI**——在小型 CLI 项目中，简单的闭包模式比 DI container 更清晰、更易生成
4. **Constitution 中不应有实现细节、临时决策或模糊愿景**——它只包含项目生命周期内的不变量
5. **Claude Code 将 Constitution 作为最高优先级的行为约束**——写好它等于给 AI agent 安装了一个可靠的"规则引擎"

---

## Next (下一章)

[Chapter 20: 编写功能规格 →](./20-specification.md)
