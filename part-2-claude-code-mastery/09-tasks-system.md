# 任务系统 (Task System for SDD)

> Spec 定义 WHAT，Plan 定义 HOW，Task 是最小可执行单元——Task System 将蓝图变为一步步可追踪的行动。

---

## 为什么 Task System 重要

在 SDD 工作流中，从 Spec 到最终代码之间存在一个关键转化：

```
Spec (做什么) → Plan (怎么做) → Tasks (具体执行步骤)
```

没有 Task System，你面临两个问题：

1. **粒度过粗**：Plan 说"实现用户认证模块"，但 Claude Code 需要知道先做什么、后做什么、每步具体交付什么
2. **进度不可见**：一个长时间运行的实现过程中，你不知道完成了多少、还剩多少、是否卡住了

Task System 将模糊的"实现计划"转化为**有序的、可追踪的、原子化的执行单元**。每个 Task 都是一个独立的 checkpoint（检查点）——完成即可验证，失败即可定位。

---

## Claude Code 的 Task API

Claude Code 提供了内置的 TodoWrite 工具来管理任务列表。它的核心操作：

### 创建任务列表

```typescript
// Claude Code 内部使用 TodoWrite 来管理任务
// 你通过自然语言指令来驱动它

// 示例指令：
"请根据 specs/auth-login.md 创建一个实现任务列表"
```

Claude Code 会生成结构化的任务列表，每个任务包含：

| 字段 | 说明 | 示例 |
|------|------|------|
| `id` | 任务唯一标识 | `task-001` |
| `subject` | 简短祈使句标题 | "Create User schema with Drizzle ORM" |
| `status` | 当前状态 | `pending` / `in_progress` / `completed` |
| `description` | 详细说明 | 具体要做什么、验证标准 |

### Task 生命周期

```
pending → in_progress → completed
   │                       ▲
   │                       │
   └──── (blocked) ────────┘
         等待依赖完成
```

每个 Task 在执行时遵循严格的状态转换：

- **pending**：等待执行（可能被其他 Task 阻塞）
- **in_progress**：正在执行中
- **completed**：已完成并通过验证

---

## Task 结构设计

### 一个好的 Task 长什么样

```markdown
## Task: Create authentication service

**Subject**: Implement AuthService with login and token generation

**Description**:
Create `src/auth/auth-service.ts` implementing:
- `login(email, password)` → validates credentials, returns JWT pair
- `verifyToken(token)` → validates JWT and returns decoded payload
- Uses bcrypt for password verification (cost factor 12)
- JWT signed with RS256, access token expires in 24h, refresh in 7d

**Acceptance Criteria**:
- AuthService.login returns Result<TokenPair, AuthError>
- AuthService.verifyToken returns Result<UserPayload, TokenError>
- All methods are pure (no side effects beyond arguments)
- Unit tests achieve 100% branch coverage for this file

**Blocks**: task-005 (integration tests depend on this service)
**Blocked By**: task-002 (needs User type definition)
```

### Subject 的写作规范

Task subject 应当是**简短的祈使句**（imperative mood）：

```markdown
# 好的 subject
- Create user repository with CRUD operations
- Add input validation to login endpoint
- Write integration tests for auth flow
- Configure JWT signing with RS256 keys

# 不好的 subject
- User repository (太模糊)
- The authentication system should validate users (不是祈使句)
- Maybe we need some tests for login (不确定性)
- Create user repository, add validation, write tests (多个关注点)
```

### 依赖关系 (Dependencies)

Tasks 之间通过 `blocks` / `blockedBy` 表达顺序依赖：

```markdown
task-001: Define TypeScript types (User, AuthToken, etc.)
task-002: Create database schema (Drizzle)
  blockedBy: task-001
task-003: Write unit tests for auth-service
  blockedBy: task-001
task-004: Implement auth-service
  blockedBy: task-001, task-003
task-005: Write integration tests
  blockedBy: task-002, task-004
task-006: Implement auth routes
  blockedBy: task-004
task-007: Write E2E tests
  blockedBy: task-005, task-006
```

可视化依赖图：

