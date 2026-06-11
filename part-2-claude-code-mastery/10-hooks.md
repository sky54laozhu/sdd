# Chapter 10: 钩子系统：自动化守卫 (Hooks: Automated Guardrails)

> Hooks 是在 Claude Code 生命周期事件中自动执行的 Shell 命令，它们将人工检查转化为机器执行的质量门控。

---

## 为什么需要 Hooks

想象这样的场景：你用 Claude Code 实现了一个功能，代码写得很漂亮，但忘了跑 formatter；或者 Claude 生成了一个 900 行的文件，远超你的项目规范；又或者你结束会话后才发现 TypeScript 有类型错误。

这些问题的共同特点是：**它们本可以被自动拦截**。

传统开发中，我们通过 Git hooks（pre-commit、pre-push）来守护代码质量。但 Claude Code 的工作方式不同 — 它在 Git commit 之前就已经完成了大量文件操作。等到 pre-commit hook 触发时，你可能已经积累了几十个问题。

Hooks 系统将质量门控**前移到每一次工具调用**的前后。它是 Claude Code 的 "immune system"（免疫系统）。

---

## Hook 的三种类型

Claude Code 提供三种 Hook 类型，对应不同的生命周期阶段：

```
┌─────────────────────────────────────────────────────┐
│                 Claude Code Session                   │
│                                                       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐       │
│  │PreToolUse│───▶│Tool Exec │───▶│PostToolUse│       │
│  │  (验证)   │    │ (执行)   │    │ (善后)    │       │
│  └──────────┘    └──────────┘    └──────────┘       │
│       │                                │              │
│       │  Block if invalid              │  Format/lint │
│       ▼                                ▼              │
│  [REJECT]                         [AUTO-FIX]         │
│                                                       │
│  ════════════════════════════════════════════════     │
│                                                       │
│  Session ends ──▶ ┌──────────┐                       │
│                   │   Stop   │                       │
│                   │(最终验证) │                       │
│                   └──────────┘                       │
│                        │                             │
│                        ▼                             │
│                  [Build check]                       │
└─────────────────────────────────────────────────────┘
```

### 1. PreToolUse — 执行前拦截

**触发时机**：在工具实际执行之前。

**用途**：

- 验证参数是否合规
- 阻止不符合规范的操作（如写入过大的文件）
- 检查文件路径是否在允许范围内
- 防止意外覆盖关键文件

**关键特性**：如果 Hook 命令返回非零退出码（exit code !== 0），工具调用将被**阻止**。这是唯一能"否决" Claude 行为的 Hook 类型。

### 2. PostToolUse — 执行后善后

**触发时机**：在工具执行成功之后。

**用途**：

- 自动格式化刚写入的文件
- 运行 linter 并自动修复
- 执行类型检查
- 验证 Spec 合规性

**关键特性**：PostToolUse Hook 不会阻止操作（操作已经完成），但其输出会反馈给 Claude，让它意识到问题并主动修复。

### 3. Stop — 会话结束验证

**触发时机**：Claude Code 会话即将结束时。

**用途**：

- 运行完整的生产构建
- 执行测试套件
- 验证没有遗留的 TODO
- 生成会话摘要

**关键特性**：Stop Hook 是"最后一道防线"。如果它失败了，Claude 会收到通知并可能尝试修复问题后再结束。

---

## 配置结构

Hooks 在 `.claude/settings.json`（项目级）或 `~/.claude/settings.json`（全局级）中配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "command": "node scripts/check-file-size.js",
        "description": "Block writes exceeding 800 lines"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm prettier --write \"$FILE_PATH\"",
        "description": "Format edited files"
      }
    ],
    "Stop": [
      {
        "command": "pnpm build",
        "description": "Verify production build before session ends"
      }
    ]
  }
}
```

每个 Hook 由三部分组成：

| 字段 | 说明 | 示例 |
|------|------|------|
| `matcher` | 触发条件（匹配工具名） | `"Write"`, `"Write\|Edit"`, `"Bash"` |
| `command` | 要执行的 Shell 命令 | `"pnpm eslint --fix \"$FILE_PATH\""` |
| `description` | 人类可读的描述 | `"Run ESLint on edited files"` |

> **注意**：`matcher` 使用正则匹配。`"Write|Edit"` 表示匹配 Write 或 Edit 工具。Stop Hook 不需要 matcher（它总是在会话结束时触发）。

---

## SDD 专属 Hook 用例

在 Spec-Driven Development 工作流中，Hooks 不仅仅是代码质量工具 — 它们是**规格合规性的自动执行者**。

### 用例 1：Format on Save（保存后格式化）

最基础也最常用的 Hook。确保 Claude 写入的每个文件都符合项目格式规范：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm prettier --write \"$FILE_PATH\"",
        "description": "Auto-format on every write/edit"
      }
    ]
  }
}
```

