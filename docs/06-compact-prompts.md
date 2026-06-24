# 第六章：压缩摘要 Prompt 全解析

每次与 Claude 的对话都受限于上下文窗口。当 Claude Code 的对话历史积累到窗口限制附近时，系统会启动"压缩"机制——把旧的对话历史摘要成一段结构化文字，替换掉原始消息，从而为新的工作释放空间。

这个机制涉及三类 prompt，分别对应三种不同的场景和设计目标。本章按从简单到复杂的顺序，完整分析这套压缩 prompt 体系的每一处设计决策。

> **本书贯穿的设计模式**
>
> 以下设计模式在本章中反复出现。每次出现的**最典型处**会用 `→ 原则N` 标注，完整归纳见第七章。
>
> `① 失败驱动` `② 双边约束` `③ 具体可操作` `④ 理由优于规则`
> `⑤ 决策框架` `⑥ 禁止+替代` `⑦ 认知框架` `⑧ 权限不可传递`
> `⑨ 格式即行为` `⑩ 随版本演化`

---

## 6.1 三层压缩架构概览

Claude Code 有三种不同的压缩路径，各自有不同的代价和适用场景：

```
对话历史积累
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│ 第一层：微压缩（MicroCompact）                              │
│ 目标：清理旧的工具调用结果（tool results）                     │
│ 代价：零 API 调用（内容替换，不摘要）                          │
│ 触发：① 时间触发（距上次响应超过 N 分钟）                      │
│        ② 缓存编辑触发（CACHED_MICROCOMPACT，通过 API 层删除）  │
└─────────────────────────────────────────────────────────┘
      │ 若微压缩不足以控制窗口使用率
      ▼
┌─────────────────────────────────────────────────────────┐
│ 第二层：全量压缩（Full Compact / Autocompact）              │
│ 目标：摘要整个（或部分）对话历史                               │
│ 代价：一次额外的 API 调用（用于生成摘要）                      │
│ 触发：对话 token 数超过自动压缩阈值，或用户手动 /compact        │
└─────────────────────────────────────────────────────────┘
      │ 若启用了会话记忆功能
      ▼
┌─────────────────────────────────────────────────────────┐
│ 第三层：会话记忆压缩（Session Memory Compact）              │
│ 目标：用持久化的会话记忆替代实时 LLM 摘要                     │
│ 代价：几乎零（读取已生成的文件，不重新生成摘要）                 │
│ 触发：会话记忆文件存在且非空，且 SM Compact 功能已启用          │
└─────────────────────────────────────────────────────────┘
```

本章的核心是第二层——全量压缩的 prompt 设计。第一层（微压缩）的机制在后半部分分析，第三层（会话记忆压缩）作为架构补充在最后简要描述。

---

## 6.2 `NO_TOOLS_PREAMBLE`：关键约束的双重封锁

**英文原文**（来源：`src/services/compact/prompt.ts`，第 19–26 行）

```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.

```

!!! note "中文附注"


    **严重警告**：仅用**纯文字**回应。**不要**调用任何工具。

    - 不要使用 Read、Bash、Grep、Glob、Edit、Write 或**任何**其他工具。
    - 上面的对话中你已经拥有所需的全部上下文。
    - 工具调用将被**拒绝**，并将浪费你唯一的轮次——你将无法完成任务。
    - 你的整个回应必须是纯文字：一个 `<analysis>` 块，然后是一个 `<summary>` 块。

**设计解析**

`NO_TOOLS_PREAMBLE` 是整个压缩系统中最具工程性的一段 prompt。它被放在所有压缩 prompt 的**最前面**，而不是最后面，这个位置选择背后有精确的技术原因。

源代码注释解释了全部背景：

```typescript
// Aggressive no-tools preamble. The cache-sharing fork path inherits the
// parent's full tool set (required for cache-key match), and on Sonnet 4.6+
// adaptive-thinking models the model sometimes attempts a tool call despite
// the weaker trailer instruction. With maxTurns: 1, a denied tool call means
// no text output → falls through to the streaming fallback (2.79% on 4.6 vs
// 0.01% on 4.5). Putting this FIRST and making it explicit about rejection
// consequences prevents the wasted turn.
```

> **中文解读**：cache-sharing fork 路径继承了父 Agent 的完整工具集（缓存键匹配需要）。在 Sonnet 4.6+ 的自适应思考模型上，模型有时会尝试工具调用，尽管尾部的弱约束存在。由于 `maxTurns: 1`，被拒绝的工具调用意味着没有文字输出 → 退回到流式降级（4.6 上发生率 2.79% vs 4.5 上的 0.01%）。把此内容放在**最前面**，并明确说明拒绝后果，可以防止轮次浪费。

这段注释揭示了三个设计决策的相互依赖：

