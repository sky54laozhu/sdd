# 第二十八章：团队与规模化 (Scaling SDD to Teams)

> 个人的 SDD 是一把利剑；团队的 SDD 是一支军队——需要统一的号令和阵型。

---

## 28.1 Solo SDD vs Team SDD：规模化带来的变化

当 SDD 从一个人的实践变成团队的方法论时，三个新维度出现了：

```
Solo SDD:
  Developer ←→ AI Agent
  (一个人的对话，自己的规则)

Team SDD:
  Multiple Developers ←→ Multiple AI Agents
  Shared Standards ←→ Shared Specs ←→ Shared Verification
  (需要一致性、协作、治理)
```

具体变化：

| 维度 | Solo | Team |
|------|------|------|
| CLAUDE.md | 个人偏好 | 组织标准 + 团队约定 + 项目规则 |
| Spec ownership | 自己写、自己用 | 多人协作、PR review |
| Quality gate | 自我判断 | CI/CD 自动化 + peer review |
| Convention drift | 无所谓 | 必须防止 |
| Onboarding | 不需要 | 关键需求 |

---

## 28.2 分层 Constitution (Shared Constitutions)

在团队环境中，CLAUDE.md 需要分层管理：

```
┌─────────────────────────────────────────────┐
│  Organization Level (~/.claude/CLAUDE.md)    │
│  ─────────────────────────────────────────  │
│  - Security policies (no secrets in code)   │
│  - Code style standards (formatting, naming)│
│  - Language/framework mandates              │
│  - Compliance requirements                  │
└─────────────────┬───────────────────────────┘
                  │ inherits
┌─────────────────v───────────────────────────┐
│  Team Level (team-repo/.claude/CLAUDE.md)   │
│  ─────────────────────────────────────────  │
│  - Architecture patterns (hexagonal, etc.)  │
│  - Tech stack specifics (Next.js 15, etc.) │
│  - Team-specific conventions                │
│  - Shared agent configurations              │
└─────────────────┬───────────────────────────┘
                  │ inherits
┌─────────────────v───────────────────────────┐
│  Project Level (project/.claude/CLAUDE.md)  │
│  ─────────────────────────────────────────  │
│  - Project-specific rules                   │
│  - Feature flags and toggles               │
│  - Migration notes                          │
│  - Override team rules when needed          │
└─────────────────────────────────────────────┘
```

### Inheritance Rule（继承规则）

Project overrides Team overrides Org。就像 CSS specificity——更具体的规则优先。

### 示例：Organization Level CLAUDE.md

```markdown
# Organization Standards

## Security (MANDATORY)
- Never commit secrets, API keys, or tokens to source code
- All user input must be validated before processing
- Use parameterized queries for all database operations
- Authentication changes require security-reviewer approval

## Code Quality
- Maximum file size: 800 lines
- Maximum function length: 50 lines  
- Minimum test coverage: 80%
- All public APIs must have documentation

## Git
- Conventional commits format required
- Feature branches from main
- Squash merge to main
```

### 示例：Team Level CLAUDE.md

```markdown
# Backend Team Standards

Extends: organization CLAUDE.md

## Architecture
- Hexagonal architecture (ports and adapters)
- Repository pattern for data access
- Service layer for business logic
- DTOs for API boundaries

## Stack
- Runtime: Node.js 22 LTS
- Framework: Fastify 5
- Database: PostgreSQL 16 + Drizzle ORM
- Testing: Vitest + Supertest

## API Conventions
- RESTful with JSON:API envelope
- Versioning via URL path (/api/v1/)
- Error format: { error: { code, message, details } }
```

### 分发策略

```bash
# 方案 1: Git submodule
git submodule add git@github.com:org/claude-standards.git .claude/org-standards

# 方案 2: 安装脚本
# install-claude-standards.sh
curl -sL https://internal.org/claude/org.md > ~/.claude/CLAUDE.md
curl -sL https://internal.org/claude/team-backend.md > .claude/CLAUDE.md

# 方案 3: Package manager
npm install --dev @org/claude-config
# postinstall script copies CLAUDE.md to .claude/
```

