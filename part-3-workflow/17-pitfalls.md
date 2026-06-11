# Chapter 17: 常见陷阱与解法 (Common Pitfalls and How to Avoid Them)

> 每种方法论都有其失败模式。了解 SDD 的 10 个常见陷阱，能让你在掉入坑里之前就看到路标。

---

## 为什么需要这一章？

SDD 的五阶段流程看起来简洁明了——但简洁不等于简单。团队在实践中会遇到各种"退化模式（degeneration patterns）"——流程表面上在运行，但已经失去了核心价值。

本章列出 10 个最常见的陷阱，每个包含：
- **Pattern（模式描述）**：陷阱是什么样的
- **Warning Signs（警告信号）**：如何早期发现
- **Solution（解法）**：怎么修
- **Prevention（预防）**：怎么避免再次发生

---

## Pitfall 1: CLAUDE.md Bloat（宪法膨胀）

### Pattern

CLAUDE.md 从简洁的项目原则文件，逐渐膨胀成一个 500 行的"项目百科全书"——什么都往里塞：feature 细节、临时决策、会议记录摘要、debug 笔记。

```markdown
# 膨胀的 CLAUDE.md（反例）

## Tech Stack
(正常内容)

## Architecture  
(正常内容)

## Sprint 3 Decisions
- 2024-12-01: 决定用 Redis 做缓存因为 MongoDB 太慢
- 2024-12-05: 临时关闭 rate limiting 给 loadtest
- 2024-12-08: John 说通知功能先用轮询不用 WebSocket

## Debug Notes
- User service 有个 race condition，暂时用 setTimeout 绕过
- 如果 CI 挂了先重跑一次可能就好了

## All API Endpoints
(50 行 endpoint 列表)
...
```

### Warning Signs

- CLAUDE.md 超过 200 行
- 包含日期性信息（"Sprint 3"、"本周"）
- 包含实现细节（具体函数名、行号）
- AI agent 经常忽略 CLAUDE.md 中的某些规则（信息过载导致注意力分散）

### Solution

**拆分为层级结构：**

```
CLAUDE.md (< 100 lines)     — 只有不可变原则
├── .claude/skills/          — 可复用的专项知识
│   ├── auth-patterns.md
│   ├── database-conventions.md
│   └── testing-standards.md
└── .claude/agents/          — 专用 subagent 定义
    ├── verifier.md
    └── researcher.md
```

**规则**：如果一条规则不是"项目级别的、长期有效的约束"，它就不属于 CLAUDE.md。

### Prevention

```bash
# 定期检查膨胀
claude "Review CLAUDE.md and flag:
1. Any line that references a specific feature or sprint
2. Any line that will become outdated within 1 month
3. Any section longer than 20 lines
Recommend: keep, move to skill, or delete."
```

---

## Pitfall 2: Spec Drift（规格漂移）

### Pattern

Spec 写好了，通过了 gate review，然后实现过程中发现"spec 说的不太对"。开发者直接改了代码，但没有回去更新 spec。三个月后，spec 和代码完全不一致。

```
时间线：
T0: Spec 说 "返回 201 with user ID"
T1: 实现时发现还需要返回 email，直接加了
T2: 另一个 feature 依赖了 email 字段
T3: 有人看 spec，以为只有 user ID
T4: 基于错误 spec 写了新 feature → bug
```

### Warning Signs

- Code review 时有人说"spec 不是这么写的但代码是对的"
- 新成员读 spec 后实现的功能和现有代码不一致
- Verifier agent 报告 spec-code 偏差越来越多

### Solution

1. **自动化 compliance hooks**：每次代码提交时检查是否偏离 spec

```bash
# PostToolUse hook: 检查 spec 偏离
claude "Compare the implementation in [file] against its spec.
If the implementation does something the spec doesn't mention, 
or omits something the spec requires, report as DRIFT:
- ADDITION: code does X but spec doesn't mention X
- OMISSION: spec requires Y but code doesn't implement Y
- CONTRADICTION: spec says Z but code does W"
```

2. **Spec 更新作为 task**：如果实现必须偏离 spec，创建一个"更新 spec"的 task

```markdown
### Task 3.5: Update spec AC-01 to include email in response
- **Reason**: During implementation, discovered email is needed for client redirect
- **Action**: Update spec AC-01 from "returns 201 with user ID" 
  to "returns 201 with user ID and email"
- **Gate**: Re-validate affected tests match updated spec
```

