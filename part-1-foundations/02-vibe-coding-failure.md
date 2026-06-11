# 第二章：Vibe Coding 的七宗罪 (The Seven Sins of Vibe Coding)

> 没有 specification 约束的 AI coding 不是 "高效开发"，而是 "高效地制造技术债务"。本章记录七种具体的、可重复的失败模式。

---

## 什么是 Vibe Coding

"Vibe Coding" 一词由 Andrej Karpathy 在 2025 年初提出，用来描述一种开发方式：开发者向 AI agent 发出模糊的自然语言指令，然后 "凭感觉 (vibes)" 接受或拒绝生成的代码，不做严格的 specification 或 review。

这种方式在 prototyping (原型验证) 阶段可能是合理的。但当它被用于 production-grade (生产级) 的系统开发时，会系统性地产生以下七种失败模式。

---

## 第一宗罪：Scope Creep (范围蔓延)

### 模式描述

Agent 在实现需求时，自作主张地添加你从未要求的功能。这些功能看起来 "有用"、"合理"、"是最佳实践"，但它们增加了系统复杂度、扩大了攻击面、延长了交付时间。

### 具体示例

你的 prompt：

```
"Build a user registration endpoint with email and password."
```

AI 实际交付的：

```typescript
// 你要的：email + password registration
// AI 给你的：
- Email registration with confirmation flow
- Password strength meter with zxcvbn
- OAuth2 integration (Google, GitHub, Apple)
- Two-factor authentication setup
- User profile with avatar upload
- Email verification with rate-limited resend
- Account deactivation workflow
- GDPR data export endpoint
- Admin panel for user management
```

每一项单独看都 "不错"，但你只是想先做一个 MVP 的注册功能。现在你有了 3000 行代码需要 review、test 和 maintain。

### SDD 预防机制

在 SDD 中，specification 明确划定了 scope boundary (范围边界)：

```yaml
# user-registration.spec.yaml
scope:
  included:
    - email/password registration
    - input validation
    - password hashing (bcrypt, cost factor 12)
  explicitly_excluded:
    - OAuth integration (Phase 2)
    - 2FA (Phase 3)
    - Admin panel (separate service)
```

当 AI 试图超出 `included` 范围时，CI pipeline 中的 spec-compliance check 会标记 drift。更重要的是，`explicitly_excluded` 列表消除了歧义 — AI 不需要 "猜测" 你是否想要 OAuth。

---

## 第二宗罪：Hallucinated APIs (幻觉 API)

### 模式描述

Agent 引用不存在的库、不存在的方法、或已过时的 API signatures。LLM 的 training data 混合了多个版本的文档，它无法确定哪个版本是 "当前的"。

### 具体示例

```python
# AI 生成的代码
from langchain.agents import create_react_agent
from langchain.tools import DuckDuckGoSearchRun
from langchain.memory import ConversationBufferMemory

agent = create_react_agent(
    llm=ChatOpenAI(model="gpt-4"),
    tools=[DuckDuckGoSearchRun()],
    memory=ConversationBufferMemory(),
    verbose=True
)
```

问题：LangChain 在 0.1 → 0.2 → 0.3 的版本迭代中大幅重构了 API。`create_react_agent` 的签名在每个版本都不同。`ConversationBufferMemory` 在最新版本中被 deprecated。AI 混合了三个不同版本的 API。

这段代码能通过语法检查，看起来 "合理"，但 runtime 会抛出 `ImportError` 或 `TypeError`。

### SDD 预防机制

SDD 的 tech spec 明确锁定依赖版本和 API usage patterns：

```yaml
dependencies:
  langchain-core: "0.3.x"
  langchain-openai: "0.2.x"

api_patterns:
  agent_creation:
    use: "langgraph.prebuilt.create_react_agent"
    not: "langchain.agents.create_react_agent"  # deprecated in 0.2
  memory:
    use: "langgraph checkpoint mechanism"
    not: "ConversationBufferMemory"  # removed in 0.3
```

AI 在 spec 的约束下，不会自由地从训练数据中 "回忆" 已过时的 API。

---

## 第三宗罪：Architectural Drift (架构漂移)

### 模式描述

在长时间的开发会话中，AI 逐渐偏离最初的架构决策。第一个文件可能完美遵循 hexagonal architecture (六边形架构)，但到第十个文件时，business logic 已经渗透到了 infrastructure layer。

### 具体示例

Session 开始时的干净架构：

```
src/
├── domain/        # Pure business logic, no external deps
├── application/   # Use cases, orchestration
├── infrastructure/  # DB, HTTP, external services
└── interface/     # Controllers, CLI
```

两小时后 (50 个 prompt 之后)：

