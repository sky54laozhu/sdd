# CLAUDE.md：你的项目宪法 (CLAUDE.md as Project Constitution)

> CLAUDE.md 是 Claude Code 每次启动时必读的项目级配置文件，它定义了 AI 与你协作的全部规则——相当于项目的宪法。

---

## 为什么需要项目宪法

每次你启动 Claude Code，它面对的是一片空白的 Context（上下文）。没有 CLAUDE.md，Claude Code 不知道：

- 这个项目是什么（一个 CLI 工具？一个 Web 应用？一个库？）
- 你用了什么技术栈（React 19 还是 Vue 3？Express 还是 Fastify？）
- 你的团队约定了什么代码风格
- 哪些事情绝对不能做（比如：绝不能删除生产数据库的 migration）

这就像雇了一位新工程师，但不给他看任何文档，直接让他写代码。结果可想而知——代码风格不统一、技术选型不对、架构混乱。

CLAUDE.md 的价值在于：**一次编写，每次会话自动生效**。它是一份 persistent instruction（持久化指令），不需要你每次都在 prompt 里重复。

---

## 文件层级系统 (File Hierarchy)

Claude Code 支持三级 CLAUDE.md 层级，从最宽到最窄：

| 层级 | 路径 | 作用域 | 典型内容 |
|------|------|--------|----------|
| User-level | `~/.claude/CLAUDE.md` | 所有项目通用 | 个人偏好、全局编码风格 |
| Project-level | `./CLAUDE.md` | 整个项目 | 技术栈、架构、工作流 |
| Directory-level | `./src/CLAUDE.md` | 特定目录 | 子模块专属约定 |

**合并规则**：Claude Code 会将所有层级的 CLAUDE.md 合并为一份指令集。当发生冲突时，**更具体的层级优先**（Directory > Project > User），这与 CSS 的 specificity（特异性）原则一致。

```
~/.claude/CLAUDE.md          ← "我偏好 2 空格缩进"
./CLAUDE.md                  ← "本项目使用 TypeScript + Express"
./src/database/CLAUDE.md     ← "此目录的文件不能超过 200 行"
```

### 何时使用 Directory-level CLAUDE.md

并非每个目录都需要自己的 CLAUDE.md。只有在以下场景才值得创建：

- **子模块有独立技术约束**：比如 `./packages/legacy/` 使用 CommonJS 而项目其余部分用 ESM
- **安全敏感目录**：比如 `./src/auth/CLAUDE.md` 声明 "此目录的所有变更必须经过 security review"
- **不同代码生成策略**：比如 `./migrations/CLAUDE.md` 声明 "此目录只允许追加文件，不允许修改已有 migration"

---

## 有效 CLAUDE.md 的解剖 (Anatomy of an Effective CLAUDE.md)

一份结构良好的 CLAUDE.md 包含以下七个模块：

### 1. 项目身份 (Project Identity)

```markdown
## 项目身份

这是一个 **实时协作白板应用**，面向远程设计团队。
核心价值是低延迟同步和直觉化交互。
```

告诉 Claude Code "你在做什么"。这决定了它在面对模糊决策时的倾向——比如在一个性能敏感的实时应用中，它会更倾向于选择 WebSocket 而非轮询。

### 2. 技术栈 (Tech Stack)

```markdown
## 技术栈

- **Runtime**: Node.js 20 LTS
- **Language**: TypeScript 5.4 (strict mode)
- **Framework**: Fastify 4.x
- **Database**: PostgreSQL 16 + Drizzle ORM
- **Testing**: Vitest + Playwright
- **Package Manager**: pnpm 9.x
```

精确到版本号。Claude Code 的训练数据包含多个版本的 API——如果你不指定 "Drizzle ORM"，它可能会生成 Prisma 代码；如果你不说 "pnpm"，它可能用 npm。

### 3. 架构约定 (Architecture Conventions)

```markdown
## 架构约定

- 目录组织按 feature 而非 file type
- 每个 feature 包含: routes、handlers、services、repositories
- 依赖注入使用 constructor injection
- 所有数据库访问必须通过 repository 层
```

