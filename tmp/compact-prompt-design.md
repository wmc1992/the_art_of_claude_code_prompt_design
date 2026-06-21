# Compact 压缩摘要的 Prompt 设计深度分析

---

## 一、整体架构：三层压缩，各有分工

首先要理解，compact 不是一个单一操作，而是三层递进策略：

```
微压缩 (microcompact)          ← 无 LLM，直接清除旧工具结果内容
  ↓ 不够用时
全量压缩 (compact)             ← LLM 生成对话摘要
  ↓ session memory 实验功能
会话记忆压缩 (SM compact)      ← 用持续维护的记忆文件，免去 LLM 调用
```

**微压缩**：把旧的工具调用结果替换为 `'[Old tool result content cleared]'` 占位字符串，无需 LLM，只是内容清除。触发条件包括计数阈值和时间阈值（用户离开超过 N 分钟，cache 已过期，提前清除是合理的）。

**全量压缩**：才是真正的 prompt 工程问题，也是本文重点。

**关键文件**：
- `src/services/compact/prompt.ts` — 压缩 prompt 的所有文本
- `src/services/compact/compact.ts` — 压缩流程编排
- `src/services/compact/microCompact.ts` — 微压缩逻辑
- `src/services/compact/sessionMemoryCompact.ts` — 会话记忆压缩

---

## 二、全量压缩 Prompt 的核心设计

### 2.1 双重封锁关键约束

`prompt.ts` 里最醒目的设计是：工具禁用指令出现了**两次**——开头一次，结尾一次：

```typescript
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
`

const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'
```

注释里解释了为什么要这样做：

> *"Aggressive no-tools preamble. The cache-sharing fork path inherits the parent's full tool set (required for cache-key match), and on Sonnet 4.6+ adaptive-thinking models the model sometimes attempts a tool call despite the weaker trailer instruction. With maxTurns: 1, a denied tool call means no text output → falls through to the streaming fallback (2.79% on 4.6 vs 0.01% on 4.5)."*

这是一个精确的失败模式打补丁：压缩任务 fork 出来的子 agent 继承了父 agent 的全部工具集（为了 prompt cache key 匹配），而新模型（Sonnet 4.6）有自适应思考能力，会在思考过程中"想到"要调用工具，即使 prompt 已经说了不要。问题是 `maxTurns: 1`，一旦工具调用被拒，那一轮就没有文字输出，整个摘要失败。

**2.79% 的失败率 vs 0.01% 的对比**，说明这是经过精确测量后的针对性修复。

### 2.2 Scratchpad 模式：用后即弃的思考空间

压缩 prompt 要求模型先写 `<analysis>` 再写 `<summary>`，但最终只保留 summary：

```typescript
// 摘要生成后，analysis 被代码自动剥离
export function formatCompactSummary(summary: string): string {
  // Strip analysis section — it's a drafting scratchpad that improves summary
  // quality but has no informational value once the summary is written.
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/, ''
  )
  // ...
}
```

Analysis 的作用是**强制模型在写最终摘要前做一遍系统性回顾**：

```
1. Chronologically analyze each message and section of the conversation.
   For each section thoroughly identify:
   - The user's explicit requests and intents
   - Your approach to addressing the user's requests
   - Key decisions, technical concepts and code patterns
   - Specific details like: file names, full code snippets, function signatures, file edits
   - Errors that you ran into and how you fixed them
   - Pay special attention to specific user feedback...
