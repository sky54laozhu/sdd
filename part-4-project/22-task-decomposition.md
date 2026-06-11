# Chapter 22: 任务分解 (Task Decomposition)

> 将架构计划翻译成一份有序、可追溯、可独立执行的原子任务列表——这是从"想清楚"到"做出来"的桥梁。

---

## 从 Plan 到 Tasks：为什么需要这一步？

在前面几章中，我们已经为 SpecTask 项目完成了三件事：

1. **Constitution**（第 19 章）：项目宪法 CLAUDE.md 确立了技术栈、分层架构和编码约定
2. **Spec**（第 20-21 章）：用 EARS 表示法定义了 Task CRUD 和 Auth 两个 feature 的精确需求
3. **Plan**（第 21 章尾部）：选定了 Commander.js + better-sqlite3 + Vitest 的技术方案，规划了分层文件结构

但 Plan 仍然是一份"方案文档"——它描述了**怎么做**，却没有告诉 Claude Code **具体做哪一步**。这就像建筑师画完了图纸，但施工队需要的是一份施工日程表：先挖地基、再浇筑梁柱、然后砌墙、最后装修。

Task Decomposition（任务分解）就是这份施工日程表。它将宏观计划拆解为**原子任务（atomic tasks）**——每个任务小到可以在一次 Claude Code 会话中完成，明确到任何人（或任何 AI agent）读了描述就知道该做什么。

---

## 任务分解四原则

好的任务分解必须满足四个特性：

### 1. 可追溯性 (Traceability)

每个 task 必须能追溯到一个或多个 spec requirement。如果某个 task 找不到对应的 spec 条目，要么 spec 有遗漏（需补充），要么 task 是多余的（需删除）。

```
Task → Spec Requirement → User Story → Business Goal
```

这条链不能断。断了就意味着你在做没有规格依据的事情——这正是 SDD 要消灭的问题。

### 2. 测试先行 (Test-First Ordering)

遵循 TDD 的 RED → GREEN → REFACTOR 节奏：

- **RED 任务**：编写测试（此时实现不存在，测试应该 fail 或无法 compile）
- **GREEN 任务**：编写实现使测试通过
- **REFACTOR 任务**：可选，清理代码但不改变行为

在任务列表中，测试任务永远排在对应实现任务的前面。

### 3. 原子性 (Atomicity)

一个 task 只关注一个关注点：

- **好的**："实现 TaskRepository 的 CRUD 方法"
- **坏的**："实现 TaskRepository 和 TaskService 并连接 CLI"

原子性的判断标准：**一个 task 的 acceptance criteria 应该可以用一条 test 命令验证**。

### 4. 可验证性 (Verifiability)

每个 task 有明确的 Done Criteria（完成标准），通常表现为：

- 一条 bash 命令可以运行并得到 pass/fail 结果
- 或者一个可以目视检查的输出

模糊的完成标准如"代码写好了"是不可接受的。必须是"运行 `pnpm test src/repositories/task.test.ts` 全部通过"这样具体的判定。

---

## 任务排序策略

任务之间存在依赖关系。正确的排序遵循**由内而外、由底而上**的原则：

```
Infrastructure → Types → Storage → Tests → Implementation → Integration → Verification
(基础设施)    (类型)   (存储层)   (测试)   (业务实现)      (集成)       (验证)
```

具体到 SpecTask 项目的分层架构：

```
┌─────────────────────────────────────────────┐
│  Phase 1: Scaffold                          │  ← 项目骨架，没有它其他都跑不起来
├─────────────────────────────────────────────┤
│  Phase 2: Types & Storage                   │  ← 最内层，无外部依赖
├─────────────────────────────────────────────┤
│  Phase 3: Repository Layer (TDD)            │  ← 依赖 Types + Storage
├─────────────────────────────────────────────┤
│  Phase 4: Service Layer (TDD)               │  ← 依赖 Repository
├─────────────────────────────────────────────┤
│  Phase 5: CLI Layer                         │  ← 依赖 Service
├─────────────────────────────────────────────┤
│  Phase 6: Integration & Verification        │  ← 依赖所有层
└─────────────────────────────────────────────┘
```

---

## SpecTask v1 完整任务列表

