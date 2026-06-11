# Chapter 20: 编写功能规格 (Writing Feature Specifications)

> Spec 是 SDD 方法论中信息密度最高的文档——它用 EARS 句式将模糊的需求转化为可测试的约束，使 Claude Code 能够产出确定性的实现。

---

## 本章目标

基于已完成的 Constitution（Chapter 19），撰写 SpecTask 的两份核心功能规格：

1. **Task CRUD Spec** — 任务的创建、查看、更新、删除（核心功能）
2. **Auth Spec** — 本地用户注册与认证（基础设施功能）

**产出物**：
- `project/specs/task-crud-spec.md`
- `project/specs/auth-spec.md`

---

## 先写哪个 Spec？

这不是随机决策。选择 spec 撰写顺序的原则是：**先写最核心、被依赖最多的功能**。

在 SpecTask 中：
- Task CRUD 是项目的核心价值——没有任务管理，其他功能无意义
- Auth 是基础设施——Task CRUD 需要知道"谁在操作"
- Spec Linking、Notifications、Hooks 都构建在 Task CRUD 之上

因此撰写顺序为：Task CRUD → Auth → 其余。但要注意：**撰写 Task CRUD spec 时会发现它依赖 Auth 的 userId**，这正好驱动我们接下来撰写 Auth spec。依赖发现（dependency discovery）是 spec 撰写过程的自然副产物。

---

## Task CRUD Spec 撰写过程

### Step 1: Feature Overview（功能概述）

从用户视角描述功能的核心价值。注意不要写实现细节——这里是"做什么"而非"怎么做"。

**思考过程**：SpecTask 的 Task CRUD 有什么区别于 todo.txt 或 Todoist 的独特之处？答案是 **spec linking** 和 **structured metadata**（priority、tags、due dates）。Overview 应该点明这一差异。

### Step 2: User Stories（用户故事）

用 "As a... I want... so that..." 格式描述用户需求。User stories 不是 requirements——它们是需求的**来源**，帮助我们理解背后的动机。

**写作技巧**：每个 story 应对应一个可独立交付的功能切片。如果一个 story 需要拆成多个 PR 才能完成，它太大了。

### Step 3: EARS Requirements（EARS 格式需求）

这是 spec 的核心——用 EARS 五种句式精确表达每一条需求。

**撰写策略**：
1. 先列出所有 Ubiquitous requirements（始终成立的约束）
2. 然后按功能操作（create、read、update、delete）写 Event-Driven requirements
3. 识别系统状态约束，写 State-Driven requirements
4. 添加 Optional features（可选行为）
5. 最后处理错误场景——Unwanted behavior handling

### Step 4: Acceptance Criteria（验收标准）

Given/When/Then 格式的验收标准直接映射为测试用例。Claude Code 在 TDD 阶段会直接从这些 AC 生成测试代码。

---

## 完整 Task CRUD Spec

以下是 `project/specs/task-crud-spec.md` 的完整内容。基于 [spec template](../templates/spec-template.md) 定制：

