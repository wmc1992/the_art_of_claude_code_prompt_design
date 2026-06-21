# 第七章：Prompt 设计的十条核心原则

前六章逐节分析了 Claude Code 的每一段 prompt，现在退出细节，向上看规律。

在这些分析中，相同的设计模式反复出现。某条规则"失败了 N 次才有了这个约束"；某个选择"用命令名而非类别描述"；某处"同时禁止不足和过度"——这些不是偶然，而是在工业压力下反复锤炼出来的设计原则。

本章将这些原则提炼成十条，每条原则配：**定义**、**Claude Code 中的具体体现**、**为什么有效**、**如何迁移到你的设计**。

---

## 原则一：失败驱动

### 定义

好的 prompt 不是描述"理想行为"，而是**针对已知的失败模式**写出的防线。每条规则背后都有一个可以想象的（往往是真实发生过的）失败场景。

### Claude Code 中的体现

**体现 A：虚假声明率的数字驱动**

第四章分析的"真实性报告规则"是最典型的案例。代码注释里有：

```typescript
// @[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302) — un-gate once
// validated on external via A/B
```

Capybara v8 的虚假声明率从 v4 的 16.7% 上升到 29–30%——不是凭感觉写的规则，而是测量到了具体的失败数字，针对这个数字写出了规则：

```
Report outcomes faithfully: if tests fail, say so... Never claim "all tests pass" when output
shows failures... never characterize incomplete or broken work as done.
```

**体现 B：commit 安全协议里的具体事故**

BashTool 的 git commit 协议里有一条：

```
CRITICAL: Always create NEW commits rather than amending, unless the user explicitly
requests a git amend. When a pre-commit hook fails, the commit did NOT happen — so
--amend would modify the PREVIOUS commit...
```

这条规则描述的是一个精确的失败场景：hook 失败 → 模型误以为是提交失败 → 用 `--amend` 修复 → 实际上修改了前一个提交（不是当前任务的提交）→ 数据静默损坏。这不是预防性的假设，是被真实发生过的事故驱动的。

**体现 C：`-uall` 标志的内存事故**

```
IMPORTANT: Never use the -uall flag as it can cause memory issues on large repos.
```

这么具体的 flag 约束，背后一定有真实事件：在大型仓库上运行 `git status -uall` 导致进程 OOM。

**体现 D：PR 描述里的三个感叹号**

```
making sure to look at all relevant commits (NOT just the latest commit, but ALL commits
that will be included in the pull request!!!)
```

感叹号在 Anthropic 的 prompt 里极其罕见。三个感叹号意味着这个错误（只分析最新提交）被反复犯过很多次。

### 为什么有效

描述理想行为（"要彻底、要准确、要负责任"）对语言模型几乎没有作用——因为模型已经被训练成"尽力做到彻底和准确"，但它对什么是"彻底"的理解与人类不同。针对失败场景的规则给出了精确的失败识别条件（"当输出显示失败时"）和正确行为的边界（"不要说'所有测试通过'"）。

认知机制：具体的失败场景比抽象的理想行为更能激活模型中的相关模式。"当 pre-commit hook 失败时不要用 --amend" 比 "小心处理 git 操作" 更能在模型的注意力机制里精确锚定到正确的推理路径。

### 如何迁移

设计每条规则时，问自己：**"这条规则对应的失败场景是什么？"** 如果说不出失败场景，这条规则可能是多余的，或者可以更精确。

收集真实的失败案例（日志、用户反馈、模型在 eval 上的错误输出），把这些失败转化为具体的约束。最有效的 prompt 是"问题报告转化的规则集"，而非"理想行为的描述"。

---

## 原则二：双边约束

### 定义

对于任何一种需要约束的行为，**同时设置两个方向的边界**：不要做得不够，也不要做得过度。单向约束只能防止一种失败，双边约束防止两种对立的失败。

### Claude Code 中的体现

**体现 A：真实性报告的双边设计**（第四章）

这是全书中最清晰的双边约束案例：

```
前半段（压制虚报成功）：
Never claim "all tests pass" when output shows failures
Never characterize incomplete or broken work as done

后半段（同时压制虚报失败/过度保守）：
When a check did pass, state it plainly
Do not hedge confirmed results with unnecessary disclaimers
Do not downgrade finished work to "partial"
```

