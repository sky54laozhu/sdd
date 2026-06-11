# Chapter 25: 回顾与输出 (Retrospective and Deliverables)

> 项目结束不等于学习结束——复盘 SDD 流程的效能，将经验编码为可复用的 Skill，让下一个项目从你这个项目的终点开始。

---

## 项目完成回顾

SpecTask 项目从 Constitution 到 Verification 走完了 SDD 的全部五个阶段。现在是时候退后一步，审视整个流程的效能（effectiveness）了。

### 各阶段时间分布

| 阶段 | 活动 | 时间 | 占比 |
|------|------|------|------|
| Constitution | 编写 CLAUDE.md | 1 hour | 8% |
| Specify | 编写 Task CRUD spec + Auth spec | 2.5 hours | 20% |
| Plan | 架构选型、分层设计 | 1.5 hours | 12% |
| Tasks | 任务分解、依赖分析 | 1 hour | 8% |
| Implement | 17 个 task 逐一执行 | 5.5 hours | 44% |
| Verify | Verifier agent + 修复 gap | 1 hour | 8% |
| **Total** | | **12.5 hours** | **100%** |

关键洞察：**Implementation 只占总时间的 44%**。剩下的 56% 都在"想清楚"。

这与直觉相反——很多开发者认为"写代码"是主要工作。但在 SDD 中，规格编写和计划才是主要工作。代码只是规格的自然衍生物。

### Spec 准确性

| 指标 | 数值 | 说明 |
|------|------|------|
| 原始 spec requirement 总数 | 14 | REQ-U01~04, REQ-E01~08, REQ-O01~02 |
| 实现中需要修订的 requirement | 2 | 14% 修订率 |
| 新增的 requirement | 1 | 实现中发现遗漏 |
| 最终 requirement 总数 | 15 | |

被修订的 2 个 requirement：
- **REQ-E07**（账户锁定）：原 spec 说"锁定 15 分钟"，实现时发现 CLI 工具没有持久化 daemon，改为"锁定直到显式解锁"
- **REQ-O02**（截止日期通知）：原 spec 说"发送通知"，改为"在 list 输出中高亮显示"——CLI 没有后台通知能力

新增的 1 个 requirement：
- **REQ-X01**（数据目录配置）：原 spec 未考虑 `.db` 文件应该存在哪里，实现中添加了 `SPECTASK_DATA_DIR` 环境变量支持

**14% 的修订率**对于首次 SDD 实践来说是很好的成绩。成熟团队通常能做到 5-10%。

---

## What Went Well (做得好的)

### 1. TDD 几乎消灭了实现 bug

在 TASK-005（实现 TaskRepository）到 TASK-011（实现 AuthService）的 4 个 GREEN phase 中，**首次运行测试就全部通过的有 3 个**。只有 AuthService 有一个 bcrypt 异步/同步的问题需要一次修复。

这说明 RED phase 写出的测试足够精确，给了 AI 明确的行为契约（behavioral contract）。

### 2. 分层架构防止了 Architecture Drift

整个实现过程中，Verifier 没有发现任何 layer violation。CLAUDE.md 中明确的分层约束（CLI → Service → Repository → Storage）配合 TypeScript 的 import 可见性，有效防止了"走捷径"。

### 3. Verifier Agent 发现了一个关键遗漏

REQ-E07（账户锁定）在 task 分解时遗漏了——没有对应的 task。如果没有 Verifier，这个 requirement 会在 PR review 或更晚才被发现。Verifier 在实现完成后 5 分钟内就标出了这个 GAP。

---

## What Was Challenging (遇到的挑战)

### 1. Spec 粒度拿捏

REQ-E07 写得太抽象（"锁定账户"），没有考虑 CLI 的技术限制。教训：**Spec 应该考虑技术可行性**，但不等于提前做技术设计。正确的做法是在 Plan 阶段发现不可行性，然后回退修订 Spec。