```markdown
# Task CRUD Specification

## Metadata / 元数据

| Field | Value |
|-------|-------|
| **Spec ID** | SPEC-001 |
| **Feature** | Task CRUD (Create, Read, Update, Delete) |
| **Author** | SDD Tutorial |
| **Created** | 2024-12-01 |
| **Status** | Approved |
| **Priority** | P0-Critical |

---

## Feature Overview / 功能概述

Task CRUD is the core feature of SpecTask. It enables users to create, list, view, update, and delete tasks through a CLI interface. Each task carries structured metadata including priority level, tags, due dates, and an optional reference to the specification file that originated it. This spec-linking capability distinguishes SpecTask from generic task managers by providing full traceability from requirements to implementation tasks.

---

## User Stories / 用户故事

1. As a developer, I want to create tasks with priority and tags, so that I can organize my work by importance and category.
2. As a developer, I want to list tasks with filters (status, priority, tag), so that I can focus on what matters right now.
3. As a developer, I want to link a task to its origin spec file, so that I can trace why a task exists and verify it against requirements.
4. As a developer, I want to update task status and properties, so that I can track progress without recreating tasks.
5. As a developer, I want to delete tasks I no longer need, so that my task list stays clean and relevant.

---

## Requirements / 需求 (EARS Format)

### Ubiquitous Requirements / 普适型需求

- **REQ-U01**: The system shall store each task with the following fields: id (string), title (string), description (string, optional), status (enum), priority (enum), tags (string array), dueDate (ISO 8601 string, optional), specRef (file path string, optional), createdAt (ISO 8601 string), updatedAt (ISO 8601 string).

- **REQ-U02**: The system shall associate every task with exactly one userId representing the currently authenticated user.

- **REQ-U03**: The system shall generate a unique 21-character nanoid for each new task's id field.

- **REQ-U04**: The system shall persist all task data in a SQLite database file at `~/.spectask/data.db`.

- **REQ-U05**: The system shall enforce the following status enum values: 'pending', 'in-progress', 'completed', 'archived'.

- **REQ-U06**: The system shall enforce the following priority enum values: 'low', 'medium', 'high', 'critical'.

### Event-Driven Requirements / 事件驱动需求

- **REQ-E01**: When the user executes the `add` command with a title, the system shall create a new task with status 'pending', priority 'medium' (default), empty tags array, and current timestamp for createdAt and updatedAt.

- **REQ-E02**: When the user executes the `add` command with --priority, --tags, --due, or --spec flags, the system shall override the corresponding default values with the provided values.

- **REQ-E03**: When the user executes the `list` command, the system shall display all tasks belonging to the current user, sorted by createdAt descending.

- **REQ-E04**: When the user executes the `list` command with --status, --priority, or --tag filter flags, the system shall display only tasks matching ALL provided filters (AND logic).

- **REQ-E05**: When the user executes the `show <id>` command, the system shall display all fields of the specified task in a formatted view.

- **REQ-E06**: When the user executes the `update <id>` command with one or more field flags, the system shall modify only the specified fields and update the updatedAt timestamp.

- **REQ-E07**: When the user executes the `delete <id>` command, the system shall permanently remove the task from storage after user confirmation.

- **REQ-E08**: When the user executes the `delete <id> --force` command, the system shall permanently remove the task without confirmation.

### State-Driven Requirements / 状态驱动需求

- **REQ-S01**: While a task has status 'completed', the system shall only allow status transitions to 'archived'.

- **REQ-S02**: While a task has status 'archived', the system shall prevent any field modifications and return an error indicating the task is archived.

- **REQ-S03**: While no user is authenticated (no valid session), the system shall reject all task operations and display a message instructing the user to log in.

### Optional Requirements / 可选型需求

- **REQ-O01**: Where the user provides a --spec flag with a file path, the system shall validate that the file exists and store the path as specRef.

- **REQ-O02**: Where the user provides a --json flag on any list or show command, the system shall output the result as a JSON array or object instead of formatted text.

- **REQ-O03**: Where the user provides a --description or -d flag on the add command, the system shall store the provided text as the task description.

### Unwanted Behavior Handling / 排除型需求

- **REQ-X01**: If the user attempts to create a task with a title exceeding 200 characters, the system shall reject the operation and display an error: "Title must not exceed 200 characters (got: {n})".

- **REQ-X02**: If the user attempts to view, update, or delete a task with an id that does not exist, the system shall display an error: "Task not found: {id}".

- **REQ-X03**: If the user attempts to set a due date in the past when creating a task, the system shall display a warning but still create the task (non-blocking).

- **REQ-X04**: If the user provides an invalid priority value, the system shall display an error listing valid options: "Invalid priority '{value}'. Valid: low, medium, high, critical".

- **REQ-X05**: If the user provides more than 10 tags on a single task, the system shall reject the operation and display: "Maximum 10 tags per task (got: {n})".

- **REQ-X06**: If the user provides a --spec path that does not exist, the system shall reject the operation and display: "Spec file not found: {path}".

---

## Acceptance Criteria / 验收标准

### AC-01: Create task with defaults
- **Given** a user is authenticated
- **When** the user runs `spectask add "Write unit tests"`
- **Then** a new task is created with title "Write unit tests", status "pending", priority "medium", empty tags, no dueDate, no specRef, and valid timestamps

### AC-02: Create task with all options
- **Given** a user is authenticated
- **When** the user runs `spectask add "Fix auth bug" --priority high --tags auth,urgent --due 2024-12-31 --spec specs/auth-spec.md`
- **Then** a new task is created with all specified values correctly stored

### AC-03: List tasks with filters
- **Given** a user has 5 tasks: 2 pending/high, 1 pending/low, 1 completed/high, 1 archived/medium
- **When** the user runs `spectask list --status pending --priority high`
- **Then** exactly 2 tasks are displayed

### AC-04: Update task status
- **Given** a task exists with status "pending"
- **When** the user runs `spectask update <id> --status in-progress`
- **Then** the task status is "in-progress" and updatedAt is updated

### AC-05: Prevent modification of archived tasks
- **Given** a task exists with status "archived"
- **When** the user runs `spectask update <id> --title "New title"`
- **Then** the operation fails with error "Cannot modify archived task: {id}"

### AC-06: Delete with confirmation
- **Given** a task exists with id "abc123"
- **When** the user runs `spectask delete abc123`
- **Then** the system prompts "Delete task 'abc123'? (y/N)" and only deletes if confirmed

### AC-07: Reject invalid task creation
- **Given** a user is authenticated
- **When** the user runs `spectask add ""` (empty title)
- **Then** the operation fails with error "Title must not be empty"

### AC-08: JSON output format
- **Given** tasks exist for the current user
- **When** the user runs `spectask list --json`
- **Then** output is valid JSON array with all task fields

---

## Constraints & Assumptions / 约束与假设

### Constraints / 约束条件

- Storage: SQLite via better-sqlite3 (file-based, no server)
- No network access required for any task operation
- Maximum 10,000 tasks per user (SQLite performance boundary)
- Single-user per machine (no concurrent access concerns)
- CLI response time: < 100ms for all operations (local SQLite)

### Assumptions / 假设

- User has completed authentication before task operations
- Node.js >= 18 is available on the user's machine
- File system at ~/.spectask/ is writable
- Spec files referenced by --spec flag are readable from the project directory

---

## Out of Scope / 范围外

- Multi-user collaboration or shared task boards
- Cloud synchronization or remote storage
- Graphical user interface (GUI/TUI)
- Task dependencies or subtask hierarchy
- Recurring tasks or task templates
- File attachment support
- Real-time notifications (only check on command execution)

---

## Dependencies / 依赖

### Upstream Dependencies / 上游依赖

| Dependency | Type | Status |
|-----------|------|--------|
| Auth service (SPEC-002) | Internal | In Progress |
| better-sqlite3 | Library | Available |
| nanoid | Library | Available |
| date-fns | Library | Available |
| chalk | Library | Available |

### Downstream Impact / 下游影响

| Affected System | Impact |
|----------------|--------|
| Notification system | Will query tasks by dueDate |
| Spec linking feature | Will use specRef field |
| Plugin hook system | Will emit task lifecycle events |
```

