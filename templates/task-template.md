# Task Decomposition Template / 任务分解模板

> **用途 / Usage**: 将实施计划分解为可独立执行的原子任务。每个任务应该小到可以在一次 Claude Code 会话中完成。
> Decompose the implementation plan into independently executable atomic tasks. Each task should be small enough to complete in a single Claude Code session.
>
> **输入 / Input**: 已批准的实施计划 (plan)
> **输出 / Output**: 有序的任务列表，Claude Code 逐个执行
>
> **命名约定 / Naming**: `tasks/SPEC-XXX-tasks.md`

---

## Metadata / 元数据

| Field | Value |
|-------|-------|
| **Task List ID** | [e.g., TASKS-001] |
| **Spec Reference** | [e.g., SPEC-001] |
| **Plan Reference** | [e.g., PLAN-001] |
| **Total Tasks** | [数量 / Count] |
| **Created** | [YYYY-MM-DD] |

---

## Task Card Format / 任务卡格式

<!-- 
  每个任务使用以下格式。关键规则：
  Each task uses the format below. Key rules:
  
  1. 测试任务排在实现任务前面 / Test tasks come before implementation tasks
  2. 每个任务有明确的验收标准 / Each task has clear acceptance criteria
  3. 任务间的依赖必须显式声明 / Dependencies between tasks must be explicit
  4. 一个任务只修改一个关注点 / One task modifies only one concern
-->

---

## Task Ordering Rules / 任务排序规则

```
1. Schema/Migration 先于 Application Code
2. Types/Interfaces 先于 Implementation
3. Test Files 先于 Production Code (TDD)
4. Utility/Helper 先于 Consumer Code
5. Inner Layer 先于 Outer Layer (Domain → App → Infra → UI)
6. Independent tasks can be parallelized (标记 parallel-group)
```

---

## Status Legend / 状态图例

| Status | Meaning |
|--------|---------|
| `pending` | 未开始 / Not started |
| `in_progress` | 进行中 / In progress |
| `completed` | 已完成 / Completed |
| `blocked` | 被阻塞 / Blocked by dependency |
| `skipped` | 跳过 / Not needed |

---

## Tasks / 任务列表

### Phase 1: [阶段名称 / Phase Name]

#### TASK-001: [任务标题 / Task Title]

| Field | Value |
|-------|-------|
| **Status** | `pending` |
| **Spec Requirement** | [e.g., REQ-E01, REQ-U02] |
| **Complexity** | [S / M / L / XL] |
| **Dependencies** | [None / TASK-XXX] |
| **Parallel Group** | [e.g., group-A / None] |

**Description / 描述**:
[一段话描述任务目标 / One paragraph describing the task goal]

**Files to Create/Modify / 涉及文件**:
- `[path/to/file]` — [操作: create/modify] — [变更描述 / Change description]
- `[path/to/file]` — [操作: create/modify] — [变更描述 / Change description]

**Acceptance Criteria / 验收标准**:
- [ ] [可验证的条件 / Verifiable condition]
- [ ] [可验证的条件 / Verifiable condition]
- [ ] [可验证的条件 / Verifiable condition]

**Verification Command / 验证命令**:
```bash
[e.g., pnpm test src/module/file.test.ts]
```

---

#### TASK-002: [任务标题 / Task Title]

| Field | Value |
|-------|-------|
| **Status** | `pending` |
| **Spec Requirement** | [e.g., REQ-S01] |
| **Complexity** | [S / M / L / XL] |
| **Dependencies** | [TASK-001] |
| **Parallel Group** | [None] |

**Description / 描述**:
[...]

**Files to Create/Modify / 涉及文件**:
- `[path/to/file]` — [create/modify] — [...]

**Acceptance Criteria / 验收标准**:
- [ ] [...]
- [ ] [...]

**Verification Command / 验证命令**:
```bash
[...]
```

---

### Phase 2: [阶段名称 / Phase Name]

#### TASK-003: [Write Tests for X / 为 X 编写测试]

| Field | Value |
|-------|-------|
| **Status** | `pending` |
| **Spec Requirement** | [REQ-XXX] |
| **Complexity** | [M] |
| **Dependencies** | [TASK-001, TASK-002] |
| **Parallel Group** | [group-B] |