结尾总结点睛："The goal is an accurate report, not a defensive one."

**体现 B：任务追踪的"exactly one in_progress"**（第五章）

```
Exactly ONE task must be in_progress at any time (not less, not more)
```

"not less"：不能在工作但没有任何 in_progress 任务（漏标）
"not more"：不能同时有多个 in_progress 任务（多标）

**体现 C：失败处理框架**（第四章）

```
Don't retry the identical action blindly,
but don't abandon a viable approach after a single failure either.
```

同时抑制两种对立行为：盲目重试（没有诊断就重做）和过度放弃（一次失败就切换策略）。

**体现 D：代码风格规则的隐含双边**（第四章）

```
Three similar lines of code is better than a premature abstraction.
```

```
no speculative abstractions, but no half-finished implementations either.
```

每条代码风格规则都有"但不是..."的隐含对立面：不要过度抽象（但也不要没有抽象），不要错误处理（但只在边界处验证），不要为未来需求设计（但也不要留下残缺的实现）。

### 为什么有效

语言模型有一种"激活补偿"的倾向：当一个方向被强力抑制时，模型倾向于向反方向偏移。单独说"不要说测试失败时通过"，模型可能开始过度谨慎，把所有通过也标记为"可能有问题"。双边约束锁定了行为空间的两侧，让模型只能在两个边界之间的准确区域运行。

工程原因：模型在对立错误之间寻找最优点的能力，远强于在单侧约束下"到底该去哪里"的能力。给出两个边界，比给出一个边界和说"找到中间"更有效。

### 如何迁移

对于每个"不要 X"的规则，问：**"如果模型过度避开 X，它会往哪里走？那个方向是否也需要约束？"**

如果存在对立错误，把两个约束都写进 prompt，并用"but"或"equally"明确连接两者（"不要 A，同样也不要 B"）。

---

## 原则三：具体可操作

### 定义

规则中的关键词应该是**具体可识别的词语**（工具名、命令名、标志名），而非抽象类别（"文件操作类命令"、"破坏性操作"）。

### Claude Code 中的体现

**体现 A：工具优先级映射的精确命令名**（第五章）

```
Read files: Use Read (NOT cat/head/tail)
Edit files: Use Edit (NOT sed/awk)
Write files: Use Write (NOT echo >/cat <<EOF)
File search: Use Glob (NOT find or ls)
Content search: Use Grep (NOT grep or rg)
```

每个映射都有精确的"不使用"命令名：`cat`、`head`、`tail`、`sed`、`awk`、`echo`、`find`、`ls`、`grep`、`rg`。不是说"不要用系统命令替代专用工具"，而是枚举出模型最可能选择的具体命令名。

**体现 B：Git Safety Protocol 里的具体标志**（第五章）

```
NEVER skip hooks (--no-verify, --no-gpg-sign, etc)
NEVER run destructive git commands (push --force, reset --hard, checkout ., restore ., clean -f, branch -D)
```

精确枚举了每个危险操作的精确标志形式：`--no-verify`、`--no-gpg-sign`、`push --force`、`reset --hard`、`checkout .`、`restore .`、`clean -f`、`branch -D`。

**体现 C：Sleep 避免规则里的具体 check 命令**（第五章）

```
If you must poll an external process, use a check command (e.g. `gh run view`) rather than
sleeping first.
```

不只是说"不要 sleep"，给出了替代方案，并附上了一个具体例子（`gh run view`），让规则立刻可操作。

**体现 D：`file_path:line_number` 格式**（第四章）

```
When referencing specific functions or pieces of code include the pattern file_path:line_number
```

不是"引用代码时要包含位置信息"，而是指定了精确的格式 `file_path:line_number`。这个格式要求具体到可以直接套用，不需要模型再做解释。

### 为什么有效

语言模型的词语匹配能力远强于概念匹配能力。当 prompt 里出现 `cat`，模型能在处理"读文件"任务时精确地把这个词语识别出来并应用约束；而"系统文件读取命令"这个抽象类别，模型需要额外的推理步骤才能确定 `cat` 属于这个类别。

