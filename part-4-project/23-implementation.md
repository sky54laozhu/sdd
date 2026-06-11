# Chapter 23: 用 Claude Code 实现 (Implementation with Claude Code)

> 进入 Implementation 阶段：按照任务列表，逐个执行 TDD 循环，让 AI 在 Spec 的轨道上将代码一步步构建出来。

---

## 实现阶段的核心原则

到这一步，你手里已经有了一份完整的任务列表（第 22 章）。现在要做的是**逐个执行**这些任务。听起来简单，但 Implementation 阶段有一个容易被忽视的纪律：

**每个任务的执行都是一次独立的 Claude Code 交互，遵循固定的 per-task workflow。**

这个 workflow 是：

```
1. 读取任务描述和关联的 spec requirement
2. 执行任务（写测试 or 写实现）
3. 运行验证命令
4. 确认通过后标记完成
5. Commit with conventional commit message
```

绝不跳步。绝不在一次交互中做两个 task。为什么？因为如果你把三个 task 合并在一次请求中完成，当出问题时你无法精确定位是哪个 task 引入了 bug。原子性在执行阶段同样重要。

---

## 第一步：项目初始化 (TASK-001)

### 你对 Claude Code 说什么

```
初始化 SpecTask 项目。

要求：
- package.json: name=spectask, type=module, 入口 src/cli/index.ts
- 依赖: commander, better-sqlite3, chalk, nanoid, bcryptjs
- Dev 依赖: typescript, vitest, @types/better-sqlite3, @types/bcryptjs, tsx
- tsconfig: target ES2022, module ESNext, moduleResolution bundler, strict
- vitest.config.ts: 基本配置
- scripts: build (tsc), test (vitest), dev (tsx src/cli/index.ts)
- 创建 src/ 目录结构: cli/, services/, repositories/, storage/, types/, utils/

完成后运行 pnpm tsc --noEmit 验证。
```

Claude Code 会自动创建文件、安装依赖、验证编译。这就是 TASK-001 的完整执行过程。

---

## TASK-002：定义 TypeScript Interfaces

这是一个类型定义任务，不涉及测试——因为类型本身由 TypeScript compiler 验证。

### 目标文件：`src/types/index.ts`

```typescript
// src/types/index.ts

// Result type — consistent error handling envelope
export type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };

// Task domain
export type TaskStatus = 'todo' | 'in_progress' | 'done';

export interface Task {
  id: string;
  userId: string;
  title: string;
  description?: string;
  status: TaskStatus;
  specLink?: string;
  dueDate?: string; // ISO 8601
  createdAt: string;
  updatedAt: string;
}

export interface CreateTaskInput {
  title: string;
  description?: string;
  specLink?: string;
  dueDate?: string;
}

export interface UpdateTaskInput {
  title?: string;
  description?: string;
  status?: TaskStatus;
  specLink?: string;
  dueDate?: string;
}

// User domain
export interface User {
  id: string;
  username: string;
  passwordHash: string;
  createdAt: string;
}

// Session
export interface Session {
  userId: string;
  username: string;
  token: string;
  expiresAt: string;
}
```

注意几个设计决策：

- **`Result<T>` 是 discriminated union**：通过 `success` 字段区分成功和失败，替代 try-catch 在 service 层间传递错误
- **所有日期用 ISO 8601 字符串**：SQLite 没有原生 Date 类型，字符串存储最简单
- **`specLink` 是可选字段**：对应 TASK-014 的 spec-linking 功能

### 验证

```bash
pnpm tsc --noEmit
# 通过 → TASK-002 完成
```

---

## TASK-004 → TASK-005：Repository 层 TDD 循环

这是 SDD 中最有教学价值的部分——一个完整的 RED → GREEN 循环。

### RED Phase (TASK-004)：编写测试

你对 Claude Code 说：