### 用例 2：Type Check（类型检查）

TypeScript 项目中，确保每次编辑后类型系统保持一致：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm tsc --noEmit --pretty false 2>&1 | head -20",
        "description": "Type-check after edits (show first 20 errors)"
      }
    ]
  }
}
```

> **性能提示**：完整的 `tsc --noEmit` 在大型项目中可能需要几秒钟。对于频繁编辑，考虑使用 `--incremental` 标志或限制输出行数。

### 用例 3：File Size Guard（文件大小守卫）

这是一个 **PreToolUse** Hook — 它在写入发生之前检查内容大小。如果超过 800 行，直接阻止操作：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const c=i.tool_input?.content||'';const lines=c.split('\\n').length;if(lines>800){console.error('[Hook] BLOCKED: File exceeds 800 lines ('+lines+' lines)');console.error('[Hook] Split into smaller modules');process.exit(2)}console.log(d)})\"",
        "description": "Block writes exceeding 800 lines"
      }
    ]
  }
}
```

当 Claude 尝试写入一个 900 行的文件时，它会收到类似这样的反馈：

```
[Hook] BLOCKED: File exceeds 800 lines (912 lines)
[Hook] Split into smaller modules
```

Claude 会理解这个约束并自动将文件拆分为更小的模块。

### 用例 4：Lint 自动修复

ESLint 配合 `--fix` 标志，自动修复简单的 lint 错误：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm eslint --fix \"$FILE_PATH\" 2>&1 | tail -5",
        "description": "Auto-fix lint issues"
      }
    ]
  }
}
```

### 用例 5：Build Verification（构建验证）

会话结束时运行完整构建，确保 Claude 没有留下无法编译的代码：

```json
{
  "hooks": {
    "Stop": [
      {
        "command": "pnpm build 2>&1 | tail -20",
        "description": "Verify production build on session end"
      }
    ]
  }
}
```

---

## Spec 合规性 Hook（SDD 核心模式）

这是 SDD 工作流中最独特的 Hook 模式 — 一个 PostToolUse Hook 自动检查代码是否符合其对应的 Spec。

### 设计思路

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Claude 编辑  │────▶│ PostToolUse  │────▶│ Spec Checker │
│  src/auth.ts │     │   触发       │     │   脚本       │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                                                 ▼
                                          ┌──────────────┐
                                          │ 比对 Spec    │
                                          │ specs/auth   │
                                          │  .spec.md    │
                                          └──────────────┘
                                                 │
                                          ┌──────┴──────┐
                                          │             │
                                          ▼             ▼
                                       [PASS]       [WARN]
                                                  输出偏差信息
                                                  → Claude 收到
                                                    反馈并修复
```

### 实现：Spec Compliance Verifier

创建一个验证脚本 `scripts/verify-spec-compliance.sh`：

