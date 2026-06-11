# 编写有效规格 (Writing Effective Specifications)

> 好的 Spec 精确到 AI 可以直接实现，同时清晰到人类可以快速审查——EARS 符号法是达成这一平衡的利器。

---

## Spec 的黄金标准

什么才算"有效"的 Spec？答案是同时满足两个约束：

1. **对 AI 足够精确**：Claude Code 读完后能产出确定性的实现，不需要反复追问
2. **对人类足够可读**：团队成员 5 分钟内能理解并审查

这两个约束看似矛盾——精确往往意味着冗长，可读往往意味着模糊。EARS 符号法（Easy Approach to Requirements Syntax）提供了一个优雅的解决方案：用固定的句式模板来表达需求，既保持了自然语言的可读性，又通过结构化语法消除了歧义。

---

## EARS 符号法详解

EARS 由 Alistair Mavin 于 2009 年在 Rolls-Royce 创建，最初用于航空航天系统的需求工程。它的核心思想是：**用固定的 keyword trigger（关键词触发器）区分不同类型的需求**。

### 五种基本句型

#### 1. Ubiquitous（普遍型）

```
The system shall [action].
```

- **无触发条件**，系统应始终满足
- 适合表达不变量（invariants）和基础约束

```markdown
REQ-001: The system shall encrypt all data at rest using AES-256.
REQ-002: The system shall respond to any API request within 500ms (p99).
REQ-003: The system shall log all authentication attempts with timestamp and IP.
```

#### 2. Event-Driven（事件驱动型）

```
WHEN [event], the system shall [action].
```

- **一次性触发**：某个事件发生时执行一次
- 适合表达用户操作的响应、webhook 处理等

```markdown
REQ-010: WHEN a user submits the login form, the system shall validate
         credentials against the user database.
REQ-011: WHEN a payment webhook is received, the system shall update the
         order status to "paid" and emit an OrderPaid event.
REQ-012: WHEN a file upload exceeds 10MB, the system shall reject the
         request with HTTP 413 and message "File size exceeds limit".
```

#### 3. State-Driven（状态驱动型）

```
WHILE [state], the system shall [action].
```

- **持续行为**：只要系统处于某状态就一直执行
- 适合表达后台任务、监控行为、UI 状态

```markdown
REQ-020: WHILE the WebSocket connection is active, the system shall send
         a heartbeat ping every 30 seconds.
REQ-021: WHILE the user has an active subscription, the system shall
         allow access to premium features.
REQ-022: WHILE the system is in maintenance mode, the system shall return
         HTTP 503 for all API requests with a retry-after header.
```

#### 4. Optional Feature（可选功能型）

```
WHERE [feature is enabled], the system shall [action].
```

- **条件激活**：仅当某功能开关打开时才生效
- 适合表达 feature flags、配置项、许可级别

```markdown
REQ-030: WHERE two-factor authentication is enabled, the system shall
         require a TOTP code after password verification.
REQ-031: WHERE the AUDIT_LOG feature flag is active, the system shall
         record all write operations to the audit_events table.
REQ-032: WHERE the user's plan includes "API access", the system shall
         issue API keys upon request.
```

#### 5. Unwanted Behaviour（异常行为型）

```
IF [unwanted condition], THEN the system shall [action].
```

- **防御性处理**：当意外或错误发生时的应对
- 适合表达错误处理、降级策略、安全防护

```markdown
REQ-040: IF the database connection pool is exhausted, THEN the system
         shall queue incoming requests for up to 5 seconds before
         returning HTTP 503.
REQ-041: IF a user fails authentication 5 times within 10 minutes, THEN
         the system shall lock the account for 30 minutes and notify
         the user via email.
REQ-042: IF the external payment gateway is unreachable, THEN the system
         shall retry 3 times with exponential backoff (1s, 2s, 4s) and
         then fail with a user-friendly error message.
```

---

## 复合句型 (Complex Combinations)

真实需求往往涉及多个条件。EARS 支持组合：

### WHILE + WHEN

```markdown
REQ-050: WHILE the user is on the dashboard page, WHEN new data arrives
         via WebSocket, the system shall update the displayed metrics
         without full page reload.
```

### WHERE + IF

```markdown
REQ-051: WHERE rate limiting is enabled, IF a client exceeds 100 requests
         per minute, THEN the system shall return HTTP 429 with a
         X-RateLimit-Reset header indicating when to retry.
```

### WHILE + IF

```markdown
REQ-052: WHILE the system is processing a batch import, IF any single
         record fails validation, THEN the system shall skip that record,
         log the error with row number and reason, and continue processing
         remaining records.
```

