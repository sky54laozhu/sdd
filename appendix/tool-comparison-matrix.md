# SDD 工具对比矩阵 / Tool Comparison Matrix

> 截至 2026 年中，主流 Spec-Driven Development 工具的横向对比。

## 工具概览 / Overview

| # | 工具名称 | 类型 | 核心理念 |
|---|----------|------|----------|
| 1 | GitHub Spec Kit | 开源 CLI | 规格即代码，CI/CD 集成 |
| 2 | AWS Kiro | Agentic IDE | EARS 原生支持，AWS 深度集成 |
| 3 | BMAD-METHOD | 多代理框架 | 角色分工，全流程覆盖 |
| 4 | Claude Code Native | CLI Agent | CLAUDE.md 宪法 + Tasks + Hooks |
| 5 | Cursor | AI IDE | Plan Mode + AGENTS.md |
| 6 | OpenSpec | 轻量级框架 | 极简主义，棕地项目友好 |

---

## 详细对比 / Detailed Comparison

### 1. GitHub Spec Kit

| 维度 | 详情 |
|------|------|
| **方法论** | Spec-as-Code：规格文件存储在仓库中，与代码同等对待 |
| **开发方式** | 开源 CLI 工具，可集成到任何 CI/CD 管道 |
| **规格格式** | Markdown + YAML front matter，支持自定义 schema |
| **IDE 锁定** | 无锁定，任何编辑器均可使用 |
| **Agent 支持** | 30+ 集成（GitHub Actions, Claude, GPT, Gemini 等） |
| **团队功能** | GitHub PR 审查流程，规格变更追踪，CODEOWNERS 支持 |
| **学习曲线** | 中等 — 需理解 spec 文件结构和 CLI 命令 |
| **成本** | 免费开源，企业版提供额外治理功能 |
| **最适合** | 已有 GitHub 工作流的团队，希望渐进式采用 SDD |

**核心特性:**
- `spec lint` — 规格格式验证
- `spec test` — 从规格自动生成验收测试
- `spec drift` — 检测实现与规格的偏离
- `spec generate` — 从规格生成代码骨架

---

### 2. AWS Kiro

| 维度 | 详情 |
|------|------|
| **方法论** | Spec-First：强制先写规格，再由 Agent 实现 |
| **开发方式** | 独立 Agentic IDE（基于 VS Code fork） |
| **规格格式** | EARS 原生语法 + Steering 文件 (.kiro/) |
| **IDE 锁定** | 高 — 必须使用 Kiro IDE |
| **Agent 支持** | 内置多代理系统（Spec Agent, Code Agent, Test Agent） |
| **团队功能** | 规格审批流程，Hook 系统，AWS 资源自动配置 |
| **学习曲线** | 低 — 对 EARS 不熟悉需短期学习 |
| **成本** | 免费预览期；预计按 AWS 使用量计费 |
| **最适合** | AWS 生态内的团队，希望一站式 SDD 体验 |

**核心特性:**
- 自动将自然语言需求转换为 EARS 格式
- Spec → Task 自动分解
- 内置 Acceptance Criteria 验证循环
- 与 AWS CDK / CloudFormation 深度集成

---

### 3. BMAD-METHOD

| 维度 | 详情 |
|------|------|
| **方法论** | Multi-Agent Spec-Driven：多角色代理协作完成全流程 |
| **开发方式** | 框架/方法论，可嵌入任何 AI 编码工具 |
| **规格格式** | 分层 Markdown（PRD → Architecture → Stories → Tasks） |
| **IDE 锁定** | 无锁定，适配 Claude Code / Cursor / Windsurf 等 |
| **Agent 支持** | 7+ 内置角色（PM, Architect, PO, Dev, QA, SM, Analyst） |
| **团队功能** | 角色模板可定制，支持团队协作流程 |
| **学习曲线** | 高 — 需理解完整方法论和角色分工 |
| **成本** | 免费开源（MIT License） |
| **最适合** | 大型项目、需要全流程管控的团队 |

**核心特性:**
- BMad Orchestrator 统一调度
- Story → Task 自动分解
- 内置质量门控（Definition of Done）
- 46K+ GitHub Stars，活跃社区

---

### 4. Claude Code Native

| 维度 | 详情 |
|------|------|
| **方法论** | Spec-Anchored：CLAUDE.md 作为宪法，规格锚定开发过程 |
| **开发方式** | CLI Agent，运行在终端中 |
| **规格格式** | Markdown（CLAUDE.md + 自定义规格文件） |
| **IDE 锁定** | 无锁定，任何终端和编辑器均可 |
| **Agent 支持** | 原生 Task 并行子代理 + Hook 系统 |
| **团队功能** | .claude/ 目录约定，settings.json 权限控制 |
| **学习曲线** | 中等 — 需学习 CLAUDE.md 约定和 Hook/Skill 系统 |
| **成本** | Claude API 使用量计费（Max 订阅或 API） |
| **最适合** | 个人开发者和小团队，灵活且可高度定制 |

