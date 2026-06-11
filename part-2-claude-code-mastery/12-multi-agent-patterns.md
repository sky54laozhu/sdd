# Chapter 12: 多代理编排模式 (Multi-Agent Orchestration Patterns)

> 当单个 Agent 的上下文有限、自我验证不可靠、顺序执行太慢时，多代理编排提供了分治、对抗和流水线三大武器。

---

## 为什么需要多代理

单个 Claude Code 会话是强大的 — 但它有三个结构性限制：

**1. 上下文窗口有限**

即使有 200k token 的窗口，一个复杂项目的完整上下文（Spec + Plan + 源代码 + 测试 + 文档）很容易超出。一个 Agent 试图"什么都记住"，结果是什么都记不牢。

**2. 自我验证不可靠**

让同一个 Agent 既写代码又验证代码，就像让学生批改自己的作业 — 同样的思维盲点、同样的假设、同样的上下文偏见（context bias）。它会倾向于"认为"自己的代码是正确的。

**3. 顺序执行太慢**

分析安全性、检查性能、验证类型 — 这些是独立的任务，但单 Agent 只能逐个做。3 个 5 分钟的任务 = 15 分钟等待。

多代理编排（Multi-Agent Orchestration）通过**角色分离**和**并行执行**解决这些问题。

---

## 模式总览

```
┌─────────────────────────────────────────────────────────┐
│                 Multi-Agent Patterns                      │
├────────────┬────────────┬──────────┬──────────┬─────────┤
│  Verifier  │  Parallel  │Orchestr- │   GAN    │Pipeline │
│   Agent    │  Research  │  ator    │Generator/│         │
│            │            │          │Evaluator │         │
├────────────┼────────────┼──────────┼──────────┼─────────┤
│ 对抗验证   │ 并行探索   │ 集中调度  │ 迭代对抗  │ 链式传递 │
│            │            │          │          │         │
│ Implement  │ Q1  Q2  Q3 │  Main    │ Gen→Eval │ A→B→C→D │
│     ↓      │  ↓   ↓   ↓ │  / | \  │  ↓    ↑  │  ↓ ↓ ↓ ↓│
│  Verify    │ [Aggregate]│ A  B  C  │ [Loop]   │[Gate][Gate]│
└────────────┴────────────┴──────────┴──────────┴─────────┘
```

---

## Pattern 1: Verifier Agent (验证代理模式)

**这是最被低估的模式。** 大多数开发者让 Claude 自己检查自己的代码，然后疑惑为什么 bug 还是漏出去了。

### 为什么自我验证失败

```
┌──────────────────────────────────────────────┐
│           Same Agent = Same Blind Spots       │
│                                               │
│  Implementer:                                 │
│    "I wrote this auth check, looks good"      │
│                                               │
│  Same Agent as Verifier:                      │
│    "Let me check... yes, auth check exists"   │
│    (Same assumption: the check is correct)    │
│                                               │
│  VERSUS                                       │
│                                               │
│  Independent Verifier:                        │
│    "Auth check exists BUT: doesn't handle     │
│     expired tokens, no rate limiting,         │
│     missing CSRF on state-changing routes"    │
└──────────────────────────────────────────────┘
```

核心问题是 **confirmation bias**（确认偏差）：同一个 Agent 在同一个上下文中，会倾向于证实自己已做的决策，而非质疑它们。

### 实现方式

使用 Claude Code 的 Task tool 启动一个独立的 Verifier Agent：

```typescript
// verification-prompt.md
// 这是传递给 Verifier Subagent 的 prompt

You are a STRICT code verifier. Your job is to find problems, not to approve.

## Your Spec
{spec_content}

## Code Under Review
{implementation_files}

## Verification Protocol

For EACH "SHALL" constraint in the spec:
1. Locate the implementation (file:line)
2. Determine status: PASS | FAIL | PARTIAL | NOT_FOUND
3. If not PASS, explain precisely what's wrong
4. Classify: AUTO_FIXABLE | HUMAN_REQUIRED

## Report Format

```json
{
  "summary": {
    "total": 23,
    "pass": 18,
    "fail": 3,
    "partial": 2,
    "compliance_rate": 0.783
  },
  "issues": [
    {
      "spec_id": "FR-007",
      "status": "FAIL",
      "constraint": "System SHALL validate email format before storage",
      "location": "src/services/user.ts:45",
      "finding": "No email validation. Raw input passed directly to DB.",
      "classification": "AUTO_FIXABLE",
      "fix_hint": "Add zod email() validator in CreateUserInput schema"
    }
  ]
}
```

## Rules
- You MUST check every single SHALL constraint. No skipping.
- Default to FAIL when in doubt (strict, not generous).
- "Code exists" is not the same as "requirement met".
- Look for edge cases the implementer likely missed.
```