**第一，压缩 prompt 的上下文里有完整的工具集。** 压缩 prompt 通过 cache-sharing fork 路径执行（共享父 Agent 的缓存前缀，以节省 cache_creation token），这意味着模型在执行压缩时，仍然"看到"完整的工具列表。单纯告诉模型"不要用工具"是不够的——模型知道工具在那里，面对复杂的摘要任务时，自然倾向于"先读几个文件来确认细节"。

**第二，maxTurns: 1 使工具调用的代价变成了任务失败。** 压缩操作只允许一轮响应。如果模型在这一轮里调用了工具（被系统拒绝），那这一轮就结束了，没有文字输出，压缩失败。2.79% 的失败率是真实测量数据——这不是假设的风险。

**第三，PREAMBLE 放在最前面是预防，TRAILER 放在最后面是强化。** 两者共同形成"双重封锁"：模型在开始读 prompt 时就接收到最强约束（CRITICAL + 工具名枚举），在读完全部内容后再次收到提醒（REMINDER）。对于一个思维链可能在中途改变决策的模型，首尾两处约束比单一约束更可靠。

**"Tool calls will be REJECTED and will waste your only turn — you will fail the task."** 这句话的设计很精妙：它不只是说"不要用工具"，而是让模型理解后果的完整因果链（被拒绝 → 浪费轮次 → 任务失败）。让模型理解"为什么不能"比单纯命令"不要做"更有效，因为模型可以用这个理由在自己的推理中抑制工具调用冲动。

`→ 原则①：失败驱动` — 2.79%（4.6）vs 0.01%（4.5）是实测数据，PREAMBLE 的首位置设计和 TRAILER 强化都来自对这个失败率的测量，见第七章 1.1 节。

`→ 原则⑩：随版本演化` — 注释里明确了版本差异（4.6+ adaptive-thinking 行为不同于 4.5），prompt 设计随模型版本更新而调整，见第七章 10.1 节。

---

## 6.3 `DETAILED_ANALYSIS_INSTRUCTION_BASE`：引导式分析模板

**英文原文（BASE 版本）**（来源：`src/services/compact/prompt.ts`，第 31–44 行）

```
Before providing your final summary, wrap your analysis in <analysis> tags to organize your thoughts and ensure you've covered all necessary points. In your analysis process:

1. Chronologically analyze each message and section of the conversation. For each section thoroughly identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like:
     - file names
     - full code snippets
     - function signatures
     - file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback that you received, especially if the user told you to do something differently.
2. Double-check for technical accuracy and completeness, addressing each required element thoroughly.
```

**英文原文（PARTIAL 版本）**（来源：`src/services/compact/prompt.ts`，第 46–59 行）

```
Before providing your final summary, wrap your analysis in <analysis> tags to organize your thoughts and ensure you've covered all necessary points. In your analysis process:

1. Analyze the recent messages chronologically. For each section thoroughly identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like:
     - file names
     - full code snippets
     - function signatures
     - file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback that you received, especially if the user told you to do something differently.
2. Double-check for technical accuracy and completeness, addressing each required element thoroughly.
```

!!! note "中文附注"


    在提供最终摘要之前，将你的分析包裹在 `<analysis>` 标签中，以整理思路并确保覆盖了所有必要的要点。在分析过程中：

    1. 按时间顺序分析对话的每条消息和每个部分。对每个部分彻底识别：
        - 用户的明确请求和意图
        - 你处理用户请求的方法
        - 关键决策、技术概念和代码模式
        - 具体细节，如：文件名、完整代码片段、函数签名、文件编辑
        - 你遇到的错误以及如何修复它们
        - **特别关注**你收到的具体用户反馈，尤其是如果用户告诉你要用不同的方式做某事
    2. 仔细检查技术准确性和完整性，彻底处理每个必要元素。

**设计解析**

**`<analysis>` 标签是草稿便签，不是输出内容。** 源代码注释说：

```typescript
// Two variants: BASE scopes to "the conversation", PARTIAL scopes to "the
// recent messages". The <analysis> block is a drafting scratchpad that
// formatCompactSummary() strips before the summary reaches context.
```

模型被要求在 `<analysis>` 里自由思考，但这个思考过程最终会被 `formatCompactSummary()` 函数删除——它永远不会出现在注入回对话的摘要里。这是"Chain-of-Thought（思维链）剥离"的实际工程实现：利用 CoT 提升输出质量，但不让中间思维过程占用宝贵的上下文空间。

`→ 原则⑨：格式即行为` — `<analysis>` 块规定了模型的"打草稿"行为，`<summary>` 块规定了输出内容，两个标签名本身就是行为指令，见第七章 9.1 节。

**"按时间顺序分析"的约束** 防止了一个常见的摘要失败：自由摘要时，模型倾向于按重要性排列（把最重要的事放前面），而不是按时间顺序。但对于需要续接工作的摘要，时间顺序比重要性顺序更关键——"最近做了什么"比"最重要的是什么"更有续接价值。

**"完整的代码片段"而非"代码描述"** 是这段 prompt 里最有工程价值的细节：