---

## Auth Spec 撰写过程

Auth spec 比 Task CRUD 更简单——它是一个基础设施功能，为其他功能提供"谁在操作"的信息。

**设计约束**（来自 Constitution）：
- 本地存储，不涉及网络认证
- 密码使用单向哈希
- Session 通过文件维护

---

## 完整 Auth Spec

以下是 `project/specs/auth-spec.md` 的完整内容：

```markdown
# Authentication Specification

## Metadata / 元数据

| Field | Value |
|-------|-------|
| **Spec ID** | SPEC-002 |
| **Feature** | Local User Authentication |
| **Author** | SDD Tutorial |
| **Created** | 2024-12-01 |
| **Status** | Approved |
| **Priority** | P0-Critical |

---

## Feature Overview / 功能概述

Local authentication provides single-user identity management for SpecTask. Users register a local profile with username and password, then log in to create a session. The session persists across CLI invocations until explicit logout. This is intentionally simple — no network auth, no tokens, no expiration — optimized for the single-developer, local-machine use case.

---

## User Stories / 用户故事

1. As a developer, I want to register a local profile, so that my tasks are associated with my identity.
2. As a developer, I want to log in once and stay logged in across terminal sessions, so that I don't re-authenticate on every command.
3. As a developer, I want to see who is currently logged in, so that I can verify I'm operating as the correct user.

---

## Requirements / 需求 (EARS Format)

### Ubiquitous Requirements / 普适型需求

- **REQ-U01**: The system shall store user profiles with the following fields: id (nanoid), username (string, unique), passwordHash (string), createdAt (ISO 8601 string).

- **REQ-U02**: The system shall store passwords using bcrypt with a cost factor of 12.

- **REQ-U03**: The system shall persist session state in a JSON file at `~/.spectask/session.json` containing the authenticated userId and username.

### Event-Driven Requirements / 事件驱动需求

- **REQ-E01**: When the user executes `spectask register --username <name> --password <pass>`, the system shall create a new user profile with hashed password and display a success message.

- **REQ-E02**: When the user executes `spectask login --username <name> --password <pass>`, the system shall verify the password against the stored hash and, on success, write the session file.

- **REQ-E03**: When the user executes `spectask logout`, the system shall delete the session file and display a confirmation message.

- **REQ-E04**: When the user executes `spectask whoami`, the system shall read the session file and display the current username, or indicate no active session.

### State-Driven Requirements / 状态驱动需求

- **REQ-S01**: While a valid session file exists, the system shall treat the stored userId as the current authenticated user for all task operations.

- **REQ-S02**: While no session file exists or the session file is corrupted, the system shall treat the user as unauthenticated and block task operations.

### Unwanted Behavior Handling / 排除型需求

- **REQ-X01**: If the user attempts to register with a username that already exists, the system shall display: "Username '{name}' is already taken".

- **REQ-X02**: If the user attempts to login with incorrect credentials, the system shall display: "Invalid username or password" (generic message, no hint about which field is wrong).

- **REQ-X03**: If the user attempts to register with a username shorter than 3 characters or longer than 32 characters, the system shall display: "Username must be 3-32 characters".

- **REQ-X04**: If the user attempts to register with a password shorter than 8 characters, the system shall display: "Password must be at least 8 characters".

- **REQ-X05**: If the session file exists but references a userId not in the database, the system shall delete the invalid session file and treat user as unauthenticated.

---

## Acceptance Criteria / 验收标准

### AC-01: Successful registration
- **Given** no user with username "alice" exists
- **When** the user runs `spectask register --username alice --password mypassword123`
- **Then** a new user is created, and output shows "User 'alice' registered successfully"

### AC-02: Successful login
- **Given** user "alice" is registered with password "mypassword123"
- **When** the user runs `spectask login --username alice --password mypassword123`
- **Then** session file is created, and output shows "Logged in as alice"

### AC-03: Session persistence
- **Given** user "alice" is logged in
- **When** the user opens a new terminal and runs `spectask whoami`
- **Then** output shows "Currently logged in as: alice"

### AC-04: Logout clears session
- **Given** user "alice" is logged in
- **When** the user runs `spectask logout`
- **Then** session file is deleted, and output shows "Logged out successfully"

### AC-05: Reject duplicate registration
- **Given** user "alice" already exists
- **When** another registration attempt with "alice" is made
- **Then** operation fails with "Username 'alice' is already taken"

### AC-06: Reject invalid credentials
- **Given** user "alice" is registered
- **When** login is attempted with wrong password
- **Then** operation fails with "Invalid username or password"

---

## Constraints & Assumptions / 约束与假设

### Constraints

- Single user per machine (no multi-session support)
- Password hashing uses bcrypt (cost 12) — acceptable latency for CLI
- Session has no expiration (local tool, physical access = authorization)
- No password recovery mechanism (user can re-register if database is reset)

### Assumptions

- File system at ~/.spectask/ is writable by the current OS user
- bcrypt library is available (native binding via better-sqlite3 build toolchain)
- Terminal can securely accept password input (no echo)

---

## Out of Scope / 范围外

- Multi-user access control or roles
- Password reset or recovery flow
- Two-factor authentication
- OAuth or external identity providers
- Session expiration or refresh tokens
- Account deletion (manual database reset instead)

---

## Dependencies / 依赖

### Upstream Dependencies

| Dependency | Type | Status |
|-----------|------|--------|
| better-sqlite3 | Library | Available |
| bcrypt (or bcryptjs) | Library | Available |
| nanoid | Library | Available |

### Downstream Impact

| Affected System | Impact |
|----------------|--------|
| Task CRUD (SPEC-001) | Requires authenticated userId for all operations |
| All future features | Session validation as prerequisite |
```

