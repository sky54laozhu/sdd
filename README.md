# SDD 教程系列：Spec-Driven Development with Claude Code

> 从零开始掌握规格驱动开发（Spec-Driven Development），用结构化的 Spec 驱动 AI 编写生产级代码。

---

## 这是什么

本教程是一个 29 章的完整 SDD 方法论指南，教你如何：

- 用 **Spec（规格文档）** 替代即兴 prompt，获得可预测、可复现的 AI 输出
- 构建 **CLAUDE.md** 作为项目宪法，让 Claude Code 理解你的意图
- 掌握 **多代理协作模式**，将复杂项目分解为可管理的任务流
- 在真实项目中实践完整的 SDD 工作流

## 为谁而写

- 使用 Claude Code 进行日常开发的工程师
- 希望从"对话式编程"升级到"工程化 AI 开发"的开发者
- 团队技术负责人，需要建立 AI 辅助开发规范

## 如何使用

每章包含：概念讲解 → 实战示例 → 模板 → 练习。建议边读边在 `project/` 目录中实践。

---

## 阅读路径

### 新手完整路径

> 从未用过 Claude Code 或刚入门 AI 辅助开发

**Part 1** → **Part 2** → **Part 3** → **Part 4** → **Part 5**

按顺序阅读，每章的实践部分不要跳过。

### 有经验的 Claude Code 用户

> 已熟悉 Claude Code 基础操作，想学习 SDD 方法论

从 [Ch.13 核心工作流](part-3-workflow/13-core-workflow.md) 开始 → 完成 **Part 4** 实战项目，遇到不熟悉的概念时回溯 **Part 2** 对应章节。

### 只要模板

> 已了解 SDD，只需要即用模板

直接进入 [`templates/`](templates/) 目录，每个模板文件都是自包含的。

---

## 目录

### Part 1: 理论基础 (Foundations)

| # | 章节 | 主题 |
|---|------|------|
| 01 | [什么是 SDD](part-1-foundations/01-what-is-sdd.md) | SDD 定义、历史脉络、核心价值主张 |
| 02 | [Vibe Coding 的七宗罪](part-1-foundations/02-vibe-coding-failure.md) | AI 编码 7 种失败模式与 SDD 对策 |
| 03 | [SDD vs TDD vs BDD vs DDD](part-1-foundations/03-sdd-vs-tdd-bdd-ddd.md) | 方法论对比、外环/内环模型、协同使用 |
| 04 | [三层严格度模型](part-1-foundations/04-three-rigor-levels.md) | Spec-First / Spec-Anchored / Spec-As-Source |
| 05 | [SDD 工具生态](part-1-foundations/05-sdd-ecosystem.md) | Spec Kit、Kiro、BMAD、Claude Code、Cursor、OpenSpec |

### Part 2: Claude Code 深度掌握 (Claude Code Mastery)

| # | 章节 | 主题 |
|---|------|------|
| 06 | [CLAUDE.md 项目宪法](part-2-claude-code-mastery/06-claude-md-constitution.md) | 文件层级、有效结构、活文档原则、反模式 |
| 07 | [编写有效规格](part-2-claude-code-mastery/07-writing-specs.md) | EARS 记法、验收标准、规格结构、实战练习 |
| 08 | [子代理系统](part-2-claude-code-mastery/08-subagents.md) | 上下文隔离、三种 SDD 代理模式、并行执行 |
| 09 | [任务系统](part-2-claude-code-mastery/09-tasks-system.md) | Task API、生命周期、依赖管理、TDD 集成 |
| 10 | [钩子系统](part-2-claude-code-mastery/10-hooks.md) | PreToolUse/PostToolUse/Stop、自动化守卫 |
| 11 | [技能系统](part-2-claude-code-mastery/11-skills.md) | SKILL.md、工作流封装、SDD 技能定义 |
| 12 | [多代理编排模式](part-2-claude-code-mastery/12-multi-agent-patterns.md) | 验证器、并行研究、编排器、GAN、流水线 |

### Part 3: SDD 工作流 (Workflow)

| # | 章节 | 主题 |
|---|------|------|
| 13 | [核心五阶段流程](part-3-workflow/13-core-workflow.md) | Constitution → Specify → Plan → Tasks → Implement |
| 14 | [阶段门审核](part-3-workflow/14-phase-gates.md) | 质量关卡、审核标准、自动化验证 |
| 15 | [SDD + TDD 协同](part-3-workflow/15-tdd-integration.md) | 外环/内环、EARS→测试、Spec-Test-Code 三角 |
| 16 | [存量项目改造](part-3-workflow/16-brownfield-sdd.md) | 增量 SDD、Spec the Delta、渐进采用 |
| 17 | [常见陷阱与解法](part-3-workflow/17-pitfalls.md) | 10 种反模式及对策 |

### Part 4: 实战项目 SpecTask (Project Walkthrough)