以下是 SpecTask 项目的完整 17 个任务，分为 6 个 phase：

### Phase 1: 项目骨架 (Scaffold)

#### TASK-001: 项目初始化

| Field | Value |
|-------|-------|
| **Spec Requirement** | Infrastructure (implicit) |
| **Complexity** | S |
| **Dependencies** | None |

**描述**：创建 package.json、tsconfig.json、vitest.config.ts，安装核心依赖（typescript、vitest、commander、better-sqlite3、chalk、nanoid、bcrypt）。配置 scripts: build, test, dev。

**验收标准**：
- [ ] `pnpm install` 成功
- [ ] `pnpm tsc --noEmit` 无错误
- [ ] `pnpm test` 输出 "no test files found"（正常，因为还没有测试）

---

#### TASK-002: 定义 TypeScript Interfaces

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-U01 (Task schema), REQ-U05 (User schema) |
| **Complexity** | S |
| **Dependencies** | TASK-001 |

**描述**：在 `src/types/index.ts` 中定义所有核心类型：Task、User、Result<T>、TaskStatus、CreateTaskInput、UpdateTaskInput 等。

**验收标准**：
- [ ] `pnpm tsc --noEmit` 通过
- [ ] 所有 spec 中提到的字段都有对应类型

---

### Phase 2: 存储层 (Storage)

#### TASK-003: 实现 SQLite 存储层与 Schema Migration

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-S01 (Persistent storage) |
| **Complexity** | M |
| **Dependencies** | TASK-002 |

**描述**：在 `src/storage/database.ts` 中实现 Database 类，负责 SQLite 连接管理和 schema 初始化。创建 tasks 和 users 表。

**验收标准**：
- [ ] Database 实例化后在指定路径创建 `.db` 文件
- [ ] tasks 和 users 表结构与 spec 中定义的字段一致
- [ ] 支持 `:memory:` 模式用于测试

---

### Phase 3: Repository 层 (TDD)

#### TASK-004: 编写 TaskRepository 测试 (RED)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-U01, REQ-U02, REQ-U03, REQ-U04 |
| **Complexity** | M |
| **Dependencies** | TASK-003 |

**描述**：在 `tests/repositories/task.test.ts` 中编写测试用例，覆盖：create、findAll、findById、update、delete 操作。使用 `:memory:` 数据库。

**验收标准**：
- [ ] 测试文件 compile 通过 (`pnpm tsc --noEmit`)
- [ ] 运行测试失败（RED phase — 实现尚不存在）
- [ ] 覆盖 happy path 和 error cases

---

#### TASK-005: 实现 TaskRepository (GREEN)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-U01, REQ-U02, REQ-U03, REQ-U04 |
| **Complexity** | L |
| **Dependencies** | TASK-004 |

**描述**：在 `src/repositories/task-repository.ts` 中实现 TaskRepository 类，使测试全部通过。

**验收标准**：
- [ ] `pnpm test tests/repositories/task.test.ts` 全部通过
- [ ] 无 type error

---

#### TASK-006: 编写 UserRepository 测试 (RED)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-U05, REQ-U06 |
| **Complexity** | M |
| **Dependencies** | TASK-003 |
| **Parallel Group** | group-A (可与 TASK-004 并行) |

**描述**：在 `tests/repositories/user.test.ts` 中编写测试，覆盖：register（含密码 hash）、findByUsername、验证唯一性约束。

**验收标准**：
- [ ] 测试文件 compile 通过
- [ ] 测试 fail（RED phase）

---

#### TASK-007: 实现 UserRepository (GREEN)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-U05, REQ-U06 |
| **Complexity** | M |
| **Dependencies** | TASK-006 |

**描述**：在 `src/repositories/user-repository.ts` 中实现 UserRepository 类。

**验收标准**：
- [ ] `pnpm test tests/repositories/user.test.ts` 全部通过

---

### Phase 4: Service 层 (TDD)

#### TASK-008: 编写 TaskService 测试 (RED)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-E01 through REQ-E04 |
| **Complexity** | M |
| **Dependencies** | TASK-005 |

**描述**：在 `tests/services/task.test.ts` 中测试业务逻辑：创建任务时自动生成 ID、更新不存在的任务返回 error、删除不存在的任务返回 error、按用户过滤等。

