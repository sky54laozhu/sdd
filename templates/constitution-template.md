# CLAUDE.md Constitution Template / CLAUDE.md 宪法模板

> **用途 / Usage**: 复制此文件到项目根目录并重命名为 `CLAUDE.md`。填写每个占位符部分。
> Copy this file to your project root and rename it to `CLAUDE.md`. Fill in each placeholder section.
>
> **原则 / Principle**: CLAUDE.md 是 Claude Code 的"宪法"——它定义项目的边界、约定和工作流程。
> CLAUDE.md is the "constitution" for Claude Code — it defines the project's boundaries, conventions, and workflow.

---

## Project Identity / 项目身份

<!-- 
  填写说明：用 1-3 句话描述项目是什么、解决什么问题、面向谁。
  Instructions: Describe in 1-3 sentences what the project is, what problem it solves, and who it serves.
-->

**Project**: [项目名称 / Project Name]
**Description**: [一句话描述 / One-line description]
**Domain**: [业务领域 / Business domain, e.g., fintech, e-commerce, developer tools]
**Stage**: [开发阶段 / Development stage: greenfield | active | maintenance]

---

## Tech Stack / 技术栈

<!-- 
  填写说明：列出核心技术选型。Claude 会据此选择正确的模式和库。
  Instructions: List core technology choices. Claude uses this to select correct patterns and libraries.
-->

| Layer | Technology | Version |
|-------|-----------|---------|
| Language | [e.g., TypeScript] | [e.g., 5.x] |
| Runtime | [e.g., Node.js] | [e.g., 20 LTS] |
| Framework | [e.g., Next.js] | [e.g., 14.x] |
| Database | [e.g., PostgreSQL] | [e.g., 16] |
| ORM | [e.g., Prisma] | [e.g., 5.x] |
| Testing | [e.g., Vitest + Playwright] | |
| Package Manager | [e.g., pnpm] | |
| CI/CD | [e.g., GitHub Actions] | |

---

## Architecture Conventions / 架构约定

<!-- 
  填写说明：描述项目的架构模式、分层策略和模块边界。
  Instructions: Describe the project's architectural patterns, layering strategy, and module boundaries.
-->

### Pattern / 架构模式
[e.g., Clean Architecture / Hexagonal / Feature-sliced / Monolith / Microservices]

### Layering / 分层
```
[描述你的分层结构 / Describe your layering structure]
e.g.:
  presentation → application → domain → infrastructure
```

### Module Boundaries / 模块边界
- [规则1 / Rule 1: e.g., Modules communicate only through public APIs]
- [规则2 / Rule 2: e.g., No circular dependencies between modules]
- [规则3 / Rule 3: e.g., Domain layer has zero external dependencies]

---

## Code Standards / 编码标准

<!-- 
  填写说明：定义代码风格的硬性规则。这些规则 Claude 会严格遵守。
  Instructions: Define hard rules for code style. Claude will follow these strictly.
-->

### Naming / 命名
- Files: [e.g., kebab-case for files, PascalCase for components]
- Variables: [e.g., camelCase, descriptive, no abbreviations]
- Types/Interfaces: [e.g., PascalCase, no I-prefix]

### Style / 风格
- Max file length: [e.g., 400 lines]
- Max function length: [e.g., 40 lines]
- Max nesting depth: [e.g., 3 levels]
- Immutability: [e.g., Always prefer immutable patterns]
- Error handling: [e.g., Use Result type, never throw in domain layer]

### Imports / 导入
- [规则 / Rule: e.g., Use path aliases (@/), group by external → internal → relative]

### Forbidden Patterns / 禁止模式
- [e.g., No `any` type]
- [e.g., No `console.log` in production code]
- [e.g., No default exports]
- [e.g., No barrel files (index.ts re-exports)]

---

## File Structure / 文件结构

<!-- 
  填写说明：展示项目的目录结构。Claude 会按照此结构放置新文件。
  Instructions: Show the project's directory structure. Claude will place new files according to this.
