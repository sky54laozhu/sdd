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
