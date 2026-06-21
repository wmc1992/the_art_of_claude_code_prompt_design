# Claude Code Prompt 设计完整学习方案

---

## 学习目标

读完这个方案后，你应该能回答三个问题：
1. 这套 prompt 是**怎么组织**的（架构）
2. 它是**怎么构造**的（工程机制）
3. 它**为什么有效**（设计原则）

---

## 第一部分：架构——prompt 是怎么组织的

### 1.1 分层结构

Claude Code 的 system prompt 不是一段固定文字，而是由多个独立 section 动态拼接而成。每个 section 只负责一类职责：

```
getSimpleIntroSection()        # 角色定义："你是谁"
getSimpleSystemSection()       # 基础规则："工具调用、标签处理"
getSimpleDoingTasksSection()   # 任务执行规范："怎么做任务"
getActionsSection()            # 高危操作决策："什么情况要确认"
getUsingYourToolsSection()     # 工具选择逻辑："用哪个工具"
getSimpleToneAndStyleSection() # 输出风格
getOutputEfficiencySection()   # 输出效率
--- DYNAMIC BOUNDARY ---
loadMemoryPrompt()             # 用户记忆（动态）
computeSimpleEnvInfo()         # 运行时环境（动态）
getMcpInstructionsSection()    # MCP 服务器指令（动态）
```

**关键文件**：`src/constants/prompts.ts`（主组装逻辑）

### 1.2 工具级 prompt

每个工具都有独立的 prompt 文件，描述：用途、参数约束、边界情况、使用禁忌。工具 prompt 是给模型的"使用手册"，质量直接影响工具被调用的准确性。

**关键目录**：`src/tools/*/prompt.ts`

---

## 第二部分：工程机制——prompt 是怎么构造的

### 2.1 两类 section：缓存 vs 易失

```typescript
// 计算一次，缓存到 /clear 或 /compact
systemPromptSection('memory', () => loadMemoryPrompt())

// 每 turn 重算，会打破 prompt cache（需要理由）
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns'
)
```

这背后是 Anthropic API 的 prompt cache 机制——system prompt 的前缀如果不变，可以复用缓存 token，节省成本也降低延迟。所以 prompt 被切成"静态内容"和"动态内容"两段，以 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 为界：

- **边界之前**：角色定义、工具规则等稳定内容，打上 `scope: 'global'` 跨 org 可复用
- **边界之后**：当前时间、cwd、用户记忆等每次可能变化的内容

**关键文件**：`src/constants/systemPromptSections.ts`

### 2.2 启动时并行预取

```typescript
// main.tsx 最顶部，在所有 import 之前执行
startMdmRawRead()        // 并行读取 MDM 系统策略（macOS/Windows）
startKeychainPrefetch()  // 并行读取 keychain（OAuth + API key）
```

prompt 的动态部分（环境信息、记忆）需要从系统读取数据。通过在模块加载期间就触发异步读取，把 I/O 等待时间和模块加载时间重叠掉。

### 2.3 Feature Flag 做死代码消除

```typescript
// VOICE_MODE 未激活时，整个分支在打包时被删除
const voiceCommand = feature('VOICE_MODE') ? require('./commands/voice') : null

// 自主模式下的 prompt 完全不同
if (feature('PROACTIVE') && proactiveModule?.isProactiveActive()) {
  return ['You are an autonomous agent. Use the available tools to do useful work.', ...]
}
```

不同功能模式下，模型看到的 prompt 是完全不同的版本，并非靠 if 判断在同一段 prompt 里加条件，而是直接走不同的组装路径。

### 2.4 用户分层（内部 vs 外部）

```typescript
process.env.USER_TYPE === 'ant'
  ? `Default to writing no comments. Only add one when the WHY is non-obvious...`
  : []  // 外部用户不加这条
```

Anthropic 内部员工（`ant`）和外部用户看到不同的 prompt，因为内部需要更严格的工程纪律要求，外部需要更友好的默认行为。这在 prompt 工程里是一个重要思路：**同一个模型可以通过 prompt 差异为不同用户群体提供差异化行为**。

---

## 第三部分：设计原则——prompt 为什么有效

这是最重要的部分。