### Prevention

- **Spec 是 source of truth**：如果 code 和 spec 不一致，先更新 spec 再改 code
- **定期 spec audit**：每个 sprint 结束检查活跃 spec 的 drift rate
- **Bidirectional links**：代码注释引用 spec ID，spec 引用实现文件

---

## Pitfall 3: Context Exhaustion（上下文耗尽）

### Pattern

在一个长会话中，AI agent 处理了太多信息——前面的 CLAUDE.md 规则、spec 细节、plan 决策逐渐被"挤出" context window 的有效注意力范围。后面的 task 开始违反前面确立的规则。

```
Session 开始: Agent 完美遵循 CLAUDE.md
Task 1-5: 正常
Task 6-8: 开始忘记某些约定
Task 9-12: 代码风格和前面明显不一致
```

### Warning Signs

- 后期 task 的代码风格和前期不一致
- Agent 开始做 CLAUDE.md 明确禁止的事
- 相同问题在会话后期重复出现

### Solution

1. **Subagent 隔离**：每个 task 用独立的 subagent session

```bash
# 不要在一个长会话里做所有 tasks
# 而是每个 task 一个独立调用

claude "Execute Task 6. Context:
- CLAUDE.md: [attached]
- Spec ref: AC-03
- Plan ref: notification service module
- Dependencies: Task 5 output (notificationService.js exists)
Complete this task and stop."
```

2. **Session checkpointing**：长会话中定期重申关键约束

```bash
# 每 5 个 tasks 后，重申核心规则
claude "Before continuing, re-read CLAUDE.md and confirm you will follow:
1. Immutable patterns (no mutation)
2. Functions < 50 lines
3. All errors handled explicitly
4. No hardcoded values

Now execute Task 8."
```

3. **Context budget management**：监控 token 使用，在 80% 时新建会话

### Prevention

- 将 tasks 拆分得足够小，单个 task 不会占满 context
- 使用 CLAUDE.md 中的规则作为 PostToolUse hook（机械执行，不依赖 agent 记忆）
- 复杂项目中每 3-5 个 tasks 开新会话

---

## Pitfall 4: Over-Specification（过度规格化）

### Pattern

Spec 不再描述"系统应该做什么（what）"，而是规定"系统应该怎么做（how）"。它变成了伪代码，限制了实现自由度。

```markdown
# 过度规格化（反例）
## Requirement
When user registers, the system shall:
1. Call bcrypt.hash(password, 12) to hash the password
2. Create a new Prisma client instance
3. Call prisma.user.create({ data: { email, username, passwordHash } })
4. Return { id: result.id } with status 201
```

### Warning Signs

- Spec 中出现函数名、库名、具体 API 调用
- Spec 修改时需要改实现（它们过度耦合了）
- 技术选型变更会导致 spec 全部重写
- Spec 看起来像伪代码

### Solution

**保持 spec behavioral（行为化）：**

```markdown
# 正确的 spec（行为导向）
## Requirement
When user submits valid registration data, the system shall:
- Create a new account with the provided email and username
- Store the password in a non-reversible hashed form
- Return the new account's unique identifier with HTTP 201

## Acceptance Criteria
- Password is not stored in plaintext (verify: hash differs from input)
- Unique identifier is a valid UUID v4
- Duplicate email returns 409
```

注意区别：行为化 spec 说"非可逆 hash"而非"bcrypt cost 12"。具体的 hash 算法选择属于 Plan 阶段。

### Prevention

**Spec 写作检查清单：**
- 能否在不改 spec 的情况下切换数据库？如果不能，spec 过度规格化了
- 能否在不改 spec 的情况下重构内部架构？如果不能，spec 过度规格化了
- Spec 中有没有库名或函数名？有的话，可能过度规格化了

---

## Pitfall 5: Under-Specification（规格不足）

### Pattern

Spec 太模糊，AI agent 必须做大量假设才能实现。这些假设可能对也可能错——但没人知道哪些是对的，直到 code review 时发现"我不是这个意思"。

```markdown
# 规格不足（反例）
## Requirement
The system should handle user registration nicely.

## Acceptance Criteria
- Registration works
- Errors are handled properly
```

### Warning Signs

- Agent 问"你是指...还是...？"（它在猜测）
- Code review 频繁出现"这不是我想要的"
- 不同 agent session 对同一 spec 产出不同实现
- Spec 中有"properly"、"nicely"、"appropriately"等模糊词