---

## Spec 撰写中的常见陷阱

### 陷阱 1: 混淆 "What" 和 "How"

```markdown
<!-- 错误：描述了实现方式 -->
REQ-E01: When user creates a task, the system shall INSERT INTO tasks VALUES...

<!-- 正确：描述了期望行为 -->
REQ-E01: When user creates a task, the system shall persist the task with a unique ID and current timestamp.
```

Spec 描述**行为**（behavior），不描述**机制**（mechanism）。"INSERT INTO" 是实现细节，属于 plan 阶段。

### 陷阱 2: 遗漏边界情况

观察我们的 Task CRUD spec：它不仅描述了"成功创建任务"，还明确了标题为空、标题过长、priority 无效、tags 过多、spec 文件不存在等所有边界情况。

**经验法则**：每个 Event-Driven requirement 对应至少一个 Unwanted behavior requirement。如果你的 REQ-E 数量远超 REQ-X，说明 error handling 考虑不足。

### 陷阱 3: 验收标准与需求脱节

每条 AC 应该可以追溯到至少一条 REQ。如果你有一条 AC 找不到对应的 REQ，说明需求不完整；如果有一条 REQ 没有 AC 覆盖，说明验证方案不完整。

### 陷阱 4: Out of Scope 太模糊

```markdown
<!-- 模糊 -->
- Advanced features

<!-- 清晰 -->
- Multi-user collaboration or shared task boards
- Cloud synchronization or remote storage
```