---

## 28.3 Spec 作为协作制品 (Spec as Collaboration Artifact)

在团队中，spec 不再是一个人写完就丢给 AI 的东西。它变成了一个 **living document**，多角色协作完成：

```
Product Manager          Tech Lead              Developer
     │                      │                      │
     v                      v                      v
┌──────────┐         ┌──────────┐         ┌──────────────┐
│ Business │         │Technical │         │ Acceptance   │
│ Require- │────────>│Constraints│────────>│  Criteria    │
│  ments   │         │& Design  │         │ Refinement   │
└──────────┘         └──────────┘         └──────────────┘
     │                      │                      │
     └──────────────────────┴──────────────────────┘
                            │
                            v
                    ┌──────────────┐
                    │  Final Spec  │
                    │ (PR merged)  │
                    └──────────────┘
```

### 角色分工

**Product Manager 贡献：**
- Business context（为什么做这个 feature）
- User stories
- Priority 和 scope boundaries
- Non-functional requirements（性能、可用性）

**Tech Lead 贡献：**
- System constraints（现有架构限制）
- Technical approach（推荐的实现方式）
- Integration points（与其他系统的交互）
- Risk assessment

**Developer 贡献：**
- Acceptance criteria refinement（确保可测试）
- Edge case identification
- Implementation feasibility feedback
- Test strategy

### 协作工作流

```
1. PM creates spec draft (business requirements)
   → Opens PR: "specs/feature-x-spec.md"

2. Tech Lead reviews, adds technical section
   → Commits to same PR

3. Developer reviews, refines acceptance criteria
   → Commits to same PR

4. All three approve → PR merged to main

5. Developer uses final spec for SDD implementation
```

---

## 28.4 Spec Review 即 Code Review (Spec Review as Code Review)

Spec 应该和 code 接受同等严格的 review。

### Spec Review Checklist

```markdown
## Spec Review Checklist

### Clarity（清晰性）
- [ ] 每个 requirement 是否有唯一 ID (REQ-N)?
- [ ] 是否使用了 EARS 模式 (Event, Action, Response, State)?
- [ ] 是否避免了歧义词 ("appropriate", "reasonable", "etc.")?

### Testability（可测试性）
- [ ] 每个 requirement 是否有至少一个 acceptance criterion?
- [ ] Acceptance criteria 是否可以自动验证?
- [ ] 是否定义了 happy path 和 error cases?

### Scope（范围）
- [ ] 是否明确了 in-scope 和 out-of-scope?
- [ ] 单个 spec 是否可以在 1-3 天内实现?
- [ ] 是否避免了 scope creep (范围蔓延)?

### Completeness（完整性）
- [ ] 是否覆盖了所有 user-facing 场景?
- [ ] 是否定义了 error handling 行为?
- [ ] 是否说明了 edge cases?

### Technical Feasibility（技术可行性）
- [ ] 是否与现有架构兼容?
- [ ] 是否识别了依赖和风险?
- [ ] 性能约束是否现实?
```

### Spec Diff Review 的优势

Markdown spec 文件在 GitHub/GitLab 的 diff view 中天然可读：

```diff
## REQ-3: Password Reset

- Users SHALL be able to reset their password via email.
+ Users SHALL be able to reset their password via email link.
+ The reset link SHALL expire after 1 hour.

### Acceptance Criteria
  - AC-3.1: Request with valid email sends reset email
  - AC-3.2: Request with invalid email returns 200 (no info leak)
+ - AC-3.3: Reset link expires after 60 minutes
+ - AC-3.4: Used reset link cannot be reused
```

每一行变更都清晰可见，reviewers 可以对特定的 requirement 进行 inline comment。

---