```bash
#!/bin/bash
# verify-spec-compliance.sh
# Check if the edited file has a corresponding spec and validate basic compliance

FILE_PATH="$1"

# Map source file to its spec
SPEC_PATH=""
if [[ "$FILE_PATH" == src/* ]]; then
  # Convert src/features/auth/login.ts → specs/features/auth/login.spec.md
  RELATIVE="${FILE_PATH#src/}"
  BASENAME="${RELATIVE%.*}"
  SPEC_PATH="specs/${BASENAME}.spec.md"
fi

# If no spec found, skip silently
if [[ -z "$SPEC_PATH" ]] || [[ ! -f "$SPEC_PATH" ]]; then
  exit 0
fi

# Extract exported function names from source
EXPORTS=$(grep -oP '(?<=export (function|const|class) )\w+' "$FILE_PATH" 2>/dev/null)

# Extract required interfaces from spec
REQUIRED=$(grep -oP '(?<=SHALL export )\w+' "$SPEC_PATH" 2>/dev/null)

# Check each required export
MISSING=""
for req in $REQUIRED; do
  if ! echo "$EXPORTS" | grep -q "^${req}$"; then
    MISSING="${MISSING}  - Missing: ${req}\n"
  fi
done

if [[ -n "$MISSING" ]]; then
  echo "[Spec Compliance] WARNING: ${FILE_PATH}"
  echo "  Spec: ${SPEC_PATH}"
  echo "  Missing required exports:"
  echo -e "$MISSING"
  exit 0  # Warning only, don't block
fi

echo "[Spec Compliance] PASS: ${FILE_PATH} matches ${SPEC_PATH}"
```

对应的 Hook 配置：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "bash scripts/verify-spec-compliance.sh \"$FILE_PATH\"",
        "description": "Verify code aligns with its spec"
      }
    ]
  }
}
```

---

## Hook 执行顺序

当你配置了多个同类型的 Hook 时，它们按照数组中的**声明顺序**依次执行。推荐的排列顺序：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm prettier --write \"$FILE_PATH\"",
        "description": "1. Format first"
      },
      {
        "matcher": "Write|Edit",
        "command": "pnpm eslint --fix \"$FILE_PATH\"",
        "description": "2. Then lint (on formatted code)"
      },
      {
        "matcher": "Write|Edit",
        "command": "pnpm tsc --noEmit --pretty false 2>&1 | head -10",
        "description": "3. Then type check (on linted code)"
      },
      {
        "matcher": "Write|Edit",
        "command": "bash scripts/verify-spec-compliance.sh \"$FILE_PATH\"",
        "description": "4. Finally verify spec compliance"
      }
    ],
    "Stop": [
      {
        "command": "pnpm build",
        "description": "5. Full build verification on exit"
      }
    ]
  }
}
```

排序逻辑：

1. **Format** — 让代码变得规范（纯粹的格式转换）
2. **Lint** — 在格式化的代码上检查/修复逻辑问题
3. **Type Check** — 在 lint 后验证类型一致性
4. **Spec Compliance** — 最后验证业务逻辑是否符合规格
5. **Build** — 最终的集成验证（仅在 Stop 阶段）

---

## 完整项目配置示例