Out of Scope 的目的是**防止 scope creep**。模糊的排除等于没有排除。

---

## Spec 之间的依赖关系

注意两份 spec 之间的引用关系：

```
SPEC-001 (Task CRUD)
  └── REQ-S03: "While no user is authenticated..."
  └── Dependencies: "Auth service (SPEC-002)"
       ↓
SPEC-002 (Auth)
  └── Downstream Impact: "Task CRUD requires authenticated userId"
```

这种显式的依赖声明帮助 Claude Code 在 plan 阶段确定实现顺序：Auth 必须在 Task CRUD 之前完成（至少是 auth interface 必须先定义）。

---

## 审查练习

在继续下一章之前，尝试回答以下问题（答案在自己心中验证）：

1. **REQ-U01 中的 tags 字段存储为 "string array"——但 SQLite 没有数组类型。这里是否应该在 spec 中指定 JSON 序列化？** （提示：spec 描述逻辑模型，物理存储方案属于 plan）

2. **REQ-S01 说 completed 状态只能转到 archived。但如果用户错误标记了 completed 想回退到 in-progress 呢？** （提示：这是一个设计决策——spec 可以选择允许或禁止）

3. **Auth spec 中 session "无过期时间"——对本地 CLI 工具合理吗？如果设备被盗呢？** （提示：threat model 决定安全级别。本地 CLI 的 threat model 中，物理访问等于完全控制）

这些问题没有唯一正确答案——它们的价值在于迫使你**主动审视设计决策**，而不是被动接受 spec 中的每一条规则。

---

## Key Takeaways (要点回顾)

1. **Spec 撰写顺序遵循依赖关系**——先写核心功能（被依赖最多），再写基础设施功能（支撑核心的）
2. **EARS 五种句式覆盖所有需求类型**——Ubiquitous（不变量）、Event-Driven（触发行为）、State-Driven（状态约束）、Optional（可选功能）、Unwanted（错误处理）
3. **每个 Event-Driven requirement 应有对应的 Unwanted behavior**——如果 happy path 和 error path 数量严重不对等，说明考虑不充分
4. **Spec 描述 "What"（行为），Plan 描述 "How"（机制）**——不要在 spec 中写实现细节
5. **Acceptance Criteria 直接映射为测试用例**——Claude Code 在 TDD 阶段会将 AC 转化为 test code

---

## Next (下一章)

[Chapter 21: 架构与实现计划 →](./21-planning.md)