**验收标准**：
- [ ] 测试 compile 通过
- [ ] 测试 fail（RED phase）

---

#### TASK-009: 实现 TaskService (GREEN)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-E01 through REQ-E04 |
| **Complexity** | L |
| **Dependencies** | TASK-008 |

**描述**：在 `src/services/task-service.ts` 中实现 TaskService，封装业务规则。

**验收标准**：
- [ ] `pnpm test tests/services/task.test.ts` 全部通过

---

#### TASK-010: 编写 AuthService 测试 (RED)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-E05 through REQ-E08 |
| **Complexity** | M |
| **Dependencies** | TASK-007 |

**描述**：在 `tests/services/auth.test.ts` 中测试：注册成功、重复用户名失败、登录成功、登录密码错误、session 管理。

**验收标准**：
- [ ] 测试 compile 通过
- [ ] 测试 fail（RED phase）

---

#### TASK-011: 实现 AuthService (GREEN)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-E05 through REQ-E08 |
| **Complexity** | L |
| **Dependencies** | TASK-010 |

**描述**：在 `src/services/auth-service.ts` 中实现认证逻辑，包括 password hashing 和 session token 管理。

**验收标准**：
- [ ] `pnpm test tests/services/auth.test.ts` 全部通过

---

### Phase 5: CLI 层

#### TASK-012: 实现 Task CLI Commands

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-U01 through REQ-U04 |
| **Complexity** | L |
| **Dependencies** | TASK-009 |

**描述**：在 `src/cli/` 下实现 `task add`、`task list`、`task update`、`task delete`、`task done` 命令。使用 Commander.js，输出使用 chalk 着色。

**验收标准**：
- [ ] `spectask task add "Buy milk"` 输出确认信息
- [ ] `spectask task list` 显示任务列表
- [ ] `spectask task done <id>` 标记完成

---

#### TASK-013: 实现 Auth CLI Commands

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-U05, REQ-U06 |
| **Complexity** | M |
| **Dependencies** | TASK-011 |

**描述**：实现 `auth register`、`auth login`、`auth logout` 命令。登录状态持久化到本地 config 文件。

**验收标准**：
- [ ] `spectask auth register --username alice --password secret` 成功注册
- [ ] `spectask auth login --username alice --password secret` 成功登录
- [ ] 登录后 task 命令关联当前用户

---

#### TASK-014: 添加 Spec-linking 功能 (--spec flag)

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-O01 (Optional) |
| **Complexity** | S |
| **Dependencies** | TASK-012 |

**描述**：为 `task add` 添加 `--spec <spec-id>` 选项，允许将任务关联到 spec requirement。

**验收标准**：
- [ ] `spectask task add "Implement auth" --spec REQ-E05` 成功
- [ ] `spectask task list --spec REQ-E05` 按 spec 过滤

---

#### TASK-015: 添加 Notification 系统

| Field | Value |
|-------|-------|
| **Spec Requirement** | REQ-O02 (Optional - due date) |
| **Complexity** | M |
| **Dependencies** | TASK-012 |

**描述**：支持 `--due` 日期选项，`task list` 时高亮即将到期和已过期的任务。

**验收标准**：
- [ ] `spectask task add "Report" --due 2024-12-01` 设置截止日期
- [ ] 过期任务以红色显示

---

### Phase 6: 集成与验证

#### TASK-016: Integration Tests

| Field | Value |
|-------|-------|
| **Spec Requirement** | All REQs (cross-cutting) |
| **Complexity** | L |
| **Dependencies** | TASK-012, TASK-013 |

**描述**：在 `tests/integration/` 中编写端到端集成测试：注册 → 登录 → 创建任务 → 列出任务 → 完成任务 → 删除任务的完整流程。

**验收标准**：
- [ ] `pnpm test tests/integration/` 全部通过
- [ ] Coverage >= 80%

---

#### TASK-017: 运行 Verifier Agent

| Field | Value |
|-------|-------|
| **Spec Requirement** | All (verification) |
| **Complexity** | M |
| **Dependencies** | TASK-016 |