一个 SDD TypeScript 项目的完整 Hooks 配置：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Write",
        "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const c=i.tool_input?.content||'';const lines=c.split('\\n').length;if(lines>800){console.error('[Guard] BLOCKED: '+lines+' lines exceeds 800-line limit');process.exit(2)}})\"",
        "description": "Block oversized file writes"
      },
      {
        "matcher": "Write",
        "command": "node -e \"let d='';process.stdin.on('data',c=>d+=c);process.stdin.on('end',()=>{const i=JSON.parse(d);const p=i.tool_input?.file_path||'';if(p.match(/\\.(env|key|pem|secret)/)){console.error('[Guard] BLOCKED: Cannot write to sensitive file: '+p);process.exit(2)}})\"",
        "description": "Block writes to sensitive files"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "pnpm prettier --write \"$FILE_PATH\" 2>/dev/null",
        "description": "Auto-format"
      },
      {
        "matcher": "Write|Edit",
        "command": "pnpm eslint --fix \"$FILE_PATH\" 2>&1 | tail -5",
        "description": "Auto-lint"
      },
      {
        "matcher": "Write|Edit",
        "command": "pnpm tsc --noEmit --pretty false 2>&1 | head -10",
        "description": "Type check"
      },
      {
        "matcher": "Write|Edit",
        "command": "bash scripts/verify-spec-compliance.sh \"$FILE_PATH\"",
        "description": "Spec compliance check"
      }
    ],
    "Stop": [
      {
        "command": "pnpm build 2>&1 | tail -20",
        "description": "Production build verification"
      },
      {
        "command": "pnpm test --run 2>&1 | tail -10",
        "description": "Run test suite"
      }
    ]
  }
}
```

---

## 注意事项与陷阱

### Hooks 同步执行

所有 Hooks 都是**同步阻塞**的。在 Hook 命令执行完毕之前，Claude Code 不会继续。这意味着：

- **快速 Hook**（< 1s）：formatter、简单的 grep 检查 — 没问题
- **中速 Hook**（1-5s）：TypeScript 编译、单元测试 — 可以接受
- **慢速 Hook**（> 5s）：完整构建、E2E 测试 — 仅用于 Stop

> **经验法则**：PostToolUse Hook 应在 3 秒内完成。超过这个时间，把它移到 Stop 阶段。

### 避免 Hook 循环

如果 PostToolUse Hook 本身修改了文件（如 `prettier --write`），要确保不会触发另一个 Hook 循环。Claude Code 内置了循环检测，但最好从设计上避免这种情况。

### 环境变量

Hook 命令可以使用以下环境变量：

| 变量 | 说明 | 适用 Hook 类型 |
|------|------|---------------|
| `$FILE_PATH` | 被操作文件的路径 | PreToolUse, PostToolUse |
| `$TOOL_NAME` | 触发的工具名 | PreToolUse, PostToolUse |

### PreToolUse 的 stdin

PreToolUse Hook 通过 **stdin** 接收工具调用的完整参数（JSON 格式）。这允许你检查工具即将执行的内容而不是文件系统的当前状态。

### 调试技巧

当 Hook 不按预期工作时：

1. 在命令前加 `echo "[DEBUG]"` 确认它被触发
2. 检查 matcher 正则是否正确
3. 确保脚本有执行权限（`chmod +x`）
4. 将复杂逻辑提取到独立脚本文件中，避免 JSON 转义问题

---

## 与传统 Git Hooks 的对比

| 维度 | Git Hooks | Claude Code Hooks |
|------|-----------|-------------------|
| 触发时机 | commit/push 时 | 每次工具调用时 |
| 反馈速度 | 延迟（积累多个问题） | 即时（逐个操作检查） |
| 作用范围 | 仓库所有操作 | 仅 Claude Code 操作 |
| 阻止能力 | pre-commit 可阻止 | PreToolUse 可阻止 |
| 配置位置 | `.git/hooks/` | `.claude/settings.json` |
| 对 AI 的反馈 | 无（AI 看不到） | 有（直接反馈给 Claude） |

两者是**互补关系**，不是替代关系。Claude Code Hooks 处理 AI 编码过程中的实时质量保障；Git Hooks 处理最终提交时的门控。

---

## 练习

### 练习 1：配置基础 Hook 链

为你的项目创建 `.claude/settings.json`，配置以下 Hook 链：
1. PreToolUse：阻止写入超过 500 行的文件
2. PostToolUse：自动格式化（使用你项目的 formatter）
3. Stop：运行测试套件

验证方式：让 Claude Code 生成一个大文件，观察它是否被阻止。

### 练习 2：实现 Spec Compliance Hook

1. 创建一个简单的 Spec 文件，定义一个模块必须导出的函数名
2. 编写验证脚本（参考上面的 `verify-spec-compliance.sh`）
3. 配置为 PostToolUse Hook
4. 让 Claude 实现该模块，观察它是否收到合规性反馈

---

## Key Takeaways (要点回顾)

- **Hooks 是生命周期守卫**：PreToolUse 拦截、PostToolUse 善后、Stop 最终验证
- **PreToolUse 是唯一的否决机制**：通过非零退出码阻止不合规的操作
- **执行顺序至关重要**：format → lint → type check → spec compliance → build
- **性能是硬约束**：PostToolUse Hook 必须快速（< 3s），慢操作放到 Stop
- **Spec Compliance Hook 是 SDD 的独特武器**：自动验证代码是否偏离规格文档

---

## Next (下一章)

[Chapter 11: 技能系统：工作流封装 (Skills: Workflow Packaging)](11-skills.md) — 学习如何将重复的工作流打包为可复用的 Skill，让 Claude Code 通过 slash command 一键执行复杂的 SDD 流程。
