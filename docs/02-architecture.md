# 第二章：Prompt 的组织架构

如果你曾经写过一个 LLM 应用，大概率是这样组织 system prompt 的：把所有指令拼成一个字符串，存在某个变量里，每次调用 API 时传过去。这个做法在小型应用里完全够用，但在 Claude Code 这个量级的系统里，它有三个致命问题：

**第一，难以维护。** 当一个字符串同时承担角色定义、安全约束、工具指南、输出格式等十几个职责时，修改任何一处都需要在整段文字里找位置，极易破坏其他部分的逻辑。

**第二，无法按需组合。** 不同的运行模式需要不同的 prompt——自主模式需要"偏向行动"，交互模式需要"谨慎确认"；内部用户需要严格的工程规范，外部用户需要友好的默认行为。一段硬编码的字符串无法灵活应对这些差异。

**第三，缓存效率极低。** Anthropic API 支持 prompt cache，可以将 system prompt 的 token 成本大幅降低。但缓存的前提是 prompt 前缀不变——如果每次调用都因为某个动态值（比如当前时间或工作目录）而改变整段 prompt，缓存就完全失效了。

Claude Code 的解决方案是把 system prompt 设计成一个**分段组装系统**，每个 segment 独立管理自己的内容、缓存行为和可见范围。本章将详细介绍这个系统的设计。

---

## 2.1 Section：职责隔离的基本单元

Claude Code 的 system prompt 由若干**Section**（节）组成。每个 Section 是一个相互独立的字符串片段，只负责一类行为约束。它们在运行时被组装成最终的 system prompt 数组，传给 Anthropic API。

这一机制的核心定义在 `src/constants/systemPromptSections.ts`：

```typescript
type ComputeFn = () => string | null | Promise<string | null>

type SystemPromptSection = {
  name: string
  compute: ComputeFn
  cacheBreak: boolean
}
```

每个 Section 有三个属性：

- **`name`**：Section 的唯一标识符，用于在缓存中查找和存储对应的内容
- **`compute`**：一个函数，调用时返回这个 Section 的实际文字内容（或 `null`，表示该 Section 在当前配置下不适用）
- **`cacheBreak`**：布尔值，控制这个 Section 的缓存行为

`compute` 返回 `null` 的设计很关键。它意味着 Section 可以根据运行时条件决定自己是否出现——比如语言偏好 Section 只在用户设置了语言时才出现，MCP 指令 Section 只在有连接的 MCP 服务器时才出现。**Section 的"不存在"是一种有意义的状态，而不是错误。**

---

## 2.2 两种 Section：缓存与易失

创建 Section 有两个工厂函数，设计上刻意形成对比：

**`systemPromptSection`**（缓存版）

```typescript
/**
 * Create a memoized system prompt section.
 * Computed once, cached until /clear or /compact.
 */
export function systemPromptSection(
  name: string,
  compute: ComputeFn,
): SystemPromptSection {
  return { name, compute, cacheBreak: false }
}
```

> **中文附注**：创建一个记忆化的 system prompt 节。计算一次后缓存，直到 `/clear` 或 `/compact` 命令被执行时才重新计算。

**`DANGEROUS_uncachedSystemPromptSection`**（易失版）

```typescript
/**
 * Create a volatile system prompt section that recomputes every turn.
 * This WILL break the prompt cache when the value changes.
 * Requires a reason explaining why cache-breaking is necessary.
 */
export function DANGEROUS_uncachedSystemPromptSection(
  name: string,
  compute: ComputeFn,
  _reason: string,
): SystemPromptSection {
  return { name, compute, cacheBreak: true }
}
```

> **中文附注**：创建一个每轮都重新计算的易失节。当值发生变化时，**这将会打破 prompt cache**。要求提供一个理由字符串，解释为什么必须打破缓存。

注意两个设计细节：

**第一，函数名使用全大写 `DANGEROUS`。** 这不是随意的命名风格，而是一个刻意的警告信号——调用这个函数的开发者必须意识到，他们正在做一个有性能代价的决定。在一个有数十个 Section 的系统里，如果开发者不假思索地把所有 Section 都设为易失，就会导致 prompt cache 完全失效，每次 API 调用都要重新处理全部 token。

