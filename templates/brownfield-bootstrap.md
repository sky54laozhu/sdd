# Brownfield Bootstrap Guide / 存量项目 SDD 导入指南

> **用途 / Usage**: 为已有项目（brownfield）引入 SDD 工作流的分步指南。
> 不需要一次性重写所有内容——渐进式引入，只对新增和变更部分使用规格驱动。
> Step-by-step guide for introducing SDD workflow to existing projects (brownfield).
> No need to rewrite everything at once — introduce incrementally, using spec-driven only for new additions and changes.
>
> **核心原则 / Core Principle**: Spec the delta, not the world.
> 只规格化增量变更，不试图规格化整个世界。

---

## Phase 0: Exploration / 阶段零：探索

**目标 / Goal**: 理解现有项目的架构、约定和状态。在做任何改变之前，先理解系统。
Understand the existing project's architecture, conventions, and state. Understand before changing.

### Exploration Checklist / 探索检查清单

#### Architecture / 架构

- [ ] Identify the primary architecture pattern (MVC, Clean Architecture, Feature-sliced, etc.)
- [ ] Map the directory structure and understand the organization principle
- [ ] Identify module boundaries and their communication patterns
- [ ] Document the layering (presentation → business → data)
- [ ] Find entry points (main, routes, CLI commands)

```bash
# 有用的探索命令 / Useful exploration commands
# 目录结构概览 / Directory structure overview
find src/ -type d -maxdepth 3 | head -50

# 文件类型分布 / File type distribution  
find src/ -type f | sed 's/.*\.//' | sort | uniq -c | sort -rn

# 最大文件（可能需要拆分）/ Largest files (may need splitting)
find src/ -type f -name "*.ts" | xargs wc -l | sort -rn | head -20
```

#### Tech Stack / 技术栈

- [ ] Language and runtime version
- [ ] Framework and its version
- [ ] Database type and ORM/query builder
- [ ] Testing framework and coverage tool
- [ ] Build system and bundler
- [ ] Package manager
- [ ] CI/CD pipeline
- [ ] Deployment target

#### Code Conventions / 代码约定

- [ ] Naming patterns (camelCase, snake_case, PascalCase)
- [ ] Import style and ordering
- [ ] Error handling patterns (try/catch, Result type, error codes)
- [ ] State management approach
- [ ] Logging conventions
- [ ] Comment style and documentation

```bash
# 发现命名约定 / Discover naming conventions
# 检查文件命名 / Check file naming
ls src/**/*.ts | head -20

# 检查导出模式 / Check export patterns
grep -r "export default" src/ --include="*.ts" | wc -l
grep -r "export {" src/ --include="*.ts" | wc -l
```

#### Testing Status / 测试状态

- [ ] What test framework is used?
- [ ] What is the current coverage percentage?
- [ ] Which areas have good test coverage?
- [ ] Which areas have no tests?
- [ ] Are there E2E tests? Integration tests?
- [ ] How are tests organized?

```bash
# 测试覆盖率 / Test coverage
[your test runner] --coverage

# 测试文件分布 / Test file distribution
find . -name "*.test.*" -o -name "*.spec.*" | wc -l
find src/ -name "*.ts" -not -name "*.test.*" -not -name "*.spec.*" | wc -l
```

#### Technical Debt / 技术债务

- [ ] Known bugs or issues (check issue tracker)
- [ ] Deprecated dependencies
- [ ] Areas marked with TODO/FIXME/HACK
- [ ] Areas with low test coverage
- [ ] Performance bottlenecks

```bash
# 技术债务标记 / Tech debt markers
grep -rn "TODO\|FIXME\|HACK\|XXX" src/ | wc -l
grep -rn "TODO\|FIXME\|HACK\|XXX" src/ | head -30
```

---

## Phase 1: Constitution Drafting / 阶段一：宪法起草

**目标 / Goal**: 从现有约定中提炼出 CLAUDE.md，使其反映"项目实际是什么样"而非"应该是什么样"。
Distill CLAUDE.md from existing conventions, reflecting "how the project actually is" not "how it should be."

### Steps / 步骤

#### 1.1 Document What Exists / 记录现状

从探索结果中提取：
Extract from exploration results:

```markdown
## Tech Stack (从 package.json / requirements.txt / go.mod 提取)
## Architecture (从目录结构和导入模式推断)
## Naming (从现有代码中观察到的模式)
## Testing (从现有测试中总结的约定)
```

