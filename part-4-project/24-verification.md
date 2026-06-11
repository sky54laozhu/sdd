# Chapter 24: 验证与质量保证 (Verification and Quality Assurance)

> 实现者不能可靠地验证自己的工作——你需要一个独立的 Verifier Agent 来对照 Spec 逐条检查实现的完整性和正确性。

---

## 为什么需要独立验证？

你可能会想："我每个 task 都跑了测试，全绿了，还不够吗？"

不够。原因有三：

1. **测试可能遗漏了 spec requirement**：你写了 7 个测试覆盖了 REQ-U01 到 REQ-U04，但 REQ-E07（"密码错误次数超过 5 次锁定账户"）有没有测试？单靠 task 粒度很难发现这种遗漏。

2. **实现可能满足了测试但偏离了 spec**：测试说"返回 error"，实现确实返回了 error，但 spec 说的是"返回 error 并记录 audit log"——测试通过了，spec 没有被完全满足。

3. **AI 的确认偏差（confirmation bias）**：同一个 agent 既写测试又写实现，它天然倾向于让两者一致——但不一定与 spec 一致。

解决方案是**关注点分离**（Separation of Concerns）应用到验证过程：**实现者和验证者必须是不同的 agent**。

---

## Verifier Agent 模式

Verifier Agent 是一个**只读代理（read-only agent）**——它没有写权限，只能读取文件和运行测试命令。它的职责是：

```
读取 Spec → 读取实现代码 → 读取测试代码 → 对照检查 → 生成报告
```

### 核心设计原则

| 原则 | 说明 |
|------|------|
| **只读** | 验证者不修改代码，只报告问题 |
| **独立** | 不依赖实现者的判断，直接对照 spec 原文 |
| **结构化输出** | 生成可操作的报告，而非模糊的"看起来不错" |
| **分级严重性** | 区分 CRITICAL（必须修复）和 LOW（建议改进） |

---

## 定义 Verifier Agent

在项目中创建 `.claude/agents/verifier.md` 文件。以下是完整的、可直接复制使用的 agent 定义：

```markdown
---
name: sdd-verifier
description: |
  Verifies SpecTask implementation against spec files.
  Checks requirement coverage, test adequacy, and architectural compliance.
  对照 spec 文件验证 SpecTask 实现的正确性和完整性。
model: sonnet
tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# SDD Verifier Agent

You are a verification agent. Your job is to check whether the implementation
correctly and completely satisfies the spec requirements.

## Protocol

### Step 1: Load Artifacts

1. Read all spec files from `specs/` directory
2. Read all source files from `src/` directory
3. Read all test files from `tests/` directory
4. Read the task list from `tasks/` if available

### Step 2: Build Traceability Matrix

For each requirement (REQ-*) in the spec files:
1. Search for implementation code that satisfies it
2. Search for test code that verifies it
3. Record the file and line number for both
4. Mark as TRACED (both found), PARTIAL (only one found), or GAP (neither found)

### Step 3: Verify Acceptance Criteria

For each acceptance criterion in the spec:
1. Find the corresponding test assertion
2. Verify the assertion checks the correct behavior
3. If possible, run the test and confirm it passes

### Step 4: Check Code Quality

Verify against project CLAUDE.md:
1. No file exceeds 400 lines
2. Naming conventions followed (camelCase functions, PascalCase types)
3. Layer boundaries respected (CLI → Service → Repository → Storage)
4. No direct database access from Service or CLI layers
5. All functions return Result<T> instead of throwing
6. No hardcoded secrets

### Step 5: Test Adequacy

1. Run `pnpm test --coverage` and check coverage percentage
2. Verify each requirement has at least one test
3. Check for error-case tests (not just happy path)
4. Detect assertion-free tests (test exists but doesn't assert anything meaningful)

### Step 6: Generate Report

Output the verification report in the format specified below.

## Report Format

```
# SDD Verification Report

**Project**: SpecTask
**Date**: [current date]
**Verdict**: [PASS / FAIL / PARTIAL]

## Requirement Traceability

| Req ID | Description | Implementation | Test | Status |
|--------|-------------|----------------|------|--------|
| REQ-U01 | ... | src/file:L## | tests/file:L## | TRACED |
| REQ-E07 | ... | — | — | GAP |

## Test Adequacy

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Line coverage | 80% | ??% | PASS/FAIL |
| Branch coverage | 70% | ??% | PASS/FAIL |
| Requirements with tests | 100% | ??% | PASS/FAIL |

## Code Quality

| Check | Status | Details |
|-------|--------|---------|
| File lengths | PASS/FAIL | ... |
| Layer violations | PASS/FAIL | ... |
| Error handling | PASS/FAIL | ... |

## Issues Found

### CRITICAL
- [blocking issues]

### HIGH
- [should fix before merge]

### MEDIUM
- [consider fixing]

### LOW
- [optional improvements]

