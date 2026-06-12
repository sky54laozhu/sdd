# Karpathy Guidelines：LLM 编码行为四大原则

> Andrej Karpathy 总结的 LLM 编码缺陷与四条行为准则——SDD 实现阶段的编码纪律参考。

---

## 背景

2025 年，Andrej Karpathy 在一条[广为传播的推文](https://x.com/karpathy/status/2015883857489522876)中总结了 LLM 编码代理的系统性缺陷：

> "The models make wrong assumptions on your behalf and just run along with them without checking. They don't manage their confusion, don't seek clarifications, don't surface inconsistencies, don't present tradeoffs, don't push back when they should."

> "They really like to overcomplicate code and APIs, bloat abstractions, don't clean up dead code... implement a bloated construction over 1000 lines when 100 would do."

这些观察与本教程 [Chapter 02: Vibe Coding 的七宗罪](../part-1-foundations/02-vibe-coding-failure.md) 中记录的失败模式高度一致。社区开发者 [forrestchang](https://github.com/forrestchang) 将这些观察提炼为四条可执行的行为准则，发布为 [andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) Claude Code Plugin。

本附录将这四条准则完整呈现，并映射到 SDD 工作流中对应的保障机制。

---

## 四大原则

### 原则一：Think Before Coding（编码前先思考）

**核心要求：不要假设，不要隐藏困惑，主动暴露权衡。**

LLM 最常见的失败模式是"静默假设"——遇到歧义时默默选择一个解释然后一路执行，而不是停下来确认。

规则：
- 明确陈述你的假设。不确定时，**问**而非**猜**
- 存在多种解释时，列出选项，不要静默选择
- 存在更简单的方案时，主动提出。该反对时要反对
- 遇到困惑时，停下来，说明困惑点，请求澄清

**示例对比：**

用户请求："添加一个导出用户数据的功能。"

```python
# 静默假设（反面示例）
def export_users(format='json'):
    users = User.query.all()            # 假设导出所有用户
    with open('users.json', 'w') as f:  # 假设写本地文件
        json.dump([u.to_dict() for u in users], f)  # 假设字段结构
```

```
# 主动澄清（正面示例）
实现之前需要确认几点：
1. 范围：导出全部用户还是筛选子集？（涉及隐私）
2. 格式：浏览器下载文件？后台任务+邮件通知？还是 API 返回数据？
3. 字段：包含哪些用户字段？（部分可能敏感）
4. 数据量：通常多少用户？（影响技术方案选择）

最简方案：添加一个分页 JSON API endpoint。
文件导出需要更多信息。你倾向哪种？
```

### 原则二：Simplicity First（简单优先）

**核心要求：用最少的代码解决问题，不做推测性设计。**

LLM 天然倾向于过度工程化——生成 Strategy Pattern、Factory、配置系统，即使当前只需要一个函数。

规则：
- 不添加需求之外的功能
- 不为只使用一次的代码创建抽象
- 不添加未被要求的"灵活性"和"可配置性"
- 不为不可能发生的场景添加错误处理
- 如果 200 行代码能用 50 行完成，重写它

**检验标准：一个 senior engineer 会不会说"这太复杂了"？如果会，简化。**

**示例对比：**

用户请求："添加一个计算折扣的函数。"

```python
# 过度工程化（反面示例）—— 46 行
class DiscountStrategy(ABC):
    @abstractmethod
    def calculate(self, amount: float) -> float: ...

class PercentageDiscount(DiscountStrategy):
    def __init__(self, percentage: float):
        self.percentage = percentage
    def calculate(self, amount: float) -> float:
        return amount * (self.percentage / 100)

class FixedDiscount(DiscountStrategy): ...

@dataclass
class DiscountConfig:
    strategy: DiscountStrategy
    min_purchase: float = 0.0
    max_discount: float = float('inf')

class DiscountCalculator:
    def __init__(self, config: DiscountConfig): ...
    def apply_discount(self, amount: float) -> float: ...
```

```python
# 简单方案（正面示例）—— 3 行
def calculate_discount(amount: float, percent: float) -> float:
    """Calculate discount amount. percent should be 0-100."""
    return amount * (percent / 100)
```

**当需要多种折扣类型时再引入复杂度，不要提前。**

### 原则三：Surgical Changes（外科手术式修改）

**核心要求：只改必须改的，只清理自己造成的 mess。**

LLM 在修复一个 bug 时，经常"顺手"改动周围的代码——重新格式化、添加 type hints、修改注释、重构不相关的逻辑。

编辑现有代码时的规则：
- 不要"改进"相邻的代码、注释或格式
- 不要重构没有坏的东西
- 匹配现有风格，即使你更喜欢另一种
- 发现无关的 dead code 时，**提及**它，不要删除它

当你的修改产生了孤立代码时：
- 删除**你的修改**导致不再使用的 imports/变量/函数
- 不要删除之前就存在的 dead code（除非被要求）

**检验标准：每一行变更都应该直接追溯到用户的请求。**

**示例对比：**

用户请求："修复空 email 导致 validator 崩溃的 bug。"

```diff
# 改太多了（反面示例）
- def validate_user(user_data):
-     # Check email format
-     if not user_data.get('email'):
+ def validate_user(user_data):
+     """Validate user data."""                    # 加了 docstring
+     email = user_data.get('email', '').strip()
+     if not email:
          raise ValueError("Email required")
-     if '@' not in user_data['email']:
+     if '@' not in email or '.' not in email.split('@')[1]:  # 增强了验证
          raise ValueError("Invalid email")
-     # Check username
-     if not user_data.get('username'):
+     # Validate username                          # 改了注释
+     username = user_data.get('username', '').strip()
+     if not username:
          raise ValueError("Username required")
+     if len(username) < 3:                        # 加了 username 验证
+         raise ValueError("Username too short")
```

```diff
# 外科手术式修复（正面示例）
  def validate_user(user_data):
      # Check email format
-     if not user_data.get('email'):
+     email = user_data.get('email', '')
+     if not email or not email.strip():
          raise ValueError("Email required")

      # Basic email validation
-     if '@' not in user_data['email']:
+     if '@' not in email:
          raise ValueError("Invalid email")
```

**只改了导致 bug 的那几行。**

### 原则四：Goal-Driven Execution（目标驱动执行）

**核心要求：定义成功标准，循环直到验证通过。**

Karpathy 的核心洞察：

> "LLMs are exceptionally good at looping until they meet specific goals... Don't tell it what to do, give it success criteria and watch it go."

将命令式指令转化为可验证的目标：

| 命令式（弱） | 目标式（强） |
|-------------|-------------|
| "添加验证" | "为非法输入写测试，然后让测试通过" |
| "修复 bug" | "写一个复现 bug 的测试，然后让测试通过" |
| "重构 X" | "确保重构前后测试全部通过" |

对于多步骤任务，陈述简要计划：

```
1. [步骤] → 验证: [检查方式]
2. [步骤] → 验证: [检查方式]
3. [步骤] → 验证: [检查方式]
```

**强成功标准让 AI 能独立循环。弱标准（"让它能用"）需要反复人工澄清。**

---

## SDD 映射：四大原则在工作流中的位置

这四条准则并非独立于 SDD，而是被 SDD 各阶段的机制所支撑和强化：

| Karpathy 原则 | 对应 SDD 阶段 | SDD 保障机制 |
|---------------|-------------|-------------|
| Think Before Coding | **Specify** | EARS Notation 消除歧义；Out-of-Scope 声明阻止静默假设 |
| Simplicity First | **Plan + Tasks** | Task 原子化限制单次实现范围；`do_not_build` 列表阻止 yak-shaving |
| Surgical Changes | **Implement** | 每次只执行一个 Task；PostToolUse Hook 自动检查 conventions |
| Goal-Driven Execution | **Implement + Verify** | Spec 的 Acceptance Criteria 定义成功标准；TDD RED→GREEN 循环 |

换言之：**SDD 在流程层面解决了 Karpathy 在行为层面指出的问题。** 两者是互补的——SDD 提供结构，Karpathy Guidelines 提供执行纪律。

---

## 集成到你的项目

### 方式一：写入 CLAUDE.md

将以下内容添加到项目 CLAUDE.md 的编码标准部分：

```markdown
## AI Coding Behavior / AI 编码行为

1. **Think Before Coding**: 有歧义时先问，不要静默假设
2. **Simplicity First**: 最少代码解决问题，不做推测性设计
3. **Surgical Changes**: 只改必须改的行，匹配现有风格
4. **Goal-Driven Execution**: 定义可验证的成功标准，循环直到通过
```

### 方式二：安装 Claude Code Plugin

```bash
# 在 Claude Code 中执行
/plugin marketplace add forrestchang/andrej-karpathy-skills
/plugin install andrej-karpathy-skills@karpathy-skills
```

Plugin 安装后，准则作为 Skill 全局生效，无需在每个项目中重复配置。

---

## 关键洞察

Karpathy Guidelines 的示例中，"过度工程化"的代码并非明显错误——它们遵循设计模式和最佳实践。问题在于**时机**：在复杂度实际需要之前就引入了复杂度。

这正是 SDD 和 Karpathy Guidelines 的共同主张：

> **好代码是简单地解决今天问题的代码，而不是提前解决明天问题的代码。**

---

## 参考链接

- [Karpathy 原始推文](https://x.com/karpathy/status/2015883857489522876)
- [andrej-karpathy-skills GitHub 仓库](https://github.com/forrestchang/andrej-karpathy-skills)
- [Chapter 02: Vibe Coding 的七宗罪](../part-1-foundations/02-vibe-coding-failure.md)
- [Chapter 17: 常见陷阱与解法](../part-3-workflow/17-pitfalls.md)