```
- Specific details like:
  - file names
  - full code snippets
  - function signatures
  - file edits
```

"full code snippets"——注意是"完整代码片段"，不是"代码描述"或"代码摘要"。摘要里如果写"修改了认证逻辑"，续接时模型不知道具体改了什么；但如果有"把 `if (token)` 改为 `if (token && !token.expired)`"，模型可以直接基于这个上下文继续工作，不需要重新读文件。

**"Pay special attention to specific user feedback"** 是防止"意图漂移"的关键规则。在长对话中，用户可能多次修正模型的方向（"不对，我说的是..."，"不要用那种方式，改用..."）。如果摘要只记录了结果而不记录用户的反馈，续接后的模型可能重蹈覆辙，又走回被纠正过的旧路。

**BASE vs PARTIAL 的唯一区别** 是范围描述：

- BASE：`"Chronologically analyze each message and section of the conversation"` — 整个对话
- PARTIAL：`"Analyze the recent messages chronologically"` — 仅最近的消息

这个差异对应两种不同的压缩模式（见下节分析）。

---

## 6.4 `BASE_COMPACT_PROMPT`：全量压缩的 9 个 Section

**英文原文**（来源：`src/services/compact/prompt.ts`，第 61–143 行）

```
Your task is to create a detailed summary of the conversation so far, paying close attention to the user's explicit requests and your previous actions.
This summary should be thorough in capturing technical details, code patterns, and architectural decisions that would be essential for continuing development work without losing context.

[DETAILED_ANALYSIS_INSTRUCTION_BASE]

Your summary should include the following sections:

1. Primary Request and Intent: Capture all of the user's explicit requests and intents in detail
2. Key Technical Concepts: List all important technical concepts, technologies, and frameworks discussed.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created. Pay special attention to the most recent messages and include full code snippets where applicable and include a summary of why this file read or edit is important.
4. Errors and fixes: List all errors that you ran into, and how you fixed them. Pay special attention to specific user feedback that you received, especially if the user told you to do something differently.
5. Problem Solving: Document problems solved and any ongoing troubleshooting efforts.
6. All user messages: List ALL user messages that are not tool results. These are critical for understanding the users' feedback and changing intent.
7. Pending Tasks: Outline any pending tasks that you have explicitly been asked to work on.
8. Current Work: Describe in detail precisely what was being worked on immediately before this summary request, paying special attention to the most recent messages from both user and assistant. Include file names and code snippets where applicable.
9. Optional Next Step: List the next step that you will take that is related to the most recent work you were doing. IMPORTANT: ensure that this step is DIRECTLY in line with the user's most recent explicit requests, and the task you were working on immediately before this summary request. If your last task was concluded, then only list next steps if they are explicitly in line with the users request. Do not start on tangential requests or really old requests that were already completed without confirming with the user first.
                       If there is a next step, include direct quotes from the most recent conversation showing exactly what task you were working on and where you left off. This should be verbatim to ensure there's no drift in task interpretation.

[示例输出结构]

Please provide your summary based on the conversation so far, following this structure and ensuring precision and thoroughness in your response. 

There may be additional summarization instructions provided in the included context. If so, remember to follow these instructions when creating the above summary. Examples of instructions include:
<example>
## Compact Instructions
When summarizing the conversation focus on typescript code changes and also remember the mistakes you made and how you fixed them.
</example>

<example>
# Summary instructions
When you are using compact - please focus on test output and code changes. Include file reads verbatim.
</example>
```

!!! note "中文附注"


    你的任务是创建迄今为止对话的详细摘要，密切关注用户的明确请求和你之前的行动。此摘要在捕捉技术细节、代码模式和架构决策方面应该是全面的，这些对于在不丢失上下文的情况下继续开发工作至关重要。

    你的摘要应包括以下部分：

    1. **主要请求和意图**：详细捕捉用户所有明确的请求和意图
    2. **关键技术概念**：列出所有重要的技术概念、技术和框架
    3. **文件和代码段**：列举被检查、修改或创建的特定文件和代码段。特别关注最近的消息并在适用时包含完整代码片段，并包含说明为什么此文件读取或编辑重要的摘要。
    4. **错误和修复**：列出你遇到的所有错误，以及如何修复它们。特别关注你收到的具体用户反馈，尤其是如果用户告诉你要用不同的方式做某事。
    5. **问题解决**：记录已解决的问题和任何正在进行的故障排除工作。
    6. **所有用户消息**：列出所有不是工具结果的用户消息。这些对于理解用户的反馈和变化的意图至关重要。
    7. **待处理任务**：概述你被明确要求处理的任何待处理任务。
    8. **当前工作**：详细描述在此摘要请求之前正在进行的工作，特别关注用户和助手的最新消息。在适用时包含文件名和代码片段。
    9. **可选的下一步**：列出与你最近工作相关的下一步。重要：确保此步骤与用户最近的明确请求和你在此摘要请求之前正在处理的任务**直接**一致。如果下一步存在，包含来自最近对话的**直接引用**，准确显示你正在处理的任务和你停下来的地方。这应该是逐字引用，以确保任务解释没有漂移。