2. Double-check for technical accuracy and completeness
```

这是 chain-of-thought 的一种工程化用法：**质量收益保留，token 成本在落地时回收**。Analysis 帮模型整理了思路，但对后续 context 没有贡献，所以可以安全丢弃。

### 2.3 摘要的 9 个 Section 设计

```
1. Primary Request and Intent    ← 用户的核心意图
2. Key Technical Concepts        ← 技术关键词列表
3. Files and Code Sections       ← 文件 + 完整代码片段
4. Errors and fixes              ← 错误和修复过程
5. Problem Solving               ← 已解决的问题
6. All user messages             ← 所有用户消息（非工具结果）
7. Pending Tasks                 ← 待办任务
8. Current Work                  ← 压缩前正在做的事
9. Optional Next Step            ← 下一步（带原文引用）
```

每个 section 的选取都有其逻辑：

**Section 6（All user messages）** 是最特别的：

> *"List ALL user messages that are not tool results. These are critical for understanding the users' feedback and changing intent."*

用户的反馈、纠正、意图转变，是最高价值的信息，也是最容易在摘要中被"概括掉"的。专门要求保留用户原话，是防止信息失真的关键设计。

**Section 9（Optional Next Step）** 有一个反直觉的设计：

> *"Include direct quotes from the most recent conversation showing exactly what task you were working on and where you left off. This should be verbatim to ensure there's no drift in task interpretation."*

要求逐字引用，而不是"概括"。这是**反漂移设计**——摘要被当作 prompt 传回模型时，模型对任务的理解必须与原始对话完全一致，任何重新措辞都可能引入语义偏移。

### 2.4 三种 Prompt 变体，对应不同压缩场景

```typescript
BASE_COMPACT_PROMPT          // 压缩整段对话
PARTIAL_COMPACT_PROMPT       // 压缩最近部分（前面保留）
PARTIAL_COMPACT_UP_TO_PROMPT // 压缩到某个点（后面还有新消息会接续）
```

第三个变体最有意思，它的 Section 8-9 设计不同：

- BASE / PARTIAL：`8. Current Work` + `9. Optional Next Step`
- UP_TO：`8. Work Completed` + `9. Context for Continuing Work`

因为 UP_TO 的摘要会被放在对话**开头**，后面还有真实的新消息接续。所以不需要"下一步要做什么"，而是需要"后续消息需要哪些上下文才能理解"。Prompt 的意图和摘要的使用场景是严格对齐的。

---

## 三、摘要如何重新注入到对话

压缩完成后，摘要通过 `getCompactUserSummaryMessage()` 被包装成一条 user 消息重新注入：

```typescript
// 普通模式（用户手动触发）
`This session is being continued from a previous conversation that ran out of context.
The summary below covers the earlier portion of the conversation.

${formattedSummary}`

// 无需追问模式（自动触发）
`${baseSummary}
Continue the conversation from where it left off without asking the user any further questions.
Resume directly — do not acknowledge the summary, do not recap what was happening,
do not preface with "I'll continue" or similar.
Pick up the last task as if the break never happened.`

// 自主模式的额外指令
`You are running in autonomous/proactive mode. This is NOT a first wake-up —
you were already working autonomously before compaction.
Continue your work loop: pick up where you left off based on the summary above.
Do not greet the user or ask what to work on.`
```

这里有三个精细的反行为设计：

1. **"do not acknowledge the summary"** — 防止模型说"好的，根据摘要，我之前在做..."
2. **"do not recap what was happening"** — 防止模型重新描述已知状态
3. **"This is NOT a first wake-up"** — 防止自主模式在压缩后错误地重置为初始状态（greeting 用户、询问要做什么）

---

## 四、自定义压缩指令的扩展点

Compact prompt 末尾预留了用户自定义指令的注入口：

```typescript
export function getCompactPrompt(customInstructions?: string): string {
  let prompt = NO_TOOLS_PREAMBLE + BASE_COMPACT_PROMPT
  if (customInstructions && customInstructions.trim() !== '') {
    prompt += `\n\nAdditional Instructions:\n${customInstructions}`
  }
  prompt += NO_TOOLS_TRAILER
  return prompt
}
```

Prompt 里甚至给出了示例，引导用户写出有效的自定义指令：

```
<example>
## Compact Instructions
When summarizing the conversation focus on typescript code changes
and also remember the mistakes you made and how you fixed them.
</example>

<example>
# Summary instructions
When you are using compact - please focus on test output and code changes.
Include file reads verbatim.
</example>
```

这是**元 prompt 设计**：用 prompt 教用户怎么写 prompt。

---

## 五、设计原则总结

从 compact 的 prompt 设计中可以提炼出以下通用原则：

| 原则 | Compact 中的体现 |
|---|---|
| **关键约束双重封锁** | NO_TOOLS 出现在头部和尾部，因为模型在长输出中会遗忘头部约束 |
| **用后即弃的思考空间** | `<analysis>` 提升质量，`formatCompactSummary()` 丢弃它，不浪费后续 token |
| **高价值内容显式点名** | "All user messages"、"specific user feedback" 反复强调，防止被概括掉 |
| **反漂移的逐字引用** | Next Step 要求原文引用，防止摘要重写引入语义偏移 |
| **prompt 意图与使用场景对齐** | 三种变体的 section 设计各自对应摘要被放置的位置 |
| **反行为约束** | 注入摘要时，明确禁止模型"致谢摘要"、"重述状态"等无效行为 |
| **上下文感知的状态恢复** | 自主模式有额外指令防止压缩后错误重置 |
| **扩展点 + 示例引导** | 自定义指令用 example 教用户写法，而非让用户自己摸索格式 |