### 调用方式

在主 Claude Code 会话中，通过 Task tool 调用 Verifier：

```
Human: 功能已经实现完毕，请启动独立的 Verifier Agent 检查合规性。

Claude: 我将启动一个独立的 Verifier Subagent 来检查实现是否符合 Spec。

[Task tool call with verification-prompt.md content + spec + source files]
```

### Issue 分类：AUTO_FIXABLE vs HUMAN_REQUIRED

Verifier 的输出将问题分为两类：

| 分类 | 含义 | 处理方式 |
|------|------|----------|
| **AUTO_FIXABLE** | 代码层面的问题，AI 可以自动修复 | 主 Agent 接收报告后自动修复 |
| **HUMAN_REQUIRED** | 需求歧义、架构决策、业务规则确认 | 暂停，等待人工判断 |

这个分类让自动化修复和人工干预有了清晰的边界。

---

## Pattern 2: Parallel Research (并行研究模式)

当你需要同时回答多个独立问题时，启动多个 Subagent 并行探索。

### 适用场景

```
┌─────────────────────────────────────────────┐
│            Main Session (Orchestrator)        │
│                                              │
│  "分析这个新模块的质量"                       │
│                                              │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ │
│  │ Agent 1   │ │ Agent 2   │ │ Agent 3   │ │
│  │ Security  │ │Performance│ │ Type-safe │ │
│  │ Analysis  │ │  Review   │ │  Check    │ │
│  │           │ │           │ │           │ │
│  │ - OWASP   │ │ - N+1    │ │ - strict  │ │
│  │ - Injection│ │ - Memory │ │ - any 类型 │ │
│  │ - Auth    │ │ - Cache  │ │ - 断言    │ │
│  └─────┬─────┘ └─────┬─────┘ └─────┬─────┘ │
│        │              │              │       │
│        ▼              ▼              ▼       │
│  ┌─────────────────────────────────────┐    │
│  │        Aggregate Results             │    │
│  │  Security: 2 HIGH, 1 MEDIUM         │    │
│  │  Performance: 1 HIGH                 │    │
│  │  Types: 3 MEDIUM                     │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘
```

### 实际调用

```
Human: 请并行分析 src/features/payment/ 模块的安全性、性能和类型安全。

Claude: 我将启动 3 个独立的分析 Agent 并行工作：

Agent 1 — Security Analysis:
[Task: 分析 payment 模块的安全漏洞，重点检查 OWASP Top 10]

Agent 2 — Performance Review:
[Task: 分析 payment 模块的性能问题，检查 N+1 查询、未缓存的热路径]

Agent 3 — Type Safety Audit:
[Task: 检查 payment 模块的类型安全，寻找 any 类型、类型断言、未处理的 null]
```

三个 Agent 并行执行，各自在独立上下文中工作。主 Agent 收集结果后汇总为统一报告。

### 关键优势

- **时间**：3 个 5 分钟任务 = 5 分钟（并行）而非 15 分钟（串行）
- **深度**：每个 Agent 只关注一个维度，可以更深入
- **独立性**：一个 Agent 的发现不会影响另一个的判断

---

## Pattern 3: Orchestrator (编排器模式)

主 Claude Code 会话作为"项目经理"，将任务分发给专家 Agent，收集结果后做决策。

### 结构

```
┌──────────────────────────────────────────────────┐
│              Orchestrator (Main Session)           │
│                                                   │
│  Responsibilities:                                │
│  - 理解全局目标                                    │
│  - 分解为子任务                                    │
│  - 分发给专家 Agent                               │
│  - 收集结果                                       │
│  - 做出最终决策                                    │
│  - Phase Gate 审查                                │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │Researcher│  │Implementer│  │ Reviewer │       │
│  │          │  │          │  │          │       │
│  │"调查最佳 │  │"按照方案  │  │"检查实现  │       │
│  │ 实践方案"│  │ 实现代码" │  │ 质量"    │       │
│  └──────────┘  └──────────┘  └──────────┘       │
│       │              │              │             │
│       ▼              ▼              ▼             │
│  [Research      [Source Code]   [Review         │
│   Report]                        Report]         │
│       │              │              │             │
│       └──────────────┼──────────────┘             │
│                      ▼                            │
│            Orchestrator Decision:                  │
│            "Review 有 2 个 HIGH 问题，             │
│             让 Implementer 修复后                  │
│             再次提交 Review"                       │
└──────────────────────────────────────────────────┘
```