### Solution

**使用 EARS notation 强制具体化：**

```markdown
# 修正后的 spec
## Requirement (Event-Driven)
When a user submits registration data with valid email (RFC 5322), 
username (3-30 chars, ^[a-zA-Z0-9_]+$), and password (min 8 chars),
the system shall create an account and return HTTP 201 with JSON body 
containing the account UUID.

## Acceptance Criteria
- [ ] POST /api/auth/register with valid body → 201 + { id: UUID }
- [ ] Invalid email format → 400 + { error: "Invalid email format" }
- [ ] Username < 3 chars → 400 + { error: "Username must be 3-30 characters" }
- [ ] Password < 8 chars → 400 + { error: "Password must be at least 8 characters" }
- [ ] Duplicate email → 409 + { error: "Email already registered" }
```

### Prevention

- **Concrete values test**：每个 AC 是否包含具体的数字、字符串或状态码？
- **Multiple-reader test**：两个不同的人读这个 spec，会实现一样的东西吗？
- **EARS compliance**：每条需求是否能归入 EARS 五种类型之一？

---

## Pitfall 6: Contradictory Specs（规格矛盾）

### Pattern

CLAUDE.md 说"所有 endpoint 需要认证"，但某个 feature spec 说"注册 endpoint 无需认证"。或者两个 spec 对同一个字段定义了不同的验证规则。

### Warning Signs

- Agent 实现了 A，reviewer 说应该是 B，两者都能在文档中找到依据
- Gate 审查通过但实现总是需要返工
- "根据哪个文档来？"成为团队 FAQ

### Solution

**建立 spec hierarchy（优先级层次）：**

```markdown
# Spec Hierarchy (in CLAUDE.md)

When documents conflict, follow this priority order:
1. CLAUDE.md (project constitution) — highest priority
2. Security requirements — override feature specs
3. Feature spec (most recently approved version)
4. Plan document
5. Task description — lowest priority

Exception: Feature specs may explicitly override CLAUDE.md rules 
with [OVERRIDE: rule-name] annotation and rationale.
```

```markdown
# 在 feature spec 中显式标注覆盖
## Registration Endpoint

[OVERRIDE: auth-required]
Rationale: Registration must be accessible to unauthenticated users 
by definition. This is not a security hole but a necessary exception.

Requirement: POST /api/auth/register does NOT require authentication.
```

### Prevention

- **Single source of truth**：每个概念只在一个地方定义
- **OVERRIDE 标注**：如果必须例外，显式标注并说明理由
- **Gate 2 交叉检查**：Plan → Tasks gate 应验证无矛盾

---

## Pitfall 7: Cargo-Cult SDD（形式主义 SDD）

### Pattern

团队走完了所有 SDD 步骤——写了 CLAUDE.md、spec、plan、tasks——但这些文档没有实质内容。它们是 template 的复制粘贴，填了字段但没有思考。

```markdown
# 形式主义 spec（反例）
## Requirement
The system shall implement the notification feature as described.

## Acceptance Criteria  
- [ ] Feature works correctly
- [ ] Tests pass
- [ ] Code is clean

## Out of Scope
- Things not in this spec
```

### Warning Signs

- Spec 通过 gate 只需要 2 分钟（太快了——没人在认真读）
- 所有 spec 看起来一模一样（只是换了 feature 名）
- Task list 都是"实现 X"、"测试 X"没有细节
- 团队抱怨"SDD 是浪费时间"

### Solution

**回归第一原则——每个阶段的真正目的是什么？**

| 阶段 | 形式主义 | 真正价值 |
|------|----------|----------|
| Spec | 填完模板字段 | 消除歧义，让两个人读出一样的意思 |
| Plan | 画个框图 | 提前发现架构风险 |
| Tasks | 列个清单 | 确保 TDD 顺序和原子性 |
| Gates | 签字盖章 | 在便宜时发现错误 |

**Key question（关键问题）**：如果去掉这个阶段，agent 的实现质量会下降吗？如果不会，说明这个阶段没有提供真正的信息。

### Prevention

- **Anti-template 思维**：template 是起点不是终点。如果填完 template 没有新的认知产生，就是在走形式
- **价值测试**：每个 spec 能否作为新人的唯一输入，让他实现正确的功能？
- **删除无价值仪式**：低风险变更可以跳过中间阶段（见 [Chapter 14](./14-phase-gates.md) 的"何时跳过 gates"）