```
task-001 (types)
  ├─→ task-002 (schema)
  │       └─→ task-005 (integration tests)
  ├─→ task-003 (unit tests)
  │       └─→ task-004 (implementation)
  │               ├─→ task-005 (integration tests)
  │               └─→ task-006 (routes)
  │                       └─→ task-007 (E2E tests)
  └─→ task-004 (implementation)
```

---

## SDD 任务分解规则

将 Spec 转化为 Tasks 时，遵循以下五条规则：

### 规则一：每个 Task 追溯到 Spec 需求

```markdown
# 好：明确关联
task-004: Implement account lockout after 5 failed attempts
  traces: REQ-041

# 坏：来源不明
task-004: Add some security stuff
```

每个 Task 必须能回答"这满足了 Spec 中的哪条需求？"。如果一个 Task 追溯不到任何 REQ，它要么是不必要的，要么说明 Spec 遗漏了需求。

### 规则二：测试 Task 在实现 Task 之前

这是 TDD（Test-Driven Development）在 Task 层面的体现：

```markdown
# 正确顺序
task-003: Write unit tests for AuthService.login (RED)
task-004: Implement AuthService.login (GREEN)
task-005: Refactor AuthService for readability (REFACTOR)

# 错误顺序
task-003: Implement AuthService.login
task-004: Write tests for AuthService.login  ← 失去了 TDD 价值
```

测试先行的 Task 顺序确保：
- 实现有明确的"完成"标准（测试通过）
- 不会遗漏测试（测试是 Task 而非事后补充）
- Claude Code 可以用测试结果验证自己的实现

### 规则三：每个 Task 只有一个关注点

```markdown
# 好：单一关注点
task-006: Add input validation for POST /auth/login
task-007: Add rate limiting middleware to auth routes

# 坏：多个关注点混合
task-006: Add validation, rate limiting, and error handling to auth routes
```

单一关注点让 Task 更容易验证——"它做了该做的事吗？"变成了一个 yes/no 问题。

### 规则四：每个 Task 可独立验证

每个 Task 完成后，必须有一种方式验证它是否正确完成：

```markdown
task-003: Write unit tests for AuthService.login
  verify: "pnpm test src/auth/auth-service.test.ts" 应该有 5 个测试
          全部 FAIL（因为还没实现）

task-004: Implement AuthService.login
  verify: "pnpm test src/auth/auth-service.test.ts" 全部 PASS
```

### 规则五：粒度适中

太粗的 Task 等于没分解：
```markdown
# 太粗
task-001: Implement user authentication
```

太细的 Task 制造管理开销：
```markdown
# 太细
task-001: Create src/auth/ directory
task-002: Create auth-service.ts file
task-003: Add import statement for bcrypt
task-004: Define login function signature
```

**适中的粒度**：每个 Task 对应 15-60 分钟的实现工作，产出 1-3 个文件的变更。

---

## 实战示例：User Authentication 分解

基于 [Ch.07](07-writing-specs.md) 中的 User Authentication Spec，完整的 Task 分解：