#### 1.2 Identify Aspirational vs Actual / 区分愿景与现实

在 CLAUDE.md 中明确区分：
Clearly distinguish in CLAUDE.md:

```markdown
## Code Standards (ACTUAL — 现有代码遵循的)
- Files use camelCase naming
- Errors are handled with try/catch

## Code Standards (ASPIRATIONAL — 新代码应遵循的)
- Max file length: 400 lines (current average: 600)
- Prefer Result type over try/catch for new code
```

#### 1.3 Start Conservative / 保守开始

第一版 CLAUDE.md 应该：
The first version of CLAUDE.md should:

- **DO**: Document patterns that 80%+ of existing code follows
- **DO**: Mark aspirational rules as "for new code only"
- **DON'T**: Impose rules that contradict the majority of existing code
- **DON'T**: Require refactoring existing code to match new rules

#### 1.4 Template

Use `constitution-template.md` and fill in based on exploration findings.

---

## Phase 2: Incremental Spec Coverage / 阶段二：渐进式规格覆盖

**目标 / Goal**: 只对增量变更使用 SDD。不试图为已存在的功能补写规格。
Use SDD only for incremental changes. Do not try to retroactively spec existing features.

### Strategy: Spec the Delta / 策略：规格化增量

```
已有功能 (Existing) → 不需要规格 (No spec needed)
新增功能 (New feature) → 完整规格 (Full spec)
修改已有功能 (Modify existing) → 增量规格 (Delta spec)
Bug 修复 (Bug fix) → 最小规格 (Minimal spec)
```

### 2.1 New Features / 新功能

完整使用 SDD 流程：
Use the full SDD flow:

```
spec-template.md → plan-template.md → task-template.md → implement → verify
```

### 2.2 Modifications to Existing Features / 修改已有功能

写增量规格（只描述变化的部分）：
Write a delta spec (only describe what changes):

```markdown
## Delta Spec: [SPEC-XXX-delta]

### Context / 上下文
[Which existing feature is being modified]
[Link to relevant source files]

### Current Behavior / 当前行为
[How it works now — brief description]

### Desired Behavior / 期望行为
[How it should work after the change — EARS format]

### Requirements (Delta Only) / 需求（仅增量）
- REQ-E01: When [new event], the system shall [new action].
- REQ-X01: If [new error case], the system shall [handle].

### Acceptance Criteria / 验收标准
[Only for the changed behavior]

### Affected Files / 受影响的文件
[List files that will be modified]
```

### 2.3 Bug Fixes / Bug 修复

最小规格——明确"是什么"和"应该是什么"：
Minimal spec — clarify "what is" vs "what should be":

```markdown
## Bug Fix Spec: [BUG-XXX]

### Observed Behavior / 观察到的行为
[What happens now — the bug]

### Expected Behavior / 期望行为
[What should happen — correct behavior]

### Reproduction Steps / 复现步骤
1. [Step 1]
2. [Step 2]
3. [Observe: incorrect result]

### Root Cause (if known) / 根因（如已知）
[Technical explanation]

### Fix Requirement / 修复需求
- REQ-X01: If [condition that causes bug], the system shall [correct behavior].

### Verification / 验证
- [ ] Regression test added that fails without fix
- [ ] Test passes with fix applied
```

---

## Phase 3: Coverage Expansion / 阶段三：覆盖率扩展

**目标 / Goal**: 随时间推移，逐步扩大 SDD 覆盖范围。
Over time, gradually expand SDD coverage.

### Priority Order / 优先级顺序

1. **Security-critical paths** / 安全关键路径 — Auth, payments, user data
2. **High-change-frequency areas** / 高变更频率区域 — Code that changes often
3. **High-bug-density areas** / 高 bug 密度区域 — Code that has many bug fixes
4. **Core business logic** / 核心业务逻辑 — Revenue-generating features
5. **Everything else** / 其他 — Lower priority, spec when touched

### Tracking Coverage / 跟踪覆盖率

在项目根目录维护一个简单的跟踪文件：
Maintain a simple tracking file in project root:

```markdown
<!-- specs/COVERAGE.md -->
# SDD Coverage Tracker

## Fully Specified (完整规格)
- [x] User authentication (SPEC-001)
- [x] Payment processing (SPEC-002)
- [ ] Search (SPEC-003, in progress)

## Delta Specified (增量规格)
- [x] Profile update (SPEC-001-delta-001)
- [x] Cart quantity limits (BUG-042)

## Not Specified (未规格化)
- Notification system (low change frequency, low priority)
- Admin panel (internal tool, spec when modifying)
- Legacy reporting (scheduled for deprecation)
```