### 2. 测试与实现的同步维护

当 REQ-E07 的实现方案从"锁定 15 分钟"改为"锁定直到显式解锁"时，对应的测试也要改。修改链是：**Spec → Task → Test → Implementation**。如果跳过了 Spec 修订直接改代码，追溯链就断了。

### 3. CLI 层的测试策略

CLI 代码本质是 I/O 密集的（读 stdin、写 stdout、读文件系统），不好做单元测试。最终采用的策略是：
- Service 层做完整单元测试
- CLI 层只做集成测试（实际调用命令，检查 stdout 输出）
- 接受 CLI 层覆盖率低于 80% 的现实

---

## 产出 SKILL.md：将 SDD 编码为可复用 Skill

SpecTask 项目最有价值的产出不是代码——是**经验的结构化表达**。我们将整个 SDD 工作流编码为一个 Claude Code Skill，以便在未来的项目中一键复用。

### 什么是 Skill？

Skill 是 Claude Code 的一种复用机制。它是一份 markdown 文件，定义了一组结构化的工作流步骤。当用户说出触发条件中的关键词时，Claude Code 会自动加载该 skill 的指令。

### 完整的 SKILL.md 定义

将以下文件保存到 `project/.claude/skills/sdd-workflow/SKILL.md`：

```markdown
---
name: sdd-workflow
description: |
  Spec-Driven Development workflow for feature implementation.
  Guides the full cycle: Constitution → Spec → Plan → Tasks → Implement → Verify.
trigger:
  - "new feature"
  - "implement feature"
  - "sdd workflow"
  - "spec-driven"
  - "start feature"
---

# SDD Workflow Skill

## When to Use

Activate this skill when:
- Starting a new feature from scratch
- Implementing a significant change that affects multiple files
- Working on a project that has CLAUDE.md and specs/ directory

## Prerequisites

Before starting, verify:
- [ ] Project has a CLAUDE.md (constitution)
- [ ] Project has a specs/ directory (or will create one)
- [ ] Project has a plans/ directory (or will create one)
- [ ] Testing framework is configured

## Workflow Steps

### Phase 1: Spec

1. Create spec file at `specs/SPEC-XXX-<feature-name>.md`
2. Define requirements using EARS notation:
   - Ubiquitous: "The system shall..."
   - Event-driven: "When [event], the system shall..."
   - State-driven: "While [state], the system shall..."
   - Optional: "Where [condition], the system shall..."
   - Unwanted: "If [unwanted], the system shall..."
3. Each requirement must be:
   - Uniquely identified (REQ-U01, REQ-E01, etc.)
   - Testable (can write a test that verifies it)
   - Unambiguous (only one interpretation)
4. Define acceptance criteria for each requirement
5. Review spec for completeness before proceeding

### Phase 2: Plan

1. Create plan file at `plans/SPEC-XXX-plan.md`
2. Make architecture decisions (with rationale)
3. Identify files to create/modify
4. Choose libraries and patterns
5. Map requirements to technical approach
6. Identify risks and unknowns

### Phase 3: Task Decomposition

1. Create task list following ordering rules:
   - Infrastructure before application code
   - Types/interfaces before implementation
   - Tests before implementation (TDD)
   - Inner layers before outer layers
2. Each task must have:
   - Spec requirement reference
   - Clear acceptance criteria
   - Verification command
   - Dependency list
3. Verify all spec requirements are covered by tasks

### Phase 4: Implementation

For each task in order:

1. **Read** the task description and linked spec requirement
2. **Execute** the task:
   - If test task (RED): Write tests, verify they compile, confirm they fail
   - If implementation task (GREEN): Write code, verify tests pass
3. **Verify** using the task's verification command
4. **Commit** with conventional commit message:
   - test: ... (for RED phase tasks)
   - feat: ... (for GREEN phase tasks)
   - refactor: ... (for cleanup tasks)

### Phase 5: Verification

1. Run all tests: `pnpm test`
2. Check coverage: `pnpm test --coverage` (target: 80%+)
3. Type check: `pnpm tsc --noEmit`
4. Run verifier agent: `@sdd-verifier` against all specs
5. Fix any CRITICAL or HIGH issues
6. Re-run verifier until PASS

## Common Pitfalls

- **Skipping spec**: "I'll just code it" → leads to rework
- **Spec too vague**: "Handle errors" → must be specific and testable
- **Merging tasks**: Doing 3 tasks in one prompt → hard to debug failures
- **Ignoring verifier**: "It's just a warning" → warnings become bugs
- **Architecture drift**: Service calling DB directly → fix immediately

## Quality Gates

| Gate | Condition | Action if Failed |
|------|-----------|-----------------|
| Spec Review | All REQs testable | Revise spec |
| Plan Review | No unresolved risks | Address risks |
| Task Review | All REQs covered | Add tasks |
| Implementation | Tests pass | Fix implementation |
| Verification | No CRITICAL gaps | Fix and re-verify |

## Output Artifacts

After completing this workflow, the project should have:
- `specs/SPEC-XXX-<name>.md` — Feature specification
- `plans/SPEC-XXX-plan.md` — Implementation plan
- `tasks/SPEC-XXX-tasks.md` — Task decomposition (optional file)
- `src/**` — Implementation code
- `tests/**` — Test code with 80%+ coverage
- Verification report (in PR description or separate file)
```

---

## 更新 CLAUDE.md

项目完成后，将学到的经验反馈到 Constitution 中。为 SpecTask 的 CLAUDE.md 追加：

```markdown
## Lessons Learned

### Spec Writing
- CLI 工具的 spec 必须考虑"无 daemon"约束——不能假设后台进程
- Optional requirements (REQ-O*) 仍然需要 task 覆盖，不能遗漏

### Testing Strategy
- Repository + Service 层：完整单元测试（目标 90%+）
- CLI 层：集成测试为主（目标 65%+，接受较低覆盖率）
- 集成测试覆盖完整用户流程（register → login → CRUD → logout）

### Architecture
- Result<T> 类型在整个 Service 层传播，不使用 throw
- CLI 层是唯一允许 process.exit() 的层
- Session 持久化用 JSON 文件，不用数据库（简单优先）

### Workflow
- Task 分解时必须交叉检查 spec——遗漏的 requirement 在这一步最容易发现
- Verifier agent 不是可选的——它是 Implementation 完成的判定条件
```

---

## 效能指标 (Metrics)

量化 SDD 工作流的效能，用于与未来项目对比：

| 指标 | SpecTask v1 | 说明 |
|------|-------------|------|
| **Spec Coverage** | 100% | 所有 feature 都有 spec |
| **First-Pass Accuracy** | 88% | 17 个 task 中 15 个一次通过 |
| **Test Coverage (line)** | 83% | 超过 80% target |
| **Spec Revision Rate** | 14% | 15 个 requirement 中 2 个被修订 |
| **Verification Attempts** | 2 | 首次 PARTIAL，修复后 PASS |
| **Planning:Implementation Ratio** | 56:44 | 规划略多于实现 |

---

## 元课程：SDD 的价值曲线

SDD 的投入产出比与项目复杂度成正比：

```
Value of SDD
     ^
     |                                    ╱
     |                              ╱─────
     |                        ╱─────
     |                  ╱─────
     |            ╱─────
     |      ╱─────
     |─────╱
     |   ╱   Break-even
     |  ╱    point
     | ╱
     |╱
     +──────────────────────────────────────> Project Complexity
     1 file    5 files    20 files    50+ files
```

### 小项目（1-5 个文件）

SDD 可能感觉是 overhead。一个单文件脚本不需要 spec、plan、task 分解——直接写就行。但即使在小项目中，花 10 分钟写一个简短的 spec 也能避免"做到一半发现需求理解错了"的情况。

### 中型项目（5-20 个文件）

这是 **break-even point（收支平衡点）**。SDD 的前期投入（规格 + 计划约 4-5 小时）在实现阶段收回——因为你几乎不需要返工、不需要"重新理解需求"、不需要修复 architecture drift。

经验法则：**当你的项目有 3 个以上的 feature 或 2 个以上的开发者时，SDD 开始产生正 ROI**。

### 大型项目（20+ 个文件）

SDD 的价值呈**超线性增长**。原因：
- Spec 成为团队沟通的单一信源（single source of truth）
- Plan 防止了多人并行开发时的架构冲突
- Task 分解允许并行执行（不同人做不同 task）
- Verifier 替代了大量的人工 code review

一个 50 文件的项目如果不用 SDD，返工率通常在 30-40%（写了 10 个文件，发现 3-4 个要重做）。用 SDD，返工率降到 10-15%。

---

## 下一步行动

### 1. 在自己的项目中应用 SDD

不需要从 greenfield（全新项目）开始。**Brownfield（存量项目）入口**：

```
1. 选一个即将开发的新 feature
2. 为它写一份 spec（不需要对整个项目补 spec）
3. 走完 Plan → Tasks → Implement → Verify
4. 观察效果，然后决定是否对更多 feature 采用 SDD
```

### 2. 定制模板

SpecTask 的模板是通用的。你的团队可能有特殊需求：

- 电商项目：spec 模板可能需要"支付合规性"字段
- 医疗项目：spec 模板需要"HIPAA 合规检查"部分
- API 项目：spec 模板重心在 endpoint 定义和 schema validation

复制 `templates/` 目录到你的项目，根据领域特点修改。

### 3. 分享 SKILL.md

将 `sdd-workflow/SKILL.md` 放入团队共享的 `~/.claude/skills/` 目录。这样团队中任何人说"implement feature"时，Claude Code 都会自动引导 SDD 流程。

### 4. 迭代改进

每次完成一个 SDD 项目，更新你的 SKILL.md：
- 新增发现的 pitfall
- 修正时间估算
- 添加领域特定的检查项

这就是**持续改进循环（continuous improvement loop）**：每个项目的 retrospective 喂入下一个项目的 constitution 和 skill。

---

## 总结：SDD 不是银弹，而是纪律

SDD 不会让差的代码变好——它防止好的意图变成差的代码。它的核心价值不在于任何单一的 artifact（spec、plan、task），而在于**信息的逐步精炼和显式传递**。

当你对一个 AI agent 说"实现一个任务管理器"，你是在赌博。当你给它一份有 15 个 EARS requirement 的 spec、一份分层架构 plan、和一份 17 个原子 task 的列表时，你是在工程化地降低风险。

```
SDD 的一句话总结：

把"希望 AI 理解我"变成"确保 AI 理解我"。
```

---

## Key Takeaways (要点回顾)

1. **Planning 占总时间 56%，这不是浪费而是投资** —— 它将实现阶段的返工率从 30-40% 压缩到 10-15%
2. **Spec 修订率是工作流健康度的指标** —— 5-15% 说明 spec 足够好但不僵化；> 30% 说明 spec 太草率
3. **将经验编码为 SKILL.md** —— 使 SDD 工作流可复用、可分享、可迭代改进
4. **SDD 的 break-even point 是 3 个 feature 或 2 个开发者** —— 小于此规模可以轻量使用，大于此规模建议完整采用
5. **每次 retrospective 的输出应喂入 Constitution 和 Skill** —— 这就是 AI 辅助开发的"持续改进循环"

---

## Next (下一章)

[Part 5: 进阶话题](../part-5-advanced/26-autonomous-pipelines.md) —— 探索自主流水线、团队协作、大型项目治理等高级 SDD 模式。
