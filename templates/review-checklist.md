# Phase Gate Review Checklist / 阶段门审查清单

> **用途 / Usage**: 在 SDD 工作流的每个阶段转换时使用此清单进行审查。
> 每个"门"(gate) 是一个质量关卡——通过后才能进入下一阶段。
> Use this checklist to review at each phase transition in the SDD workflow.
> Each "gate" is a quality checkpoint — pass before moving to the next phase.
>
> **工作流 / Workflow**:
> ```
> Specify → [Gate 1] → Plan → [Gate 2] → Tasks → [Gate 3] → Implement → [Gate 4] → Done
> ```

---

## Gate 1: Specify → Plan / 规格 → 计划

**目的 / Purpose**: 确认规格足够完整和清晰，可以开始技术规划。
Confirm the spec is complete and clear enough to begin technical planning.

### Review Questions / 审查问题

| # | Question | Pass? |
|---|----------|-------|
| 1 | 每条需求是否使用 EARS 格式且无歧义？ / Is every requirement in EARS format and unambiguous? | [ ] |
| 2 | 验收标准是否可独立测试？ / Are acceptance criteria independently testable? | [ ] |
| 3 | 范围外是否已明确界定？ / Is out-of-scope clearly defined? | [ ] |
| 4 | 所有待确认问题是否已解决？ / Are all open questions resolved? | [ ] |
| 5 | 依赖项是否已识别且状态明确？ / Are dependencies identified with clear status? | [ ] |
| 6 | 用户故事是否覆盖所有角色？ / Do user stories cover all affected roles? | [ ] |
| 7 | 约束条件（性能、兼容性）是否量化？ / Are constraints (performance, compatibility) quantified? | [ ] |
| 8 | 优先级是否已分配？ / Is priority assigned? | [ ] |

### Common Red Flags / 常见危险信号

- **Vague requirements** / 模糊需求: "系统应该快速响应" → 快是多快？定义具体数字。
  "The system should respond quickly" → How quickly? Define specific numbers.
- **Missing error cases** / 缺少错误场景: 只有正常流程，没有 REQ-X (Unwanted) 类型需求。
  Only happy path, no REQ-X (Unwanted) type requirements.
- **Scope creep signals** / 范围蔓延信号: 规格中出现 "and also", "might also", "future possibility"。
  Spec contains "and also", "might also", "future possibility".
- **Untestable criteria** / 不可测试的标准: "用户体验应该好" → 无法自动化验证。
  "User experience should be good" → Cannot be verified automatically.
- **Missing stakeholder sign-off** / 缺少利益相关者确认: 规格没有经过产品/设计确认。
  Spec not confirmed by product/design.

### Decision / 决定

- [ ] **PASS** — Proceed to Plan / 通过 — 进入计划阶段
- [ ] **REVISE** — Return to Specify with feedback / 修订 — 带反馈返回规格阶段

**Notes / 备注**: [...]

---

## Gate 2: Plan → Tasks / 计划 → 任务

**目的 / Purpose**: 确认技术方案合理、完整，可以分解为具体任务。
Confirm the technical approach is sound and complete enough for task decomposition.

### Review Questions / 审查问题

| # | Question | Pass? |
|---|----------|-------|
| 1 | 架构决策是否有明确理由？ / Do architecture decisions have clear rationale? | [ ] |
| 2 | 数据模型是否覆盖所有规格中的实体？ / Does the data model cover all entities from the spec? | [ ] |
| 3 | 文件结构是否符合 CLAUDE.md 约定？ / Does the file structure follow CLAUDE.md conventions? | [ ] |
| 4 | 依赖图是否无循环？ / Is the dependency graph acyclic? | [ ] |
| 5 | 风险是否有对应的缓解策略？ / Do risks have corresponding mitigation strategies? | [ ] |
| 6 | 每个组件的职责是否单一明确？ / Is each component's responsibility singular and clear? | [ ] |
| 7 | API 设计是否向后兼容（如适用）？ / Is the API design backward-compatible (if applicable)? | [ ] |
| 8 | 性能考虑是否合理？ / Are performance considerations reasonable? | [ ] |
| 9 | 实施阶段的验证方式是否明确？ / Are phase verification methods defined? | [ ] |