**设计解析**

9 个 Section 的设计不是随机排列——它们构成了一个从**背景 → 状态 → 续接**的信息漏斗。

**Section 1-5：背景信息层**

这五个 Section 提供了对话的全局背景：用户的请求是什么、涉及什么技术、文件如何变化、遇到了什么问题。这是续接工作的基础知识层。

**Section 6（All user messages）** 是整个 9 个 Section 里最特殊的一个：

```
6. All user messages: List ALL user messages that are not tool results. These are critical
for understanding the users' feedback and changing intent.
```

"**LIST ALL**"——不是摘要，不是重要的，而是**全部**。这个要求看起来与"摘要"的目标相矛盾（摘要的目的不是压缩吗？）。背后的逻辑是：用户消息是整个对话中**信息密度最高的部分**——每条用户消息都代表了一次明确的意图声明、方向修正或反馈。摘要这些消息会引入解释性偏差（摘要者的理解可能与实际意图不符）。完整保留用户消息，续接时的模型可以原汁原味地理解用户的历史意图，而不是通过摘要者的中间层。

**Section 8（Current Work）和 Section 9（Optional Next Step）** 是续接的核心：

```
9. Optional Next Step: ... If there is a next step, include direct quotes from the most recent
conversation showing exactly what task you were working on and where you left off. This should
be verbatim to ensure there's no drift in task interpretation.
```

"**verbatim**"（逐字）——这是全文中唯一要求用原话引用而非转述的地方。为什么？因为摘要中的任何转述都可能引入"任务漂移"（task drift）——续接后的模型基于转述的任务理解开始工作，而不是基于用户说的原话。随着压缩次数增加，每次转述都会累积一层偏差，最终模型做的事情与用户最初想要的相去甚远。verbatim 引用是防止这种语义漂移的最可靠方法。

**"Optional Next Step"而非"Next Step"** 是另一个精细设计：

```
If your last task was concluded, then only list next steps if they are explicitly in line
with the users request. Do not start on tangential requests or really old requests that were
already completed without confirming with the user first.
```

摘要不应该凭空为续接模型生成议程。如果对话中的任务已经完成，续接模型应该等待用户的新指令，而不是从摘要里找"还没做的旧请求"来执行。这条规则防止了一种自动化陷阱：模型在压缩后自行恢复执行用户早就改变主意的旧任务。

**自定义压缩指令的支持**：

```
There may be additional summarization instructions provided in the included context.
```

用户可以在 CLAUDE.md 里写 `## Compact Instructions`，Claude Code 会在压缩时把这些指令追加到 prompt 里。这是一种元定制：用户可以改变压缩摘要的重点（"关注 TypeScript 代码变更"、"包含 verbatim 的文件读取内容"）。这个设计的实现方式是 `getCompactPrompt(customInstructions?)` 函数签名——自定义指令作为可选参数追加在基础 prompt 之后，不影响基础结构。

---

## 6.5 `PARTIAL_COMPACT_PROMPT`：局部压缩变体

**英文原文（关键差异）**（来源：`src/services/compact/prompt.ts`，第 145–203 行，节选）

```
Your task is to create a detailed summary of the RECENT portion of the conversation — the messages that follow earlier retained context. The earlier messages are being kept intact and do NOT need to be summarized. Focus your summary on what was discussed, learned, and accomplished in the recent messages only.

[...]

Please provide your summary based on the RECENT messages only (after the retained earlier context), following this structure and ensuring precision and thoroughness in your response.
```

!!! note "中文附注"


    你的任务是创建对话**最近部分**的详细摘要——跟在之前保留的上下文之后的消息。之前的消息保持完整，**不需要**摘要。只关注最近消息中讨论、学习和完成的内容。

    请仅基于**最近的消息**（在保留的早期上下文之后）提供你的摘要。

**设计解析**

局部压缩（`PARTIAL_COMPACT_PROMPT`，对应 `direction: 'from'`）解决了这样一个场景：对话历史很长，但已经有了一个"压缩边界"——边界之前的内容已经被摘要过，边界之后的新内容需要被摘要并合并。

这种场景下，全量压缩 BASE 的问题是：它会把已经摘要过的旧摘要再次摘要，层层套叠，逐渐失真。PARTIAL 的设计是"只摘要新部分"，保留旧部分的摘要不动，然后把两者拼接起来。

PARTIAL 与 BASE 的结构差异非常小（只有范围描述不同），但这个微小的措辞差异（"recent messages only" vs "the conversation so far"）对模型的摘要行为有显著影响——模型会遵从这个范围约束，不去重新处理它被告知"已保留完整"的早期内容。

---

## 6.6 `PARTIAL_COMPACT_UP_TO_PROMPT`：反向压缩变体