## 28.5 CI/CD Integration (持续集成集成)

将 SDD 嵌入 CI/CD pipeline 确保 spec discipline 不会随着时间退化。

### Pre-commit Hook: Spec Format Validation

```bash
#!/bin/bash
# .husky/pre-commit

# Check spec format for any modified spec files
MODIFIED_SPECS=$(git diff --cached --name-only | grep "specs/.*\.md$")

if [ -n "$MODIFIED_SPECS" ]; then
  for spec in $MODIFIED_SPECS; do
    # Check: has at least one REQ-N
    if ! grep -q "REQ-[0-9]" "$spec"; then
      echo "ERROR: $spec missing requirement IDs (REQ-N format)"
      exit 1
    fi
    
    # Check: has acceptance criteria
    if ! grep -q "AC-[0-9]" "$spec"; then
      echo "ERROR: $spec missing acceptance criteria (AC-N format)"
      exit 1
    fi
    
    # Check: no ambiguous words
    if grep -qi "appropriate\|reasonable\|as needed\|etc\." "$spec"; then
      echo "WARNING: $spec contains ambiguous language"
    fi
  done
fi
```

### CI Pipeline: Spec Compliance Check

```yaml
# .github/workflows/spec-check.yml
name: Spec Compliance

on: [pull_request]

jobs:
  spec-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Validate spec format
        run: |
          for spec in specs/*.md; do
            echo "Checking $spec..."
            
            # Must have requirements section
            grep -q "## REQ-" "$spec" || \
              (echo "FAIL: $spec missing requirements" && exit 1)
            
            # Must have acceptance criteria
            grep -q "## Acceptance Criteria\|### AC-" "$spec" || \
              (echo "FAIL: $spec missing acceptance criteria" && exit 1)
            
            # Must have scope section
            grep -q "## Scope\|## In Scope" "$spec" || \
              (echo "WARN: $spec missing scope definition")
          done

  spec-coverage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Check spec coverage
        run: |
          TOTAL_FEATURES=$(find src/features -maxdepth 1 -type d | wc -l)
          SPECS=$(find specs -name "*.md" | wc -l)
          COVERAGE=$((SPECS * 100 / TOTAL_FEATURES))
          
          echo "Spec coverage: $COVERAGE% ($SPECS/$TOTAL_FEATURES)"
          
          if [ "$COVERAGE" -lt 60 ]; then
            echo "WARNING: Spec coverage below 60%"
          fi
```

### Post-merge: Update Metrics

```yaml
  update-metrics:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - name: Calculate SDD metrics
        run: |
          echo "spec_count=$(find specs -name '*.md' | wc -l)" >> metrics.env
          echo "spec_with_tests=$(..." >> metrics.env
          echo "first_pass_rate=..." >> metrics.env
          # Push to metrics dashboard
```

---

## 28.6 多仓库 SDD (Multi-Repo SDD)

当系统由多个 service 组成时，SDD 的挑战变得更复杂。

### Shared Spec Registry（共享规格注册表）

```
spec-registry/              # 独立仓库
├── contracts/
│   ├── user-service-api.md      # User service 暴露的 API spec
│   ├── payment-service-api.md   # Payment service 的 API spec
│   └── notification-events.md   # Event bus 的 event schema
├── shared-types/
│   ├── user.schema.json         # Shared user type definition
│   └── payment.schema.json      # Shared payment type
└── versioning.md                # Spec versioning policy
```

### API Spec 作为服务间契约

```markdown
# User Service API Contract v2.1

## GET /api/v2/users/:id

### Response (200)
```json
{
  "data": {
    "id": "uuid",
    "email": "string",
    "name": "string",
    "createdAt": "ISO8601"
  }
}
```

### Consumers
- Payment Service (reads user email for receipts)
- Notification Service (reads user name for personalization)

### Breaking Change Policy
- New fields: non-breaking, can add freely
- Remove/rename fields: BREAKING, requires version bump
- Consumers must handle unknown fields gracefully
```