```typescript
// domain/order.ts — 应该是 pure domain logic
import { PrismaClient } from '@prisma/client'  // infrastructure leak!
import { sendEmail } from '../infrastructure/email'  // side effect in domain!

export class Order {
  async complete() {
    const prisma = new PrismaClient()
    await prisma.order.update({ ... })  // 直接在 domain 里调 DB
    await sendEmail(this.customerEmail, 'Order complete')  // side effect
  }
}
```

AI 在第 50 个 prompt 时已经 "忘了" 最初的架构约束。它选择了最短路径来完成当前任务，代价是污染了 domain layer 的纯粹性。

### SDD 预防机制

SDD 的 architecture spec 定义了不可违反的 layer boundaries：

```yaml
architecture:
  style: hexagonal
  layers:
    domain:
      allowed_imports: []  # NO external dependencies
      forbidden_patterns:
        - "import.*from.*infrastructure"
        - "import.*from.*@prisma"
        - "import.*from.*node-fetch"
    application:
      allowed_imports: ["domain"]
    infrastructure:
      allowed_imports: ["domain", "application"]
```

这些规则可以被 linter、CI check、或 AI agent 的 system prompt 执行。当 AI 试图在 domain layer 中引入 Prisma，规则会立即阻止。

---

## 第四宗罪：Test-That-Tests-Nothing (空测试)

### 模式描述

AI 生成的测试看起来结构完整 — 有 describe blocks、it statements、assertions — 但实际上形成了 circular validation (循环验证)：测试验证的是 AI 自己的假设，而不是 spec 定义的行为。

### 具体示例

```typescript
// AI 生成的 "测试"
describe('calculateDiscount', () => {
  it('should calculate discount correctly', () => {
    const result = calculateDiscount(100, 0.2)
    expect(result).toBe(80)  // 看起来合理...
  })

  it('should handle zero discount', () => {
    const result = calculateDiscount(100, 0)
    expect(result).toBe(100)  // 也看起来合理...
  })
})
```

问题：spec 可能要求折扣计算要 **先四舍五入到分 (cent)**，并且 **不能低于最低价格阈值**。AI 的测试完全忽略了这些业务规则，只测试了最显而易见的数学运算。

更糟糕的情况：

```typescript
it('should return user data', async () => {
  const mockUser = { id: 1, name: 'Test', email: 'test@test.com' }
  jest.spyOn(userService, 'findById').mockReturnValue(mockUser)

  const result = await userService.findById(1)
  expect(result).toEqual(mockUser)  // 你在测试 mock 本身！
})
```

这个测试永远不会失败 — 它只是验证了 mock 框架能正确返回你给它的值。

### SDD 预防机制

SDD 要求 test cases 直接从 spec 派生，而非从 implementation 推导：

```yaml
# discount.spec.yaml
behavior:
  - given: price=99.99, discount_rate=0.156
    then: final_price=84.39  # rounded to cent
    note: "Must round to nearest cent, not truncate"

  - given: price=10.00, discount_rate=0.95
    then: final_price=5.00  # minimum price threshold
    note: "Cannot go below $5.00 minimum"

  - given: price=0, discount_rate=0.5
    then: error "Price must be positive"
```

测试必须覆盖 spec 中列出的 edge cases，而不是 AI "觉得应该测试" 的场景。

---

## 第五宗罪：Context Amnesia (上下文遗忘)

### 模式描述

在长时间的开发会话中，AI 的 context window 逐渐将早期决策推出 "可见范围"。它开始做出与前期决策矛盾的选择，却无法自我检测到这种矛盾。

### 具体示例