**英文原文**（来源：`src/services/compact/prompt.ts`，第 208–267 行）

```
Your task is to create a detailed summary of this conversation. This summary will be placed at the start of a continuing session; newer messages that build on this context will follow after your summary (you do not see them here). Summarize thoroughly so that someone reading only your summary and then the newer messages can fully understand what happened and continue the work.

[DETAILED_ANALYSIS_INSTRUCTION_BASE]

Your summary should include the following sections:

1. Primary Request and Intent: Capture the user's explicit requests and intents in detail
2. Key Technical Concepts: List important technical concepts, technologies, and frameworks discussed.
3. Files and Code Sections: Enumerate specific files and code sections examined, modified, or created. Include full code snippets where applicable and include a summary of why this file read or edit is important.
4. Errors and fixes: List errors encountered and how they were fixed.
5. Problem Solving: Document problems solved and any ongoing troubleshooting efforts.
6. All user messages: List ALL user messages that are not tool results.
7. Pending Tasks: Outline any pending tasks.
8. Work Completed: Describe what was accomplished by the end of this portion.
9. Context for Continuing Work: Summarize any context, decisions, or state that would be needed to understand and continue the work in subsequent messages.
```

!!! note "中文附注"


    你的任务是创建对话的详细摘要。此摘要将被放置在继续会话的**开头**；在此上下文之后，建立在此基础上的**更新的消息**将随之而来（你在这里看不到它们）。彻底摘要，以便只阅读你的摘要然后阅读更新消息的人能够完全理解发生了什么，并继续工作。

    第 9 节（最大不同）：**继续工作的上下文**：总结理解和继续后续消息中的工作所需的任何上下文、决策或状态。

**设计解析**

`PARTIAL_COMPACT_UP_TO_PROMPT`（`direction: 'up_to'`）是三个变体中最特殊的，因为它的信息流方向与另外两个相反。

理解它需要先理解"up_to"压缩的使用场景。代码注释说：

```typescript
// 'up_to': model sees only the summarized prefix (cache hit). Summary will
// precede kept recent messages, hence "Context for Continuing Work" section.
```

在缓存命中时，模型只"看到"被摘要的前段对话（通过缓存读取），不需要重新处理这部分内容。"up_to"压缩就是对这个前段进行摘要。摘要完成后，它会被放在对话的开头，后面跟着**模型尚未处理的新消息**。

这就解释了为什么"up_to"变体的第 9 Section 从"Optional Next Step"变成了"Context for Continuing Work"：

**BASE / PARTIAL（from）的 Section 9**：
```
9. Optional Next Step: List the next step that you will take...
```
这里的"next step"是压缩后**续接模型要执行**的操作——摘要是为了让续接模型直接开始工作。

**PARTIAL（up_to）的 Section 9**：
```
9. Context for Continuing Work: Summarize any context, decisions, or state that would be
needed to understand and continue the work in subsequent messages.
```
这里没有"下一步"，因为"up_to"摘要后面跟着的是**已知的新消息**——续接模型会从这些新消息里直接知道下一步是什么。所以 Section 9 变成了"状态交接"而非"任务指令"：提供理解后续消息所需的上下文背景。

另一个差异：**Section 8 从 "Current Work" 变成了 "Work Completed"**。BASE/PARTIAL 的"Current Work"描述的是中断时刻"正在做什么"；"up_to"的"Work Completed"描述的是这个前段"完成了什么"——因为从后续消息的角度看，这个前段的所有工作都已经完成了。

---

## 6.7 `NO_TOOLS_TRAILER`：尾部重复约束

**英文原文**（来源：`src/services/compact/prompt.ts`，第 269–272 行）

```
REMINDER: Do NOT call any tools. Respond with plain text only — an <analysis> block followed by a <summary> block. Tool calls will be rejected and you will fail the task.
```

!!! note "中文附注"


    **提醒**：**不要**调用任何工具。仅用纯文字回应——一个 `<analysis>` 块，然后是一个 `<summary>` 块。工具调用将被拒绝，你将无法完成任务。

**设计解析**

TRAILER 是 PREAMBLE 的回响。完整的压缩 prompt 结构是：

```
NO_TOOLS_PREAMBLE（最前）
     ↓
BASE/PARTIAL/UP_TO_COMPACT_PROMPT（主体）
     ↓
[可选：自定义指令]
     ↓
NO_TOOLS_TRAILER（最后）
```

两处约束的表述有细微差异：

**PREAMBLE 用"CRITICAL"（严重警告）** + 详细列举工具名 + 解释全部后果：
```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
```

**TRAILER 用"REMINDER"（提醒）** + 仅重申核心约束：
```
REMINDER: Do NOT call any tools. Respond with plain text only...
Tool calls will be rejected and you will fail the task.
```

