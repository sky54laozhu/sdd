# 第五章：SDD 工具生态全景 (The SDD Tool Landscape)

> 2025-2026 年间，SDD 从理论主张变为实用工具生态。本章纵览六个主要 SDD 工具/框架，帮你做出 informed choice。

---

## 工具选型的前置思考

在评估 SDD 工具之前，你需要回答三个问题：

1. **你的团队在 [三层严格度模型](04-three-rigor-levels.md) 中处于哪一层？** — Spec-First 不需要重型工具；Spec-As-Source 需要深度集成
2. **你的现有 workflow 是什么？** — IDE-centric (Cursor/VS Code)、terminal-centric (CLI)、还是 CI-centric？
3. **你的 AI agent 策略是什么？** — 单一 agent (一个 LLM 做所有事)、multi-agent (多个专责 agent)、还是 hybrid？

带着这些问题，让我们逐一审视当前生态中的主要选手。

---

## 1. GitHub Spec Kit

### 来源与背景

GitHub 在 2025 年 9 月以 open-source CLI 工具的形式发布了 Spec Kit。它是 GitHub 对 "AI coding needs structure" 这个认知的直接产品化。Spec Kit 不绑定特定 AI model — 它是一个 spec-writing 和 spec-management 的框架。

### 核心功能

**三个核心命令：**

```bash
# 从需求生成结构化 specification
gh specify "Add webhook delivery to notification service"

# 从 spec 生成实现计划 (分步骤的 task breakdown)
gh plan specs/webhook-delivery.md

# 从 plan 生成可执行的 task list (可直接分配给 AI agent)
gh tasks plans/webhook-delivery-plan.md
```

**关键设计决策：**

- **Constitution file (宪法文件)**：`constitution.md` 定义了项目级别的规则和约束，所有 spec 都必须遵守
- **Agent-agnostic (不绑定 agent)**：生成的 specs/plans/tasks 可以喂给 Copilot、Claude、GPT 或任何 agent
- **30+ integrations**：与 GitHub Issues, Projects, Actions 深度集成
- **Spec format**：Markdown-based, human-readable, version-controlled

### 工作流

```
需求讨论 → gh specify → spec.md → 人类 review
                                      ↓
                              gh plan → plan.md → 人类 approve
                                                     ↓
                                             gh tasks → tasks.json
                                                          ↓
                                                   分配给 AI agents
```

### 优势

- 与 GitHub 生态无缝集成 (Issues, PRs, Actions)
- Open source (MIT license), 社区贡献活跃
- 不锁定 AI provider
- 低学习曲线 — 基于 Markdown, 不需要学新 DSL
- Constitution file 提供了 project-level guardrails

### 局限

- Spec 质量依赖于 initial prompt 的质量 (garbage in, garbage out)
- 没有 runtime drift detection — 只在 generation time 起作用
- 对 brownfield (存量) 项目的支持有限
- Constitution 的 enforcement 主要依赖 AI agent 的 compliance，没有硬性校验

### 最适合

- GitHub-centric 团队
- 需要 agent-agnostic 方案
- 正在从 "zero specification" 向 "Spec-First" 过渡
- 想要最低的工具切换成本

---

## 2. AWS Kiro

### 来源与背景

Amazon 在 2026 年 5 月发布了 Kiro — 一个基于 VS Code 的 "agentic IDE"。Kiro 不仅是一个编辑器，它内置了完整的 SDD workflow，从需求澄清到代码生成到持续验证。Kiro 是 AWS 对 "AI-native IDE" 的理解。

### 核心功能

**EARS Notation (Easy Approach to Requirements Syntax)：**

Kiro 采用了 EARS — 一种结构化的需求书写格式，比自由文本更精确，比形式化方法更易学：

```
WHEN <trigger>
THE SYSTEM SHALL <action>
UNLESS <exception>

# 示例：
WHEN user submits login form with valid credentials
THE SYSTEM SHALL create a session token with 24h expiry
UNLESS the account is locked due to 5+ failed attempts
```

**三层架构：**

| 层 | 功能 | 对应 artifact |
|----|------|---------------|
| **Specs** | 定义系统行为 | EARS requirements + acceptance criteria |
| **Steering** | 指导 AI 实现 | Design docs, architecture constraints |
| **Hooks** | 持续验证 | Pre-commit checks, CI integration |

**内置 AI models：**
- Claude Sonnet (primary reasoning)
- Amazon Nova (fast tasks, indexing)
- Dual-model 协作 for different task types

### 工作流

```
需求输入 → Kiro 引导式 spec writing (EARS)
  ↓
Spec review → 人类确认 acceptance criteria
  ↓
Steering docs → 架构约束、技术栈选择
  ↓
AI 实现 → 在 Specs + Steering 的约束下生成代码
  ↓
Hooks → 每次修改后自动验证 spec compliance
```