激活层面：具体词语（如工具名、命令名）在模型的嵌入空间里与其使用场景有直接关联；抽象类别的关联是间接的，需要更多上下文才能激活。

### 如何迁移

写规则时，问：**"这个规则里最重要的词是否精确到可以识别？"** 如果规则里有抽象类别（"可能危险的操作"、"不必要的输出"），试着枚举出最常见的具体例子。

对于"不要用 X，用 Y 替代"这类规则，列出 X 的所有常见变体（如果有多个类似物），并确保 Y 也具体到可以直接使用（工具名、命令名，而非"适当的替代方案"）。

---

## 原则四：理由优于规则

### 定义

告诉模型**为什么**，比只告诉模型**做什么**更有效。携带理由的规则可以泛化到没有被枚举的情况，而无理由的规则只能覆盖规则本身明确的情况。

### Claude Code 中的体现

**体现 A：工具优先级的理由说明**（第五章）

```
Using dedicated tools allows the user to better understand and review your work.
This is CRITICAL to assisting the user.
```

不只说"用 Read 不用 cat"，还说了为什么：**用户可见性和可审查性**。这个理由让模型在遇到规则未覆盖的边界情况时（比如，一个不在列表里的 shell 命令），可以自问："用这个命令是否有助于用户理解和审查？"——理由成了判断的标准。

**体现 B：URL 生成约束的"unless you are confident"**（第四章）

```
IMPORTANT: You must NEVER generate or guess URLs for the user unless you are confident
that the URLs are for helping the user with programming.
```

"unless you are confident"而非"unless explicitly instructed"——这个措辞把判断权交给了模型，并隐含了理由：不确定的 URL 会产生幻觉链接，而编程文档 URL 是有价值的。模型理解了这个理由，才能在"URL"和"编程帮助"之间做出正确判断。

**体现 C：会话级缓存设计中的理由**（第三章和第六章）

```typescript
/**
 * Session-variant guidance that would fragment the cacheScope:'global'
 * prefix if placed before SYSTEM_PROMPT_DYNAMIC_BOUNDARY. Each conditional
 * here is a runtime bit that would otherwise multiply the Blake2b prefix
 * hash variants (2^N).
 */
```

这段注释不只是说"这些 Section 放在边界之后"，还说了**为什么**：如果放在静态区域，条件分支会让全局缓存 key 按 2^N 方式爆炸。理解了这个理由，工程师在添加新的动态 Section 时就知道应该放在边界之后，不需要每次都查规范。

**体现 D：CronCreate 避免整点的理由**（第五章）

```
Every user who asks for "9am" gets `0 9`, and every user who asks for "hourly" gets `0 *`
— which means requests from across the planet land on the API at the same instant.
```

不只说"避免整点"，还解释了为什么：全球用户的集中触发会造成流量洪峰。理解了这个理由，模型在遇到"大约X点"和"恰好X点"时，能自己判断是否需要偏移，而不需要规则枚举每种情况。

### 为什么有效

规则的力量来自覆盖范围。规则只覆盖被明确枚举的情况，理由覆盖**所有类似情况**。当模型理解了"为什么"，它就拥有了一个可以应用于未枚举情况的判断原则——这是规则无法提供的泛化能力。

认知机制：理由在模型的推理层面工作，而规则在模式匹配层面工作。模式匹配快速但脆弱（边界情况容易失效），推理慢一些但鲁棒（可以处理新情况）。理由让规则从"模式匹配规则"升级为"推理原则"。

### 如何迁移

对于每条重要规则，在规则之后（或之前）加一句"because..."或"this is important because..."。理由不需要很长，一句话就够。

审视你现有的规则列表，找出那些只有"做什么"而没有"为什么"的条目。如果你自己说不清楚某条规则的理由，这条规则可能是多余的，或者需要重新思考。

---

## 原则五：决策框架优于规则列表

### 定义

与其列举"X 情况下做 A，Y 情况下做 B，Z 情况下做 C"，不如给出一个**判断框架**，让模型根据框架自己推导出应对任何情况的正确行为。

### Claude Code 中的体现

**体现 A："可逆性+爆炸半径"框架**（第四章）

```
Carefully consider the reversibility and blast radius of actions. Generally you can
freely take local, reversible actions like editing files or running tests. But for
actions that are hard to reverse, affect shared systems beyond your local environment,
or could otherwise be risky or destructive, check with the user before proceeding.
```