### Phase Gate Reviews

Orchestrator 的核心能力是**阶段门控**（Phase Gate）：

```markdown
## Phase Gate Protocol

### Gate 1: Research Complete
- [ ] 至少 3 种方案被评估
- [ ] 每种方案有 pros/cons 分析
- [ ] 推荐方案有明确理由
→ 通过后进入 Implementation 阶段

### Gate 2: Implementation Complete
- [ ] 所有计划中的文件已创建
- [ ] 无编译错误
- [ ] 基础测试通过
→ 通过后进入 Review 阶段

### Gate 3: Review Passed
- [ ] 无 CRITICAL 问题
- [ ] HIGH 问题已修复或有例外说明
- [ ] 代码覆盖率 >= 80%
→ 通过后进入 Integration 阶段
```

如果某个 Gate 未通过，Orchestrator 会将问题反馈给对应的 Agent 并要求修复。

---

## Pattern 4: GAN-Style Generator/Evaluator (生成-评估模式)

灵感来自 GAN（Generative Adversarial Network）的对抗训练：一个 Agent 生成，另一个 Agent 评估，循环直到质量达标。

### 核心机制

```
┌──────────────────────────────────────────────┐
│          GAN-Style Iteration Loop             │
│                                               │
│  ┌──────────────┐      ┌──────────────┐     │
│  │  Generator   │      │  Evaluator   │     │
│  │              │      │              │     │
│  │ "实现功能"   │      │ "打分 + 反馈" │     │
│  └──────┬───────┘      └──────┬───────┘     │
│         │                      │             │
│         │   Implementation     │             │
│         ├─────────────────────▶│             │
│         │                      │             │
│         │   Score + Feedback   │             │
│         │◀─────────────────────┤             │
│         │                      │             │
│         │   Revised Impl.      │ score < 8?  │
│         ├─────────────────────▶│ → iterate   │
│         │                      │             │
│         │   Score: 9/10 ✓      │ score >= 8? │
│         │◀─────────────────────┤ → accept    │
│         │                      │             │
│  ════════════════════════════════════════     │
│  Iteration 1: Score 5/10 (missing error      │
│    handling, no input validation)             │
│  Iteration 2: Score 7/10 (error handling     │
│    added, but edge cases missed)             │
│  Iteration 3: Score 9/10 (all checks pass)  │
│  → ACCEPTED                                  │
└──────────────────────────────────────────────┘
```

### Evaluator Rubric（评估标准）

Evaluator Agent 不是随意打分 — 它有明确的评估矩阵（rubric）：

```markdown
## Evaluation Rubric

Score each dimension 1-10, then average:

### Correctness (权重: 3x)
- All spec constraints satisfied?
- Edge cases handled?
- Error paths covered?

### Code Quality (权重: 2x)
- Functions < 50 lines?
- No deep nesting?
- Clear naming?
- Immutable patterns?

### Security (权重: 2x)
- Input validated?
- No injection vectors?
- Secrets handled correctly?

### Performance (权重: 1x)
- No obvious N+1?
- Appropriate data structures?
- No unnecessary allocations in hot paths?

### Testing (权重: 2x)
- Unit tests exist?
- Edge cases tested?
- Coverage >= 80%?

## Scoring
- Weighted average >= 8.0: ACCEPT
- Weighted average 6.0-7.9: REVISE (provide specific feedback)
- Weighted average < 6.0: REJECT (major issues, provide rewrite guidance)
```

### 实际交互流程

```
Orchestrator → Generator:
  "Implement the user authentication module per spec FR-001 through FR-012"

Generator → Output:
  [auth.ts, auth.test.ts, types.ts]

Orchestrator → Evaluator:
  "Score this implementation against the rubric. Spec: {spec}. Code: {files}"

Evaluator → Score:
  {
    "score": 6.2,
    "verdict": "REVISE",
    "feedback": [
      "FR-007: No email validation",
      "FR-011: Token expiry not checked on refresh",
      "Code: validatePassword() is 67 lines, split it"
    ]
  }

Orchestrator → Generator:
  "Revise based on feedback: {evaluator_feedback}"

[Cycle continues until score >= 8.0 or max_iterations reached]
```

### 何时使用

- 高质量要求的关键模块（认证、支付、数据处理）
- 第一次实现某个复杂模式时（学习性迭代）
- 代码将长期维护、无法频繁重写时

### 何时不用