### Spec Versioning

```
规则：
- MAJOR (v1 → v2): breaking changes to existing fields
- MINOR (v2.0 → v2.1): new optional fields, new endpoints
- PATCH (v2.1.0 → v2.1.1): clarification, typo fixes

所有 consumers 必须 pin 到 MAJOR version
Producer 必须支持 N-1 MAJOR version (backward compatibility)
```

---

## 28.7 组织级 Managed Agents（托管代理）

在团队规模下，标准化 agent 配置可以确保一致性。

### Organization-level Agent Definitions

```markdown
# org-agents/verifier.md

## Role: Spec Verifier Agent

You are a verification agent for [OrgName]. Your job is to verify
implementation against spec.

### Rules:
1. Check EVERY acceptance criterion, not just some
2. Use the EXACT test commands defined in the project
3. Report results in this format:
   - PASS: AC-X.Y verified ✓
   - FAIL: AC-X.Y failed — [reason]
   - SKIP: AC-X.Y cannot be tested automatically — [reason]
4. Never mark PASS unless test actually passes
5. Never skip a criterion without explaining why
```

### Shared Skill Definitions

团队可以共享 custom skill 定义：

```
org-skills/
├── spec-writing.md        # How to write specs in our format
├── api-design.md          # API design conventions
├── database-migration.md  # Database change process
└── security-review.md     # Security review checklist
```

这些 skill 通过 CLAUDE.md 中的引用分发给所有团队成员的 AI agent：

```markdown
# Project CLAUDE.md

## Skills Reference
When writing specs, follow: @org-skills/spec-writing.md
When designing APIs, follow: @org-skills/api-design.md
```

---

## 28.8 度量仪表盘 (Metrics Dashboard)

度量让 SDD 的价值可见、可追踪。

### 核心指标

| 指标 | 定义 | 目标 |
|------|------|------|
| Spec Coverage | features with specs / total features | > 80% |
| Spec-to-Code Drift | specs outdated vs code | < 10% |
| First-Pass Accuracy | tasks completed without rework | > 70% |
| Phase Gate Pass Rate | tasks passing verification on first try | > 60% |
| Mean Time to Implementation | spec merged → feature complete | trending down |
| Rework Rate | tasks requiring > 1 retry | < 30% |

### 数据收集

```jsonl
{"date":"2026-06-01","feature":"user-crud","spec_coverage":1.0,"first_pass":0.83,"rework_rate":0.17,"mttr_hours":4.2}
{"date":"2026-06-03","feature":"payment-flow","spec_coverage":1.0,"first_pass":0.71,"rework_rate":0.29,"mttr_hours":6.8}
{"date":"2026-06-05","feature":"notification","spec_coverage":0.8,"first_pass":0.60,"rework_rate":0.40,"mttr_hours":8.1}
```

### 趋势分析

```
First-Pass Accuracy Over Time:

100% │
 90% │                              ╭──
 80% │                    ╭────────╯
 70% │          ╭────────╯
 60% │    ╭────╯
 50% │───╯
     └────────────────────────────────────
     Week 1  Week 2  Week 3  Week 4  Week 5

解读：随着 spec 质量提升和团队 SDD 经验积累，
first-pass accuracy 应该呈上升趋势。
```

如果 first-pass accuracy 停滞或下降，通常意味着：
- Spec 质量不够高（歧义太多）
- Task 分解粒度不对（太大或太小）
- CLAUDE.md 的 convention 与实际代码不一致

---

## 28.9 新人入职 (Onboarding New Team Members)

SDD 的一个隐藏好处：它天然产生文档。

### Spec 即文档

新团队成员可以通过阅读 spec 文件来理解：
- 系统做什么（business requirements）
- 如何做的（technical approach）
- 为什么这样做（design decisions in tech notes）
- 怎么验证（acceptance criteria = expected behavior）