### 4. 编码标准 (Code Standards)

```markdown
## 编码标准

- 函数命名: camelCase，动词开头（getUserById, validateInput）
- 类型命名: PascalCase（UserRepository, CreateUserInput）
- 文件命名: kebab-case（user-repository.ts）
- 最大函数长度: 40 行
- 最大文件长度: 300 行
- 错误处理: 使用 Result<T, E> pattern，禁止 throw 用于控制流
- 禁止使用 any 类型
- 所有公共函数必须有 JSDoc 注释
```

### 5. 测试要求 (Testing Requirements)

```markdown
## 测试要求

- 最低覆盖率: 80%
- 每个 feature 必须有: unit tests + integration tests
- 测试文件与源文件同目录，命名 *.test.ts
- 使用 TDD 工作流: 先写测试 → 实现 → 重构
- E2E 测试覆盖所有 critical user path
```

### 6. 安全边界 (Security Boundaries)

```markdown
## 安全边界（绝对禁止）

- 不得在代码中硬编码任何密钥、token 或密码
- 不得使用 eval() 或动态代码执行
- 不得禁用 TypeScript strict mode
- 不得直接拼接 SQL 字符串
- 不得在错误响应中暴露内部实现细节
- 不得跳过 input validation
```

**绝对禁止**（Security Boundaries）这一节至关重要。Claude Code 非常善于遵守明确的禁令——在这里写下的规则，它几乎不会违反。

### 7. 工作流定义 (Workflow Definition)

```markdown
## 工作流

本项目使用 SDD (Spec-Driven Development):

1. Spec: 先写规格文档，定义要做什么
2. Plan: 从 Spec 生成实现计划
3. Implement: 按计划 TDD 实现
4. Verify: 验证实现是否符合 Spec
```

---

## Living Document 原则

CLAUDE.md 不是写完就存档的静态文件。它是一份 **living document（活文档）**，应该随项目演进而更新。

一个强大的特性是：**Claude Code 可以自己更新 CLAUDE.md**。你可以在会话中说：

```
"我们决定从 Express 迁移到 Fastify。请更新 CLAUDE.md 中的技术栈部分。"
```

Claude Code 会用 Edit tool 修改 CLAUDE.md，下一次会话自动读取新版本。这形成了一个正反馈循环：

```
发现规则缺失 → 在会话中补充 → CLAUDE.md 更新 → 下次自动生效
```

### 何时更新 CLAUDE.md

- 技术栈变更（升级框架、更换库）
- 发现 Claude Code 反复犯同一个错误（说明规则缺失）
- 团队约定了新的编码规范
- 引入新的开发工作流
- 发现某条规则太模糊导致执行不一致

---

## 反模式 (Anti-Patterns)

### 1. 过长 (Too Long)

```markdown
<!-- 反模式：200+ 行的 CLAUDE.md -->
```

超过 200 行的 CLAUDE.md 会导致 instruction following（指令遵循度）下降。Claude Code 的 attention（注意力）会分散，后面的规则容易被忽略。

**解决方案**：保持 CLAUDE.md 在 100-150 行。把详细说明放在独立文件（如 `docs/architecture.md`），CLAUDE.md 只做索引和要点概述。

### 2. 过于模糊 (Too Vague)

```markdown
<!-- 反模式 -->
## 编码标准
写好的代码。遵循最佳实践。
```

"好的代码"和"最佳实践"对 Claude Code 来说毫无意义——它需要具体、可执行的规则。

**解决方案**：每条规则都应该是可判定的——给出一段代码，你能明确回答"这违反了规则吗？"

### 3. 自相矛盾 (Contradictory Rules)

```markdown
<!-- 反模式 -->
- 所有函数必须有完整 JSDoc 注释
- 代码应该 self-documenting，减少注释
```

Claude Code 面对矛盾规则时会随机选择一个执行，导致输出不一致。

**解决方案**：编写 CLAUDE.md 后，以"矛盾检测"的视角重新审阅一遍。