**编写组合句型的原则**：最外层是时间/状态条件（WHILE/WHERE），内层是触发/异常条件（WHEN/IF）。保持同一条需求只表达一个行为。

---

## Acceptance Criteria（验收标准）

EARS 需求定义"系统应做什么"，Acceptance Criteria 定义"如何验证它做到了"。使用 **Given/When/Then** 格式，并填入具体的值：

```markdown
### REQ-041 的验收标准

**Scenario: Account lockout after failed attempts**

Given a user "alice@example.com" with a valid account
When the user submits incorrect password 5 times within 10 minutes
Then the account status changes to "locked"
And the lock expires after exactly 30 minutes
And an email is sent to "alice@example.com" with subject containing "account locked"
And subsequent login attempts (even with correct password) return HTTP 403
  with body {"error": "account_locked", "retryAfter": <seconds_remaining>}

**Scenario: Counter reset after successful login**

Given a user has failed authentication 3 times
When the user successfully authenticates
Then the failure counter resets to 0
```

注意 Given/When/Then 中使用了**具体值**（`5 times`、`10 minutes`、`30 minutes`、`HTTP 403`）而不是模糊描述。这让 Claude Code 可以直接将这些值写入测试用例。

---

## 完整 Spec 结构

一份 SDD Spec 的完整骨架：

```markdown
# Feature: [Feature Name]

## Overview
[2-3 句话说明这个功能是什么、为谁服务、解决什么问题]

## User Stories
- As a [role], I want to [action], so that [benefit].
- As a [role], I want to [action], so that [benefit].

## Requirements (EARS)

### Ubiquitous
- REQ-001: The system shall ...
- REQ-002: The system shall ...

### Event-Driven
- REQ-010: WHEN ..., the system shall ...

### State-Driven
- REQ-020: WHILE ..., the system shall ...

### Optional Features
- REQ-030: WHERE ..., the system shall ...

### Unwanted Behaviour
- REQ-040: IF ..., THEN the system shall ...

## Acceptance Criteria

### REQ-010
Given ...
When ...
Then ...

## Constraints & Assumptions
- [技术约束：如必须兼容现有 PostgreSQL 14 数据库]
- [性能约束：如 p99 latency < 200ms]
- [假设：如用户已完成邮箱验证]

## Out of Scope
- [明确列出不做的事情]
- [防止 scope creep 的关键部分]

## Dependencies
- [依赖的其他 feature 或服务]
- [依赖的第三方 API]
```

### Out of Scope 的重要性

**Out of Scope（范围之外）** 是 Spec 中最容易被忽略、却最重要的部分之一。它的作用是：

- 防止 Claude Code "自作主张"添加你没要求的功能
- 让审查者明确知道哪些事情是刻意不做的
- 避免后续的 scope creep（范围蔓延）

```markdown
## Out of Scope

- OAuth / social login（将在 v2 独立 spec 中处理）
- Password reset via SMS（仅支持 email）
- Session management for mobile clients（本 spec 仅覆盖 web）
- Admin ability to unlock accounts manually（使用 CLI tool 处理）
```

---

## 常见错误 (Common Mistakes)

### 1. 过于模糊

```markdown
<!-- 错误 -->
REQ-X: The system should handle errors gracefully.

<!-- 正确 -->
REQ-X: IF an unhandled exception occurs in any request handler, THEN the
       system shall return HTTP 500 with body {"error": "internal_error",
       "requestId": "<uuid>"} and log the full stack trace at ERROR level.
```

### 2. 过于实现化

```markdown
<!-- 错误：指定了实现细节 -->
REQ-X: The system shall use a Redis SORTED SET with score = timestamp
       to implement the rate limiter.

<!-- 正确：指定行为，不指定实现 -->
REQ-X: IF a client exceeds 100 requests per minute, THEN the system
       shall return HTTP 429. The rate limit window is sliding.
```

除非有明确的技术约束（如必须使用特定中间件），否则 Spec 应该描述 **what**（做什么）而不是 **how**（怎么做）。让 Claude Code 自由选择最佳实现。

### 3. 缺少边界条件

```markdown
<!-- 不完整 -->
REQ-X: WHEN a user uploads a file, the system shall store it.

<!-- 完整 -->
REQ-X: WHEN a user uploads a file, the system shall store it in the
       configured storage backend with a unique identifier.
REQ-X+1: IF the uploaded file exceeds 50MB, THEN the system shall reject
         with HTTP 413.
REQ-X+2: IF the uploaded file type is not in [jpg, png, pdf, docx], THEN
         the system shall reject with HTTP 415.
REQ-X+3: IF storage write fails, THEN the system shall retry once and
         then return HTTP 502 with error details.
```