- 简单的 CRUD 操作
- 原型/POC 阶段
- 时间紧迫时（每次迭代消耗额外 Token 和时间）

---

## Pattern 5: Pipeline (流水线模式)

将 Agent 链接为流水线，每个 Agent 的输出是下一个的输入。这是 SDD 工作流的自然形态。

### 结构

```
┌─────────┐    ┌─────────┐    ┌────────────┐    ┌──────────┐    ┌──────────┐
│Researcher│───▶│ Planner │───▶│Implementer │───▶│ Verifier │───▶│ Reviewer │
│          │    │         │    │            │    │          │    │          │
│ Output:  │    │ Output: │    │ Output:    │    │ Output:  │    │ Output:  │
│ Research │    │ Plan    │    │ Code +     │    │ Verify   │    │ Review   │
│ Report   │    │ Doc     │    │ Tests      │    │ Report   │    │ Report   │
└─────────┘    └─────────┘    └────────────┘    └──────────┘    └──────────┘
     │              │               │                │               │
     ▼              ▼               ▼                ▼               ▼
  [GATE 1]      [GATE 2]       [GATE 3]         [GATE 4]       [GATE 5]
  "Research     "Plan           "Code            "Compliance    "Quality
   complete?"   approved?"      compiles?"       >= 90%?"       >= 8/10?"
```

### Phase Gate 定义

```json
{
  "pipeline": {
    "stages": [
      {
        "name": "research",
        "agent": "researcher",
        "gate": {
          "criteria": "At least 2 viable approaches documented",
          "output": "research-report.md"
        }
      },
      {
        "name": "plan",
        "agent": "planner",
        "input_from": "research",
        "gate": {
          "criteria": "All spec constraints mapped to implementation files",
          "output": "implementation-plan.md"
        }
      },
      {
        "name": "implement",
        "agent": "implementer",
        "input_from": "plan",
        "gate": {
          "criteria": "pnpm build passes && pnpm test passes",
          "output": "src/**/*.ts"
        }
      },
      {
        "name": "verify",
        "agent": "verifier",
        "input_from": ["implement", "spec"],
        "gate": {
          "criteria": "compliance_rate >= 0.9",
          "output": "verification-report.json"
        }
      },
      {
        "name": "review",
        "agent": "reviewer",
        "input_from": "implement",
        "gate": {
          "criteria": "No CRITICAL issues, max 2 HIGH issues",
          "output": "review-report.md"
        }
      }
    ]
  }
}
```

---

## 质量追踪：Audit Trail

无论使用哪种模式，都应该记录质量轨迹。使用 `quality-timeline.jsonl` 格式：

```jsonl
{"timestamp":"2026-06-10T14:23:00Z","stage":"implement","agent":"generator","action":"created","files":["src/auth.ts","src/auth.test.ts"],"metrics":{"lines":234,"functions":8}}
{"timestamp":"2026-06-10T14:24:30Z","stage":"verify","agent":"verifier","action":"evaluated","result":{"compliance_rate":0.78,"failures":["FR-007","FR-011","NFR-002"]}}
{"timestamp":"2026-06-10T14:25:00Z","stage":"implement","agent":"generator","action":"revised","files":["src/auth.ts"],"feedback_applied":["FR-007","FR-011"]}
{"timestamp":"2026-06-10T14:26:00Z","stage":"verify","agent":"verifier","action":"evaluated","result":{"compliance_rate":0.95,"failures":["NFR-002"]}}
{"timestamp":"2026-06-10T14:27:00Z","stage":"review","agent":"reviewer","action":"scored","result":{"score":8.4,"verdict":"ACCEPT","issues_remaining":1}}
```

这个审计轨迹让你可以：

- 追踪质量如何随迭代提升
- 发现哪些约束类型最容易被漏掉
- 评估多代理模式的 ROI（额外 Token 成本 vs 质量提升）

---

## 何时不使用多代理

多代理不是万能的。以下场景应避免使用：

| 场景 | 原因 | 替代方案 |
|------|------|----------|
| 简单的单文件修改 | Overhead > Benefit | 直接在主会话中做 |
| 紧急修复（hotfix） | 时间压力 | 单 Agent + 快速验证 |
| 上下文很小的任务 | 不需要分治 | 单 Agent 足够 |
| 探索性/实验性工作 | 目标不明确，无法定义 rubric | 对话式协作 |
| Token 预算紧张 | 多 Agent = 多倍成本 | 选择性使用关键模式 |

**经验法则**：如果任务可以在单个 Claude Code 会话中 10 分钟内完成，不要使用多代理。

