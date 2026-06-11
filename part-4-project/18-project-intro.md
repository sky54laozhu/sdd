# Chapter 18: 实战项目：SpecTask (Project Introduction)

> 纸上得来终觉浅，绝知此事要躬行——接下来四章我们将从零构建一个完整的 CLI task manager，让 SDD 的每一步都落地为可运行的代码。

---

## 为什么需要一个实战项目？

前面十七章我们讨论了 SDD 方法论的理论基础、Claude Code 的深度使用技巧、以及五阶段工作流的完整设计。但方法论的真正检验不在于理论是否自洽，而在于**能否在真实的工程场景中稳定产出高质量代码**。

本章开始，我们将构建 **SpecTask**——一个基于 SDD 方法论的 CLI task manager。这个项目有三重意义：

1. **方法论验证**：用 SDD 流程构建软件，本身就是对方法论的一次端到端测试
2. **元参照性（meta-referential）**：一个 task manager 构建过程中产生的 task，恰好可以被自身管理
3. **完整示范**：从 Constitution 到最终代码，展示每一步的具体产出物

---

## SpecTask 是什么？

SpecTask 是一个 spec-aware CLI task manager——它不仅管理任务，还能将任务与产生它们的 specification 文件关联起来。这让开发者可以追溯每个任务的来源，实现从需求到实现的全链路可追溯性（full traceability）。

### 核心功能一览

| # | Feature | 简述 |
|---|---------|------|
| 1 | Local User Profiles | 简单的本地认证，支持 username + password hash |
| 2 | Task CRUD | 创建、查看、更新、删除任务，支持 priority、tags、due dates |
| 3 | Spec Linking | 任务可关联到来源 spec 文件，实现需求追溯 |
| 4 | Notification System | due-date 提醒，通过 terminal output 输出 |
| 5 | Plugin Hook System | lifecycle hooks（task.created, task.completed 等），支持扩展 |

### 为什么选这个项目？

**非平凡（non-trivial）但范围有界（bounded scope）**：

- 涉及多个 feature module（auth、CRUD、notifications、plugins）
- 需要设计 data model、service layer、storage layer
- 有足够的复杂度展示 SDD 的分阶段交付优势
- 总代码量约 500-800 行，不会因规模失控而模糊方法论重点

**与 SDD 元相关（meta-relevant）**：

- SpecTask 自身就是一个 task 管理工具
- 构建它的过程产生 specs、plans、tasks——这些恰好是它的数据模型
- 读者在学习的同时，能感受到"规格驱动"带来的质的飞跃

---

## 技术栈选型

| Layer | Technology | 选型理由 |
|-------|-----------|---------|
| Language | TypeScript 5.x | 类型安全，适合展示 strict 模式下的开发 |
| Runtime | Node.js >= 18 | LTS 版本，原生支持 ESM |
| CLI Framework | Commander.js | 成熟、文档完善、社区活跃 |
| Storage | better-sqlite3 | 零配置、文件级存储、同步 API 简化 CLI 场景 |
| Date Handling | date-fns | 模块化、immutable、tree-shakeable |
| Testing | Vitest | 快速、TypeScript 原生、兼容 Jest API |
| Output Formatting | chalk | Terminal 彩色输出，提升 CLI 体验 |

为什么不用 PostgreSQL 或 MongoDB？因为 SpecTask 是一个**本地 CLI 工具**。better-sqlite3 的"零服务器"特性意味着用户 `npm install` 后即可使用，无需配置数据库服务。

---

## 项目结构预览

```
project/
├── CLAUDE.md              # Project constitution — SDD 的起点
├── specs/                 # Feature specifications (EARS format)
│   ├── task-crud-spec.md  # 任务 CRUD 功能规格
│   └── auth-spec.md       # 用户认证功能规格
├── plans/                 # Technical implementation plans
│   └── v1-plan.md         # 第一版实施计划
├── .claude/               # Agent & skill definitions
│   └── settings.json      # Claude Code project settings
└── src/                   # Implementation code
    ├── cli/               # Command definitions (Commander.js)
    ├── services/          # Business logic layer
    ├── repositories/      # Data access interface
    ├── storage/           # SQLite setup & migrations
    ├── types/             # TypeScript interfaces & type definitions
    └── utils/             # Shared utilities (Result type, validators, etc.)
```

注意这个结构体现了 SDD 的核心理念：**specs/ 和 plans/ 与 src/ 平级**，而不是藏在某个 docs/ 子目录里。它们是第一等公民（first-class citizens），与代码同等重要。

---

## 功能详细说明

### Feature 1: Local User Profiles

最简化的本地认证系统。不涉及网络通信、OAuth、JWT 等复杂机制。

```bash
spectask register --username alice --password ****
spectask login --username alice --password ****
spectask whoami
# => Currently logged in as: alice
```

**设计要点**：
- Password 使用 bcrypt 或 scrypt 哈希存储
- Session 通过本地文件（`~/.spectask/session.json`）维护
- 无 token expiry（本地工具不需要）

### Feature 2: Task CRUD