这个框架提供了两个维度：**可逆性**和**爆炸半径**。有了这两个维度，模型可以对任何操作做出是否需要确认的判断，无论这个操作是否在规则里被枚举。然后跟着是具体例子（破坏性操作、难以撤销的操作等），但这些例子是**框架的说明**，而非规则的全集。

**体现 B：多命令执行决策树**（第五章）

```
If the commands are independent and can run in parallel, make multiple Bash tool calls
If the commands depend on each other, use a single Bash call with '&&'
Use ';' only when you need sequential but don't care if earlier commands fail
```

三个条件分别对应三个策略，覆盖了所有多命令场景。这不是枚举，而是按"独立性"和"失败容忍"两个维度构建的决策矩阵。

**体现 C：Fork vs 新 Agent 的判断框架**（第五章）

```
Fork yourself (omit `subagent_type`) when the intermediate tool output isn't worth
keeping in your context. The criterion is qualitative — "will I need this output again"
— not task size.
```

明确给出判断标准："我将来还需要这个输出吗？"——这是一个可以应用于任何具体任务的判断框架，而不是"研究任务用 fork，实现任务用新 agent"之类的规则枚举（后者会有很多边界情况）。

**体现 D：EnterPlanMode 的双版本触发条件**（第五章）

外部用户版的七条触发条件不是规则列表，而是一个"扩展的判断框架"：任何一个条件满足 → 进入规划模式。最后一条甚至给出了框架内部的等价条件："如果你会用 AskUserQuestion，就改用 EnterPlanMode"——这是框架的一致性保证，而非额外规则。

### 为什么有效

规则列表的覆盖范围是有限的，任何规则列表都有"没有被枚举"的边界情况。决策框架的覆盖范围是原则上无限的——只要是框架维度覆盖的情况，模型都可以做出判断。

工程原因：维护成本不同。规则列表随着新情况的出现需要不断添加新规则；决策框架在设计好之后，新情况自动被框架覆盖，不需要更新。

### 如何迁移

当你发现自己在添加越来越多的"如果 X 就做 A，如果 Y 就做 B"规则时，停下来问：**"X 和 Y 有什么共同的判断维度？"** 把这个维度提炼出来，把具体的 X/Y 变成维度的例子而非规则的全集。

好的框架往往只有 1-2 个判断维度，并且对几乎所有情况都能给出明确答案。如果框架有很多边界情况，可能维度选得不对，需要重新抽象。

---

## 原则六：禁止与替代成对

### 定义

每个"不要 X"的约束，都应该配一个"改用 Y"或"X 之外的正确做法是..."。**禁止是封堵，替代是出路**——只有出路的禁止才是完整的约束。

### Claude Code 中的体现

**体现 A：工具优先级的完整三元组**（第五章）

每条工具优先级规则都是完整的三元组：
```
[应用场景] — [禁止] — [替代]
Read files — NOT cat/head/tail — Use Read
Edit files — NOT sed/awk — Use Edit
```

没有任何一条只说"不要用 cat"而不给替代工具。

**体现 B：Sleep 避免规则里的替代方案**（第五章）

```
If your command is long running and you would like to be notified when it finishes —
use `run_in_background`. No sleep needed.
```

```
If you must poll an external process, use a check command (e.g. `gh run view`)
rather than sleeping first.
```

两个常见的 sleep 使用场景，各有具体的替代方案：长时任务用 `run_in_background`，轮询外部状态用 `gh run view`。

**体现 C：Glob 的自我限制与替代引导**（第五章）

```
When you are doing an open ended search that may require multiple rounds of globbing
and grepping, use the Agent tool instead.
```

Glob 在自身的 prompt 里主动告知："这种情况下不用我，用 Agent"。禁止（不用 Glob）配上替代（用 Agent），而且这条"禁止"来自于被禁止的工具本身——这是"自我限制"的设计。

**体现 D：注释规范的禁止与替代**（第四章，内部用户）

```
Default to writing no comments. Only add one when the WHY is non-obvious.
```

禁止（默认不写注释）+ 替代条件（当 WHY 不明显时才写）+ 替代标准（隐藏约束、细微不变量、特定 bug 的变通方法）。