## Verdict Rationale
[Why PASS/FAIL/PARTIAL]
```

## Severity Rules

- **CRITICAL**: Any spec requirement with no implementation (GAP in traceability)
- **HIGH**: Acceptance criterion not covered by test, or test fails
- **MEDIUM**: Coverage below target, or missing edge-case tests
- **LOW**: Style/naming deviations, minor convention issues

## Auto-Fail Conditions

Verdict is automatically FAIL if ANY of:
- A requirement has GAP status (no implementation at all)
- Line coverage is below 80%
- Any CRITICAL issue exists
- Layer violations exist (architecture integrity)
```

将这个文件放在 `project/.claude/agents/verifier.md`。

---

## 运行验证

### 启动 Verifier Agent

在 Claude Code 中调用：

```
@sdd-verifier 请验证 SpecTask 实现是否满足 specs/ 目录下的所有 spec requirement。
生成完整的验证报告。
```

### 验证过程示例

Verifier 会按照协议逐步执行。以下是一次典型验证的输出片段：

```markdown
# SDD Verification Report

**Project**: SpecTask
**Date**: 2024-12-15
**Verdict**: PARTIAL

## Requirement Traceability

| Req ID | Description | Implementation | Test | Status |
|--------|-------------|----------------|------|--------|
| REQ-U01 | Create task with auto-generated ID | src/repositories/task-repository.ts:L12-30 | tests/repositories/task.test.ts:L15 | TRACED |
| REQ-U02 | List tasks by user | src/repositories/task-repository.ts:L32-40 | tests/repositories/task.test.ts:L45 | TRACED |
| REQ-U03 | Update task fields | src/repositories/task-repository.ts:L42-70 | tests/repositories/task.test.ts:L60 | TRACED |
| REQ-U04 | Delete task, error on non-existent | src/repositories/task-repository.ts:L72-85 | tests/repositories/task.test.ts:L82 | TRACED |
| REQ-E05 | Register with unique username | src/services/auth-service.ts:L15-35 | tests/services/auth.test.ts:L12 | TRACED |
| REQ-E06 | Login with correct password | src/services/auth-service.ts:L37-55 | tests/services/auth.test.ts:L30 | TRACED |
| REQ-E07 | Lock after 5 failed attempts | — | — | GAP |
| REQ-E08 | Logout invalidates session | src/services/auth-service.ts:L57-65 | tests/services/auth.test.ts:L50 | TRACED |
| REQ-O01 | Spec-link feature | src/cli/commands/task.ts:L25 | — | PARTIAL |
| REQ-O02 | Due date notification | src/cli/commands/task.ts:L55 | tests/integration/due-date.test.ts:L8 | TRACED |

## Test Adequacy

| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Line coverage | 80% | 83% | PASS |
| Branch coverage | 70% | 72% | PASS |
| Requirements with tests | 100% | 80% | FAIL |

## Issues Found

### CRITICAL
1. **REQ-E07 not implemented**: "If login fails 5 consecutive times, the system shall lock the account for 15 minutes." — No implementation or test found.

### HIGH
2. **REQ-O01 partial**: Spec-link feature implemented in CLI but has no dedicated test verifying the `--spec` flag behavior.

### MEDIUM
3. Missing edge-case test: What happens when `task list` is called with both `--status` and `--spec` filters simultaneously?

### LOW
4. `src/utils/session.ts` uses `any` type on line 12 — should be typed as `Session | null`.

## Verdict Rationale

PARTIAL: One CRITICAL gap (REQ-E07 account locking) prevents full PASS.
The spec-link feature needs test coverage to move from PARTIAL to TRACED.
Overall architecture is sound and coverage target is met.
```

---

## 处理验证失败

验证报告中的每个 issue 都有对应的处理策略：

### CRITICAL Issues → 立即修复

发现 REQ-E07 是 GAP（完全未实现），处理步骤：

1. **补充 task**：创建新的 TASK-018（写测试）和 TASK-019（写实现）
2. **执行 TDD 循环**：照常 RED → GREEN
3. **重新验证**：修复后再次运行 verifier

```
发现 CRITICAL gap: REQ-E07 (account locking) 未实现。
请创建两个新任务：
- TASK-018: 编写 account locking 测试 (RED)
- TASK-019: 实现 account locking (GREEN)

先执行 TASK-018。
```

### HIGH Issues → 应当修复

REQ-O01 有实现但缺测试：

```
REQ-O01 (spec-link) 已实现但缺少测试。
请为 --spec flag 编写测试：
1. spectask task add "x" --spec REQ-U01 应成功创建并关联
2. spectask task list --spec REQ-U01 应只返回关联的任务
3. --spec 值为空时应忽略该字段
```

### MEDIUM Issues → 酌情修复

缺少边界条件测试：

```
请为 task list 的组合过滤（--status + --spec 同时使用）补充测试。
这是 edge case，但可能暴露 SQL 查询拼接的问题。
```