**第二，`_reason` 参数在函数体内未被使用（下划线前缀表示有意忽略）。** 它存在的唯一目的是强制调用者写出理由。这是一种代码层面的文档约束——你不是在告诉运行时为什么，而是在告诉下一个阅读这段代码的人为什么。比如：

```typescript
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',  // ← 强制写出的理由
)
```

这段代码来自 `src/constants/prompts.ts`，对应的注释进一步解释了设计背景：

```
// When delta enabled, instructions are announced via persisted
// mcp_instructions_delta attachments (attachments.ts) instead of this
// per-turn recompute, which busts the prompt cache on late MCP connect.
// Gate check inside compute (not selecting between section variants)
// so a mid-session gate flip doesn't read a stale cached value.
```

> **中文附注**：当 delta 功能启用时，指令通过持久化的 `mcp_instructions_delta` 附件（attachments.ts）来传达，而不是用这个每轮重算的方式——因为后者会在 MCP 服务器延迟连接时打破 prompt cache。检查 gate 条件放在 compute 函数内部（而非在两个 section 变体之间选择），是为了防止在会话中途的 gate 切换读到过期的缓存值。

这段注释展示了 Anthropic 工程师的思考层次：他们不只是"MCP 在运行时可能变化，所以用易失 Section"，而是精确地分析了在什么条件下易失是必要的（MCP 服务器的晚期连接），并且在更优解可用时（`delta` 机制）转向它。这种对缓存成本的精细核算贯穿了整个 prompt 工程的实现。

---

## 2.3 Section 的解析：缓存优先

所有 Section 的实际内容解析由 `resolveSystemPromptSections` 完成：

```typescript
/**
 * Resolve all system prompt sections, returning prompt strings.
 */
export async function resolveSystemPromptSections(
  sections: SystemPromptSection[],
): Promise<(string | null)[]> {
  const cache = getSystemPromptSectionCache()

  return Promise.all(
    sections.map(async s => {
      if (!s.cacheBreak && cache.has(s.name)) {
        return cache.get(s.name) ?? null
      }
      const value = await s.compute()
      setSystemPromptSectionCacheEntry(s.name, value)
      return value
    }),
  )
}
```

> **中文附注**：解析所有 system prompt 节，返回 prompt 字符串数组。对每个 section，如果它不是易失的（`!s.cacheBreak`）且缓存中有值，则直接返回缓存；否则调用 `compute()` 函数重新计算，并将结果存入缓存。

这段代码很简洁，但逻辑层次清晰：

1. 获取当前的 Section 缓存（存储在全局 state 中，随 `/clear` 或 `/compact` 重置）
2. 对每个 Section 并行处理（`Promise.all`）
3. 缓存命中 → 直接返回；缓存未命中或 `cacheBreak: true` → 重新计算，存入缓存

`Promise.all` 的使用意味着所有需要重新计算的 Section 是**并发**执行的，而不是顺序等待。在一次对话的早期，可能有多个 Section 需要同时从文件系统或配置中读取数据，并发执行可以显著降低准备 prompt 的延迟。

---

## 2.4 静态与动态的边界

在 Section 机制之上，还有一个更粗粒度的划分：**静态内容**和**动态内容**的边界。

这个边界由一个特殊的字符串常量标记：