**体现 E：AskUserQuestion 的规划模式替代**（第五章）

```
If you would use AskUserQuestion to clarify the approach, use EnterPlanMode instead.
```

当 AskUserQuestion 被限制使用时，明确给出了替代工具（EnterPlanMode），以及替代的条件（需要澄清实现方式时）。

### 为什么有效

没有出路的禁止会导致两种问题：要么模型仍然做被禁止的事情（因为它不知道该怎么完成任务），要么模型在需要某个能力时陷入困境（知道不能做但不知道可以做什么）。

"禁止"告诉模型哪条路被封堵，"替代"告诉模型哪条路是正确的。两者一起才能可靠地引导行为。

### 如何迁移

审视所有"不要做 X"的规则，问：**"如果模型不能做 X，它用什么方式完成同样的目标？"** 如果你没有答案，可能规则太宽了，或者需要澄清在什么场景下才适用这个约束。

最好在同一句话里写完禁止和替代（"不要用 X，改用 Y"），而不是把它们放在不同的段落里——模型在处理"不要 X"时，紧接着看到"用 Y"比之后再找替代方案更有效。

---

## 原则七：认知框架

### 定义

给模型提供**思维工具和角色隐喻**，而不只是外部规则约束。认知框架让模型从内部驱动行为，而非被动服从规则。

### Claude Code 中的体现

**体现 A："好同事"的角色隐喻**（第五章 AgentTool）

```
Brief the agent like a smart colleague who just walked into the room — it hasn't seen
this conversation, doesn't know what you've tried, doesn't understand why this task matters.
```

"刚走进房间的聪明同事"不是一条规则，而是一个认知框架。当模型在写 Agent prompt 时，它可以用这个角色来检验自己的写作：这段话对一个"刚走进房间"的人说清楚了吗？这个框架比任何具体的写作规则都更容易记住和应用。

**体现 B：自主模式的"好同事"框架**（第四章 getProactiveSection）

```
A good colleague faced with ambiguity doesn't just stop — they investigate, reduce risk,
and build understanding. Ask yourself: what don't I know yet? What could go wrong? What
would I want to verify before calling this done?
```

三个自问问题形成了一个持续工作的思维循环。不是规则（"遇到歧义时去读更多文件"），而是认知框架（"好同事会怎么想？"）——模型可以把任何未预见的场景套入这个框架。

**体现 C：Kairos 模式的"terminalFocus"校准**（第四章）

```
Unfocused: The user is away. Lean heavily into autonomous action.
Focused: The user is watching. Be more collaborative.
```

这不是规则，而是一个**行为校准模型**：根据用户是否在场来调整自主程度。模型获得了一个思维模型（"我是在独立工作还是在被观察？"），而不只是两条规则。

**体现 D：压缩 prompt 里的 `<analysis>` scratchpad**（第六章）

```
Before providing your final summary, wrap your analysis in <analysis> tags to organize
your thoughts and ensure you've covered all necessary points.
```

`<analysis>` 标签不是对模型行为的约束，而是一个**认知工具**——给模型提供了一个明确的"草稿空间"，让它可以在这里自由展开思考，而不必直接输出最终结论。这个工具实际上提升了摘要质量（比直接要求输出摘要效果更好）。

**体现 E：BASE_COMPACT_PROMPT 里的"直接引用防止漂移"**（第六章）

```
include direct quotes from the most recent conversation showing exactly what task you were
working on and where you left off. This should be verbatim to ensure there's no drift in
task interpretation.
```

"verbatim to ensure there's no drift"——这里给了模型一个认知框架：摘要不是为了表达你的理解，而是为了**防止解释偏差**。模型理解了这个框架，就知道在摘要任务里，忠实复现原文比用自己的语言改写更重要。

### 为什么有效

外部规则（"做 A，不做 B"）依赖于模型记住并执行规则的每个条目；认知框架（"想象你是一个好同事"）改变了模型对**任务本身的理解**，使正确行为成为理解任务的自然结果，而非服从规则的结果。

服从规则和理解任务有根本区别：规则可以被遗漏（多条规则同时工作时，模型可能只关注了部分）；任务理解是整体性的，一旦建立，就渗透到所有相关决策中。