```markdown
# Task List: User Authentication (Email/Password)
# Spec: specs/auth-login.md

## Phase 1: Foundation (基础设施)

### task-001: Define core types
- subject: Define TypeScript types for auth domain
- description: Create `src/auth/types.ts` with:
  - `LoginInput` (email, password)
  - `TokenPair` (accessToken, refreshToken)
  - `UserPayload` (decoded JWT payload)
  - `AuthError` enum (INVALID_CREDENTIALS, ACCOUNT_LOCKED, etc.)
- traces: REQ-001, REQ-002, REQ-003
- verify: `pnpm tsc --noEmit` passes

### task-002: Create database schema
- subject: Add auth-related columns to user table schema
- description: In Drizzle schema, ensure user table has:
  - `passwordHash` (text, not null)
  - `failedAttempts` (integer, default 0)
  - `lockedUntil` (timestamp, nullable)
  Generate and apply migration.
- blockedBy: task-001
- traces: REQ-001, REQ-041
- verify: Migration runs successfully, `pnpm drizzle-kit push` clean

## Phase 2: Tests First (测试先行)

### task-003: Write unit tests for password hashing
- subject: Write tests for bcrypt hash/verify utilities
- description: Test cases:
  - Hash generates valid bcrypt string (cost 12)
  - Verify returns true for matching password
  - Verify returns false for wrong password
  - Hash never equals plaintext
- traces: REQ-001
- verify: `pnpm test src/auth/password.test.ts` — 4 tests, all FAIL

### task-004: Write unit tests for auth service
- subject: Write tests for AuthService.login behavior
- description: Test cases:
  - Valid credentials → returns TokenPair
  - Invalid password → returns INVALID_CREDENTIALS error
  - Non-existent email → returns INVALID_CREDENTIALS error (same as invalid password)
  - Locked account → returns ACCOUNT_LOCKED error with retryAfter
  - 5th failed attempt → locks account
  - Successful login → resets failure counter
- traces: REQ-010, REQ-011, REQ-020, REQ-040, REQ-041
- blockedBy: task-001
- verify: `pnpm test src/auth/auth-service.test.ts` — 6 tests, all FAIL

### task-005: Write unit tests for token service
- subject: Write tests for JWT generation and verification
- description: Test cases:
  - Generate returns valid JWT with correct claims
  - Token expires after 24h
  - Verify rejects expired token
  - Verify rejects tampered token
  - Refresh token expires after 7d
- traces: REQ-003, REQ-010
- blockedBy: task-001
- verify: `pnpm test src/auth/token-service.test.ts` — 5 tests, all FAIL

## Phase 3: Implementation (实现)

### task-006: Implement password utilities
- subject: Implement bcrypt hash and verify functions
- description: Create `src/auth/password.ts`:
  - `hashPassword(plain)` → bcrypt hash with cost 12
  - `verifyPassword(plain, hash)` → boolean
- blockedBy: task-003
- traces: REQ-001
- verify: `pnpm test src/auth/password.test.ts` — all PASS

### task-007: Implement token service
- subject: Implement JWT generation and verification
- description: Create `src/auth/token-service.ts`:
  - `generateTokenPair(user)` → { accessToken, refreshToken }
  - `verifyAccessToken(token)` → Result<UserPayload, TokenError>
  - RS256 signing, keys from env vars
- blockedBy: task-005
- traces: REQ-003, REQ-043
- verify: `pnpm test src/auth/token-service.test.ts` — all PASS

### task-008: Implement auth service
- subject: Implement AuthService with login logic
- description: Create `src/auth/auth-service.ts`:
  - `login(input)` → Result<TokenPair, AuthError>
  - Orchestrates: find user → verify password → check lock → generate tokens
  - Handles failure counting and account lockout
- blockedBy: task-004, task-006, task-007
- traces: REQ-010, REQ-011, REQ-020, REQ-040, REQ-041
- verify: `pnpm test src/auth/auth-service.test.ts` — all PASS

## Phase 4: Integration (集成)

### task-009: Implement auth routes
- subject: Create POST /auth/login Fastify route
- description: Create `src/auth/routes.ts`:
  - POST /auth/login with TypeBox schema validation
  - Proper HTTP status codes (200, 400, 401, 403, 500)
  - Error response format: { "error": "error_code" }
- blockedBy: task-008
- traces: REQ-010, REQ-040, REQ-041, REQ-042
- verify: Route registers without errors

### task-010: Write integration tests
- subject: Write integration tests for auth endpoint
- description: Using supertest + test database:
  - Full login flow: register → login → use token
  - Account lockout flow: 5 failures → locked → wait → unlocked
  - Invalid input validation
- blockedBy: task-002, task-009
- traces: All REQs
- verify: `pnpm test tests/integration/auth.test.ts` — all PASS

## Phase 5: Verification (验证)

### task-011: Run spec compliance check
- subject: Verify all REQ-xxx requirements are satisfied
- description: Use verifier agent to check specs/auth-login.md against
  src/auth/ implementation. All requirements must be PASS.
- blockedBy: task-010
- verify: Verification report shows 0 FAIL, 0 NOT_FOUND
```