---

## Pitfall 8: Phase Gate Fatigue（审核疲劳）

### Pattern

所有变更无论大小都需要通过完整的 4 道 gate。两行 CSS 修复也要写 spec、plan、tasks。团队越来越厌倦，开始草草通过审核——gate 失去了质量把关作用。

### Warning Signs

- Gate review 从 30 分钟降到 2 分钟（没人认真看了）
- "LGTM" 成为唯一的 review 内容
- 团队开始绕过 gate（"先发了，事后补文档"）
- 小修改的 lead time 和大 feature 一样长

### Solution

**风险分级的 gate 校准：**

```markdown
## Gate Calibration Matrix

| Change Risk Level | Required Gates | Example |
|-------------------|---------------|---------|
| Critical | All 4 gates + security review | Payment flow, auth changes |
| High | Gate 1 + Gate 3 + Gate 4 | New API endpoint, DB schema change |
| Medium | Gate 3 + Gate 4 | New service method, refactoring |
| Low | Gate 4 only (automated) | Bug fix, config change, UI tweak |
| Trivial | No gates | Typo fix, comment update |

Risk Assessment Criteria:
- Affects money? → Critical
- Affects auth? → Critical  
- New public API? → High
- Internal refactor? → Medium
- Single file fix? → Low
- Documentation only? → Trivial
```

### Prevention

- **Gate 强度与风险成正比**：不要一刀切
- **自动化 low-risk gates**：用 hooks 代替人工审查
- **定期回顾 gate policy**：每月问"上个月有多少次 gate 发现了真正的问题？"

---

## Pitfall 9: The "Awareness ≠ Adherence" Problem

### Pattern

Claude 读了 CLAUDE.md，"知道"你的规则，但不总是"遵循"它们。特别是在长会话、复杂逻辑、或规则与常见模式冲突时。

```
CLAUDE.md: "Never use default exports"
Claude: *creates a file with export default*

CLAUDE.md: "All errors must use AppError class"
Claude: *throws generic Error("something went wrong")*
```

### Warning Signs

- Code review 重复发现相同类型的违规
- 相同的规则违规在不同会话中反复出现
- Agent 在被提醒后会修正，但下次又犯

### Solution

**Documentation 定义期望，Hooks 强制执行：**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "node scripts/check-conventions.js \"$FILE_PATH\"",
        "description": "Enforce CLAUDE.md conventions mechanically"
      }
    ]
  }
}
```

```javascript
// scripts/check-conventions.js
const fs = require('fs');
const filePath = process.argv[2];
const content = fs.readFileSync(filePath, 'utf8');
const violations = [];

// Rule: No default exports
if (content.includes('export default')) {
  violations.push('VIOLATION: Default export detected. Use named exports.');
}