```
执行 TASK-004：编写 TaskRepository 的测试。

Spec requirements:
- REQ-U01: 创建任务时自动生成唯一 ID，存入 tasks 表
- REQ-U02: 列出当前用户的所有任务
- REQ-U03: 更新任务字段（title, description, status）
- REQ-U04: 删除任务；删除不存在的任务返回错误

使用 :memory: SQLite 数据库。遵循 AAA pattern。
这是 RED phase：测试应该 fail 因为 TaskRepository 还不存在。
```

Claude Code 生成的测试文件：

```typescript
// tests/repositories/task.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { Database } from '../../src/storage/database';
import { TaskRepository } from '../../src/repositories/task-repository';
import type { CreateTaskInput } from '../../src/types';

describe('TaskRepository', () => {
  let db: Database;
  let repo: TaskRepository;

  beforeEach(() => {
    db = new Database(':memory:');
    repo = new TaskRepository(db);
  });

  describe('create', () => {
    it('should create a task with auto-generated ID', () => {
      // Arrange
      const input: CreateTaskInput = { title: 'Buy groceries' };
      const userId = 'user-1';

      // Act
      const result = repo.create(userId, input);

      // Assert
      expect(result.success).toBe(true);
      if (result.success) {
        expect(result.data.id).toBeDefined();
        expect(result.data.id.length).toBeGreaterThan(0);
        expect(result.data.title).toBe('Buy groceries');
        expect(result.data.userId).toBe('user-1');
        expect(result.data.status).toBe('todo');
      }
    });

    it('should generate unique IDs for different tasks', () => {
      const input: CreateTaskInput = { title: 'Task' };
      const r1 = repo.create('user-1', input);
      const r2 = repo.create('user-1', input);

      if (r1.success && r2.success) {
        expect(r1.data.id).not.toBe(r2.data.id);
      }
    });
  });

  describe('findAllByUser', () => {
    it('should return only tasks belonging to the specified user', () => {
      repo.create('user-1', { title: 'Task A' });
      repo.create('user-2', { title: 'Task B' });
      repo.create('user-1', { title: 'Task C' });

      const result = repo.findAllByUser('user-1');

      expect(result.success).toBe(true);
      if (result.success) {
        expect(result.data).toHaveLength(2);
        expect(result.data.every(t => t.userId === 'user-1')).toBe(true);
      }
    });
  });

  describe('update', () => {
    it('should update task fields and return updated task', () => {
      const created = repo.create('user-1', { title: 'Original' });
      if (!created.success) throw new Error('Setup failed');

      const result = repo.update(created.data.id, { title: 'Updated', status: 'done' });

      expect(result.success).toBe(true);
      if (result.success) {
        expect(result.data.title).toBe('Updated');
        expect(result.data.status).toBe('done');
      }
    });

    it('should return error when updating non-existent task', () => {
      const result = repo.update('non-existent-id', { title: 'X' });

      expect(result.success).toBe(false);
      if (!result.success) {
        expect(result.error).toContain('not found');
      }
    });
  });

  describe('delete', () => {
    it('should delete an existing task', () => {
      const created = repo.create('user-1', { title: 'To delete' });
      if (!created.success) throw new Error('Setup failed');

      const result = repo.delete(created.data.id);
      expect(result.success).toBe(true);

      const found = repo.findAllByUser('user-1');
      if (found.success) {
        expect(found.data).toHaveLength(0);
      }
    });

    it('should return error when deleting non-existent task', () => {
      const result = repo.delete('non-existent-id');

      expect(result.success).toBe(false);
      if (!result.success) {
        expect(result.error).toContain('not found');
      }
    });
  });
});
```

此时运行测试：

```bash
pnpm test tests/repositories/task.test.ts
# ❌ FAIL — Cannot find module '../../src/repositories/task-repository'
# 这正是 RED phase 的预期结果！
```

TASK-004 完成。Commit：

```bash
git add tests/repositories/task.test.ts
git commit -m "test: add TaskRepository tests (RED phase)

Covers REQ-U01 through REQ-U04: create with auto-ID, findAllByUser,
update fields, delete with error on non-existent."
```