### Common Red Flags / 常见危险信号

- **Over-engineering** / 过度设计: 为简单功能创建复杂抽象层。
  Creating complex abstraction layers for simple features.
- **Missing error paths** / 缺少错误路径: 系统设计只展示正常流程。
  System design only shows happy path.
- **Unresolved dependencies** / 未解决的依赖: 计划依赖"待确认"的外部服务。
  Plan depends on "TBD" external services.
- **Schema drift** / Schema 偏离: 数据模型与规格中的实体不对应。
  Data model doesn't correspond to entities in the spec.
- **No rollback plan** / 无回滚计划: 数据库迁移没有逆向方案。
  Database migrations have no reverse path.
- **Tech stack mismatch** / 技术栈不匹配: 引入 CLAUDE.md 中未定义的技术。
  Introducing technology not defined in CLAUDE.md.

### Decision / 决定

- [ ] **PASS** — Proceed to Tasks / 通过 — 进入任务分解
- [ ] **REVISE** — Return to Plan with feedback / 修订 — 带反馈返回计划阶段

**Notes / 备注**: [...]

---

## Gate 3: Tasks → Implement / 任务 → 实现

**目的 / Purpose**: 确认任务分解合理，顺序正确，可以开始实现。
Confirm task decomposition is reasonable, correctly ordered, and ready for implementation.

### Review Questions / 审查问题

| # | Question | Pass? |
|---|----------|-------|
| 1 | 测试任务是否排在对应实现任务之前？ / Do test tasks precede their implementation tasks? | [ ] |
| 2 | 每个任务是否小到一次会话可完成？ / Is each task small enough for one session? | [ ] |
| 3 | 任务依赖是否正确声明？ / Are task dependencies correctly declared? | [ ] |
| 4 | 每个规格需求是否至少被一个任务引用？ / Is every spec requirement referenced by at least one task? | [ ] |
| 5 | 验收标准是否可自动化验证？ / Are acceptance criteria automatable? | [ ] |
| 6 | 验证命令是否可直接运行？ / Are verification commands directly runnable? | [ ] |
| 7 | 文件路径是否与计划中的文件结构一致？ / Do file paths match the file structure in the plan? | [ ] |
| 8 | 是否有可并行执行的任务组？ / Are there parallelizable task groups? | [ ] |
| 9 | 完成标准是否明确？ / Are completion criteria explicit? | [ ] |

### Common Red Flags / 常见危险信号

- **Tests after code** / 测试在代码后: 先写实现再写测试，违反 TDD。
  Writing implementation before tests, violating TDD.
- **Giant tasks** / 巨大任务: 单个任务涉及 5+ 个文件或复杂度为 XL。
  Single task touching 5+ files or marked XL complexity.
- **Missing traceability** / 缺少追溯: 任务没有引用规格需求编号。
  Task doesn't reference spec requirement IDs.
- **Implicit dependencies** / 隐式依赖: 任务 B 需要任务 A 的输出，但未声明依赖。
  Task B needs Task A's output but doesn't declare the dependency.
- **No verification** / 无验证方式: 任务没有可运行的验证命令。
  Task has no runnable verification command.
- **Wrong granularity** / 粒度错误: 任务太细碎（改个变量名）或太粗放（实现整个模块）。
  Tasks too granular (rename a variable) or too coarse (implement entire module).

### Traceability Matrix / 追溯矩阵

<!-- 确认每个规格需求都有对应任务 / Confirm every spec requirement has corresponding tasks -->

| Requirement | Task(s) | Covered? |
|------------|---------|----------|
| REQ-U01 | TASK-XXX | [ ] |
| REQ-S01 | TASK-XXX | [ ] |
| REQ-E01 | TASK-XXX, TASK-YYY | [ ] |
| REQ-O01 | TASK-XXX | [ ] |
| REQ-X01 | TASK-XXX | [ ] |

### Decision / 决定

- [ ] **PASS** — Begin Implementation / 通过 — 开始实现
- [ ] **REVISE** — Return to Tasks with feedback / 修订 — 带反馈返回任务分解