| # | 章节 | 主题 |
|---|------|------|
| 18 | [项目介绍](part-4-project/18-project-intro.md) | SpecTask CLI 任务管理器概述 |
| 19 | [撰写项目宪法](part-4-project/19-constitution.md) | 编写 CLAUDE.md、分层架构、编码标准 |
| 20 | [编写功能规格](part-4-project/20-specification.md) | Task CRUD + Auth 的 EARS 规格 |
| 21 | [架构与实现计划](part-4-project/21-planning.md) | ADR、系统设计、数据模型、文件结构 |
| 22 | [任务分解](part-4-project/22-task-decomposition.md) | 17 个原子任务、依赖图、TDD 顺序 |
| 23 | [用 Claude Code 实现](part-4-project/23-implementation.md) | RED→GREEN 循环、CLI 命令、常见问题 |
| 24 | [验证与质量保证](part-4-project/24-verification.md) | Verifier Agent、追溯矩阵、验证报告 |
| 25 | [回顾与输出](part-4-project/25-retrospective.md) | 效能分析、SKILL.md 输出、经验总结 |

### Part 5: 进阶 (Advanced)

| # | 章节 | 主题 |
|---|------|------|
| 26 | [自主多代理流水线](part-5-advanced/26-autonomous-pipelines.md) | 权限模式、自主循环、安全栏杆 |
| 27 | [上下文管理策略](part-5-advanced/27-context-management.md) | Token 预算、子代理隔离、Prompt Cache |
| 28 | [团队与规模化](part-5-advanced/28-scaling-sdd.md) | 共享宪法、CI/CD 集成、多仓库 SDD |
| 29 | [SDD 的未来](part-5-advanced/29-future.md) | Spec-As-Source、形式化验证、自然语言编程 |

### 模板 (Templates)

| 文件 | 用途 |
|------|------|
| [项目宪法模板](templates/constitution-template.md) | CLAUDE.md 起步模板 |
| [功能规格模板](templates/spec-template.md) | EARS 格式功能规格 |
| [实现计划模板](templates/plan-template.md) | 架构与实现计划 |
| [任务分解模板](templates/task-template.md) | 原子任务卡片 |
| [审核检查清单](templates/review-checklist.md) | 阶段门审核清单 |
| [验证代理模板](templates/verifier-agent.md) | 验证子代理定义 |
| [存量改造指南](templates/brownfield-bootstrap.md) | 存量项目 SDD 引导 |

### 附录 (Appendix)

| 文件 | 内容 |
|------|------|
| [术语表](appendix/glossary.md) | SDD 中英双语术语表 (50+ 术语) |
| [EARS 速查卡](appendix/ears-cheatsheet.md) | EARS 记法快速参考 |
| [工具对比矩阵](appendix/tool-comparison-matrix.md) | 6 款 SDD 工具详细对比 |
| [推荐资源](appendix/further-reading.md) | 文档、论文、教程、视频、仓库 |

### 实战项目 (Project)

| 目录/文件 | 内容 |
|-----------|------|
| [project/CLAUDE.md](project/CLAUDE.md) | SpecTask 项目宪法 |
| [project/specs/](project/specs/) | 功能规格文件 (Task CRUD + Auth) |
| [project/plans/](project/plans/) | 实现计划 (v1-plan) |
| [project/.claude/agents/](project/.claude/agents/) | 验证代理定义 |
| [project/.claude/skills/](project/.claude/skills/) | SDD 工作流技能 |

---

## 项目结构

```
sdd/
├── README.md                          ← 你在这里
├── CLAUDE.md                          ← 项目宪法
├── part-1-foundations/                 ← 基础概念 (Ch.01-05)
├── part-2-claude-code-mastery/        ← Claude Code 精通 (Ch.06-12)
├── part-3-workflow/                   ← SDD 工作流 (Ch.13-17)
├── part-4-project/                    ← 实战项目教程 (Ch.18-25)
├── part-5-advanced/                   ← 进阶话题 (Ch.26-29)
├── templates/                         ← 即用模板 (7 files)
├── project/                           ← SpecTask 实战项目
│   ├── specs/                         ← Spec 文件
│   ├── plans/                         ← 实现计划
│   └── src/                           ← 源代码
└── appendix/                          ← 附录 (4 files)
```

---

## 前置条件

开始之前，请确保：

- **Claude Code** 已安装并配置（`claude` 命令可用）
- **Node.js** >= 18（用于运行示例代码）
- **Git** 已安装（版本控制）
- 基本的 CLI/终端操作知识
- 一个代码编辑器（推荐 VS Code）

可选但推荐：

- 了解 TypeScript 基础语法（示例代码使用 TypeScript）
- GitHub 账号（用于 PR 工作流示例）

---

## 参考资源

- [Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering)
- [CLAUDE.md 最佳实践](https://docs.anthropic.com/en/docs/claude-code/claude-md)
- [Claude Code Hooks 文档](https://docs.anthropic.com/en/docs/claude-code/hooks)

---

## 许可

本教程内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可。代码示例采用 MIT 许可。