### 如何迁移

问：**"模型在做这个任务时，脑子里应该有什么思维模型？"** 然后明确写出这个思维模型，而不只是写出期望的输出格式。

有效的认知框架通常是：
- 角色隐喻（"像一个...的人"）
- 判断维度（"问自己：...？"）
- 目标框架（"这个任务的目的是...，而不是..."）

---

## 原则八：权限不可传递

### 定义

用户对某个操作的**一次授权**，仅适用于该次授权的具体范围，不能被模型解读为对类似操作的广泛许可。每次需要授权的操作都独立处理。

### Claude Code 中的体现

**体现 A：getActionsSection() 里的明确声明**（第四章）

```
A user approving an action (like a git push) once does NOT mean that they approve it in
all contexts, so unless actions are authorized in advance in durable instructions like
CLAUDE.md files, always confirm first. Authorization stands for the scope specified,
not beyond. Match the scope of your actions to what was actually requested.
```

这条规则的核心是"Authorization stands for the scope specified, not beyond"——授权范围与授权完全对应，不能外延。

**体现 B：CLAUDE.md 作为"持久授权"的单独处理**（第四章）

```
unless actions are authorized in advance in durable instructions like CLAUDE.md files
```

CLAUDE.md 里的指令被视为"持久授权"——用户在 CLAUDE.md 里写了"始终自动提交"，就是对这个操作的广泛授权。这是权限不可传递原则的**唯一例外**：经过特定渠道（持久指令文件）的授权才能扩展到多次操作。

**体现 C：BashTool 的 Git Safety Protocol**（第五章）

```
NEVER update the git config
NEVER run destructive git commands (push --force, reset --hard...) unless the user
explicitly requests these actions.
```

"explicitly requests"是关键词——每次破坏性操作都需要当次的明确请求，不能因为上次用户说了"这种情况下可以 force push"就推断"下次类似情况也可以"。

**体现 D：Sandbox 的逐命令授权**（第五章，sandbox 模式）

```
Treat each command you execute with `dangerouslyDisableSandbox: true` individually.
Even if you have recently run a command with this setting, you should default to running
future commands within the sandbox.
```

用 `dangerouslyDisableSandbox: true` 运行了一个命令，不意味着下一个命令也可以用这个设置——每个命令独立判断，授权不传递。

### 为什么有效

语言模型有强烈的"类比推理"倾向：如果情况 A 获得了授权，类似的情况 B 应该也可以。这种推理在自然对话中是有益的（上下文学习），但在 AI Agent 做高风险操作时是危险的——用户可能有充分理由在 A 情况下授权但在 B 情况下不授权，这些理由模型不一定能理解。

安全机制：权限不可传递原则把 Agent 的授权模型从"状态式授权（曾经被授权就一直有）"切换到"事件式授权（每次需要时独立授权）"。这样即使模型犯了错误（如过度类比），错误的范围也只限于当前操作，不会蔓延到后续所有类似操作。

### 如何迁移

对于 Agent 应用中的任何高风险操作，检查：**"模型是否可能从之前的授权中推断出当前操作也被授权？"** 如果可能，明确写出"每次...操作都需要独立确认"，或者定义哪些渠道的授权是持久的（类似于 CLAUDE.md 的角色）。

---

## 原则九：格式即行为

### 定义

输出格式的要求不只是让输出更易读，**格式结构直接影响模型的思维质量**。正确的格式要求既是输出约束，也是认知约束。

### Claude Code 中的体现

**体现 A：`<analysis>` + `<summary>` 的双块格式**（第六章）