---

## 实战：用 Generator + Verifier 实现一个功能

将理论付诸实践。假设我们要实现一个 "Rate Limiter" 功能：

### Step 1: 准备 Spec

```markdown
# Rate Limiter Spec (rate-limiter.spec.md)

## FR-001
System SHALL limit each API key to 100 requests per minute.

## FR-002
System SHALL return HTTP 429 with Retry-After header when limit exceeded.

## FR-003
System SHALL use sliding window algorithm (not fixed window).

## FR-004
System SHALL support configurable limits per route.

## NFR-001
Rate check SHALL complete within 5ms for in-memory store.
```

### Step 2: 主 Agent 作为 Orchestrator

```
Human: 请用 Generator + Verifier 模式实现 rate-limiter.spec.md

Orchestrator (Main Claude):
"我将使用 GAN 模式：先启动 Generator 实现，
再启动 Verifier 检查，迭代直到合规。"

→ [Launch Generator Agent]
  Input: Spec content
  Output: src/rate-limiter.ts, src/rate-limiter.test.ts

→ [Launch Verifier Agent]
  Input: Spec + Generated code
  Output: Verification report

→ [If score < threshold]
  Feed back to Generator with specific failures
  
→ [Repeat until PASS]
```

### Step 3: 结果

```
Iteration 1:
  Generator: Created rate-limiter.ts (sliding window, 100 req/min)
  Verifier: Score 6/10
    - FR-002: FAIL - No Retry-After header
    - FR-004: FAIL - Hardcoded 100, not configurable
    - NFR-001: PARTIAL - No benchmark proving < 5ms

Iteration 2:
  Generator: Revised with Retry-After header, config map, added benchmark
  Verifier: Score 9/10
    - All PASS except NFR-001: PARTIAL (benchmark shows 3ms avg, 
      but no test assertion)

Iteration 3:
  Generator: Added performance test asserting p99 < 5ms
  Verifier: Score 10/10 - ALL PASS

Total iterations: 3
Total time: ~4 minutes
Extra token cost: ~2x single implementation
Quality improvement: Caught 3 issues that single-agent would likely miss
```

---

## 模式选择决策树

```
需要多代理吗？
│
├─ 任务简单（< 10 min）？ → 不需要，单 Agent 足够
│
├─ 需要自我验证？
│   └─ YES → Pattern 1: Verifier Agent
│
├─ 有多个独立问题？
│   └─ YES → Pattern 2: Parallel Research
│
├─ 需要分角色协作？
│   └─ YES → Pattern 3: Orchestrator
│
├─ 需要迭代提升质量？
│   └─ YES → Pattern 4: GAN Generator/Evaluator
│
└─ 有明确的阶段流程？
    └─ YES → Pattern 5: Pipeline
```

---

## 练习

### 练习 1：Verifier Agent 实践

1. 编写一个简单的 Spec（5 个 SHALL 约束）
2. 让 Claude Code 实现它（单 Agent，不做验证）
3. 启动一个独立的 Verifier Agent 检查实现
4. 对比 Verifier 发现的问题与你自己 review 发现的问题
5. 思考：哪些问题只有 Verifier 发现了？

### 练习 2：Parallel Research

选择一个你计划引入的库或工具，同时启动 3 个 Agent：
- Agent 1：安全性分析（CVE 历史、依赖安全）
- Agent 2：性能评估（benchmark、bundle size 影响）
- Agent 3：维护状态（commit 频率、issue 响应时间、社区活跃度）

汇总三份报告，做出是否采用的决策。

---

## Key Takeaways (要点回顾)

- **Verifier Agent 是 SDD 的核心武器**：独立验证消除 confirmation bias，是单代理自检的根本性升级
- **并行不只是快，更是深**：每个 Agent 专注一个维度，比单 Agent 面面俱到的浅层检查质量高得多
- **GAN 模式适合关键路径**：额外的迭代成本换来的是显著的质量提升，用在值得的地方
- **Pipeline 是 SDD 的自然形态**：Specify → Plan → Implement → Verify → Review 天然是一条流水线
- **知道何时不用**：简单任务、紧急修复、探索性工作 — 单 Agent 更高效

---

## Next (下一章)

[Chapter 13: 核心工作流 (Core Workflow)](../part-3-workflow/13-core-workflow.md) — 进入 Part 3，将前面学到的所有 Claude Code 能力（Hooks、Skills、多代理）组合为完整的 SDD 工作流：Spec → Plan → Implement → Verify 的端到端实践。
