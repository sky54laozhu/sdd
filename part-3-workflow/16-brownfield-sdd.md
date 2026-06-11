# Chapter 16: 存量项目改造 (Brownfield SDD: Adding Specs to Existing Projects)

> 大多数项目不是从零开始的——它们带着历史债务、隐式约定和零散文档。Brownfield SDD 不要求你重写历史，而是"规格化增量"：只 spec 你正在改变的部分。

---

## 为什么完整逆向工程行不通

新团队成员加入时，常见的第一反应是："让我们先把整个系统文档化。" 这听起来合理，但在实践中几乎总是失败：

```
全量逆向工程的死循环：

┌────────────┐     需要 2 周     ┌─────────────┐
│ 决定文档化 │───────────────▶│ 开始分析     │
│ 整个系统   │                  │ 代码库       │
└────────────┘                  └──────┬──────┘
                                       │
                                       ▼ 发现比预期复杂
                              ┌─────────────┐
                              │ 3 周后完成   │
                              │ 60% 文档    │
                              └──────┬──────┘
                                     │
                                     ▼ 这时代码已经变了
                              ┌─────────────┐
                              │ 文档已过时   │
                              │ 放弃维护     │
                              └─────────────┘
```

**根本原因**：文档的 decay rate（腐化速度）超过了它的 creation rate（创建速度）。你无法赢得这场竞赛。

---

## 增量方法："Spec the Delta"

Brownfield SDD 的核心原则是：**只 spec 你正在改变的部分（spec the delta）**。

```
不要这样做：
"让我们为这个 50,000 行的项目写完整的 spec"

要这样做：
"让我们为接下来要加的通知功能写一份 spec"
```

这意味着：
- 第一天的 spec 覆盖率可能是 2%（只有新功能有 spec）
- 第 100 天可能是 30%（频繁改动的部分都有了）
- 永远不会达到 100%——稳定且不再改动的代码不需要 spec

**这是刻意的设计，不是懒惰。** Spec 的价值在于指导变更，稳定代码不需要变更指导。

---

## Brownfield SDD 工作流

### Phase 0: Explore（探索）

在写任何 spec 之前，先理解你要改动的区域。

```bash
# Claude Code: 探索现有架构
claude "Act as a researcher agent. Analyze the following:
1. Directory structure and file organization pattern
2. Existing patterns for authentication/authorization  
3. Database schema (check migrations, models, or schema files)
4. API routing conventions (path patterns, middleware usage)
5. Testing patterns (framework, file naming, coverage)
6. Error handling patterns (custom errors, status codes)

Focus on the areas that will be affected by adding a notification feature.
Output a structured summary I can reference during planning."
```

**关键洞察**：Explore 不是逆向工程整个系统——它聚焦在你即将改动的邻域（neighborhood）。

### Phase 1: Document（文档化当前约定）

创建一份 **minimal CLAUDE.md**——不是完整文档，而是"当前约定的快照"。

```bash
# Claude Code: 自动生成 CLAUDE.md 初稿
claude "Based on your analysis of this codebase, draft a minimal CLAUDE.md that captures:
1. Tech stack (frameworks, runtime, database)
2. Project structure conventions (where things live)
3. Naming patterns (file names, function names, variables)
4. Error handling approach
5. Testing conventions

Keep it under 100 lines. Only document what you can verify from code, 
not what you assume. Mark uncertain items with [VERIFY]."
```

初稿可能像这样：

```markdown
# CLAUDE.md (Auto-generated, needs human review)

## Tech Stack
- Runtime: Node.js 18 (from package.json engines)
- Framework: Express.js 4.x with JavaScript (no TypeScript)
- Database: MongoDB with Mongoose ODM
- Testing: Jest + Supertest

## Project Structure
src/
├── routes/          # Express routers (one per resource)
├── controllers/     # Request handlers
├── models/          # Mongoose schemas
├── middleware/      # Auth, validation, error handling
├── services/        # Business logic [VERIFY: inconsistent usage]
└── utils/           # Helper functions

## Conventions
- Route files: kebab-case (user-routes.js)
- Controllers: camelCase (userController.js)
- Models: PascalCase (User.js)
- Error handling: custom AppError class thrown to global error middleware
- Auth: JWT in Authorization header, verified by auth middleware

## Testing
- Test files: __tests__/[module].test.js
- Coverage: currently ~45% [VERIFY]
- Pattern: mostly integration tests, few unit tests
```

### Phase 2: Spec the Change（规格化变更）

为新功能写 EARS spec——参考现有代码的约定：