### 3.1 针对失败模式写 prompt，而不是针对理想行为

代码里散布着这样的注释：

```
// @[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302)
// @[MODEL LAUNCH]: False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)
// @[MODEL LAUNCH]: Update comment writing for Capybara — remove or soften once model stops over-commenting
```

**他们不是在描述理想的 AI 应该怎么做，而是在打补丁。** 某个模型版本有过度注释的倾向，就在 prompt 里专门压制它；模型会在测试失败时谎称通过，就加专门的规则。

每条规则背后都是一个观察到的真实失败：
- 模型倾向于 `git commit --amend` 已有提交 → "CRITICAL: Always create NEW commits"
- 模型会在钩子失败后用 `--amend` 污染前一个提交 → 解释了 git 底层机制作为理由
- 模型完成任务后过度 hedge → "do not hedge confirmed results with unnecessary disclaimers"

**核心启示**：prompt 工程是一个**观察 → 假设 → 修正**的迭代过程，不是一次性设计出来的。他们能做到这一点是因为有 A/B 测试基础设施（GrowthBook）和真实用户反馈。

### 3.2 双边约束：同时防止不足和过度

大量规则是"both sides"结构，因为模型有"反弹"现象——你说"不要过度自信"，它会变得过度谦虚：

> *"Report outcomes faithfully: if tests fail, say so... Never claim 'all tests pass'... **Equally**, when a check did pass or a task is complete, **state it plainly — do not hedge confirmed results with unnecessary disclaimers**, downgrade finished work to 'partial,' or re-verify things you already checked."*

> *"Don't retry the identical action blindly, **but don't abandon a viable approach after a single failure either.**"*

**写规则时永远问自己：我压制了这个行为之后，模型会走向哪个反方向？那个方向需要同时约束吗？**

### 3.3 具体 > 抽象，可操作 > 原则

**差的写法**：`Be careful with git operations`

**实际写法**：
```
NEVER run destructive git commands (push --force, reset --hard,
checkout ., restore ., clean -f, branch -D) unless the user
explicitly requests these actions.
```

**差的写法**：`Don't commit sensitive files`

**实际写法**：
```
When staging files, prefer adding specific files by name rather
than using "git add -A" or "git add .", which can accidentally
include sensitive files (.env, credentials) or large binaries
```

具体的命令名、文件名让模型不需要推理"这算不算风险操作"——直接模式匹配，消除歧义空间。

### 3.4 给出理由，让模型能举一反三

光给规则，模型只能机械执行。给了理由，模型能在未覆盖的场景下自主推理：

> *"When a pre-commit hook fails, the commit did NOT happen — so --amend would modify the PREVIOUS commit, which may result in destroying work or losing previous changes."*

这条 prompt 解释了 git 的底层机制，模型因此理解了**为什么**，在类似场景（rebase 失败、cherry-pick 冲突）也能正确判断。

**规则 = what。理由 = why。有 why 的规则才能泛化。**

### 3.5 用决策框架代替规则列表

`getActionsSection()` 没有穷举所有禁止操作，而是给出了判断框架：

> *"The cost of pausing to confirm is low, while the cost of an unwanted action can be very high."*

然后给出分类标准：
- **本地可逆操作**（编辑文件、跑测试）→ 自由执行
- **难以撤销 / 影响共享状态**（push、发消息、删分支）→ 先确认

这比列一百条"不能做 X"更有效，因为模型拿到的是**判断逻辑**，可以泛化到任何未列出的情况。

### 3.6 禁止 + 替代，成对出现

几乎每个限制都配着出口：

> *"Do NOT use the Bash tool when a dedicated tool exists"*  
> → "To read files use Read instead of cat, head, tail, or sed"

> *"Escalate to AskUserQuestion only when genuinely stuck after investigation, not as a first response to friction."*

只说"不要"会让模型陷入困境；说清楚"那应该怎样"才能导向正确行为。**约束和替代路径要同时提供。**

### 3.7 给模型一个认知框架，而非外部约束列表

Proactive 模式下，他们没有列 checklist，而是给了一个思维工具：

> *"A good colleague faced with ambiguity doesn't just stop — they investigate, reduce risk, and build understanding. Ask yourself: what don't I know yet? What could go wrong? What would I want to verify before calling this done?"*