```typescript
/**
 * Boundary marker separating static (cross-org cacheable) content from dynamic content.
 * Everything BEFORE this marker in the system prompt array can use scope: 'global'.
 * Everything AFTER contains user/session-specific content and should not be cached.
 *
 * WARNING: Do not remove or reorder this marker without updating cache logic in:
 * - src/utils/api.ts (splitSysPromptPrefix)
 * - src/services/api/claude.ts (buildSystemPromptBlocks)
 */
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

> **中文附注**：分隔静态内容（可跨组织缓存）和动态内容的边界标记。此标记**之前**的所有内容可以使用 `scope: 'global'`（全局缓存）。此标记**之后**的内容包含用户/会话特定信息，不应被缓存。**警告**：在不更新 `src/utils/api.ts` 和 `src/services/api/claude.ts` 中相关缓存逻辑的情况下，不得移除或重新排序此标记。

这个注释的最后一行是工程纪律的体现：这个字符串常量不只是一个占位符，它在运行时被代码逻辑识别和处理，牵连着两个文件中的缓存构建逻辑。移动它不是在改文字，而是在改系统行为。

**静态内容**（边界之前）是那些在同一版本的 Claude Code 里几乎不会变化的指令：角色定义、工具使用规范、代码风格要求、安全约束。这些内容可以使用 `scope: 'global'`，意味着它们的缓存可以跨不同用户、不同组织共享——只要使用相同版本的 Claude Code，任何人第一次调用时都会计算，之后所有人都能命中缓存。

**动态内容**（边界之后）是那些随用户或会话变化的信息：当前工作目录、操作系统版本、用户的语言偏好、记忆内容、MCP 服务器指令。这些内容只能做会话级缓存，不能跨用户共享。

这种分层设计的意义在于：Anthropic 可以通过精细控制哪些内容进入"全局缓存区域"，来最大化缓存命中率。理论上，所有使用相同版本 Claude Code 的用户，在发起第一次对话后，其后续对话的静态 prompt 部分都可以从缓存中读取，而不需要重新计算和传输这些 token。

---

## 2.5 最终组装：`getSystemPrompt` 的返回结构

把以上机制放在一起，`getSystemPrompt` 函数的最终返回结构如下（来自 `src/constants/prompts.ts`）：

```typescript
return [
  // --- Static content (cacheable) ---
  getSimpleIntroSection(outputStyleConfig),
  getSimpleSystemSection(),
  outputStyleConfig === null ||
  outputStyleConfig.keepCodingInstructions === true
    ? getSimpleDoingTasksSection()
    : null,
  getActionsSection(),
  getUsingYourToolsSection(enabledTools),
  getSimpleToneAndStyleSection(),
  getOutputEfficiencySection(),
  // === BOUNDARY MARKER - DO NOT MOVE OR REMOVE ===
  ...(shouldUseGlobalCacheScope() ? [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] : []),
  // --- Dynamic content (registry-managed) ---
  ...resolvedDynamicSections,
].filter(s => s !== null)
```

> **中文附注**：最终返回值是一个字符串数组，由静态内容和动态内容两部分组成，以边界标记分隔。`null` 值在最后被过滤掉，因此不适用的 Section 自动从最终 prompt 中消失。

几个值得注意的细节：

**第一，`getSimpleDoingTasksSection()` 的条件判断**。当用户设置了自定义输出风格（`outputStyleConfig !== null`），并且该风格的 `keepCodingInstructions` 不为 `true` 时，整个编码规范 Section 都会被省略。这说明 prompt 的某些组成部分是可以被用户自定义的输出风格完全替换掉的。

**第二，`shouldUseGlobalCacheScope()` 的条件判断**。边界标记并非在所有情况下都插入——它只在全局缓存 scope 被支持时才出现。这意味着在某些配置（比如不支持全局缓存的部署环境）下，整个静态/动态分层机制会退化为更简单的形式。

**第三，`.filter(s => s !== null)`**。这是整个组装系统的"清洁"步骤，把所有不适用的 Section（返回 `null` 的）从最终数组中移除。结果是一个纯净的字符串数组，每个元素都是真实的 prompt 文本。

---

## 2.6 两种路径：普通模式与自主模式

值得特别说明的是，上述组装流程只是 `getSystemPrompt` 函数的**普通模式路径**。在自主模式（Proactive/Kairos）激活时，函数会走一条完全不同的路径，直接返回一套精简的 prompt：

```typescript
if (
  (feature('PROACTIVE') || feature('KAIROS')) &&
  proactiveModule?.isProactiveActive()
) {
  return [
    `\nYou are an autonomous agent. Use the available tools to do useful work.\n\n${CYBER_RISK_INSTRUCTION}`,
    getSystemRemindersSection(),
    await loadMemoryPrompt(),
    envInfo,
    getLanguageSection(settings.language),
    getMcpInstructionsSection(mcpClients),
    getScratchpadInstructions(),
    getFunctionResultClearingSection(model),
    SUMMARIZE_TOOL_RESULTS_SECTION,
    getProactiveSection(),
  ].filter(s => s !== null)
}
```

> **中文附注**：当自主模式（PROACTIVE 或 KAIROS feature flag）激活时，系统 prompt 走完全不同的路径，返回一套精简版的自主模式专用 prompt，角色定义简化为"你是一个自主 Agent，使用可用工具做有用的工作"，并附加自主模式特有的行为指南节（`getProactiveSection()`）。

这一设计说明：**Claude Code 不是"同一个 AI 在不同场景下"，而是在不同配置下使用完全不同的 prompt 体系**。普通模式的 prompt 强调谨慎、确认、不越权；自主模式的 prompt 强调行动、主动探索、减少停顿。这两套 prompt 体系从角色定义就开始分叉，不存在共用的"基础版本"。

---

## 2.7 工具级 Prompt 的双层结构

在 system prompt 之外，Claude Code 还有另一个维度的 prompt：**工具级 prompt**。每个工具有两层 prompt 内容，服务于不同的用途。

### 第一层：`description` 字段

`description` 是传入 Anthropic API 的工具描述，模型在决定调用哪个工具时依赖它。它需要简洁，通常在一到几行之间，告诉模型这个工具是干什么的、何时应该使用。

以 GlobTool 为例（`src/tools/GlobTool/prompt.ts`）：

```typescript
export const DESCRIPTION = `- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead`
```

> **中文附注**：快速文件模式匹配工具，适用于任何规模的代码库。支持 glob 模式，如 `**/*.js` 或 `src/**/*.ts`。返回按修改时间排序的匹配文件路径。当需要通过文件名模式查找文件时使用此工具。当进行需要多轮 glob 和 grep 的开放式搜索时，改用 Agent 工具。

GrepTool 的 description 更复杂，还包含了使用约束（`src/tools/GrepTool/prompt.ts`）：

```typescript
export function getDescription(): string {
  return `A powerful search tool built on ripgrep

  Usage:
  - ALWAYS use ${GREP_TOOL_NAME} for search tasks. NEVER invoke \`grep\` or \`rg\` as a ${BASH_TOOL_NAME} command. The ${GREP_TOOL_NAME} tool has been optimized for correct permissions and access.
  - Supports full regex syntax (e.g., "log.*Error", "function\\s+\\w+")
  - Filter files with glob parameter (e.g., "*.js", "**/*.tsx") or type parameter (e.g., "js", "py", "rust")
  - Output modes: "content" shows matching lines, "files_with_matches" shows only file paths (default), "count" shows match counts
  - Use ${AGENT_TOOL_NAME} tool for open-ended searches requiring multiple rounds
  - Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping (use \`interface\\{\\}\` to find \`interface{}\` in Go code)
  - Multiline matching: By default patterns match within single lines only. For cross-line patterns like \`struct \\{[\\s\\S]*?field\`, use \`multiline: true\`
`
}
```

> **中文附注**：基于 ripgrep 的强大搜索工具。使用说明：搜索任务**始终**使用 Grep 工具，**绝不**通过 Bash 命令调用 `grep` 或 `rg`，Grep 工具已针对正确的权限和访问做了优化。支持完整正则语法。可通过 glob 或文件类型过滤。多种输出模式：`content` 显示匹配行，`files_with_matches` 只显示文件路径（默认），`count` 显示匹配数量。开放式多轮搜索使用 Agent 工具。模式语法使用 ripgrep 规则（非 grep）——字面大括号需要转义。多行匹配默认仅在单行内匹配，跨行模式需设置 `multiline: true`。

注意 GrepTool description 的写法：它不仅说"我是什么"，还明确说"什么情况下**不该**用我（转而用 Agent 工具）"以及"ALWAYS...NEVER"这种强约束。这是 description 里就在做行为约束，而不是把所有约束都推到 system prompt 里。

### 第二层：详细使用说明（`prompt.ts`）

Description 只是工具的"名片"。真正详细的行为规范存放在各工具目录下的 `prompt.ts` 文件中，这些内容通过 system prompt 或工具使用上下文传递给模型，告诉它工具的参数语义、边界情况、禁止操作、步骤顺序等。

以 FileReadTool 为例，其 `renderPromptTemplate` 函数会生成这样的内容：

```
Reads a file from the local filesystem. You can access any file directly by using this tool.
Assume this tool is able to read all files on the machine. If the User provides a path to a
file assume that path is valid. It is okay to read a file that does not exist; an error will
be returned.