**Description / 描述**:
[编写测试用例，覆盖核心行为和边界情况 / Write test cases covering core behavior and edge cases]

**Files to Create/Modify / 涉及文件**:
- `tests/[module]/[file].test.ts` — create — [测试内容 / What it tests]

**Acceptance Criteria / 验收标准**:
- [ ] Tests exist and compile (may fail — this is RED phase)
- [ ] Tests cover happy path scenarios
- [ ] Tests cover error/edge cases from spec
- [ ] Test names describe behavior clearly

**Verification Command / 验证命令**:
```bash
# Tests should compile but may fail (RED phase)
pnpm tsc --noEmit
```

---

#### TASK-004: [Implement X / 实现 X]

| Field | Value |
|-------|-------|
| **Status** | `pending` |
| **Spec Requirement** | [REQ-XXX] |
| **Complexity** | [L] |
| **Dependencies** | [TASK-003] |
| **Parallel Group** | [None] |

**Description / 描述**:
[实现功能使测试通过 / Implement feature to make tests pass (GREEN phase)]

**Files to Create/Modify / 涉及文件**:
- `src/[module]/[file].ts` — create — [功能描述 / Feature description]

**Acceptance Criteria / 验收标准**:
- [ ] All tests from TASK-003 pass
- [ ] No type errors
- [ ] Follows architecture conventions from CLAUDE.md
- [ ] Handles error cases explicitly

**Verification Command / 验证命令**:
```bash
pnpm test src/[module]/
pnpm tsc --noEmit
```

---

## Example: Typical Feature Task Breakdown / 示例：典型功能任务分解

<!-- 
  以下是一个"用户注册"功能的典型任务分解示例，展示正确的排序和粒度。
  Below is a typical task breakdown for a "user registration" feature showing correct ordering and granularity.
-->

```
Phase 1: Data Layer / 数据层
  TASK-001: [S] Create User type definitions and validation schema
  TASK-002: [M] Write database migration for users table
  TASK-003: [S] Create User repository interface

Phase 2: Business Logic / 业务逻辑 (TDD)
  TASK-004: [M] Write tests for registration service (RED)
  TASK-005: [L] Implement registration service (GREEN)
  TASK-006: [S] Refactor registration service (REFACTOR)

Phase 3: API Layer / API 层 (TDD)
  TASK-007: [M] Write tests for POST /api/auth/register endpoint (RED)
  TASK-008: [M] Implement registration endpoint (GREEN)
  TASK-009: [S] Add input validation middleware

Phase 4: Integration / 集成
  TASK-010: [M] Write integration tests (database + API)
  TASK-011: [S] Add rate limiting to registration endpoint
  TASK-012: [M] Write E2E test for registration flow

Phase 5: Polish / 完善
  TASK-013: [S] Add API documentation
  TASK-014: [S] Verify coverage >= 80%
  TASK-015: [S] Run security checklist
```

---

## Progress Summary / 进度摘要

<!-- 在执行过程中更新此表 / Update this table during execution -->

| Phase | Total | Completed | Blocked | Remaining |
|-------|-------|-----------|---------|-----------|
| Phase 1 | [n] | [n] | [n] | [n] |
| Phase 2 | [n] | [n] | [n] | [n] |
| Phase 3 | [n] | [n] | [n] | [n] |
| Phase 4 | [n] | [n] | [n] | [n] |
| **Total** | **[n]** | **[n]** | **[n]** | **[n]** |

---

## Blocked Tasks / 阻塞记录

<!-- 记录被阻塞的任务和原因 / Record blocked tasks and reasons -->

| Task | Blocked By | Reason | Resolution |
|------|-----------|--------|------------|
| [TASK-XXX] | [TASK-YYY / External] | [原因/Reason] | [解决方案/Resolution] |

---

## Completion Criteria / 完成标准

<!-- 所有任务完成后的最终检查 / Final checks after all tasks are done -->

- [ ] All tasks marked `completed` or `skipped` (with justification)
- [ ] All tests passing: `[test command]`
- [ ] Coverage meets target: `[coverage command]`
- [ ] Type check passes: `[type check command]`
- [ ] Lint passes: `[lint command]`
- [ ] All spec requirements traced to at least one task
- [ ] No `pending` or `in_progress` tasks remain