"好同事"这个类比给模型安装了一个内部决策循环。这类**角色隐喻 + 自问框架**的组合，让模型在无数具体情况下都能套用，而不是只能执行一张固定清单。

### 3.8 权限不可传递原则

> *"A user approving an action (like a git push) once does NOT mean that they approve it in all contexts. Authorization stands for the scope specified, not beyond. Match the scope of your actions to what was actually requested."*

模型的自然倾向是"用户之前同意了 X，类似情况应该也行"。这条 prompt 专门关闭了这个推理路径，把授权范围锁定在"当前被明确要求的事"。这是**防止权限滑坡**的关键设计。

### 3.9 输出格式是行为控制，不只是样式管理

内部用户和外部用户看到完全不同的输出要求：

- **外部用户**：`"Go straight to the point. Be extra concise."`
- **内部用户**：详细的散文写作规范（inverted pyramid、avoid semantic backtracking、flowing prose）

**输出格式的 prompt 不只在管格式，在管思维密度。** 要求散文式表达的模型会更完整地解释推理；要求极简的模型会跳过推理直接给答案。

数字锚定比定性描述有效：
```
Length limits: keep text between tool calls to ≤25 words.
Keep final responses to ≤100 words unless the task requires more detail.
```
模型对"简洁"的理解是模糊的，对"≤25 词"的理解是精确的。

### 3.10 prompt 随模型版本演化

```typescript
// @[MODEL LAUNCH]: Remove this section when we launch numbat.
// @[MODEL LAUNCH]: capy v8 assertiveness counterweight (PR #24302) —
//   un-gate once validated on external via A/B
```

这些注释揭示了最关键的工程哲学：**prompt 不是一次写好的，它跟模型一起成长**。每次模型升级，他们重新评估哪些规则可以放宽（模型已经自然学会了），哪些新问题需要新的规则，哪些规则要从内部 A/B 测试后推向外部用户。

---

## 推荐阅读顺序

| 阶段 | 文件 | 学习重点 |
|---|---|---|
| 1 | `src/constants/systemPromptSections.ts` | Section 抽象 + 缓存/易失机制 |
| 2 | `src/constants/prompts.ts` 第 100-500 行 | 各 section 的实际文案，重点看注释里的 `@[MODEL LAUNCH]` |
| 3 | `src/constants/prompts.ts` 第 500-900 行 | 组装逻辑、boundary marker、proactive 模式 prompt |
| 4 | `src/tools/BashTool/prompt.ts` | 最复杂的工具 prompt：git 安全协议、步骤化指令、example 用法 |
| 5 | `src/tools/FileReadTool/prompt.ts` | 简洁工具 prompt 的标准写法 |
| 6 | `src/tools/AskUserQuestionTool/prompt.ts` | 工具间的约束关系（plan mode 限制） |
| 7 | `src/constants/cyberRiskInstruction.ts` | 安全边界 prompt 的写法 + 治理模型（不能随便改） |
| 8 | `src/services/compact/compact.ts` | 压缩摘要本身也是 prompt 工程问题 |

---

## 提炼出的可迁移原则

写任何 LLM 系统的 prompt，以下原则都适用：

| 原则 | 一句话 |
|---|---|
| 失败驱动 | 每条规则背后要有一个观察到的失败，不要写"理想化"的规则 |
| 双边约束 | 压制一个行为时，同时检查反方向是否需要约束 |
| 具体可操作 | 能写命令名就不写"危险操作"，能写文件名就不写"敏感文件" |
| 理由优于规则 | 带着 why 的规则能泛化，纯 what 的规则只能机械执行 |
| 决策框架优于清单 | 给判断逻辑，不穷举所有情况 |
| 禁止 + 替代成对 | 每个约束都要提供出口 |
| 角色隐喻 + 自问框架 | 给模型认知工具，不只给外部约束 |
| 权限不传递 | 明确授权的范围不能被模型自行扩展 |
| 格式即行为 | 输出格式要求直接影响思维质量 |
| prompt 要演化 | 配合模型迭代持续观察和修正，不要期望一次写好 |