---

### GREEN Phase (TASK-005)：实现使测试通过

```
执行 TASK-005：实现 TaskRepository 使 TASK-004 的测试全部通过。

约束：
- 文件：src/repositories/task-repository.ts
- 使用 better-sqlite3 prepared statements（防 SQL injection）
- 使用 nanoid 生成 ID
- 返回 Result<T> 类型
- 遵循 CLAUDE.md 中的分层架构约定
```

Claude Code 生成实现：

```typescript
// src/repositories/task-repository.ts
import { nanoid } from 'nanoid';
import type { Database } from '../storage/database';
import type { Task, CreateTaskInput, UpdateTaskInput, Result } from '../types';

export class TaskRepository {
  constructor(private readonly db: Database) {}

  create(userId: string, input: CreateTaskInput): Result<Task> {
    const now = new Date().toISOString();
    const task: Task = {
      id: nanoid(),
      userId,
      title: input.title,
      description: input.description,
      status: 'todo',
      specLink: input.specLink,
      dueDate: input.dueDate,
      createdAt: now,
      updatedAt: now,
    };

    const stmt = this.db.connection.prepare(`
      INSERT INTO tasks (id, user_id, title, description, status, spec_link, due_date, created_at, updated_at)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
    `);

    stmt.run(
      task.id, task.userId, task.title, task.description ?? null,
      task.status, task.specLink ?? null, task.dueDate ?? null,
      task.createdAt, task.updatedAt
    );

    return { success: true, data: task };
  }

  findAllByUser(userId: string): Result<Task[]> {
    const stmt = this.db.connection.prepare(
      'SELECT * FROM tasks WHERE user_id = ? ORDER BY created_at DESC'
    );
    const rows = stmt.all(userId) as any[];
    const tasks = rows.map(this.rowToTask);
    return { success: true, data: tasks };
  }

  update(id: string, input: UpdateTaskInput): Result<Task> {
    const existing = this.db.connection
      .prepare('SELECT * FROM tasks WHERE id = ?')
      .get(id) as any | undefined;

    if (!existing) {
      return { success: false, error: `Task with id "${id}" not found` };
    }

    const now = new Date().toISOString();
    const stmt = this.db.connection.prepare(`
      UPDATE tasks SET
        title = COALESCE(?, title),
        description = COALESCE(?, description),
        status = COALESCE(?, status),
        spec_link = COALESCE(?, spec_link),
        due_date = COALESCE(?, due_date),
        updated_at = ?
      WHERE id = ?
    `);

    stmt.run(
      input.title ?? null, input.description ?? null,
      input.status ?? null, input.specLink ?? null,
      input.dueDate ?? null, now, id
    );

    const updated = this.db.connection
      .prepare('SELECT * FROM tasks WHERE id = ?')
      .get(id) as any;

    return { success: true, data: this.rowToTask(updated) };
  }

  delete(id: string): Result<void> {
    const existing = this.db.connection
      .prepare('SELECT id FROM tasks WHERE id = ?')
      .get(id);

    if (!existing) {
      return { success: false, error: `Task with id "${id}" not found` };
    }

    this.db.connection.prepare('DELETE FROM tasks WHERE id = ?').run(id);
    return { success: true, data: undefined };
  }

  private rowToTask(row: any): Task {
    return {
      id: row.id,
      userId: row.user_id,
      title: row.title,
      description: row.description ?? undefined,
      status: row.status,
      specLink: row.spec_link ?? undefined,
      dueDate: row.due_date ?? undefined,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
    };
  }
}
```

运行测试验证：

```bash
pnpm test tests/repositories/task.test.ts
# ✓ should create a task with auto-generated ID
# ✓ should generate unique IDs for different tasks
# ✓ should return only tasks belonging to the specified user
# ✓ should update task fields and return updated task
# ✓ should return error when updating non-existent task
# ✓ should delete an existing task
# ✓ should return error when deleting non-existent task
# Tests: 7 passed
```

全绿。TASK-005 完成。Commit：