### LOW Issues → 可选修复

类型不够严格：

```
src/utils/session.ts 第 12 行使用了 any 类型。
请改为 Session | null，并确保调用方正确处理 null case。
```

---

## 测试覆盖率检查

Verifier 的一个关键职责是检查覆盖率。在 SpecTask 项目中：

```bash
pnpm test --coverage
```

Vitest 的覆盖率报告会显示：

```
----------|---------|----------|---------|---------|
File      | % Stmts | % Branch | % Funcs | % Lines |
----------|---------|----------|---------|---------|
All files |   83.2  |   72.1   |   88.0  |   83.5  |
 repos/   |   95.0  |   90.0   |  100.0  |   95.0  |
 services/|   85.0  |   75.0   |   90.0  |   85.0  |
 cli/     |   65.0  |   50.0   |   70.0  |   65.0  |
 storage/ |   90.0  |   80.0   |  100.0  |   90.0  |
----------|---------|----------|---------|---------|
```

注意 CLI 层的覆盖率较低（65%）——这是正常的。CLI 层主要是胶水代码，真正的业务逻辑在 Service 层。如果整体覆盖率达到 80%，且核心层（Repository + Service）超过 85%，这是可接受的。

---

## Final Build Verification

在 verifier 报告 PASS 之后，执行最终构建验证：

```bash
# Type check — 无 type error
pnpm tsc --noEmit

# Lint — 无 lint error（如果配置了 ESLint）
pnpm lint

# All tests pass
pnpm test

# Coverage meets target
pnpm test --coverage

# Build succeeds
pnpm build
```

所有命令都必须通过，零 error。Warning 可以存在但应该尽量消除。

---

## 验证的 ROI

你可能觉得"跑一个 verifier 太重了"。但看数据：

| 不使用 Verifier | 使用 Verifier |
|----------------|---------------|
| 实现完成后直接交付 | 实现完成后验证 15-20 min |
| 后期发现 spec 遗漏时返工 2-4 hours | 验证时发现并立即修复 30-60 min |
| PR review 中被指出 "这个 requirement 没实现" | PR review 干净通过 |
| 上线后用户发现缺失功能 | 上线前 100% requirement coverage |

**Verifier 的时间投入是 20 分钟。它避免的返工是 2-4 小时。ROI 约 6-12x。**

尤其在多人协作中，Verifier 报告可以直接附在 PR description 里，作为"我确认实现了所有 spec requirement"的证据。Reviewer 不需要逐行对照 spec——verifier 已经做了这件事。

---

## 完整 Verifier Agent 文件

为方便直接使用，以下是完整的 verifier agent 文件内容。将其保存为 `project/.claude/agents/verifier.md`：

```markdown
---
name: sdd-verifier
description: |
  Verifies implementation against SDD spec files.
  Checks requirement coverage, test adequacy, and architectural compliance.
model: sonnet
tools:
  - Read
  - Glob
  - Grep
  - Bash
---

# SDD Verifier Agent

You are a verification agent for Spec-Driven Development projects. Your role
is strictly read-only: you examine source code, tests, and specs, then produce
a structured verification report.

## Input

You will be given one or more spec files to verify against.

## Protocol

1. **Load**: Read spec files, source files, test files
2. **Trace**: Map each REQ-* to implementation + test
3. **Verify**: Check acceptance criteria have matching assertions
4. **Quality**: Check CLAUDE.md conventions (file size, layers, naming)
5. **Coverage**: Run `pnpm test --coverage` and compare to target
6. **Report**: Output structured report with severity classifications

## Severity

- CRITICAL: Requirement not implemented at all
- HIGH: Acceptance criterion untested or test fails
- MEDIUM: Coverage gap or missing edge-case test
- LOW: Style deviation or minor type issue

## Auto-Fail

Verdict = FAIL if any CRITICAL issue exists or coverage < 80%.

## Output Format

Use the markdown table format with Requirement Traceability, Test Adequacy,
Code Quality, Issues Found (by severity), and Verdict Rationale sections.
```

---

## Key Takeaways (要点回顾)

1. **实现者和验证者必须分离** —— 同一个 agent 不能既写代码又判定自己的代码是否正确
2. **Verifier 是只读的** —— 它只报告问题，不修复问题；修复是 implementer 的职责
3. **Traceability matrix 是核心产出** —— 每个 spec requirement 必须能追溯到实现代码和测试代码
4. **CRITICAL issue 是 blocker** —— 任何 GAP（requirement 完全未实现）都意味着验证 FAIL
5. **20 分钟的验证可以避免数小时的返工** —— 尤其在 PR review 和上线前

---

## Next (下一章)

[Chapter 25: 回顾与输出](25-retrospective.md) —— 项目完成后的复盘：什么做得好、什么可以改进、如何将 SDD 工作流编码为可复用的 Skill。
