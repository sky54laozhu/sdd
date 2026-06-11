# EARS 快速参考卡 / EARS Cheatsheet

> EARS = Easy Approach to Requirements Syntax
> 一种让需求编写更加结构化、一致和可测试的方法。

## 起源 / Origin

- **提出者**: Alistair Mavin 等人（Rolls-Royce）
- **发表年份**: 2009
- **论文**: "Easy Approach to Requirements Syntax (EARS)" — IEEE International Requirements Engineering Conference
- **目的**: 消除自然语言需求中的歧义、不完整和不可测试性

EARS 被工业界广泛采用，特别是在航空航天、汽车和嵌入式系统领域。其核心思想是：通过少量关键字为需求句子提供结构化模板。

---

## 五种需求类型 / Five Requirement Types

### 1. Ubiquitous（无条件需求）

没有前置条件，系统在所有情况下都必须满足。

**句法模式:**
```
The [system] shall [action].
```

**特征**: 没有 WHEN / WHILE / WHERE / IF，是最基本的需求形式。

**示例:**
```
The system shall encrypt all stored passwords using bcrypt.
系统应使用 bcrypt 加密所有存储的密码。

The system shall display a loading indicator during data fetches.
系统应在数据获取期间显示加载指示器。

The system shall log all API requests with timestamp and user ID.
系统应记录所有 API 请求的时间戳和用户 ID。
```

---

### 2. Event-Driven（事件驱动需求）

由特定事件触发，使用 **WHEN** 关键字。

**句法模式:**
```
WHEN [trigger event], the [system] shall [action].
```

**特征**: 响应离散事件，通常是瞬时触发。

**示例:**
```
WHEN the user clicks the "Save" button, the system shall persist the form data to the database.
当用户点击"保存"按钮时，系统应将表单数据持久化到数据库。

WHEN a new task is created, the system shall assign a unique identifier.
当创建新任务时，系统应分配唯一标识符。

WHEN the session token expires, the system shall redirect the user to the login page.
当会话令牌过期时，系统应将用户重定向到登录页面。
```

---

### 3. State-Driven（状态驱动需求）

在特定系统状态持续期间有效，使用 **WHILE** 关键字。

**句法模式:**
```
WHILE [system state], the [system] shall [action].
```

**特征**: 在状态持续期间持续满足，而非一次性触发。

**示例:**
```
WHILE the system is in offline mode, the system shall queue all write operations locally.
当系统处于离线模式时，系统应在本地排队所有写操作。

WHILE the user has admin privileges, the system shall display the administration panel.
当用户拥有管理员权限时，系统应显示管理面板。

WHILE a file upload is in progress, the system shall display a progress bar.
当文件上传进行中时，系统应显示进度条。
```

---

### 4. Optional Feature（可选特性需求）

仅在特定功能/配置存在时适用，使用 **WHERE** 关键字。

**句法模式:**
```
WHERE [feature/configuration], the [system] shall [action].
```

**特征**: 用于可选模块、插件、许可证或配置开关。

**示例:**
```
WHERE the dark mode feature is enabled, the system shall apply the dark color palette.
在启用深色模式功能时，系统应应用深色调色板。

WHERE the enterprise license is active, the system shall allow SSO authentication.
在企业许可证激活时，系统应允许 SSO 认证。

WHERE push notifications are configured, the system shall send reminders 15 minutes before due time.
在配置了推送通知时，系统应在截止时间前 15 分钟发送提醒。
```

---

### 5. Unwanted Behavior（异常行为需求）

处理不期望的情况，使用 **IF...THEN** 结构。

**句法模式:**
```
IF [unwanted condition], THEN the [system] shall [mitigation action].
```

**特征**: 处理错误、异常、边界情况和故障恢复。

**示例:**
```
IF the database connection fails, THEN the system shall retry 3 times with exponential backoff.
如果数据库连接失败，则系统应以指数退避方式重试 3 次。

IF the user enters an invalid email format, THEN the system shall display an inline validation error.
如果用户输入无效的邮箱格式，则系统应显示内联验证错误。

IF the API response time exceeds 5 seconds, THEN the system shall cancel the request and show a timeout message.
如果 API 响应时间超过 5 秒，则系统应取消请求并显示超时消息。
```

---

## 组合模式 / Complex Combinations

EARS 类型可以组合使用，处理更复杂的场景。

### WHILE + WHEN（状态中的事件触发）

```
WHILE [state], WHEN [event], the [system] shall [action].
```

**示例:**
```
WHILE the user is editing a document, WHEN another user modifies the same document,
the system shall display a real-time conflict notification.
当用户正在编辑文档时，若另一个用户修改了同一文档，系统应显示实时冲突通知。
```

### WHERE + WHEN（可选特性中的事件触发）

```
WHERE [feature], WHEN [event], the [system] shall [action].
```

**示例:**
```
WHERE two-factor authentication is enabled, WHEN the user logs in from a new device,
the system shall require a verification code.
在启用双因素认证时，当用户从新设备登录，系统应要求输入验证码。
```