核心功能——创建、查看、更新、删除任务。

```bash
spectask add "Implement user registration" --priority high --tags auth,core --due 2024-12-31
spectask list --status pending --priority high
spectask update <task-id> --status completed
spectask delete <task-id>
spectask show <task-id>
```

**数据模型预览**：
- `id`: 唯一标识符（nanoid 生成）
- `title`: 任务标题
- `description`: 详细描述（可选）
- `status`: pending | in-progress | completed | archived
- `priority`: low | medium | high | critical
- `tags`: 字符串数组（JSON 序列化存储）
- `dueDate`: ISO 8601 日期字符串（可选）
- `specRef`: 关联的 spec 文件路径（可选）
- `createdAt` / `updatedAt`: 时间戳

### Feature 3: Spec Linking

让任务与其来源 spec 建立关联——这是 SpecTask 的差异化特性。

```bash
spectask add "Parse EARS requirements" --spec specs/task-crud-spec.md
spectask list --spec specs/task-crud-spec.md
# => Shows only tasks linked to this spec
spectask trace specs/task-crud-spec.md
# => Shows spec → tasks → status mapping
```

### Feature 4: Notification System

基于 due date 的终端提醒系统。

```bash
spectask notify
# => ⚠ Task "Submit PR review" is due tomorrow
# => 🔴 Task "Fix auth bug" is overdue by 2 days
```

**设计要点**：
- 检查时机：用户执行任何 spectask 命令时附带提醒
- 不使用后台 daemon（保持简单）
- 颜色编码：overdue = red，due today = yellow，due this week = blue

### Feature 5: Plugin Hook System

Lifecycle hooks 允许用户编写自定义逻辑响应任务事件。

```typescript
// .spectask/hooks/on-task-completed.ts
export default function onTaskCompleted(task: Task) {
  console.log(`🎉 Completed: ${task.title}`);
  // Could: update a dashboard, send a notification, log to file, etc.
}
```

**支持的 hook 事件**：
- `task.created` — 任务创建后触发
- `task.updated` — 任务更新后触发
- `task.completed` — 任务状态变为 completed 时触发
- `task.deleted` — 任务删除后触发

---

## 如何跟随本实战

接下来三章的结构与 SDD 五阶段流程精确对齐：

| 章节 | SDD Phase | 产出物 |
|------|-----------|--------|
| Chapter 19 | Constitution | `project/CLAUDE.md` |
| Chapter 20 | Specify | `project/specs/task-crud-spec.md`, `project/specs/auth-spec.md` |
| Chapter 21 | Plan | `project/plans/v1-plan.md` |

每章不仅展示最终文档，还会讲解**撰写过程中的思考逻辑**——为什么这样写，而不仅仅是写了什么。

### 建议的跟随方式

1. **先读后做**：通读每章内容，理解设计决策
2. **对比模板**：参照 `templates/` 目录中的对应模板，理解定制化的部分
3. **尝试变体**：在理解的基础上，尝试为你自己的项目写一份类似文档
4. **实际运行**：最终的 src/ 代码可以实际编译运行

---

## 前提条件

开始之前，请确保你的环境满足以下要求：

```bash
# Node.js >= 18
node --version
# => v18.x.x 或更高

# npm / pnpm
pnpm --version  # 推荐使用 pnpm
# 或
npm --version

# Claude Code 已安装
claude --version

# TypeScript (全局安装用于验证，项目内使用本地版本)
npx tsc --version
```

如果你只想阅读和学习，不需要实际运行，则无需安装任何东西——每章的产出物都以完整的 Markdown 文本呈现。

---

## 本章在整体教程中的位置

```
Part 1: Foundations (基础理论)          ← 已完成
Part 2: Claude Code Mastery (工具精通)  ← 已完成
Part 3: Workflow (工作流)               ← 已完成
Part 4: Project Walkthrough (实战)      ← 你在这里
  ├── Ch.18 — 项目介绍 (本章)
  ├── Ch.19 — 宪法撰写
  ├── Ch.20 — 功能规格
  └── Ch.21 — 架构计划
Part 5: Advanced (进阶)                 ← 后续
```

从下一章开始，我们将动手产出第一份 artifact——项目宪法 `CLAUDE.md`。它是 SDD 项目的基石，所有后续阶段都建立在它的约束之上。

---

## Key Takeaways (要点回顾)

1. **SpecTask 是一个 spec-aware CLI task manager**——它的差异化价值在于任务与规格文件的关联追溯能力
2. **项目规模经过精心控制**——500-800 行代码，5 个 feature module，足够展示 SDD 价值但不会失控
3. **技术栈选择以"本地 CLI"为核心约束**——better-sqlite3 零配置、Commander.js 成熟稳定、TypeScript 提供类型安全
4. **项目结构中 specs/ 和 plans/ 与 src/ 平级**——体现了 SDD "specification as first-class citizen" 的哲学
5. **后续三章对应 SDD 的三个阶段**——每章产出一份真实可用的工程文档

---

## Next (下一章)

[Chapter 19: 撰写项目宪法 →](./19-constitution.md)