---

## Phase 4: Process Integration / 阶段四：流程集成

**目标 / Goal**: 将 SDD 融入日常开发流程。
Integrate SDD into daily development flow.

### Git Branch Naming / Git 分支命名

```
feat/SPEC-XXX-short-description
fix/BUG-XXX-short-description
```

### PR Template Addition / PR 模板补充

Add to your PR template:

```markdown
## SDD Reference
- Spec: [SPEC-XXX or N/A for legacy code]
- Plan: [PLAN-XXX or N/A]
- Verification: [PASS/FAIL/NOT_APPLICABLE]
```

### Review Process / 审查流程

```
For new features:    Require spec before PR review
For modifications:   Require delta spec before PR review
For bug fixes:       Require bug fix spec in PR description
For refactoring:     No spec needed (document in PR description)
```

---

## Migration Timeline Template / 迁移时间线模板

<!-- 
  根据项目规模和团队大小调整时间线。
  Adjust timeline based on project size and team size.
-->

### Week 1-2: Foundation / 基础

- [ ] Complete Phase 0 exploration
- [ ] Draft initial CLAUDE.md (constitution)
- [ ] Set up `specs/`, `plans/`, `tasks/` directories
- [ ] Team agrees on SDD adoption scope

### Week 3-4: Pilot / 试点

- [ ] Pick 1-2 upcoming features for full SDD treatment
- [ ] Write first spec using spec-template.md
- [ ] Write first plan using plan-template.md
- [ ] Execute tasks and run first verification
- [ ] Retrospective: what worked, what didn't

### Month 2: Expansion / 扩展

- [ ] All new features use full SDD
- [ ] Bug fixes use minimal spec format
- [ ] Feature modifications use delta spec format
- [ ] Start tracking coverage in COVERAGE.md
- [ ] Refine CLAUDE.md based on learnings

### Month 3+: Steady State / 稳态

- [ ] SDD is default workflow for all non-trivial changes
- [ ] Coverage tracker maintained and reviewed
- [ ] Gate reviews happen naturally
- [ ] Team is comfortable with EARS notation
- [ ] Verifier agent runs before merges

---

## Common Pitfalls / 常见陷阱

### Pitfall 1: Boiling the Ocean / 煮沸海洋

**Wrong / 错误**: "Let's write specs for all 50 existing features before doing anything new."
**Right / 正确**: Spec only what you're about to change. Legacy code gets specced when touched.

### Pitfall 2: Over-Specifying / 过度规格化

**Wrong / 错误**: Writing a 10-page spec for a CSS color change.
**Right / 正确**: Match spec depth to change complexity. A bug fix needs 5 lines. A new module needs a full spec.

### Pitfall 3: Constitution as Wishlist / 宪法变愿望清单

**Wrong / 错误**: CLAUDE.md describes the ideal codebase, not the actual one.
**Right / 正确**: CLAUDE.md documents reality first, marks aspirations explicitly as "for new code."

### Pitfall 4: Ignoring Existing Tests / 忽视已有测试

**Wrong / 错误**: Discarding existing test patterns and starting fresh.
**Right / 正确**: Extend existing test patterns. Gradually improve test quality in newly written tests.

### Pitfall 5: All-or-Nothing Thinking / 非此即彼思维

**Wrong / 错误**: "If we can't do full SDD for everything, why bother?"
**Right / 正确**: Even partial SDD (spec + verify for critical paths) dramatically reduces defects.

---

## Quick Start Checklist / 快速开始清单

For teams who want to start TODAY:

```
[ ] 1. Run exploration commands (30 min)
[ ] 2. Create CLAUDE.md from findings (1 hour)
[ ] 3. Create specs/ plans/ tasks/ directories (1 min)
[ ] 4. Pick your next feature or significant bug fix
[ ] 5. Write a spec for it using spec-template.md (30 min)
[ ] 6. Write a plan using plan-template.md (30 min)
[ ] 7. Decompose into tasks using task-template.md (20 min)
[ ] 8. Implement using Claude Code, following the tasks
[ ] 9. Run verification against the spec
[ ] 10. Retrospective: refine your CLAUDE.md and process
```

Total bootstrap time for first feature: ~3 hours (including exploration).
After that, spec writing becomes faster with practice.