```markdown
# Notification Feature Spec

## Context (existing system reference)
- Auth: JWT-based, user ID available in req.user.id
- Database: MongoDB/Mongoose, existing User model
- API pattern: /api/v1/[resource], REST conventions

## Requirements (EARS notation)

### Event-Driven
- When a user receives a new follower, the system shall create 
  an in-app notification with type "new_follower" and the follower's user ID.
- When a user's post receives a comment, the system shall create 
  an in-app notification with type "new_comment" and the comment ID.

### State-Driven
- While a user has unread notifications, the system shall return 
  unread_count > 0 in the GET /api/v1/notifications response.

### Ubiquitous
- The system shall limit notification history to 90 days.
- The system shall return notifications in reverse chronological order.

### Unwanted
- If a user has muted another user, the system shall NOT create 
  notifications from the muted user.

## Acceptance Criteria
- [ ] GET /api/v1/notifications returns paginated list (20 per page)
- [ ] GET /api/v1/notifications includes unread_count field
- [ ] PATCH /api/v1/notifications/:id/read marks as read, decrements unread_count
- [ ] Notifications older than 90 days are automatically pruned
- [ ] Muted users do not generate notifications
- [ ] Response time < 200ms for notification list (indexed query)

## Out of Scope
- Push notifications (mobile/browser) — separate spec
- Email notifications — separate spec  
- Notification preferences/settings — planned for next sprint
- Real-time WebSocket delivery — separate spec
```

### Phase 3: Plan the Delta（规划增量）

Plan 只描述**新增和修改的部分**，但引用现有代码作为 context：

```markdown
# Notification Feature Plan

## New Modules
src/
├── models/Notification.js        # NEW: Mongoose schema
├── routes/notification-routes.js # NEW: Express router
├── controllers/notificationController.js # NEW
├── services/notificationService.js       # NEW
└── middleware/notification-emitter.js    # NEW: event listener

## Modified Modules
- src/routes/index.js — add notification routes mount
- src/controllers/commentController.js — emit notification event after comment
- src/controllers/followController.js — emit notification event after follow

## Data Model (new collection)
```javascript
// Notification schema
{
  _id: ObjectId,
  userId: ObjectId (indexed),     // notification recipient
  type: String (enum: ['new_follower', 'new_comment']),
  sourceUserId: ObjectId,         // who triggered it
  referenceId: ObjectId,          // follower ID or comment ID
  read: Boolean (default: false),
  createdAt: Date (indexed, TTL: 90 days)
}
```

## Architecture Decision
- **Decision**: Event emitter pattern (emit in controller, listen in service)
- **Rationale**: Existing codebase has no event system, but adding EventEmitter 
  is minimal change; keeps notification logic decoupled from core features
- **Alternative rejected**: Direct function calls (too coupled, hard to test)
```

### Phase 4: Implement with Context（带上下文实现）

Tasks 同时引用 spec 和现有代码：

```markdown
## Task List

### Task 1: Create Notification Mongoose model
- **Spec ref**: AC (pagination, 90-day TTL, unread_count)
- **Existing reference**: Follow pattern of src/models/User.js
- **Action**: Create Notification.js with schema, indexes, TTL
- **Acceptance**: Model loads without errors, indexes created

### Task 2: Write notification service tests
- **Spec ref**: AC (create notification, muted users filter)
- **Existing reference**: Test pattern in __tests__/userService.test.js
- **Action**: Create __tests__/notificationService.test.js
- **Acceptance**: Tests fail (no implementation)

### Task 3: Implement notification service
- **Spec ref**: AC (create, list, mark read)
- **Existing reference**: Follow pattern of src/services/ (if exists)
- **Action**: Create notificationService.js
- **Acceptance**: Task 2 tests pass

### Task 4: Write route integration tests
- **Spec ref**: AC (GET paginated, PATCH mark read)
- **Existing reference**: Test pattern in __tests__/userRoutes.test.js
- **Action**: Create __tests__/notificationRoutes.test.js
- **Acceptance**: Tests fail

### Task 5: Implement routes + controller
- **Spec ref**: AC (endpoints, response format)
- **Existing reference**: src/routes/user-routes.js, src/controllers/userController.js
- **Action**: Create notification routes and controller
- **Acceptance**: Task 4 tests pass

### Task 6: Add event emission to existing controllers
- **Spec ref**: Event-Driven requirements
- **Existing reference**: src/controllers/commentController.js line ~45
- **Action**: Add EventEmitter emit calls after follow/comment actions
- **Acceptance**: Notification created when follow/comment occurs

### Task 7: Regression tests for modified modules
- **Spec ref**: Unwanted requirement (muted users)
- **Action**: Add tests verifying existing follow/comment behavior unchanged
- **Acceptance**: All existing tests still pass + new regression tests pass
```