### 优势

- 最完整的 end-to-end SDD experience in an IDE
- EARS notation 比 free-form spec 更 rigorous, 比 formal methods 更 accessible
- Hooks 提供了 continuous verification (持续验证)
- VS Code 生态兼容 (extensions, keybindings, themes)
- AWS 资源支持，长期投入可预期

### 局限

- 绑定 AWS 生态 (虽然可以用于非 AWS 项目，但深度集成优势在 AWS 上)
- 目前仅支持 Claude Sonnet + Amazon Nova (model choice limited)
- IDE-centric — 不适合 terminal-first 工作流
- 2026 年 5 月刚发布，社区生态还在建设中
- EARS notation 有学习曲线

### 最适合

- VS Code 用户
- AWS 技术栈团队
- 想要 "开箱即用" 的完整 SDD 体验
- 不想自己组装工具链

---

## 3. BMAD-METHOD

### 来源与背景

BMAD-METHOD 是一个 community-driven (社区驱动) 的开源框架，2025 年初在 GitHub 上出现，迅速获得了 46K+ stars。它的核心创新是 **multi-agent team (多智能体团队)** — 不是用一个 AI 做所有事，而是模拟一个完整的软件团队。

### 核心功能

**Multi-Agent Team 架构：**

| Agent 角色 | 职责 | 产出物 |
|------------|------|--------|
| **Analyst** | 需求分析、用户研究 | PRD (Product Requirements Doc) |
| **PM** | 优先级排序、路线图 | Roadmap, Sprint plans |
| **Architect** | 系统设计、技术选型 | Architecture docs, ADRs |
| **Developer** | 代码实现 | Source code |
| **QA** | 测试策略、验证 | Test plans, test code |
| **DevOps** | 部署、监控 | CI/CD configs, infra code |

**Cross-platform 支持：**
- 可以运行在 Claude Code、Cursor、Copilot Workspace、甚至 ChatGPT 上
- Agent definitions 是 platform-agnostic 的 Markdown 文件
- 通过 system prompts 注入角色定义

### 工作流

```
用户需求 → Analyst agent (needs clarification, PRD)
  ↓
PRD → PM agent (prioritization, task breakdown)
  ↓
Tasks → Architect agent (design decisions, tech spec)
  ↓
Tech Spec → Developer agent (implementation)
  ↓
Code → QA agent (test plan, test implementation)
  ↓
Tested Code → DevOps agent (deployment config)
```

### 优势

- Open source (MIT)，社区极其活跃
- Multi-agent 模型模拟了真实团队动态
- Platform-agnostic — 不绑定任何特定工具
- 灵活 — 可以只用部分 agents，不需要全部
- 产出物丰富 (PRD, architecture, tests, deployment)

### 局限

- 配置复杂度高 — 需要理解和定制多个 agent prompts
- Agent 间的 "handoff (交接)" 质量取决于 prompt engineering
- 没有统一的 runtime — 每个 agent 可能跑在不同平台
- "46K stars" 包含大量 watchers，实际深度使用者可能较少
- 对小项目来说 overhead 过高

### 最适合

- 有 prompt engineering 经验的团队
- 想要模拟 "完整团队" 工作流的独立开发者
- 需要 cross-platform 灵活性
- 已有 multi-agent 实践经验

---

## 4. Claude Code Native

### 来源与背景

Claude Code 是 Anthropic 的 terminal-native AI coding tool。它的 SDD 能力不是通过额外插件实现的，而是通过其原生的 constitution (CLAUDE.md)、subagents (Tasks)、Hooks 和 Skills 机制来实现。

### 核心功能

**CLAUDE.md (Constitution file)：**
```markdown
# CLAUDE.md — 项目宪法

## Architecture
- Hexagonal architecture: domain/ has zero external imports
- All API responses use snake_case

## Constraints
- No ORM in domain layer
- Max file size: 800 lines
- All public functions must have JSDoc

## Conventions
- Error handling: Result<T, E> pattern, never throw
- Date format: ISO8601 everywhere
- IDs: UUIDv7
```

**Tasks (Subagents)：**
```bash
# 启动一个 spec-writing subagent
claude --task "Write a specification for the payment processing module
              following our CLAUDE.md conventions"
```

**Hooks：**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "command": "node scripts/check-spec-drift.js \"$FILE_PATH\""
    }]
  }
}
```

**Skills：**
预定义的 workflow templates，如 `/spec`, `/plan`, `/tdd-workflow` 等。

### 工作流

```
CLAUDE.md 定义 → 项目级约束 (始终有效)
  ↓