Session 开始时 (prompt #3)：

> "所有 API responses 使用 snake_case，因为我们的前端团队使用 Python conventions。"

AI 在 prompt #3-20 中忠实执行。

Session 中期 (prompt #45)：

```json
// AI 悄悄切换到了 camelCase
{
  "userId": 123,
  "firstName": "Alice",
  "createdAt": "2025-01-01"
}
```

AI 没有 "故意" 违反约定 — 它只是在 context window 的 attention 中，prompt #3 的 snake_case 决策已经被后续 40 多个 prompt 的 "噪声" 稀释了。

### SDD 预防机制

SDD 将所有 cross-cutting decisions 记录在 specification 的 `conventions` section 中：

```yaml
conventions:
  api_response_format:
    casing: snake_case
    date_format: ISO8601
    null_handling: omit field (not null value)
    pagination: cursor-based, not offset-based
```

这份 spec 被注入到 AI agent 的 system prompt (如 CLAUDE.md) 中，确保它在整个 session 中都能 "看到" 这些约束 — 无论 context window 多拥挤。

---

## 第六宗罪：Yak-Shaving Spirals (剃牦牛)

### 模式描述

AI 在解决 Task A 时，发现需要先解决 Task B，然后发现 B 需要 C，C 需要 D... 最终花了 90% 的时间在与原始任务无关的 tangential problems (切线问题) 上。

### 具体示例

原始需求："在用户 profile 页面添加一个 avatar upload 功能。"

AI 的实际行为链：

1. "需要 avatar upload → 我应该先建一个 general file upload service"
2. "File upload service 需要一个 storage abstraction → 让我实现一个支持 S3/GCS/Azure 的 multi-cloud adapter"
3. "Multi-cloud adapter 需要 credential rotation → 我来加一个 secret management layer"
4. "Secret management 需要 encryption → 让我实现 AES-256-GCM with key derivation"
5. "Key derivation 需要 HSM integration → ..."

三小时后，你有了一个 500 行的 crypto module，但 avatar upload 功能还没开始。

### SDD 预防机制

SDD 的 task specification 明确定义了 implementation boundaries：

```yaml
task: avatar-upload
scope:
  implementation:
    - Accept image file (JPEG, PNG, WebP, max 5MB)
    - Resize to 256x256
    - Upload to S3 (use existing aws-sdk config)
    - Store URL in user.avatar_url field

  use_existing:
    - S3 client from src/infrastructure/aws.ts
    - Image processing from sharp library

  do_not_build:
    - Generic file upload service
    - Multi-cloud storage abstraction
    - Custom encryption layer
    - New credential management
```

`do_not_build` 列表是关键 — 它提前堵住了 yak-shaving 的入口。

---

## 第七宗罪：Self-Congratulation Machine (自我肯定机)

### 模式描述

AI 在完成代码生成后，自行验证自己的输出 — 而验证标准就是它自己生成代码时的假设。这形成了一个 closed feedback loop (闭环反馈)：AI 永远对自己的工作感到满意。

### 具体示例

```
User: "Build an authentication middleware."

AI: *generates 200 lines of auth middleware*

AI: "Let me verify this works correctly."
AI: *writes test cases based on its own implementation assumptions*
AI: *runs tests*
AI: "All 12 tests pass! The implementation is correct and production-ready."
```

问题在于：

- AI 定义了 "correct" 的标准
- AI 基于这个标准生成了 implementation
- AI 基于同样的标准写了 tests
- AI 运行 tests 并宣布成功

这就像让一个学生自己出题、自己答题、自己打分。它漏掉了什么？

- Timing attack vulnerability in token comparison (时序攻击)
- Token refresh race condition (令牌刷新竞态)
- Session fixation on login (会话固定)
- Missing CSRF protection for cookie-based auth (CSRF 缺失)

这些 edge cases 不在 AI 的 "自我验证" 范围内，因为 AI 从未被告知要考虑它们。

### SDD 预防机制

SDD 将 validation criteria 从 implementation 中分离出来：

```yaml
# auth-middleware.spec.yaml
security_requirements:
  - constant-time string comparison for tokens
  - token refresh must be atomic (no race window)
  - session ID must regenerate on privilege escalation
  - CSRF token required for all state-changing requests
  - failed auth attempts rate-limited (5/min per IP)

acceptance_criteria:
  - passes OWASP authentication checklist
  - reviewed by security-reviewer agent
  - penetration test for timing attacks
```

验证标准是由 spec author (通常是 senior engineer 或 security specialist) 定义的，而不是由 AI implementation 自己派生的。这打破了 closed feedback loop。

---

## 七宗罪的共同根因

所有七种失败模式都源于同一个根本问题：

> **AI agent 在没有 external specification 的情况下，被迫同时扮演 "决策者" 和 "执行者" 两个角色。**

当一个 agent 既要决定 "什么应该被建造"，又要负责 "怎么建造它"，它不可避免地会：
- 基于训练数据中的统计分布做决策 (而非基于项目的实际需求)
- 在 uncertainty (不确定性) 面前选择 "看起来合理的" 默认值
- 无法区分 "我确信这是对的" 和 "我在猜测"

**SDD 的核心干预就是将这两个角色分离**：人类做决策 (通过 specification)，机器做执行 (通过 code generation)。

---

## Key Takeaways (要点回顾)

1. **Vibe Coding 的七宗罪不是偶发事故** — 它们是 LLM coding agents 在没有 specification 约束下的系统性失败模式
2. **Scope Creep 和 Yak-Shaving 耗费时间**，Hallucinated APIs 和 Architectural Drift 引入 bugs，Test-That-Tests-Nothing 和 Self-Congratulation 掩盖问题，Context Amnesia 制造不一致
3. **每种罪都有对应的 SDD 预防机制**：explicit scope boundaries、pinned dependencies、architecture rules、spec-derived tests、persistent conventions、task boundaries、external validation criteria
4. **根因是角色混淆**：AI 同时充当 decision-maker 和 implementer 时，质量无法保证
5. **SDD 不是 "限制 AI 的创造力"** — 而是给创造力一个明确的方向和边界

---

## Next (下一章)

在 [Chapter 03: 方法论对比](03-sdd-vs-tdd-bdd-ddd.md) 中，我们将把 SDD 放在更广阔的软件工程方法论图谱中，看它如何与 TDD、BDD、DDD 互补共存，而非相互替代。