### WHILE + IF...THEN（状态中的异常处理）

```
WHILE [state], IF [unwanted condition], THEN the [system] shall [action].
```

**示例:**
```
WHILE a file upload is in progress, IF the network connection drops,
THEN the system shall pause the upload and resume automatically when connectivity is restored.
当文件上传进行中时，如果网络连接断开，则系统应暂停上传并在连接恢复后自动继续。
```

---

## 常见错误与修正 / Common Mistakes

### 错误 1: 模糊动词

```
❌ The system shall handle user input appropriately.
✅ The system shall validate user input against the defined schema and reject non-conforming data with a 400 status code.
```

**问题**: "handle appropriately" 不可测试。
**修正**: 使用具体、可验证的动作。

### 错误 2: 混淆 WHEN 和 WHILE

```
❌ WHEN the user is logged in, the system shall show the dashboard.
✅ WHILE the user is authenticated, the system shall display the dashboard.
```

**问题**: "logged in" 是持续状态，不是瞬时事件。
**修正**: 持续状态用 WHILE，瞬时事件用 WHEN。

### 错误 3: 需求中包含实现细节

```
❌ WHEN the user submits the form, the system shall use Redux to update the global state and call the REST API endpoint /api/tasks with a POST method.
✅ WHEN the user submits the task form, the system shall create the task and confirm creation within 2 seconds.
```

**问题**: 规格不应绑定具体技术实现。
**修正**: 描述行为和约束，不描述实现方式。

### 错误 4: 多个行为塞进一条需求

```
❌ WHEN the user creates a task, the system shall validate the input, save to database, send notification, and update the dashboard.
✅ (拆分为 4 条独立需求)
  WHEN the user submits task creation, the system shall validate all required fields.
  WHEN a task is successfully validated, the system shall persist it to storage.
  WHEN a task is successfully created, the system shall notify assigned users.
  WHEN a task is successfully created, the system shall update the task list view.
```

**问题**: 一条需求包含多个独立行为，难以追踪和测试。
**修正**: 每条需求只描述一个可独立验证的行为。

### 错误 5: 缺少异常处理

```
❌ WHEN the user clicks "Delete", the system shall delete the task.
✅ WHEN the user clicks "Delete", the system shall request confirmation before deletion.
   IF the deletion fails due to a server error, THEN the system shall display an error message and retain the task.
```

**问题**: 只考虑 happy path，忽略失败场景。
**修正**: 为关键操作补充 IF...THEN 异常处理需求。

---

## 实战示例：任务管理应用 / Real Example: Task Manager App

以下是一个任务管理应用的 EARS 需求集合：

```markdown
## 任务创建

REQ-001 [Ubiquitous]
The system shall generate a unique ID for each task using UUID v4 format.

REQ-002 [Event-Driven]
WHEN the user submits the task creation form with valid data,
the system shall create the task and display it in the task list within 1 second.

REQ-003 [Unwanted]
IF the task title exceeds 200 characters,
THEN the system shall truncate the display title and show a tooltip with the full text.

## 任务状态

REQ-004 [Event-Driven]
WHEN the user drags a task to a different status column,
the system shall update the task status and record the transition timestamp.

REQ-005 [State-Driven]
WHILE a task status is "In Progress",
the system shall display an elapsed time counter on the task card.

## 协作功能

REQ-006 [Optional Feature]
WHERE the team collaboration module is enabled,
the system shall display assignee avatars on each task card.

REQ-007 [WHILE + WHEN]
WHILE multiple users are viewing the same project board,
WHEN any user modifies a task,
the system shall broadcast the change to all connected users within 500ms.

## 错误处理

REQ-008 [Unwanted]
IF the WebSocket connection is lost,
THEN the system shall switch to polling mode every 5 seconds
and display a "reconnecting" indicator.

REQ-009 [WHILE + IF...THEN]
WHILE the system is in polling mode,
IF connectivity is restored,
THEN the system shall re-establish the WebSocket connection
and synchronize any missed updates.
```

---

## 速查表 / Quick Reference

| Type | Keyword | 适用场景 | 信号词 |
|------|---------|----------|--------|
| Ubiquitous | (none) | 始终适用的全局约束 | always, all, every |
| Event-Driven | WHEN | 响应一次性事件 | clicks, submits, receives, triggers |
| State-Driven | WHILE | 持续状态期间 | is active, is connected, is in mode |
| Optional | WHERE | 可配置功能 | is enabled, is installed, is licensed |
| Unwanted | IF...THEN | 错误和异常处理 | fails, exceeds, times out, is invalid |

---

## 参考文献 / References

- Mavin, A. et al. (2009). "Easy Approach to Requirements Syntax (EARS)." *17th IEEE International Requirements Engineering Conference*.
- Mavin, A. & Wilkinson, P. (2010). "Big Ears: The Return of Easy Approach to Requirements Syntax." *18th IEEE International Requirements Engineering Conference*.
- Rolls-Royce EARS Guidelines (internal, widely cited in academia).