/spec 或手动写 spec → 功能级规格
  ↓
Tasks 分解 → 每个 task 都有 spec context
  ↓
实现 → Claude Code 在 CLAUDE.md + spec 的约束下生成代码
  ↓
Hooks → 每次编辑后自动验证 compliance
```

### 优势

- Terminal-native — 适合 "keyboard-first" 开发者
- CLAUDE.md 是天然的 "living spec" — 每次 Claude 都会读取
- Tasks 提供了 multi-agent 能力 (parallel execution)
- Hooks 提供了 continuous enforcement
- 与 git workflow 深度集成
- 不需要额外安装 — Claude Code 本身就够了

### 局限

- 绑定 Claude (Anthropic) 生态
- Terminal interface 的学习曲线 (对 IDE 用户)
- CLAUDE.md 的 enforcement 依赖 AI compliance，非 hard enforcement
- 大型 spec 可能超出 context window
- 目前没有 GUI-based spec editor

### 最适合

- Terminal-first 开发者
- 已经在使用 Claude Code 的团队
- 偏好 "configuration as code" 风格
- 想要最低的工具切换成本 (如果已在 Claude Code 中)

---

## 5. Cursor

### 来源与背景

Cursor 是一个 AI-native 代码编辑器 (VS Code fork)。它的 SDD 支持通过 **Plan Mode** 和 **AGENTS.md** 约定来实现。Cursor 的哲学是 "AI as pair programmer"，SDD 功能是这个哲学的自然延伸。

### 核心功能

**Plan Mode：**
在 Cursor 的 Agent mode 中，可以开启 "Plan before code" — AI 先制定实现计划，人类 approve 后才执行。

**AGENTS.md (convention file)：**
```markdown
# AGENTS.md — Agent behavior constraints

## Code Style
- Use functional React components only
- State management: Zustand for client, TanStack Query for server

## Architecture Rules
- No business logic in components
- API calls only in /services/ directory
- Types in /types/ directory, co-located with features

## When Editing Files
- Always add JSDoc to exported functions
- Maximum 300 lines per file
- Prefer composition over inheritance
```

**Rules for AI：**
Cursor 支持 `.cursorrules` 文件，作用类似 CLAUDE.md。

### 工作流

```
.cursorrules / AGENTS.md → 全局约束
  ↓
Plan Mode 激活 → AI 提出实现方案
  ↓
人类 review plan → approve / modify / reject
  ↓
AI 执行 → 在约束下实现代码
  ↓
Composer 集成 → multi-file 编辑, 保持一致性
```

### 优势

- VS Code 用户几乎零切换成本
- Plan Mode 天然是 Spec-First 思维
- GUI 友好，可视化 diff
- Composer 模式支持 multi-file coherent edits
- 活跃的社区和 plugin 生态

### 局限

- Plan 的 granularity 有限 — 不如 dedicated spec format 精确
- .cursorrules 是 unstructured text，没有 schema validation
- 不支持 formal drift detection
- Multi-agent 能力有限 (主要是单 agent workflow)
- 闭源，定价可能是限制因素

### 最适合

- VS Code 用户寻找 AI-native 升级
- 个人开发者或小团队
- 偏好 GUI over terminal
- 需要 "just works" 的开箱体验

---

## 6. OpenSpec

### 来源与背景

OpenSpec 是一个 minimalist (极简主义) 的 SDD framework，由 Fission AI 在 2025 年开源。它的核心理念是 **brownfield-first (存量项目优先)** — 不需要从零开始，可以对已有的大型代码库逐步引入 specification。

### 核心功能

**Incremental Specification：**
```yaml
# openspec.yaml — 只定义你关心的部分
modules:
  authentication:
    spec_status: complete
    spec_path: specs/auth.md
    coverage: 95%

  payments:
    spec_status: partial
    spec_path: specs/payments.md
    coverage: 60%
    gaps:
      - refund flow
      - subscription billing

  legacy_reports:
    spec_status: unspecified
    notes: "Will spec this when we refactor in Q3"
```

**Selective Drift Detection：**
只对 `spec_status: complete` 的模块运行 drift checks。未被 spec 覆盖的模块不会触发 noise alerts。

**Brownfield Adoption Tools：**
```bash
# 从现有代码反向推导 spec (reverse-engineering)
openspec infer src/auth/

# 对比推导出的 spec 与手写 spec 的差异
openspec diff specs/auth.md --inferred