### Phase 5: Verify（验证新代码 + 回归测试）

```bash
# Claude Code: 验证实现
claude "Run full verification:
1. All new tests pass (notification module)
2. All existing tests pass (regression check)
3. Coverage for new code >= 80%
4. Performance: notification list query uses index (explain plan)
5. Verify muted user filter works
6. Verify 90-day TTL is configured correctly"
```

---

## Bootstrap 宪法：从零到 CLAUDE.md

对于完全没有文档的存量项目，以下是 bootstrap 流程：

### Step 1: 自动分析

```bash
claude "Analyze this entire codebase and generate a CLAUDE.md draft.
Focus on:
1. Observable patterns (not assumptions)
2. Conventions that are consistently followed (not one-offs)
3. Tech stack versions (from lockfile/config)
4. Architecture style (if identifiable)

Mark anything uncertain with [NEEDS VERIFICATION]."
```

### Step 2: 人工精炼

自动生成的 CLAUDE.md 需要人工审查：

- 删除不确定的推断（标记了 [NEEDS VERIFICATION] 的）
- 补充只有人知道的约定（"我们从不用 ORM 做报表查询"）
- 确认 tech stack 版本准确
- 添加安全规则（这些通常不能从代码推断）

### Step 3: 迭代精进

CLAUDE.md 不需要第一天就完美。**当你观察到 AI drift（AI 行为偏离期望）时，添加规则：**

```markdown
# 观察到的 drift → 添加的规则

## Week 1
AI drift: Agent 使用了 callback 风格而非 async/await
Added rule: "All async code uses async/await, never callbacks"

## Week 2  
AI drift: Agent 创建了 300 行的 controller 文件
Added rule: "Controllers only handle HTTP concerns, < 50 lines per handler"

## Week 3
AI drift: Agent 在 service 层直接返回 Mongoose documents
Added rule: "Service layer returns plain objects, not Mongoose documents"
```

这种"观察 drift → 添加规则"的循环是 CLAUDE.md 最健康的成长方式。

---

## 衡量 Brownfield SDD 进展

### Metric 1: Spec Density（规格密度）

```
Spec Density = (有 spec 的最近变更数) / (最近变更总数) × 100%

目标：
- Month 1: > 50% (新功能有 spec)
- Month 3: > 70% (重大改动都有 spec)
- Month 6: > 80% (稳定运行)
```

### Metric 2: Drift Rate（漂移率）

```
Drift Rate = (实现偏离 spec 的次数) / (有 spec 的实现总数) × 100%

健康值: < 10%
警告值: 10-25% (spec 可能太模糊)
危险值: > 25% (spec 流程可能有问题)
```

### Metric 3: Review Efficiency（审查效率）

```
引入 SDD 前: 平均 code review 时间 = 45 分钟
引入 SDD 后: 平均 code review 时间 = 20 分钟 (因为 spec 已预审批)

Improvement = (45 - 20) / 45 = 55% faster reviews
```

### 追踪 Dashboard 示例

```markdown
## SDD Health Dashboard (Week of 2025-01-13)

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| Spec Density | 65% | >70% | ⚠️ Close |
| Drift Rate | 8% | <10% | ✅ Healthy |
| Review Time | 22min | <25min | ✅ Healthy |
| Test Coverage (new code) | 84% | >80% | ✅ Healthy |
| CLAUDE.md Rules | 47 | <60 | ✅ Manageable |
```

---

## 何时不使用 SDD

SDD 不是银弹。以下场景中，SDD 的开销超过了它的收益：

### 一次性脚本

```bash
# 不需要 spec 的例子：
# 数据迁移脚本，跑一次就删除
node scripts/migrate-user-emails.js
```

如果代码的预期生命周期 < 1 天，写 spec 是浪费。

### 探索性原型（Throwaway Prototypes）

当你还在验证"这个方向可行吗？"时，spec 会约束探索空间。**先原型，验证可行后再 spec。**

```
正确流程：
1. 原型探索（无 spec，快速试错）
2. 验证可行性
3. 丢弃原型
4. 用 SDD 从 Specify 开始正式实现

错误流程：
1. 为不确定的方向写详细 spec
2. 方向不可行
3. Spec 全部作废
```

### 紧急热修复（Emergency Hotfix）

生产环境 P0 故障时：