```
Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

这个格式要求让模型先在 `<analysis>` 里自由展开思考，再在 `<summary>` 里给出结构化摘要。如果只要求输出摘要，模型会直接跳到结论；`<analysis>` 标签创造了一个中间步骤，强制模型先"思考后写作"，提升了摘要质量。格式要求成了认知要求。

**体现 B：BASE_COMPACT_PROMPT 的 9 Section 结构**（第六章）

9 个明确命名的 Section（Primary Request, Key Technical Concepts, Files...），每个 Section 都有明确的内容要求。这个格式不只是让摘要更易读，它**迫使模型覆盖所有必要的维度**——如果没有这些 Section 标题，模型的自由摘要可能遗漏某些维度（如"所有用户消息"或"可选的下一步"）。格式起到了"思维清单"的作用。

**体现 C：HEREDOC 的提交消息格式**（第五章）

```
ALWAYS pass the commit message via a HEREDOC... to ensure good formatting
```

HEREDOC 要求不只是格式偏好——它防止了 Co-Authored-By 署名的格式在不同 shell 里被错误转义。格式要求直接保证了功能正确性。

**体现 D：`file_path:line_number` 格式**（第四章）

```
include the pattern file_path:line_number to allow the user to easily navigate to the
source code location
```

这个格式要求让代码引用从"描述"变成了"可导航的链接"。格式直接改变了输出的实用价值——不只是更美观，而是功能质变。

**体现 E：PR 创建的 HEREDOC 格式**（第五章）

```
Create PR using gh pr create with the format below. Use a HEREDOC to pass the body to
ensure correct formatting.

gh pr create --title "the pr title" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points>

## Test plan
[Bulleted markdown checklist...]
EOF
)"
```

这个具体的格式模板把 PR 创建从"自由发挥"变成了"填写模板"。`## Summary` 和 `## Test plan` 的标题强制模型在生成 PR 描述时覆盖这两个维度，而非只写一段自由描述。

**体现 F：TodoWriteTool 的 content + activeForm 双形式**（第五章）

```
IMPORTANT: Task descriptions must have two forms:
- content: The imperative form (e.g., "Run tests")
- activeForm: The present continuous form (e.g., "Running tests")
```

这个格式要求是 UI 需求，但它通过 prompt 约束来实现——格式要求直接服务于界面体验（in_progress 状态显示 activeForm）。

### 为什么有效

结构化的输出格式在认知层面创造了"格式脚手架"——模型按照格式填充内容时，格式本身引导了思考的方向和范围。就像人类填写表格比自由写作更不容易遗漏信息，格式化的 prompt 让模型的输出更完整、更少遗漏。

此外，格式化输出便于程序化处理（解析 `<summary>` 标签、提取 Section N 的内容），这是格式要求同时服务于人类读者和程序处理的双重价值。

### 如何迁移

设计 prompt 时，考虑：**"如果模型在这个任务上的思维过程有一个理想的步骤顺序，这个顺序能否通过格式要求来强制执行？"**

具体策略：
- 多步推理任务：要求先写草稿/分析块，再写最终输出（`<analysis>` + `<summary>`）
- 需要覆盖多个维度的任务：给出明确的 Section 标题（而非要求"全面覆盖"）
- 需要程序解析的输出：使用 XML 标签或 JSON 格式约束

---

## 原则十：Prompt 随模型版本演化

### 定义

Prompt 不是一次写好就永久固定的文档，而是**与模型能力和缺陷共同演化的工程产物**。每个模型版本可能引入新的失败模式，也可能修复旧的问题——prompt 需要随之更新。

### Claude Code 中的体现

**体现 A：模型启动标记和临时补丁**（第四章）

代码里有多处 `@[MODEL LAUNCH]` 标记：

```typescript
// @[MODEL LAUNCH]: Update comment writing for Capybara — remove or soften once
// the model stops over-commenting by default
```

```typescript
// @[MODEL LAUNCH]: Remove this section when we launch numbat.
```

这些标记显示：某些 prompt Section 是针对当前模型版本的**临时补丁**，预计在下一个模型版本发布时移除或修改。这是把 prompt 视为版本化工程产物的直接证据。

**体现 B：内外用户差异的模型依赖**（第四章）

```typescript
// @[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302) — un-gate once
// validated on external via A/B
```

Capybara v8 的"thoroughness counterweight"（彻底性反压）规则只针对内部用户，而且预计在 A/B 测试验证后推送给外部用户——这是一个渐进部署的 prompt 工程实践。

**体现 C：工具调用率的版本差异**（第六章）

```typescript
// 2.79% on 4.6 vs 0.01% on 4.5
```

Sonnet 4.6 在压缩场景下的工具调用率比 4.5 高了 279 倍，这个差异直接驱动了 `NO_TOOLS_PREAMBLE` 的设计（加强约束）和位置选择（放在最前面）。同样的 prompt 在不同模型版本上可能产生截然不同的行为，需要针对性调整。