// Rule: No generic Error throws
if (content.match(/throw new Error\(/)) {
  violations.push('VIOLATION: Generic Error throw. Use AppError class.');
}

// Rule: No console.log
if (content.match(/console\.(log|debug|info)\(/)) {
  violations.push('VIOLATION: Console statement detected. Use logger.');
}

if (violations.length > 0) {
  console.error(violations.join('\n'));
  process.exit(2); // Block the write
}
```

**Awareness（文档）+ Adherence（hooks）= Reliable Compliance**

### Prevention

- **每条高频违规规则都配一个 hook**：文档是软约束，hook 是硬约束
- **机械检查优先**：能用 regex/AST 检查的就不靠 agent 自觉
- **分层执行**：简单规则用 hook，复杂规则用 verifier agent

---

## Pitfall 10: Template Worship（模板崇拜）

### Pattern

团队把 template 当成目标而非工具。每个字段都填满，即使某些字段对当前场景毫无意义。结果：产出大量形式完整但内容空洞的文档。

```markdown
# 模板崇拜（反例）

## Performance Requirements
- Response time: N/A (internal tool, no SLA)
- Throughput: N/A (10 users maximum)
- Scalability: N/A (single server)

## Security Requirements  
- Authentication: N/A (internal network only)
- Authorization: N/A (all users are admins)
- Data encryption: N/A (no sensitive data)

## Accessibility Requirements
- WCAG level: N/A (CLI tool)
```

连续 6 个 "N/A"——这些字段对这个项目完全没有价值，但 template 说要填就填了。

### Warning Signs

- 文档中 "N/A" 或 "TBD" 超过 20%
- 团队花更多时间在"填 template"而非"思考问题"
- 不同项目的 spec 长得一模一样（只换了名字）
- Template 从不修改——被当成"神圣文本"

### Solution

**Template 是起点，不是终点：**

```markdown
# 健康的 template 使用方式

1. 读 template → 理解每个字段的目的
2. 问自己：这个字段对当前项目/feature 有价值吗？
3. 有价值 → 填写具体内容
4. 无价值 → 删除该字段（不是写 N/A）
5. 需要额外字段 → 添加（template 没有的不代表不需要）
```

```bash
# Claude Code: 模板适配
claude "I'm using the spec template for a CLI tool with 10 internal users.
Remove sections that don't apply:
- Performance SLA (internal, no SLA needed)
- Accessibility (CLI, not web)
- Browser compatibility (not applicable)
Add sections that are needed but not in template:
- CLI argument specification
- Error exit codes
- Installation requirements"
```

### Prevention

- **Template 有版本和修改记录**：当团队发现某些字段从不有用，更新 template
- **"Why" test**：每填一个字段，问"这个信息会影响实现决策吗？"
- **定期 template review**：每季度评估模板是否仍然适合团队需求

---

## Quick Diagnostic（快速诊断清单）

如果你遇到以下症状，检查对应的陷阱：

| 症状 | 可能的陷阱 | 检查什么 |
|------|-----------|----------|
| Agent 忽略某些规则 | #1 (Bloat) 或 #9 (Awareness≠Adherence) | CLAUDE.md 行数？有无 enforcement hooks？ |
| Code review 发现"spec 不是这么说的" | #2 (Drift) | 最后一次更新 spec 是什么时候？ |
| 会话后期代码质量下降 | #3 (Context Exhaustion) | 会话持续了多少 tokens？ |
| Agent 实现了 spec 没说的功能 | #4 (Over-Spec) 或 #5 (Under-Spec) | Spec 是描述行为还是实现？Spec 是否足够具体？ |
| 团队在同一决策上反复争论 | #6 (Contradictions) | 有无 spec hierarchy？ |
| 文档完整但代码还是有 bug | #7 (Cargo-Cult) | Spec 是否提供了真正的新信息？ |
| 团队抱怨流程太重 | #8 (Gate Fatigue) | 有无 risk-based gate calibration？ |
| 相同规则反复违规 | #9 (Awareness≠Adherence) | 规则有无 automated enforcement？ |
| 所有项目文档长得一样 | #10 (Template Worship) | Template 是否根据项目适配了？ |

---

## 陷阱关系图

这 10 个陷阱不是孤立的——它们相互关联：

```
Bloat (#1) ───────────▶ Context Exhaustion (#3)
      │                         │
      ▼                         ▼
Awareness≠Adherence (#9)     Gate Fatigue (#8)
                                │
                                ▼
                        Cargo-Cult (#7)

Over-Spec (#4) ◀─────▶ Under-Spec (#5)
      │                       │
      └──────┐     ┌──────────┘
             ▼     ▼
        Contradictions (#6)
             │
             ▼
        Spec Drift (#2)

Template Worship (#10) ──▶ Cargo-Cult (#7)
```

**两条主要链路：**
1. **膨胀链**：Bloat → Context Exhaustion → Adherence Problems → Gate Fatigue → Cargo-Cult
2. **精度链**：Over/Under-Spec → Contradictions → Drift

打断这两条链路的方法：
- 膨胀链：保持 CLAUDE.md 精炼 + 用 hooks 强制执行
- 精度链：EARS notation + concrete acceptance criteria + automated compliance checks

---

## Key Takeaways (要点回顾)

1. **CLAUDE.md 保持精炼**（<200 行），多余内容拆分到 skills 和 subagents
2. **Spec 是 source of truth**：代码偏离时先更新 spec，不是反过来
3. **Hooks 比文档更可靠**：能自动化检查的规则不要靠 agent 自觉
4. **Gate 强度与风险成正比**：低风险变更简化流程，高风险变更全部通过
5. **Template 是工具不是目标**：根据项目适配，删除无价值字段，添加必要字段

---

## Next (下一章)

本章是 Part 3（SDD 工作流）的最后一章。继续阅读 [Part 4: 实战项目](../part-4-project/) 将所有工作流知识应用到完整的项目开发中。