### CLAUDE.md 即项目指南

新人入职时读 CLAUDE.md 就能了解：
- 项目使用什么技术栈
- 代码应该怎么组织
- 测试怎么写和运行
- Commit 格式是什么
- 有哪些不可违反的规则

### 入职 Workflow

```markdown
# 新人 SDD 入职计划（第一周）

## Day 1
- 阅读 Organization CLAUDE.md（了解公司标准）
- 阅读 Project CLAUDE.md（了解项目规则）
- 运行 project setup 命令

## Day 2
- 阅读 3 个已完成的 spec 文件（了解 spec 格式）
- 阅读对应的 plan 和 task-list 文件
- 对比 spec 和最终实现的代码

## Day 3
- 为一个 small feature 写第一个 spec
- 提交 PR，接受 spec review
- 修改直到 approved

## Day 4-5
- 使用自己写的 spec 进行 SDD implementation
- 配置 CLAUDE.md 和 agent permissions
- 完成第一个 SDD-driven feature
```

---

## 28.10 案例研究：5 人团队的渐进式采用

让我们看一个真实的 adoption 轨迹。

### Week 1-2: Champion Phase（冠军阶段）

一个人先试：
- Developer A 在一个 solo feature 上尝试 SDD
- 写了第一个 spec，用 AI 实现
- 结果：feature 质量提升，但 spec 写作花了额外 30 分钟
- 分享经验给团队

### Week 3-4: Pilot Phase（试点阶段）

两个人参与：
- Developer A + Developer B 在两个相关 feature 上使用 SDD
- 发现了 spec 协作的需求（他们的 feature 有依赖关系）
- 建立了第一版团队 CLAUDE.md
- 结果：跨 feature 的 integration 更顺畅

### Week 5-6: Team Adoption（团队采用）

全团队参与：
- 所有 5 人在新 feature 上使用 SDD
- 建立了 spec review 流程
- 定义了 organization-level CLAUDE.md
- 添加了 CI spec validation
- 结果：first-pass accuracy 从 50% 提升到 70%

### Week 7-8: Optimization（优化阶段）

- 收集 metrics，识别 bottleneck
- 优化 spec template（减少重复 boilerplate）
- 引入 autonomous pipeline（第 26 章）做简单 task
- Spec coverage 达到 80%
- 结果：mean time to implementation 下降 40%

### 关键教训

```
1. 不要一步到位。先 solo 验证，再 pilot，再全团队。
2. Spec review 和 code review 同等重要。跳过 spec review = 跳过 code review。
3. CLAUDE.md 必须 evolve。每周 retro 时更新，保持与实际实践同步。
4. 度量驱动改进。没有 metrics 就无法证明 SDD 的价值。
5. Autonomous pipeline 是 reward，不是 prerequisite。先把 spec quality 和 verification 做好。
```

---

## Key Takeaways (要点回顾)

1. **分层 Constitution 是团队 SDD 的基础**：Organization → Team → Project 三层 CLAUDE.md，通过 inheritance 确保一致性同时允许 customization。

2. **Spec 是多角色协作产物**：PM 定义 business requirements，Tech Lead 添加 technical constraints，Developer 细化 acceptance criteria。PR review 流程确保质量。

3. **CI/CD 让 SDD discipline 持久化**：pre-commit hooks 验证 spec format，CI pipeline 检查 compliance，post-merge 更新 metrics。不靠人的自觉，靠系统的强制。

4. **度量使价值可见**：Spec coverage、first-pass accuracy、rework rate 等指标让你知道 SDD 是否在工作，以及哪里需要改进。

5. **渐进式采用是唯一可靠的路径**：从一个 champion 开始，pilot 验证，然后全团队。每一步都需要建立信任和积累经验。

---

下一章是本系列的最后一章。我们将展望 SDD 的未来——当 AI 能力持续增长时，specification 和 verification 的角色如何演化。