**体现 D：虚假声明率的版本跟踪**（第四章）

从代码注释可以推断，Anthropic 在每个模型版本发布时都会测量虚假声明率。从 v4（16.7%）到 v8（29-30%）的上升，触发了"真实性报告规则"的修订。这是"测量-发现问题-写规则"的正反馈循环。

**体现 E：Scratchpad 剥离的版本适配**（第六章）

```typescript
// and on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a tool call
// despite the weaker trailer instruction.
```

"Sonnet 4.6+"的 adaptive-thinking 特性让模型更倾向于尝试工具调用——这是一个模型版本特有的行为变化，驱动了压缩 prompt 的防御性加强。

### 为什么有效

这条原则本身不是"有效的设计技巧"，而是对 prompt 工程本质的认识。Prompt 不是对理想 AI 行为的规范，而是对特定模型在特定场景下的行为矫正。模型每次更新，行为就会改变，某些矫正可能变得不必要，某些新的矫正可能变得必要。

把 prompt 视为与模型版本绑定的工程产物，而非永久固定的规则集，是维持 prompt 有效性的必要认识。

### 如何迁移

**建立测量机制**：确定关键的行为指标（如某类错误的发生率）并持续测量。当指标恶化时，检查是否是新模型版本引入的行为变化，针对性地更新 prompt。

**标记临时规则**：在 prompt 注释（或文档）里标注哪些规则是针对当前模型版本的临时补丁，以及什么条件下可以移除。这避免了临时规则变成永久规则而永远不被清理。

**分离"通用原则"和"模型特定规则"**：前者（如"双边约束"、"禁止+替代"）跨模型版本有效；后者（如"不要过度添加注释"）只对特定模型的特定缺陷有效。分离这两类，便于在模型更新时快速识别哪些规则需要重新评估。

---

## 十条原则的关系

这十条原则不是孤立的，它们有内在的逻辑层次：

**基础层：认识论原则**
- 原则一（失败驱动）：确定写什么——针对失败，不描述理想
- 原则十（随版本演化）：确定何时更新——模型变了，prompt 也要变

**内容层：规则设计原则**
- 原则二（双边约束）：规则覆盖两个方向，不只防一种失败
- 原则三（具体可操作）：规则用具体名词，不用抽象类别
- 原则四（理由优于规则）：规则携带原因，具备泛化能力
- 原则六（禁止与替代成对）：每个约束都有出路

**架构层：结构设计原则**
- 原则五（决策框架）：框架优于列表，覆盖未枚举情况
- 原则八（权限不可传递）：授权是点状的，不能传播

**认知层：思维设计原则**
- 原则七（认知框架）：给模型思维工具，改变内部驱动
- 原则九（格式即行为）：格式约束思维过程，不只约束输出

在实际设计 prompt 时，这十条原则应该共同应用：从基础层确定要解决什么问题，用内容层原则设计具体规则，用架构层原则组织规则，用认知层原则提升整体效果。

---

## 小结：工业级 Prompt 的特征

Claude Code 的 prompt 体系，经过 Anthropic 工程团队在工业压力下的长期迭代，呈现出以下区别于"教程级 prompt"的特征：

**1. 精确的失败模式覆盖**：每条规则都对应一个具体的失败，不是凭感觉的"最佳实践"。

**2. 量化驱动**：有数字支撑的决策（2.79% vs 0.01%、16.7% vs 29-30%）比凭直觉的决策更可信，也更容易在版本更新时重新评估。

**3. 双边安全边界**：不只防止"做得不够"，也防止"做得过度"——对称的约束反映了对模型行为空间的完整理解。

**4. 可泛化的框架**：核心设计是判断框架和认知工具，而非穷尽的规则列表——这让系统在遇到未预见情况时仍然有行为基准。

**5. 版本化管理**：prompt 被视为与模型版本绑定的工程产物，临时补丁有标记，持久原则有区分。

这些特征不是 Claude Code 特有的——它们是任何工业级 LLM 应用在经历了足够多的失败迭代之后，自然演化出的工程实践。本书希望通过对 Claude Code 的完整解析，让读者不必经历相同的失败迭代，直接站在这套实践的肩膀上。