# 逐步提升 spec coverage
openspec coverage --report
```

### 优势

- 专为 brownfield (legacy) 项目设计
- Incremental — 不需要一次性 spec 整个系统
- Reverse-engineering 能力帮助理解现有代码
- 选择性 drift detection — 只检查 "已 spec" 的部分
- Minimalist — 学习和配置成本极低

### 局限

- 社区较小，生态不如 GitHub Spec Kit 或 BMAD
- Reverse-engineered specs 可能不准确
- 功能范围有限 — 不包含 code generation
- Fission AI 作为小公司，长期维护的确定性较低
- 文档和教程较少

### 最适合

- 大型 legacy 代码库需要逐步引入 SDD
- 团队抵触 "big bang" 转型
- 只想要 drift detection, 不需要 code generation
- "Just enough spec" 的极简主义者

---

## 决策矩阵

| 维度 | GitHub Spec Kit | AWS Kiro | BMAD-METHOD | Claude Code | Cursor | OpenSpec |
|------|----------------|----------|-------------|-------------|--------|---------|
| **开源** | MIT | 否 | MIT | 部分 | 否 | MIT |
| **IDE 集成** | CLI | 完整 | 无 (跨平台) | Terminal | 完整 | CLI |
| **Multi-agent** | 否 | 双模型 | 完整团队 | Tasks | 有限 | 否 |
| **Drift Detection** | 否 | Hooks | 否 | Hooks | 否 | 核心功能 |
| **Brownfield 支持** | 弱 | 中 | 中 | 中 | 弱 | 强 |
| **学习曲线** | 低 | 中 | 高 | 中 | 低 | 低 |
| **Spec Rigor Level** | L1-L2 | L2-L3 | L1-L2 | L2 | L1 | L1-L2 |
| **社区规模** | 大 | 中 | 极大 | 大 | 大 | 小 |
| **最佳场景** | GitHub teams | AWS shops | Multi-agent fans | Terminal devs | VS Code users | Legacy codebases |

---

## 本教程的选择：Claude Code Native

本教程系列选择 **Claude Code Native** 作为主要工具，原因如下：

1. **Terminal-native** — 最少的 GUI 依赖，适合严肃的 engineering workflow
2. **CLAUDE.md 是天然的 living spec** — 无需额外工具，specification 就在项目根目录
3. **Tasks 提供了 multi-agent 能力** — 可以 parallel 执行 spec writing, implementation, testing
4. **Hooks 提供了 continuous enforcement** — PostToolUse hooks 在每次编辑后验证 compliance
5. **Skills 提供了 repeatable workflows** — `/spec`, `/plan`, `/tdd` 等 built-in patterns

**但原则是 tool-agnostic 的。** 本教程中讲授的 SDD 思想 — spec structure, drift detection, three rigor levels — 适用于任何工具。如果你使用 Kiro、Cursor 或 GitHub Spec Kit，具体的命令不同，但底层方法论完全相同。

---

## 生态趋势观察 (2026 年中)

几个值得关注的趋势：

### Convergence (趋同)

所有工具都在向类似的架构靠拢：
- Constitution/Rules file (项目级约束)
- Spec → Plan → Tasks pipeline
- AI agent + human review loop
- Drift detection / continuous verification

### Standardization Pressure (标准化压力)

社区开始讨论 "spec format interoperability (规格格式互操作性)" — 能否定义一种通用的 spec format，让不同工具的 spec 可以互相读取？目前还没有赢家，但 OpenAPI 在 API 层面已经证明了这条路是可行的。

### IDE vs Terminal vs CI

三个主要的 "execution context" 正在分化：
- **IDE-centric** (Kiro, Cursor): Spec writing 和 verification 集成在编辑器中
- **Terminal-centric** (Claude Code, GitHub Spec Kit): Command-line driven, pipeline-friendly
- **CI-centric** (OpenSpec): 主要在 CI pipeline 中运行，开发时不干预

没有绝对优劣 — 取决于团队的 workflow 偏好。

---

## Key Takeaways (要点回顾)

1. **2026 年的 SDD 生态已经足够成熟** — 有从极简 (OpenSpec) 到完整 IDE (Kiro) 的全频谱选择
2. **没有 "最好的" 工具** — 只有 "最适合你团队的 workflow、技术栈和严格度需求" 的工具
3. **所有工具都在趋同** — Constitution file + Spec-Plan-Tasks pipeline + Continuous verification 正在成为共识架构
4. **本教程选择 Claude Code Native** — 因为它 terminal-native、配置即代码、且 multi-agent 能力强；但所有 principles 都是 tool-agnostic 的
5. **最重要的不是选哪个工具** — 而是养成 "specification before implementation" 的习惯。工具会变，习惯会留

---

## Next (下一章)

Part 1 (理论基础) 到此完成。在 Part 2 中，我们将进入实战 — 从 Claude Code 的安装配置开始，一步步构建完整的 SDD workflow。进入 Part 2: Claude Code 精通之路。