这种"强调前置，简洁收尾"的设计是合理的：
- PREAMBLE 需要强力，因为它在模型读取任何内容之前就设定了最重要的约束
- TRAILER 需要轻量，因为模型已经读完了主体内容，这里只是防止在生成响应时"忘记"约束，而不是重新建立约束

两者的共同点是：都明确了后果（"will fail the task"），不只是命令，也给出了理由。

---

## 6.8 `getCompactUserSummaryMessage()`：摘要注入消息

这个函数决定了压缩摘要如何被"重新注入"到对话上下文中。它不是压缩 prompt 的一部分，而是生成压缩后注入给模型的"你之前的对话被摘要了，以下是摘要"消息。

**英文原文（核心模式）**（来源：`src/services/compact/prompt.ts`，第 345–373 行）

```typescript
let baseSummary = `This session is being continued from a previous conversation that ran out of context. The summary below covers the earlier portion of the conversation.

${formattedSummary}`

if (transcriptPath) {
  baseSummary += `\n\nIf you need specific details from before compaction (like exact code snippets, error messages, or content you generated), read the full transcript at: ${transcriptPath}`
}

if (recentMessagesPreserved) {
  baseSummary += `\n\nRecent messages are preserved verbatim.`
}

if (suppressFollowUpQuestions) {
  let continuation = `${baseSummary}
Continue the conversation from where it left off without asking the user any further questions. Resume directly — do not acknowledge the summary, do not recap what was happening, do not preface with "I'll continue" or similar. Pick up the last task as if the break never happened.`

  if ((feature('PROACTIVE') || feature('KAIROS')) && proactiveModule?.isProactiveActive()) {
    continuation += `

You are running in autonomous/proactive mode. This is NOT a first wake-up — you were already working autonomously before compaction. Continue your work loop: pick up where you left off based on the summary above. Do not greet the user or ask what to work on.`
  }

  return continuation
}
```

**展开后的三种模式：**

**模式 A：基础摘要（`suppressFollowUpQuestions = false`）**

```
This session is being continued from a previous conversation that ran out of context.
The summary below covers the earlier portion of the conversation.

[摘要内容]

If you need specific details from before compaction (like exact code snippets, error messages,
or content you generated), read the full transcript at: [路径]
```

**模式 B：自动续接（`suppressFollowUpQuestions = true`）**

```
This session is being continued from a previous conversation that ran out of context.
The summary below covers the earlier portion of the conversation.

[摘要内容]

If you need specific details from before compaction...

Recent messages are preserved verbatim.

Continue the conversation from where it left off without asking the user any further questions.
Resume directly — do not acknowledge the summary, do not recap what was happening, do not
preface with "I'll continue" or similar. Pick up the last task as if the break never happened.
```

**模式 C：自主模式续接（自主模式且 `suppressFollowUpQuestions = true`）**

模式 B 的内容，后面追加：

```
You are running in autonomous/proactive mode. This is NOT a first wake-up — you were already
working autonomously before compaction. Continue your work loop: pick up where you left off
based on the summary above. Do not greet the user or ask what to work on.
```

!!! note "中文附注"


    **模式 A（基础）**：本会话从之前因上下文耗尽而结束的对话继续。以下摘要涵盖了对话的早期部分。如果需要压缩前的具体细节，请读取完整记录于：[路径]

    **模式 B（自动续接）**：在模式 A 基础上追加：不询问用户任何进一步问题，直接继续。不要确认摘要，不要重述正在发生的事情，不要以"我将继续"等开头。就像这个间断从未发生过一样接着最后的任务。

    **模式 C（自主模式续接）**：在模式 B 基础上追加：你正在自主/主动模式下运行。这**不是**首次唤醒——在压缩之前你已经在自主工作。继续你的工作循环：根据上面的摘要从你停下的地方继续。不要问候用户或询问要做什么。

**设计解析**

**Transcript 路径的设计** 解决了摘要不可避免的信息损失问题。任何摘要都会丢失一些细节，模型可能在续接时需要查阅具体的原始内容（如一段被修改的代码的完整版本）。提供 transcript 路径并告知模型可以通过 Read 工具访问，相当于给了模型一个"查阅原始记录"的逃生通道——当摘要里的信息不够用时，不必依赖幻觉，可以去查原始对话。

**"Recent messages are preserved verbatim"** 是局部压缩的信号。当近期消息被原样保留（不摘要）时，这条说明告诉模型：你下面看到的消息内容是完整的，不是被摘要过的。这防止了模型错误地把近期消息也视为"可能不完整的摘要"。

**"Resume directly — do not acknowledge the summary"** 是对一种模型自然行为的明确抑制：收到摘要后，模型倾向于先"承认"摘要（"好的，根据之前的摘要，我们在..."），这种承认是多余的——用户已经知道对话被压缩了，不需要模型再重述一遍。三个具体禁止（do not acknowledge / do not recap / do not preface）覆盖了承认行为的三种常见变体。