### 4. 重复代码中已有的信息 (Duplicating Code)

```markdown
<!-- 反模式 -->
## ESLint 规则
- no-unused-vars: error
- no-console: warn
- ...（复制整个 .eslintrc）
```

CLAUDE.md 不应重复 `tsconfig.json`、`.eslintrc`、`package.json` 中已有的配置。Claude Code 可以直接读取这些文件。

**解决方案**：CLAUDE.md 只写工具无法表达的意图和决策，比如"我们选择 Drizzle 而非 Prisma 的原因是需要更细粒度的查询控制"。

---

## 实战示例：完整 CLAUDE.md

以下是一个 TypeScript API 项目的完整 CLAUDE.md：

```markdown
# CLAUDE.md — TaskFlow API

## 项目身份

TaskFlow 是一个任务管理 API 服务，为多个前端客户端提供 RESTful 接口。
核心关注点：类型安全、性能、可测试性。

## 技术栈

- Runtime: Node.js 20 LTS
- Language: TypeScript 5.4 (strict mode)
- Framework: Fastify 4.x + @fastify/type-provider-typebox
- Database: PostgreSQL 16 + Drizzle ORM
- Cache: Redis 7 (ioredis)
- Testing: Vitest (unit/integration) + Playwright (E2E)
- Package Manager: pnpm 9.x
- Linting: Biome

## 架构

- 按 feature 组织: src/{feature}/{routes,handlers,services,repositories}.ts
- Controller → Service → Repository 分层
- 依赖注入: constructor injection + factory functions
- 错误处理: Result<T, AppError> pattern（不使用 throw）
- Validation: TypeBox schema 同时用于 runtime validation 和 type inference

## 编码标准

- 禁止 any 类型（使用 unknown + type narrowing）
- 函数 <= 40 行，文件 <= 300 行
- 命名: camelCase (functions), PascalCase (types), kebab-case (files)
- 纯函数优先，副作用显式标注
- 所有公共 API 必须有 JSDoc

## 测试

- 最低覆盖率: 80%
- TDD 工作流: RED → GREEN → REFACTOR
- 测试文件: *.test.ts（与源文件同目录）
- Integration tests 使用 testcontainers 启动真实 PostgreSQL

## 安全边界

- 禁止硬编码密钥（使用 env vars via @fastify/env）
- 禁止 SQL 字符串拼接（Drizzle 的 query builder 已强制参数化）
- 禁止 eval / new Function
- 所有外部输入必须经过 TypeBox schema 验证
- Error response 不暴露 stack trace 或内部路径

## 工作流 (SDD)

1. 编写 Spec（specs/ 目录）
2. 从 Spec 生成 Plan
3. TDD 实现
4. 验证符合 Spec
```

注意这个示例大约 60 行，简洁但完整。每条规则都是具体、可执行的。

---

## 启动建议 (Tips for Getting Started)

1. **从最小可行宪法开始**：只写项目身份 + 技术栈 + 3 条最重要的编码规则
2. **观察偏差再补充**：当你发现 Claude Code 的输出偏离期望时，把对应规则加入 CLAUDE.md
3. **一条规则解决一个问题**：不要预防性地添加大量规则——那只会稀释注意力
4. **定期精简**：每月审阅一次，删除已不适用的规则
5. **团队共建**：让团队成员都能提 PR 修改 CLAUDE.md，就像维护编码规范一样

---

## Key Takeaways (要点回顾)

- CLAUDE.md 是 Claude Code 每次启动时的第一读物，它定义了 AI 理解项目的全部上下文
- 三级层级（User → Project → Directory）允许从通用到具体的规则管理
- 有效的 CLAUDE.md 包含七个模块：项目身份、技术栈、架构、编码标准、测试、安全、工作流
- 保持简洁（100-150 行），避免模糊、矛盾、重复的规则
- 它是活文档——随项目演进而更新，Claude Code 自身也可以修改它

---

## Next (下一章)

[Ch.07 编写有效规格](07-writing-specs.md) — 学习如何用 EARS 符号法编写 AI 可精确执行的需求规格文档。