-->

```
src/
├── [顶层目录 / top-level dir]/
│   ├── [子目录 / sub-dir]/
│   └── ...
├── ...
tests/
├── unit/
├── integration/
└── e2e/
```

### File Placement Rules / 文件放置规则
- [规则1 / Rule 1: e.g., Each feature gets its own directory under src/features/]
- [规则2 / Rule 2: e.g., Shared utilities go in src/lib/]
- [规则3 / Rule 3: e.g., Tests mirror the src/ structure]

---

## Testing Requirements / 测试要求

<!-- 
  填写说明：定义测试策略和覆盖率目标。
  Instructions: Define testing strategy and coverage targets.
-->

### Coverage Target / 覆盖率目标
- Minimum: [e.g., 80%]
- Critical paths: [e.g., 95%]

### Testing Strategy / 测试策略
- Unit tests: [e.g., All pure functions and domain logic]
- Integration tests: [e.g., API endpoints, database queries]
- E2E tests: [e.g., Critical user journeys]

### Test Commands / 测试命令
```bash
# Unit tests / 单元测试
[e.g., pnpm test]

# Integration tests / 集成测试
[e.g., pnpm test:integration]

# E2E tests / 端到端测试
[e.g., pnpm test:e2e]

# Coverage report / 覆盖率报告
[e.g., pnpm test:coverage]
```

### TDD Workflow / TDD 工作流
When implementing features, follow RED → GREEN → REFACTOR:
1. Write failing test
2. Write minimal implementation to pass
3. Refactor while keeping tests green

---

## Security Boundaries / 安全边界

<!-- 
  填写说明：定义安全敏感区域和禁止操作。Claude 在这些区域会格外小心。
  Instructions: Define security-sensitive areas and forbidden operations. Claude will be extra careful in these areas.
-->

### Sensitive Areas / 敏感区域
- [e.g., src/auth/ — authentication logic]
- [e.g., src/payments/ — payment processing]
- [e.g., migrations/ — database schema changes]

### Forbidden Operations / 禁止操作
- [e.g., Never hardcode secrets, always use environment variables]
- [e.g., Never disable CSRF protection]
- [e.g., Never use raw SQL without parameterized queries]
- [e.g., Never commit .env files]

### Environment Variables / 环境变量
```
# Required / 必须
[e.g., DATABASE_URL=]
[e.g., API_SECRET_KEY=]

# Optional / 可选
[e.g., LOG_LEVEL=info]
```

---

## SDD Workflow / SDD 工作流定义

<!-- 
  填写说明：定义本项目使用的 SDD 阶段和规则。
  Instructions: Define the SDD phases and rules used in this project.
-->

### Phases / 阶段
```
Specify (规格) → Plan (计划) → Tasks (任务) → Implement (实现) → Verify (验证)
```

### Workflow Rules / 工作流规则
1. **No implementation without a spec** / 没有规格不实现
   - Every feature must have a spec file in `specs/` before coding begins
2. **No code without a plan** / 没有计划不写代码
   - Implementation plan must exist in `plans/` before task decomposition
3. **Tests before implementation** / 测试先于实现
   - Task ordering always puts test creation before production code
4. **Verify against spec** / 对照规格验证
   - Every PR must trace back to spec requirements

### File Locations / 文件位置
```
specs/          — Feature specifications (EARS format)
plans/          — Implementation plans
tasks/          — Task decomposition files
docs/           — Additional documentation
```

---

## Additional Instructions / 附加指令

<!-- 
  填写说明：任何不属于上述类别的项目特定规则。
  Instructions: Any project-specific rules that don't fit the categories above.
-->

- [e.g., Always use Chinese comments for business logic]
- [e.g., Prefer composition over inheritance]
- [e.g., Use feature flags for incomplete features]
- [e.g., All API responses follow envelope format { data, error, meta }]