```bash
git add src/repositories/task-repository.ts
git commit -m "feat: implement TaskRepository with CRUD operations

Satisfies REQ-U01 through REQ-U04. Uses nanoid for ID generation,
better-sqlite3 prepared statements for queries. All 7 repository
tests now passing."
```

---

## TASK-012：CLI Commands 实现

当 Service 层完成后，CLI 层的实现相对直接——它只是一层"胶水"，将用户输入映射到 service 调用。

### 你对 Claude Code 说什么

```
执行 TASK-012：实现 Task CLI commands。

要求：
- 使用 Commander.js 定义命令
- 命令列表: task add, task list, task update, task delete, task done
- 每个命令调用 TaskService 对应方法
- 输出用 chalk 着色：成功绿色，错误红色，列表有格式化表格
- 当前用户从 session 文件读取（如果未登录则提示 "please login first"）

参考 spec:
- REQ-U01: spectask task add <title> [--desc] [--spec] [--due]
- REQ-U02: spectask task list [--status] [--spec]
- REQ-U03: spectask task update <id> [--title] [--desc] [--status]
- REQ-U04: spectask task delete <id>
```

Claude Code 生成的 CLI 入口：

```typescript
// src/cli/commands/task.ts
import { Command } from 'commander';
import chalk from 'chalk';
import { TaskService } from '../../services/task-service';
import { getSession } from '../../utils/session';

export function registerTaskCommands(program: Command, taskService: TaskService): void {
  const task = program.command('task').description('Manage tasks');

  task
    .command('add <title>')
    .description('Create a new task')
    .option('--desc <description>', 'Task description')
    .option('--spec <spec-id>', 'Link to spec requirement')
    .option('--due <date>', 'Due date (YYYY-MM-DD)')
    .action((title, opts) => {
      const session = getSession();
      if (!session) {
        console.error(chalk.red('Error: Please login first (spectask auth login)'));
        process.exit(1);
      }

      const result = taskService.create(session.userId, {
        title,
        description: opts.desc,
        specLink: opts.spec,
        dueDate: opts.due,
      });

      if (result.success) {
        console.log(chalk.green(`✓ Task created: ${result.data.id}`));
        console.log(`  Title: ${result.data.title}`);
      } else {
        console.error(chalk.red(`✗ ${result.error}`));
        process.exit(1);
      }
    });

  task
    .command('list')
    .description('List all tasks')
    .option('--status <status>', 'Filter by status')
    .option('--spec <spec-id>', 'Filter by spec link')
    .action((opts) => {
      const session = getSession();
      if (!session) {
        console.error(chalk.red('Error: Please login first'));
        process.exit(1);
      }

      const result = taskService.findAll(session.userId, opts);

      if (result.success) {
        if (result.data.length === 0) {
          console.log(chalk.dim('No tasks found.'));
          return;
        }
        for (const t of result.data) {
          const status = t.status === 'done'
            ? chalk.green('✓ done')
            : t.status === 'in_progress'
            ? chalk.yellow('◐ in_progress')
            : chalk.dim('○ todo');
          const due = t.dueDate ? chalk.dim(` (due: ${t.dueDate})`) : '';
          console.log(`  ${status}  ${chalk.bold(t.id.slice(0, 8))}  ${t.title}${due}`);
        }
      } else {
        console.error(chalk.red(`✗ ${result.error}`));
      }
    });

  task
    .command('done <id>')
    .description('Mark task as done')
    .action((id) => {
      const result = taskService.update(id, { status: 'done' });
      if (result.success) {
        console.log(chalk.green(`✓ Task ${id} marked as done`));
      } else {
        console.error(chalk.red(`✗ ${result.error}`));
      }
    });
}
```

---

## Per-Task Workflow 总结

每个 task 的执行遵循严格的五步流程：

