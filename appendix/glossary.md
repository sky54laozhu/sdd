# SDD 术语表 / SDD Glossary

> 中英双语术语对照表，涵盖 Spec-Driven Development 核心概念及相关领域。

## 核心 SDD 术语 / Core SDD Terms

| English Term | 中文翻译 | Definition (中文) |
|---|---|---|
| Specification | 规格说明 | 对系统行为的精确、可验证的描述，是 SDD 的核心制品 |
| Spec-Driven Development (SDD) | 规格驱动开发 | 以规格说明为中心驱动所有开发活动的方法论 |
| Constitution | 宪法文件 | 项目级别的最高规则文件（如 CLAUDE.md），所有子任务必须遵守 |
| Phase Gate | 阶段门控 | 在开发流程中设置的检查点，只有满足条件才能进入下一阶段 |
| Acceptance Criteria | 验收标准 | 定义功能"完成"的明确条件，通常以可测试的形式表达 |
| Requirements Specification | 需求规格 | 对系统功能和非功能需求的正式文档 |
| Behavioral Contract | 行为契约 | 对组件/模块预期行为的正式约束描述 |
| Invariant | 不变量 | 在整个系统生命周期中必须始终保持为真的条件 |
| Precondition | 前置条件 | 函数/操作执行前必须满足的条件 |
| Postcondition | 后置条件 | 函数/操作成功执行后保证满足的条件 |
| Traceability | 可追溯性 | 从需求到实现再到测试的双向追踪能力 |
| Spec Decomposition | 规格分解 | 将高层规格逐步分解为可实现的子规格的过程 |
| Verifier | 验证器 | 自动检查实现是否符合规格的程序或代理 |
| Orchestrator | 编排器 | 协调多个子代理或子任务执行的控制层 |

## EARS 符号术语 / EARS Notation Terms

| English Term | 中文翻译 | Definition (中文) |
|---|---|---|
| EARS | EARS 需求句法 | Easy Approach to Requirements Syntax，一种结构化需求编写方法 |
| Ubiquitous Requirement | 无条件需求 | 系统在所有情况下都必须满足的需求，无触发条件 |
| Event-Driven Requirement | 事件驱动需求 | 由特定事件触发的需求，使用 WHEN 关键字 |
| State-Driven Requirement | 状态驱动需求 | 在特定系统状态下持续有效的需求，使用 WHILE 关键字 |
| Optional Feature Requirement | 可选特性需求 | 仅在特定配置或特性存在时适用的需求，使用 WHERE 关键字 |
| Unwanted Behavior Requirement | 异常行为需求 | 处理不期望情况的需求，使用 IF...THEN 结构 |
| Trigger | 触发器 | 引发事件驱动需求执行的具体事件 |
| System Response | 系统响应 | 需求中 shall 子句描述的系统行为 |

## Claude Code 术语 / Claude Code Terms

| English Term | 中文翻译 | Definition (中文) |
|---|---|---|
| CLAUDE.md | CLAUDE.md 文件 | Claude Code 的项目宪法文件，定义全局规则和约束 |
| Subagent | 子代理 | 主代理委派出去执行特定子任务的独立代理实例 |
| Hook | 钩子 | 在工具执行前后自动运行的脚本（PreToolUse / PostToolUse） |
| Skill | 技能 | 可复用的专业知识模块，通过 slash command 调用 |
| Task | 任务 | Claude Code 中分配给子代理的独立工作单元 |
| Plan Mode | 规划模式 | 先制定计划再执行的工作模式，适合复杂任务 |
| Extended Thinking | 扩展思考 | Claude 在生成回答前进行更深层推理的能力 |
| Context Window | 上下文窗口 | 模型单次对话能处理的最大 token 数量 |
| Compaction | 压缩 | 当上下文接近上限时，自动精简对话历史的机制 |

## 相关方法论 / Related Methodologies

| English Term | 中文翻译 | Definition (中文) |
|---|---|---|
| Test-Driven Development (TDD) | 测试驱动开发 | 先写测试再写实现的开发方法，Red-Green-Refactor 循环 |
| Behavior-Driven Development (BDD) | 行为驱动开发 | 使用自然语言描述行为场景来驱动开发的方法 |
| Domain-Driven Design (DDD) | 领域驱动设计 | 以业务领域模型为核心组织软件架构的方法 |
| Contract-First Development | 契约优先开发 | 先定义 API 契约（如 OpenAPI）再实现的方法 |
| Design-by-Contract (DbC) | 契约式设计 | 为组件定义前置/后置条件和不变量的设计方法（Bertrand Meyer 提出） |
| Formal Methods | 形式化方法 | 使用数学方法来规格化和验证软件正确性 |
| Literate Programming | 文学化编程 | 将文档和代码交织在一起的编程范式（Donald Knuth 提出） |

## AI 编程术语 / AI Coding Terms

| English Term | 中文翻译 | Definition (中文) |
|---|---|---|
| Vibe Coding | 氛围编程 | 通过自然语言描述意图让 AI 生成代码的编程方式 |
| Intent Drift | 意图漂移 | AI 在多轮交互中逐渐偏离用户原始意图的现象 |
| Context Decay | 上下文衰减 | 随着对话变长，早期信息影响力逐渐减弱的现象 |
| Hallucination | 幻觉 | AI 生成看似合理但实际不正确的代码或信息 |
| Agent | 代理 | 能自主规划和执行多步骤任务的 AI 系统 |
| Agentic Loop | 代理循环 | Agent 反复执行"思考-行动-观察"的自主工作循环 |
| Prompt Engineering | 提示工程 | 设计有效指令以获得期望 AI 输出的技术 |
| Grounding | 接地 | 将 AI 输出锚定在具体、可验证的事实或规格上 |
| Guardrails | 护栏 | 约束 AI 行为、防止越界的机制和规则 |
| Token | 令牌 | 模型处理文本的基本单位，约 3/4 个英文单词 |

## 工作流术语 / Workflow Terms

| English Term | 中文翻译 | Definition (中文) |
|---|---|---|
| Greenfield Project | 绿地项目 | 从零开始的全新项目，没有历史代码约束 |
| Brownfield Project | 棕地项目 | 在现有代码基础上进行修改和扩展的项目 |
| Spec-First | 规格优先 | 先编写完整规格再开始实现的工作方式 |
| Spec-Anchored | 规格锚定 | 实现过程中持续参照规格进行验证的工作方式 |
| CI/CD Pipeline | CI/CD 流水线 | 持续集成和持续部署的自动化工作管道 |
| Code Generation | 代码生成 | 根据规格或模板自动生成源代码 |
| Scaffold | 脚手架 | 自动生成项目基础结构和样板代码的工具 |
| Living Document | 活文档 | 随项目演进持续更新的规格或文档 |
| Single Source of Truth | 唯一真实来源 | 某类信息只有一个权威来源，避免不一致 |
| Drift Detection | 漂移检测 | 发现实现偏离规格的自动化机制 |

---

> 术语表将随教程系列更新持续扩充。如有建议或补充，欢迎贡献。