```
紧急流程：
1. 修复 → 测试 → 部署 (10 分钟)
2. 事后补 spec + 回归测试 (事后 1 小时内)

不要：
1. 写 spec → 过 gate → 写 plan → 分 task → 实现 (2 小时后系统还在挂)
```

### 判断标准

```
使用 SDD 当：
- 变更影响 > 1 个文件
- 变更涉及业务逻辑
- 变更需要 code review
- 变更会存在超过 1 周

跳过 SDD 当：
- 一行 config fix
- 纯依赖升级（无行为变更）
- 一次性脚本
- 原型验证
- 紧急修复（事后补 spec）
```

---

## 实战示例：为 Express.js API 添加通知功能

假设你加入了一个运行 6 个月的 Express.js 项目，要添加通知功能。

### Day 1: Bootstrap

```bash
# 1. 自动分析
claude "Analyze this Express.js project. Generate CLAUDE.md covering:
tech stack, project structure, naming conventions, error handling, testing."

# 2. 人工审查 10 分钟，确认/修正自动生成的内容
# 3. Commit CLAUDE.md
git add CLAUDE.md
git commit -m "docs: bootstrap CLAUDE.md for SDD adoption"
```

### Day 1-2: Spec + Plan

```bash
# 4. 写 notification spec（参考上面的 Spec 示例）
claude "Help me write an EARS-formatted spec for in-app notifications.
Requirements: follower notifications, comment notifications, 
muted user filtering, 90-day retention, paginated API."

# 5. Gate 1 审查（自审 + AI 验证）
claude "Review this spec against Gate 1 criteria. Is it ready for planning?"

# 6. 写 Plan（参考上面的 Plan 示例）
claude "Based on the spec and existing codebase conventions, 
create an implementation plan for notifications."

# 7. Gate 2 审查
claude "Cross-reference plan against spec. Any requirements without modules?"
```

### Day 2-3: Tasks + Implement

```bash
# 8. 生成 task list
claude "Decompose the notification plan into atomic tasks.
Order: infrastructure → data → tests → implementation → integration.
Each task must reference a spec AC."

# 9. Gate 3 审查
claude "Validate task list: atomic? TDD order? Spec refs present?"

# 10. 执行 tasks（TDD 内循环）
claude --agent tdd-guide "Execute Task 1: Create Notification Mongoose model.
Follow existing pattern in src/models/User.js.
Spec ref: TTL 90 days, indexed userId field."

# ... 重复 Task 2-7 ...

# 11. Gate 4 最终验证
claude "Final verification: all tests pass, coverage >= 80% for new code, 
all existing tests still pass, no regressions."
```

### 结果

- **新增文件**: 6 (model, service, controller, routes, 2 test files)
- **修改文件**: 3 (index routes, comment controller, follow controller)
- **Spec coverage**: 100% of notification ACs have tests
- **Regression**: 0 existing tests broken
- **Time**: ~2 days (vs. estimated 3-4 days without SDD, due to fewer bugs and rework)

---

## 从 Brownfield 到 Greenfield 心态

随着 spec density 增长，你的项目会逐渐从 brownfield（存量混沌）向 greenfield（规格驱动）转变：

```
Spec Density 进化：

[░░░░░░░░░░] 0%   — 传统项目，无 spec
[██░░░░░░░░] 20%  — 新功能有 spec，存量无
[████░░░░░░] 40%  — 活跃模块逐渐覆盖
[██████░░░░] 60%  — 大多数变更有 spec
[████████░░] 80%  — 接近 greenfield 效率
[██████████] 100% — 所有活跃代码有 spec（理想但非必需）
```

**不要追求 100%**。稳定且不变的代码不需要 spec——那是成本没有收益。瞄准活跃变更的 80%+ 覆盖即可。

---

## Key Takeaways (要点回顾)

1. **Spec the delta**：只规格化你正在改变的部分，不要逆向工程整个系统
2. **Bootstrap CLAUDE.md**：自动分析 + 人工精炼 + 观察 drift 持续迭代
3. **Explore 先于 Spec**：理解邻域代码是写好增量 spec 的前提
4. **衡量进展用三个指标**：Spec Density、Drift Rate、Review Efficiency
5. **知道何时不用 SDD**：一次性脚本、原型探索、紧急修复不适合完整流程

---

## Next (下一章)

[Chapter 17: 常见陷阱与解法 (Common Pitfalls and How to Avoid Them)](./17-pitfalls.md) — 学习 SDD 实践中最常见的 10 个陷阱，以及如何识别和避免它们。