**核心特性:**
- CLAUDE.md 作为 Single Source of Truth
- Task 工具实现并行子代理
- PreToolUse / PostToolUse Hook
- Skill 系统（/slash commands）
- Extended Thinking 深度推理

---

### 5. Cursor

| 维度 | 详情 |
|------|------|
| **方法论** | Plan-then-Execute：Plan Mode 规划，Agent 模式执行 |
| **开发方式** | AI-first IDE（VS Code fork） |
| **规格格式** | .cursor/rules/ + AGENTS.md |
| **IDE 锁定** | 高 — 必须使用 Cursor IDE |
| **Agent 支持** | 内置 Agent 模式，支持 MCP 协议 |
| **团队功能** | Team 版本共享规则，Usage Dashboard |
| **学习曲线** | 低 — IDE 体验流畅，上手快 |
| **成本** | Pro $20/月，Business $40/月/人 |
| **最适合** | 偏好 IDE 体验的开发者，快速原型开发 |

**核心特性:**
- Plan Mode 生成实现计划
- AGENTS.md 定义多代理行为
- Rules for AI 项目级规则
- 长上下文 + 自动 codebase 索引
- Tab 补全 + 内联编辑

---

### 6. OpenSpec

| 维度 | 详情 |
|------|------|
| **方法论** | Spec-Anchored Minimal：最小化规格，持续锚定 |
| **开发方式** | 轻量级约定 + CLI 辅助工具 |
| **规格格式** | 简约 Markdown，模块化 spec 片段 |
| **IDE 锁定** | 无锁定 |
| **Agent 支持** | 工具无关，可与任何 AI Agent 配合 |
| **团队功能** | 极简主义，依赖团队自身的 Git 工作流 |
| **学习曲线** | 低 — 概念简单，5 分钟上手 |
| **成本** | 免费开源 |
| **最适合** | 棕地项目、渐进式改善、不想引入重型框架的团队 |

**核心特性:**
- "Just enough spec" 哲学
- 模块化 spec 片段，按需添加
- 棕地友好：不要求重写现有文档
- 与现有工具链零冲突

---

## 横向对比矩阵 / Cross-Comparison Matrix

| 维度 | Spec Kit | Kiro | BMAD | Claude Code | Cursor | OpenSpec |
|------|----------|------|------|-------------|--------|---------|
| **方法论** | Spec-as-Code | Spec-First | Multi-Agent | Spec-Anchored | Plan-Execute | Minimal Spec |
| **IDE 锁定** | 无 | 高 | 无 | 无 | 高 | 无 |
| **Agent 支持** | 多工具集成 | 内置多代理 | 多角色代理 | Task 子代理 | 内置 Agent | 工具无关 |
| **EARS 支持** | 插件 | 原生 | 无 | 手动 | 无 | 无 |
| **团队协作** | GitHub 原生 | AWS 组织 | 角色模板 | .claude/ 约定 | Team 版 | Git 约定 |
| **Drift 检测** | 内置 | 内置 | 手动 | Hook 实现 | 无 | 无 |
| **学习曲线** | 中 | 低 | 高 | 中 | 低 | 低 |
| **成本** | 免费/企业版 | AWS 计费 | 免费 | API 计费 | $20-40/月 | 免费 |
| **绿地项目** | 优 | 优 | 优 | 良 | 良 | 中 |
| **棕地项目** | 良 | 中 | 中 | 优 | 良 | 优 |
| **开源** | 是 | 否 | 是 | 部分 | 否 | 是 |

---

## 选择建议 / Recommendation Guide

### 场景导向选择

| 你的情况 | 推荐工具 | 理由 |
|----------|----------|------|
| AWS 重度用户，新项目 | Kiro | 原生 AWS 集成，EARS 内置 |
| 已有 GitHub 工作流 | Spec Kit | 无缝融入现有流程 |
| 大型团队，多角色协作 | BMAD-METHOD | 角色分工明确，流程完整 |
| 个人/小团队，灵活定制 | Claude Code Native | 高度可定制，无锁定 |
| 偏好 IDE 体验 | Cursor | 上手快，体验流畅 |
| 棕地项目，渐进改善 | OpenSpec | 零侵入，按需添加 |

### 组合使用

工具之间并非互斥，常见组合：

- **Claude Code + Spec Kit**: Claude Code 写代码，Spec Kit 做 CI 验证
- **BMAD + Claude Code**: BMAD 做规划和角色分工，Claude Code 做实现
- **Kiro + Spec Kit**: Kiro 做开发，Spec Kit 做跨项目规格治理
- **OpenSpec + 任何 Agent**: OpenSpec 定义规格，搭配任何 AI 工具实现

---

## 趋势观察 / Trends (2026)

1. **规格格式趋向标准化** — EARS 逐渐成为事实标准
2. **Drift Detection 成为标配** — 自动检测实现偏离规格
3. **Multi-Agent 架构普及** — 从单一 Agent 到角色化代理团队
4. **规格与测试融合** — 从规格直接生成验收测试
5. **棕地友好性提升** — 工具不再假设从零开始
6. **IDE-agnostic 趋势** — 减少工具锁定，拥抱开放协议（如 MCP）