**描述**：调用 `@sdd-verifier` 代理，对照 spec 文件验证所有 requirement 已实现且有测试覆盖。

**验收标准**：
- [ ] Verifier 报告中无 CRITICAL 或 HIGH 级别的 GAP
- [ ] 所有 spec requirement 标记为 TRACED

---

## 依赖关系图

```
TASK-001 ──▶ TASK-002 ──▶ TASK-003 ──┬──▶ TASK-004 ──▶ TASK-005 ──▶ TASK-008 ──▶ TASK-009 ──▶ TASK-012 ──┬──▶ TASK-016 ──▶ TASK-017
                                      │                                                                     │
                                      └──▶ TASK-006 ──▶ TASK-007 ──▶ TASK-010 ──▶ TASK-011 ──▶ TASK-013 ──┘
                                                                                                    │
                                                                                             TASK-014, TASK-015
                                                                                         (parallel, depend on TASK-012)
```

注意 TASK-004/005（Task repo）和 TASK-006/007（User repo）可以并行执行——它们都只依赖 TASK-003，彼此无依赖。这是一个**并行执行机会（parallelization opportunity）**。

---

## 在 Claude Code 中创建任务

在 Claude Code 中，你可以使用自然语言指令配合 TodoWrite 工具来管理任务。以下是几个实际创建任务的示例：

### 示例 1：创建整个任务列表

```
请帮我创建 SpecTask 项目的任务列表。总共 17 个任务，按以下结构组织：

Phase 1 - Scaffold:
- TASK-001: 项目初始化 (package.json, tsconfig, vitest config)
- TASK-002: 定义 TypeScript interfaces

Phase 2 - Storage:
- TASK-003: SQLite 存储层和 schema migration

Phase 3 - Repository TDD:
- TASK-004: TaskRepository 测试 (RED)
- TASK-005: 实现 TaskRepository (GREEN)
...

每个任务标注状态为 pending，标注依赖关系。
```

### 示例 2：开始执行特定任务

```
开始执行 TASK-004。

上下文：
- Spec requirement: REQ-U01 (create task), REQ-U02 (read tasks), REQ-U03 (update task), REQ-U04 (delete task)
- 这是 RED phase：编写测试，预期测试会 fail
- 使用 :memory: SQLite 做测试数据库
- 文件：tests/repositories/task.test.ts

完成标准：测试文件 compile 通过（pnpm tsc --noEmit 无错误）
```

### 示例 3：标记完成并推进

```
TASK-004 已完成（测试文件已编写，compile 通过，确认 fail 因为实现不存在）。
现在开始 TASK-005：实现 TaskRepository 使 TASK-004 的测试全部通过。
```

---

## Phase Gate Review

在开始执行任务之前，进行一次 Phase Gate Review（阶段门审查）：

**检查清单**：

- [ ] 每个 spec requirement 都被至少一个 task 追溯
- [ ] 没有"孤儿 task"（无 spec 对应的任务）
- [ ] TDD 配对完整：每个实现 task 前面都有对应的测试 task
- [ ] 依赖关系无循环
- [ ] 每个 task 的 complexity 估算合理（S=1h, M=2-4h, L=4-8h）
- [ ] 验证 task（TASK-017）在最后

如果有 requirement 未被覆盖，**立即补充 task**。如果发现 spec 本身有缺失，**回退到 Spec 阶段**修补。SDD 允许回退，但不允许跳过。

---

## Key Takeaways (要点回顾)

1. **任务分解是 Plan 和 Implementation 之间的桥梁**——Plan 说"怎么做"，Tasks 说"具体做哪一步"
2. **四原则缺一不可**：可追溯性确保不偏离 spec、测试先行确保质量、原子性确保可管理、可验证性确保可判定完成
3. **排序遵循"由内而外"**：infrastructure → types → storage → tests → implementation → integration → verification
4. **并行机会要识别**：无依赖关系的 task 可以并行执行，加速开发节奏
5. **Phase Gate Review 是安全网**：在动手之前最后一次检查任务是否完整覆盖了所有 spec requirement

---

## Next (下一章)

[Chapter 23: 用 Claude Code 实现](23-implementation.md) —— 进入 Implementation 阶段，逐个执行任务，展示 TDD 循环的实际操作。