**自主模式的特殊追加** 是状态机设计的一个边界条件：自主模式下压缩后的模型可能误认为自己是"刚被唤醒的第一个 tick"，从而重新进入"问候用户+询问要做什么"的初始化流程（见第四章 `getProactiveSection()` 的分析）。"This is NOT a first wake-up"直接关闭了这个误判路径。

---

## 6.9 `formatCompactSummary()`：Scratchpad 剥离机制

**英文原文（代码注释）**（来源：`src/services/compact/prompt.ts`，第 305–335 行）

```typescript
/**
 * Formats the compact summary by stripping the <analysis> drafting scratchpad
 * and replacing <summary> XML tags with readable section headers.
 * @param summary The raw summary string potentially containing <analysis> and <summary> XML tags
 * @returns The formatted summary with analysis stripped and summary tags replaced by headers
 */
export function formatCompactSummary(summary: string): string {
  let formattedSummary = summary

  // Strip analysis section — it's a drafting scratchpad that improves summary
  // quality but has no informational value once the summary is written.
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/,
    '',
  )

  // Extract and format summary section
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  if (summaryMatch) {
    const content = summaryMatch[1] || ''
    formattedSummary = formattedSummary.replace(
      /<summary>[\s\S]*?<\/summary>/,
      `Summary:\n${content.trim()}`,
    )
  }

  // Clean up extra whitespace between sections
  formattedSummary = formattedSummary.replace(/\n\n+/g, '\n\n')

  return formattedSummary.trim()
}
```

**设计解析**

`formatCompactSummary()` 实现了"思维链剥离"的完整流程：

1. **删除 `<analysis>` 块**：正则 `/<analysis>[\s\S]*?<\/analysis>/` 匹配并删除整个分析过程（`[\s\S]*?` 是非贪婪匹配，防止错误删除多个分析块之间的内容）
2. **提取 `<summary>` 块**：把 `<summary>...</summary>` 替换为 `Summary:\n[内容]`——去除 XML 标签，改用自然语言标题
3. **清理多余空行**：连续两个以上的空行压缩为两个

整个处理流程的目的：模型的 `<analysis>` 块里可能有大量的中间推理（"让我先分析第 1 条消息，用户说了...然后...接下来..."），这些内容对最终摘要质量有贡献（帮助模型做更全面的思考），但注入到对话上下文里是纯粹的 token 浪费——它们对续接工作没有任何价值，还会占用宝贵的上下文空间。

通过 `<analysis>` 标签给模型提供"可以自由草稿"的空间，再通过 `formatCompactSummary()` 在注入前剥离这个草稿，实现了"思考时充分展开，存储时精简高效"的两全其美。

---

## 6.10 微压缩的工作原理

微压缩（MicroCompact）比全量压缩轻得多：它不生成语义摘要，只是把旧的工具调用结果内容替换为占位字符串。

### `TIME_BASED_MC_CLEARED_MESSAGE`

**英文原文**（来源：`src/services/compact/microCompact.ts`，第 36 行）

```typescript
export const TIME_BASED_MC_CLEARED_MESSAGE = '[Old tool result content cleared]'
```

!!! note "中文附注"

    [旧工具结果内容已清除]

这是出现在被清除工具结果位置的占位文字。第四章（4.13 节）分析的 `SUMMARIZE_TOOL_RESULTS_SECTION` 就是为了应对这个机制——提前告诉模型"工具结果可能被清除"，让模型主动在文字输出里记录重要信息。

**设计解析**

**COMPACTABLE_TOOLS 的边界** 定义了哪些工具的结果可以被微压缩：

```typescript
const COMPACTABLE_TOOLS = new Set<string>([
  FILE_READ_TOOL_NAME,    // Read
  ...SHELL_TOOL_NAMES,    // Bash, PowerShell
  GREP_TOOL_NAME,         // Grep
  GLOB_TOOL_NAME,         // Glob
  WEB_SEARCH_TOOL_NAME,   // WebSearch
  WEB_FETCH_TOOL_NAME,    // WebFetch
  FILE_EDIT_TOOL_NAME,    // Edit
  FILE_WRITE_TOOL_NAME,   // Write
])
```

注意**不在**这个集合里的工具：AgentTool、TaskTools、AskUserQuestion、ScheduleCronTool 等。这些工具的结果包含状态信息或交互结果，不能被随意清除。被清除的只是"可重现"的工具结果——如果需要，模型可以重新调用 Read 或 Grep 获取相同内容；但 Agent 的执行结果是一次性的，清除就永久丢失了。

### 时间触发型微压缩

**设计解析**（来源：`microCompact.ts`，第 411–444 行）

时间触发的逻辑：

```
距上次 assistant 消息的时间 > gapThresholdMinutes（配置值）
→ 清除所有旧的 COMPACTABLE 工具结果（保留最近 keepRecent 个）
→ 替换为 TIME_BASED_MC_CLEARED_MESSAGE
```