---

## Task 排序策略

上面的示例展示了一个通用的 Task 排序模式：

```
Infrastructure → Data Model → Tests → Implementation → Integration → Verification
(基础设施)      (数据模型)   (测试)    (实现)         (集成)        (验证)
```

这个顺序的逻辑：

1. **Infrastructure**：类型定义、schema——其他一切的基础
2. **Data Model**：数据库结构——实现需要的持久化层
3. **Tests**：在实现之前写好——TDD 的 RED 阶段
4. **Implementation**：逐个通过测试——TDD 的 GREEN 阶段
5. **Integration**：组装各部分、端到端测试
6. **Verification**：对照 Spec 做最终合规检查

### 并行机会识别

在排序时，注意同一 Phase 内的独立 Tasks 可以并行：

```markdown
# Phase 2 中可并行的 Tasks：
task-003 (password tests)  ─┐
task-004 (auth service tests) ─┼─ 并行执行
task-005 (token tests)     ─┘

# Phase 3 中的依赖关系：
task-006 (password impl) ─┐
task-007 (token impl)    ─┼─ 可并行
                          │
task-008 (auth service)  ─┘ 必须等 006 和 007 完成
```

---

## 与 Phase Gate 的集成

在 SDD 工作流中，Task List 是 **Plan 阶段的最终产物**，也是进入 Implementation 阶段的 Phase Gate（阶段门控）：

```
Spec ──→ Plan (高层设计) ──→ Task List (原子任务) ──→ Implementation
                                    ▲
                                    │
                              Phase Gate
                         "Task List 是否完整？
                          每个 Task 是否可追溯到 Spec？
                          依赖关系是否正确？
                          TDD 顺序是否满足？"
```

通过 Phase Gate 的检查清单：

- [ ] 每个 Spec 中的 REQ 都至少被一个 Task 覆盖
- [ ] 没有孤立 Task（无法追溯到任何 REQ）
- [ ] 测试 Task 排在对应实现 Task 之前
- [ ] 依赖关系形成 DAG（有向无环图），无循环依赖
- [ ] 每个 Task 的 verify 字段有具体可执行的验证方式
- [ ] Task 粒度适中（15-60 分钟估算）

---

## 生成 Task List 的实践方法

在 Claude Code 中，你可以用以下 prompt 从 Spec 生成 Task List：

```
请读取 specs/auth-login.md，并生成一个完整的 Task List：

规则：
1. 每个 Task 必须用 traces 字段追溯到 Spec 中的 REQ 编号
2. 测试 Tasks 排在实现 Tasks 之前（TDD）
3. 标注 blockedBy 依赖关系
4. 每个 Task 包含 verify 字段说明如何验证完成
5. 按 Phase 分组：Foundation → Tests → Implementation → Integration → Verification
6. 粒度：每个 Task 对应 1-3 个文件的变更

输出格式使用 Markdown，与 tasks.md 模板一致。
```

Claude Code 生成 Task List 后，你应当审查：
- 是否有遗漏的 REQ（Spec 中的需求没被任何 Task 覆盖）
- 是否有多余的 Task（做了 Spec 没要求的事）
- 依赖顺序是否合理
- TDD 顺序是否正确

审查通过后，Task List 即成为 Implementation 阶段的执行清单。

---

## Key Takeaways (要点回顾)

- Task 是 SDD 中最小的可执行和可追踪单元，将 Plan 转化为具体行动
- 每个 Task 必须满足四个条件：追溯到 Spec、单一关注点、可独立验证、粒度适中
- TDD 集成要求测试 Task 排在实现 Task 之前，确保先有验证标准再有实现
- Task 排序遵循：Infrastructure → Data Model → Tests → Implementation → Integration → Verification
- Task List 是 Plan 阶段的最终产物，通过 Phase Gate 检查后才进入 Implementation

---

## Next (下一章)

[Ch.10 Hooks 系统](10-hooks-system.md) — 学习如何用 PreToolUse/PostToolUse/Stop hooks 实现自动化质量保障，让每次代码变更都经过格式化、lint 和类型检查。