Usage:
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to 2000 lines starting from the beginning of the file
...
```

BashTool 的 `prompt.ts` 则长达 369 行，包含 Git 安全协议、沙箱约束、commit 步骤化流程、PR 创建指引等完整规范。这是"工具使用手册"维度最丰富的一个，将在第五章作为第一档工具的第一个案例详细分析。

**Description 与使用说明的关系可以这样理解**：Description 决定模型是否选择这个工具，使用说明决定模型如何正确使用这个工具。两者服务于不同的决策时刻，缺一不可。

---

## 2.8 整体结构图

把本章的所有层次放在一起，Claude Code 的 prompt 架构如下：

```
┌──────────────────────────────────────────────────────────────────┐
│                      最终 API 请求                                │
│                                                                  │
│  system: [                                                       │
│    ┌────────────────────────────────────────────────────────┐   │
│    │              静态 Sections（可全局缓存）                 │   │
│    │                                                        │   │
│    │  · getSimpleIntroSection()     角色定义                 │   │
│    │  · getSimpleSystemSection()    系统规则                 │   │
│    │  · getSimpleDoingTasksSection() 任务执行规范             │   │
│    │  · getActionsSection()         高危操作决策             │   │
│    │  · getUsingYourToolsSection()  工具选择逻辑             │   │
│    │  · getSimpleToneAndStyleSection() 输出风格              │   │
│    │  · getOutputEfficiencySection() 输出效率                │   │
│    └────────────────────────────────────────────────────────┘   │
│                    SYSTEM_PROMPT_DYNAMIC_BOUNDARY                │
│    ┌────────────────────────────────────────────────────────┐   │
│    │           动态 Sections（会话级缓存或每轮重算）           │   │
│    │                                                        │   │
│    │  · session_guidance    会话级动态指导（缓存）            │   │
│    │  · memory              用户记忆（缓存）                  │   │
│    │  · env_info_simple     环境信息（缓存）                  │   │
│    │  · language            语言偏好（缓存）                  │   │
│    │  · output_style        输出风格配置（缓存）              │   │
│    │  · mcp_instructions    MCP 指令（每轮重算）◀─DANGEROUS  │   │
│    │  · scratchpad          草稿空间指令（缓存）              │   │
│    │  · frc                 结果清理提示（缓存）              │   │
│    └────────────────────────────────────────────────────────┘   │
│  ]                                                               │
│                                                                  │
│  tools: [                                                        │
│    { name: "Bash",     description: "...", input_schema: {...} } │
│    { name: "Read",     description: "...", input_schema: {...} } │
│    { name: "Edit",     description: "...", input_schema: {...} } │
│    ... （43个工具，每个都有自己的description）                     │
│  ]                                                               │
│                                                                  │
│  messages: [ 对话历史 ]                                           │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2.9 小结

本章揭示了 Claude Code prompt 的三层架构：

**第一层：System Prompt 的 Section 机制。** 通过 `systemPromptSection` 和 `DANGEROUS_uncachedSystemPromptSection` 两个工厂函数，将 system prompt 分解为独立的、可缓存的、按需激活的单元。函数名中的 `DANGEROUS` 是设计者对调用者施加的工程约束，而非单纯的命名风格。

**第二层：静态/动态边界。** `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 将 prompt 分为可全局缓存的稳定内容和必须动态计算的用户特定内容，让缓存机制能够最大化复用率。

**第三层：工具级双层 Prompt。** 每个工具有简短的 `description`（决定模型是否调用它）和详细的使用说明（决定模型如何调用它）。两者服务于不同的决策时刻，设计原则也不同。

理解了这个架构，后续章节对具体 prompt 内容的分析就有了结构上的坐标系。第四章开始的所有 Section 分析，都将对应到上面图中的某个具体位置。

---

> **下一章**：我们将深入 prompt 的工程机制，分析 Prompt Cache 的成本模型、Feature Flag 如何实现死代码消除、用户分层如何产生差异化行为，以及启动时并行预取如何降低首次响应延迟。