时间触发的背景来自 prompt cache 机制：API 的 prompt cache 在不活跃 5 分钟后过期。如果用户暂停使用超过阈值时间，下次发送消息时 cache 已经失效，整个前缀需要重写。这时，旧的工具结果内容必然要被重新发送到 API——这是浪费 token 的最糟糕时机（不是 cache_read，而是 cache_write 或 uncached）。

时间触发微压缩在 cache 失效之前（即用户发送下一条消息时）提前清除旧工具结果，减少需要重写的内容量。这是一个"知道 cache 将要失效，提前清理"的主动优化。

### 缓存编辑型微压缩（CACHED_MICROCOMPACT）

这是更精密的变体，通过 API 层的 `cache_edits` 机制，在**不修改本地消息内容**的情况下，通知服务器端删除特定工具结果。这样做的好处是：本地消息历史保持完整（便于用户查看完整对话），但服务器端的缓存 prefix 中不再包含这些被删除的内容。

缓存编辑型微压缩适用于 cache 仍然热的情况——直接清除内容会破坏缓存 key（content hash 变了），而 cache_edits 是服务器端的"软删除"，不影响缓存 key。

---

## 6.11 会话记忆压缩：第三层架构

会话记忆压缩（Session Memory Compact）是全量压缩的高效替代路径。

**核心思路**：在每次会话进行时，系统会持续提取并更新"会话记忆"——一个存储在 `.claude/session_memory.md` 里的结构化文档，记录当前工作的关键信息。当需要压缩时，不重新调用 LLM 生成摘要，而是直接读取这个已维护好的会话记忆文件，用它作为"摘要"注入到新上下文里。

**为什么这样更好**：全量压缩需要一次额外的 API 调用（用于生成摘要），这个调用会有延迟，且摘要质量依赖于单次 LLM 运行的表现。会话记忆是**增量维护**的——每轮对话都在更新，从多个时间点积累信息，而非在压缩时一次性生成。

**配置参数**（来源：`sessionMemoryCompact.ts`，第 57–61 行）：

```typescript
export const DEFAULT_SM_COMPACT_CONFIG: SessionMemoryCompactConfig = {
  minTokens: 10_000,             // 最少保留的 token 数
  minTextBlockMessages: 5,        // 最少保留的有文字块的消息数
  maxTokens: 40_000,             // 最多保留的 token 数上限
}
```

**保留哪些消息** 的决策逻辑：
1. 从上次摘要边界之后的消息开始
2. 若保留的消息不足 10,000 token 或不足 5 条有文字的消息，则向前扩展（保留更多近期消息）
3. 若保留的消息超过 40,000 token，则停止扩展（即使还没满足最小条件）
4. 调整以不切断 tool_use/tool_result 配对（`adjustIndexToPreserveAPIInvariants()`）

---

## 6.12 压缩 Prompt 设计总结

将本章分析的所有设计决策汇总，可以提炼出以下规律：

**结构设计**

| 组件 | 位置 | 功能 | 设计原则 |
|---|---|---|---|
| NO_TOOLS_PREAMBLE | 最前 | 禁止工具调用 | 首位约束 + 后果说明 |
| BASE/PARTIAL/UP_TO prompt | 主体 | 引导摘要生成 | 结构化输出 + 具体字段 |
| 自定义指令 | 主体末 | 用户定制重点 | 追加不覆盖 |
| NO_TOOLS_TRAILER | 最后 | 重申约束 | 轻量提醒 |

**摘要的信息层次**

9 个 Section 不是随机的——它们从背景到状态到续接，形成一个完整的信息漏斗。特别关键的设计点：
- Section 6（All user messages）：保留全部而非摘要，防止解释性偏差
- Section 9（Optional Next Step）：verbatim 引用防止任务漂移

**三个变体的核心差异**

| 变体 | 方向 | 范围 | Section 9 |
|---|---|---|---|
| BASE | - | 全部对话 | Optional Next Step |
| PARTIAL (from) | → | 仅最近消息 | Optional Next Step |
| PARTIAL (up_to) | ← | 前段对话 | Context for Continuing Work |

**思维链剥离**

`<analysis>` 标签的设计将"思考过程"和"最终摘要"明确分离。前者提升质量，后者注入上下文。`formatCompactSummary()` 在注入之前完成分离，既得到了 CoT 的质量收益，又避免了 CoT 的 token 代价。

**双重封锁的工程理由**

PREAMBLE + TRAILER 的双重约束不是强迫症的产物——2.79% vs 0.01% 的工具调用率差异（Sonnet 4.6 vs 4.5）和 `maxTurns: 1` 的硬约束，使得工具调用在这个场景下的代价是"任务失败"。双重约束把这个失败率压到可接受范围内。

---

> **下一章**：从具体的 prompt 分析回到设计原则的抽象。我们将把前六章中反复出现的设计模式归纳成十条可迁移的原则，每条原则都会指向它在 Claude Code 源码中的具体体现，以及如何在你自己的 LLM 应用中应用它。