| Step | 动作 | 验证 |
|------|------|------|
| 1 | 读取任务描述 + 关联 spec | 确认理解需求 |
| 2 | 执行任务（写测试或写实现） | 代码生成 |
| 3 | 运行验证命令 | pass/fail 判定 |
| 4 | 标记任务完成 | 更新 task status |
| 5 | Commit | conventional commit message |

Claude Code 的工作就是在每一步中充当"受 spec 约束的执行者"。你不需要告诉它"怎么写一个 SQLite 查询"——这是它的能力域。你需要告诉它的是"这个查询必须满足 REQ-U02 的要求"——这是 spec 的约束域。

---

## 实现中的常见问题与对策

### 问题 1：AI 偏离 Spec

**症状**：Claude Code 添加了 spec 中没有提到的功能（如自动归档已完成任务）。

**对策**：
```
停止。回到 spec 文件 specs/task-crud-spec.md。
你添加的 auto-archive 功能在 spec 中不存在。
请只实现 REQ-U03 指定的行为：更新 status 字段。
不要添加任何 spec 未要求的行为。
```

关键点：**指向具体的 requirement ID**，而不是笼统地说"不要多做"。

### 问题 2：测试意外失败

**症状**：TASK-005 的实现写完了，但测试 fail。

**对策**：首先区分是**测试有 bug** 还是**实现有 bug**：

```
测试 "should return error when updating non-existent task" 失败了。
错误：Expected success to be false, received true.

请先检查测试逻辑是否正确（AAA pattern 各部分），
然后检查实现是否符合 spec REQ-U03 的描述：
"If [task ID does not exist], the system shall return an error message."
```

### 问题 3：Architecture Drift

**症状**：Service 层直接引用了 `better-sqlite3`，绕过了 Repository 层。

**对策**：
```
架构违规：src/services/task-service.ts 第 15 行直接 import 了 better-sqlite3。
根据 CLAUDE.md 的分层架构约定，Service 层只能依赖 Repository 接口。
请重构为通过 TaskRepository 访问数据。
```

---

## 有效的 Prompt 模式

以下是在 Implementation 阶段效果最好的 prompt 结构：

### 模式 1：Task + Context + Constraint

```
[执行哪个 task]
[关联的 spec requirement 是什么]
[技术约束是什么]
[完成标准是什么]
```

### 模式 2：Fix with Reference

```
[什么 fail 了]
[预期行为是什么（引用 spec）]
[实际行为是什么]
[请修复，不要改变其他行为]
```

### 模式 3：Refactor with Guard

```
[重构目标是什么]
[不能改变的行为边界是什么]
[运行哪个测试命令来验证行为不变]
```

---

## 执行节奏

一个合理的执行节奏（假设一个人 + Claude Code）：

| Phase | Tasks | 预估时间 |
|-------|-------|----------|
| Scaffold | 1-2 | 30 min |
| Types + Storage | 2-3 | 45 min |
| Repository TDD | 4-7 | 2 hours |
| Service TDD | 8-11 | 2 hours |
| CLI | 12-15 | 2 hours |
| Integration | 16-17 | 1.5 hours |

总计约 **8-9 小时**。注意这包括了完整的测试覆盖。如果跳过 TDD 直接写实现，可能"看起来"快 2-3 小时，但后期 debug 和回退的成本往往远超这个时间。

---

## Key Takeaways (要点回顾)

1. **每个 task 是一次独立的 Claude Code 交互** —— 原子执行，独立验证，独立 commit
2. **RED → GREEN 循环是核心节奏** —— 先写测试（fail），再写实现（pass），这不是仪式而是质量保障
3. **Prompt 中引用具体的 REQ-ID** —— 比笼统描述有效 10 倍，给 AI 明确的行为边界
4. **Architecture drift 要立即纠正** —— 分层架构是 Constitution 级别的约束，不是建议
5. **测试失败时先检查测试本身** —— 测试代码也是代码，也可能有 bug

---

## Next (下一章)

[Chapter 24: 验证与质量保证](24-verification.md) —— 实现完成后，如何用 Verifier Agent 自动对照 Spec 检查实现完整性。