### 4. 遗漏 Out of Scope

如果你不说"不做什么"，Claude Code 可能会根据常识"帮你"加上额外功能。比如你写了 "user login" 的 Spec，它可能顺便加上密码重置、邮箱验证、remember me——这些都不在你本次迭代的计划中。

---

## 实战练习：从模糊到精确

**原始需求**："加一个用户登录功能"

让我们将其转化为完整的 EARS Spec：

```markdown
# Feature: User Authentication (Email/Password)

## Overview

允许已注册用户通过邮箱和密码登录系统，获取 JWT access token
用于后续 API 调用的身份验证。

## User Stories

- As a registered user, I want to log in with my email and password,
  so that I can access my personal data and features.
- As a system administrator, I want failed login attempts to be tracked,
  so that I can detect brute-force attacks.

## Requirements (EARS)

### Ubiquitous
- REQ-001: The system shall store passwords hashed with bcrypt (cost factor 12).
- REQ-002: The system shall never return password hashes in any API response.
- REQ-003: The system shall issue JWT tokens with a maximum lifetime of 24 hours.

### Event-Driven
- REQ-010: WHEN a user submits valid credentials to POST /auth/login,
  the system shall return HTTP 200 with an access token and refresh token.
- REQ-011: WHEN a user submits valid credentials, the system shall reset
  the failed attempt counter to 0.
- REQ-012: WHEN a login succeeds, the system shall log the event with
  userId, timestamp, and IP address.

### State-Driven
- REQ-020: WHILE an account is locked, the system shall reject all login
  attempts with HTTP 403 and body including "retryAfter" in seconds.

### Unwanted Behaviour
- REQ-040: IF credentials are invalid, THEN the system shall return
  HTTP 401 with body {"error": "invalid_credentials"}. The response
  shall not indicate whether the email or password was incorrect.
- REQ-041: IF a user fails authentication 5 times within 15 minutes,
  THEN the system shall lock the account for 30 minutes.
- REQ-042: IF the request body is malformed (missing email or password),
  THEN the system shall return HTTP 400 with validation error details.
- REQ-043: IF the JWT signing key is unavailable, THEN the system shall
  return HTTP 500 and emit a CRITICAL alert.

## Acceptance Criteria

### REQ-010: Successful Login
Given a user with email "test@example.com" and password "Str0ngP@ss!"
When POST /auth/login with body {"email": "test@example.com", "password": "Str0ngP@ss!"}
Then response status is 200
And response body contains "accessToken" (valid JWT, expires in 24h)
And response body contains "refreshToken" (opaque string, expires in 7d)

### REQ-041: Account Lockout
Given a user with email "test@example.com"
When POST /auth/login with wrong password is sent 5 times within 15 minutes
Then the 6th attempt returns HTTP 403
And response body contains {"error": "account_locked", "retryAfter": <number>}
And after 30 minutes, login with correct password succeeds

## Constraints & Assumptions

- Assumes user registration is handled by a separate feature (users already exist in DB)
- JWT signing uses RS256 with key pair stored in environment variables
- Refresh token rotation: each use of a refresh token invalidates the old one
- All timestamps in UTC

## Out of Scope

- User registration / sign-up (separate spec)
- OAuth / social login (planned for v2)
- Password reset / forgot password (separate spec)
- Multi-factor authentication (separate spec)
- Session management / "remember me" (token-based only)
- Email verification at login time (handled at registration)

## Dependencies

- User table must exist with columns: id, email, password_hash, status, failed_attempts, locked_until
- Environment variables: JWT_PRIVATE_KEY, JWT_PUBLIC_KEY
- Logging service must be available
```

注意这份 Spec 从一句"加个登录"变成了约 80 行的精确规格。Claude Code 读完后可以直接开始 TDD 实现——不需要追问任何细节。

---

## Key Takeaways (要点回顾)

- EARS 符号法用五种固定句型消除需求歧义：Ubiquitous、Event-Driven、State-Driven、Optional Feature、Unwanted Behaviour
- 每条需求必须是可验证的——配合 Given/When/Then 格式的 Acceptance Criteria
- Out of Scope 与需求本身同等重要，防止 AI 和人类产生范围蔓延
- Spec 描述 what（行为），不是 how（实现）——留给 Claude Code 选择最佳方案
- 从模糊到精确的转化是 SDD 的核心技能，决定了 AI 输出的确定性

---

## Next (下一章)

[Ch.08 子代理系统](08-subagents.md) — 学习如何用 Subagent Architecture 将复杂任务分解给专职 AI 代理并行执行。