**Notes / 备注**: [...]

---

## Gate 4: Implement → Done / 实现 → 完成

**目的 / Purpose**: 确认实现符合规格，质量达标，可以交付。
Confirm implementation meets spec, quality is acceptable, and it's ready to ship.

### Review Questions / 审查问题

| # | Question | Pass? |
|---|----------|-------|
| 1 | 所有任务是否标记为 completed 或 skipped（有理由）？ / Are all tasks marked completed or skipped (with justification)? | [ ] |
| 2 | 所有测试是否通过？ / Do all tests pass? | [ ] |
| 3 | 测试覆盖率是否达标（>= 80%）？ / Does coverage meet target (>= 80%)? | [ ] |
| 4 | 类型检查是否通过？ / Does type checking pass? | [ ] |
| 5 | Lint 是否通过？ / Does lint pass? | [ ] |
| 6 | 每条验收标准是否已验证？ / Is every acceptance criterion verified? | [ ] |
| 7 | 安全检查清单是否通过？ / Does the security checklist pass? | [ ] |
| 8 | 无硬编码密钥或凭据？ / No hardcoded secrets or credentials? | [ ] |
| 9 | 错误处理是否完整？ / Is error handling comprehensive? | [ ] |
| 10 | 性能是否在约束范围内？ / Is performance within constraints? | [ ] |
| 11 | 构建是否成功？ / Does the build succeed? | [ ] |
| 12 | 文档是否已更新？ / Is documentation updated? | [ ] |

### Verification Commands / 验证命令

```bash
# 运行所有检查 / Run all checks
[test command]            # All tests pass
[coverage command]        # Coverage >= 80%
[type check command]      # No type errors
[lint command]            # No lint errors
[build command]           # Build succeeds
```

### Acceptance Criteria Verification / 验收标准验证

<!-- 逐条确认验收标准 / Verify acceptance criteria one by one -->

| AC | Description | Verified? | Evidence |
|----|------------|-----------|----------|
| AC-01 | [...] | [ ] | [test name / screenshot / log] |
| AC-02 | [...] | [ ] | [test name / screenshot / log] |
| AC-03 | [...] | [ ] | [test name / screenshot / log] |

### Common Red Flags / 常见危险信号

- **Skipped tests** / 跳过的测试: 使用 `.skip` 或注释掉测试来"通过"。
  Using `.skip` or commenting out tests to "pass".
- **Coverage gaming** / 覆盖率造假: 写无断言的测试凑覆盖率。
  Writing assertion-free tests to inflate coverage.
- **Unhandled TODOs** / 未处理的 TODO: 代码中留有 TODO/FIXME/HACK 注释。
  Code contains TODO/FIXME/HACK comments.
- **Console artifacts** / 控制台残留: 生产代码中有 console.log/debug 语句。
  Production code contains console.log/debug statements.
- **Missing migrations** / 缺少迁移: 数据模型变更但没有迁移脚本。
  Data model changes without migration scripts.
- **Broken backward compatibility** / 破坏向后兼容: API 变更没有版本策略。
  API changes without versioning strategy.

### Decision / 决定

- [ ] **PASS** — Ship it / 通过 — 可以交付
- [ ] **REVISE** — Return to Implement with issues / 修订 — 带问题返回实现
- [ ] **REJECT** — Fundamental issues, return to Plan / 驳回 — 根本性问题，返回计划阶段

**Notes / 备注**: [...]

---

## Quick Reference Card / 快速参考卡

```
Gate 1 (Spec → Plan):    规格完整吗？需求可测试吗？
Gate 2 (Plan → Tasks):   方案合理吗？风险可控吗？
Gate 3 (Tasks → Impl):   顺序正确吗？TDD 先行吗？
Gate 4 (Impl → Done):    测试通过吗？覆盖达标吗？
```

### Escalation Path / 升级路径

如果审查发现严重问题 / If review finds critical issues:
1. 记录问题和影响 / Document the issue and impact
2. 判断是修订当前阶段还是回退到更早阶段 / Decide if revising current phase or rolling back
3. 更新相关文档 / Update related documents
4. 重新进行门审查 / Re-run the gate review
