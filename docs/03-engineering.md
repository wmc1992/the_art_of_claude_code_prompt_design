# 第三章：Prompt 的工程机制

第二章介绍了 prompt 的组织架构——它是由什么构成的。本章转向另一个问题：**这套系统是如何被构建和运行的**。

这里涉及的工程机制，大多数是为了解决同一个核心矛盾：prompt 的内容越丰富，越能引导模型做出正确行为；但 prompt 的体积越大，每次 API 调用的成本越高、延迟越长。Claude Code 通过几个独立的机制来化解这个矛盾，本章逐一解析。

---

## 3.1 Prompt Cache 与成本模型

理解本章其他所有机制的前提，是理解 **Prompt Cache** 的工作原理和成本结构。

### Prompt Cache 的基本原理

Anthropic API 支持一种名为 **prompt caching** 的特性：如果两次 API 调用的 system prompt 前缀完全相同，API 会复用之前计算过的 KV（Key-Value）缓存，而不是重新处理这些 token。被缓存命中的 token 称为 **cache read token**，其计费通常只有普通 input token 的 10%；而第一次建立缓存时，需要支付稍高的 **cache write token** 费用（通常是普通 input token 的 125%）。

对于一个长期运行的对话场景，这个机制可以将 system prompt 部分的成本降低 90%。Claude Code 的 system prompt 展开后约 3,000–4,000 token，在开发者每天高频使用的场景下，这个数字乘以调用次数会变成相当可观的开销。

### 缓存失效的代价

缓存的前提是 prompt 前缀不变。一旦 system prompt 的任何部分发生改变（即使只是末尾追加了一行），整个缓存就会失效，下一次调用需要重新建立。这就是为什么 Claude Code 要如此精细地区分哪些内容是稳定的（可以安全地放在缓存区域内），哪些内容是会变化的（必须放在缓存区域之后）。

**一次不必要的缓存失效，代价是：**
- 重新处理数千个 token（原本可以命中缓存）
- 触发一次较贵的 cache write 计费（建立新缓存）
- 下一轮对话才能重新命中缓存

在高频使用场景下，这个代价会被快速放大。

---

## 3.2 全局缓存 Scope：跨用户共享的静态内容

Claude Code 在标准 prompt cache 之上，实现了一个更激进的优化：**全局缓存 scope（`scope: 'global'`）**。

在标准的 prompt cache 中，缓存是按用户（或 API key）隔离的——用户 A 的缓存对用户 B 不可见。而 `scope: 'global'` 是 Anthropic API 的一个特殊选项，允许将 system prompt 的某一部分设为跨用户、跨组织共享的全局缓存。只要两个用户发送了完全相同的内容，他们可以共用同一份 KV 缓存。

这对 Claude Code 来说意义重大：使用相同版本 Claude Code 的所有用户，其 system prompt 的静态部分（角色定义、工具规范、安全约束等）是完全相同的。理论上，全球数十万开发者，只需要有第一个人触发了这份 prompt 的缓存写入，其他所有人都能命中全局缓存，直接以极低成本获得这部分 token 的处理结果。

这个机制的实现位于 `src/utils/api.ts` 的 `splitSysPromptPrefix` 函数：

```typescript
export function splitSysPromptPrefix(
  systemPrompt: SystemPrompt,
  options?: { skipGlobalCacheForSystemPrompt?: boolean },
): SystemPromptBlock[] {
  const useGlobalCacheFeature = shouldUseGlobalCacheScope()
  // ...

  if (useGlobalCacheFeature) {
    const boundaryIndex = systemPrompt.findIndex(
      s => s === SYSTEM_PROMPT_DYNAMIC_BOUNDARY,
    )
    if (boundaryIndex !== -1) {
      // ...
      const staticJoined = staticBlocks.join('\n\n')
      if (staticJoined)
        result.push({ text: staticJoined, cacheScope: 'global' })  // ← 全局 scope
      const dynamicJoined = dynamicBlocks.join('\n\n')
      if (dynamicJoined) result.push({ text: dynamicJoined, cacheScope: null })  // ← 不缓存
      // ...
    }
  }
}
```

> **中文附注**：`splitSysPromptPrefix` 函数以 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 为分界线，将 prompt 数组切分为静态块和动态块。静态块标记 `cacheScope: 'global'`（全局缓存），动态块标记 `cacheScope: null`（不参与跨用户缓存）。这个 scope 最终会对应到 API 请求中的 `cache_control` 字段。

这也解释了为什么第二章中的 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 注释里会有这样的警告：

```typescript
/**
 * WARNING: Do not remove or reorder this marker without updating cache logic in:
 * - src/utils/api.ts (splitSysPromptPrefix)
 * - src/services/api/claude.ts (buildSystemPromptBlocks)
 */
```

这个标记一旦位置变化，`splitSysPromptPrefix` 就无法正确识别静态/动态边界，直接导致原本可以全局缓存的内容失去 `scope: 'global'` 标记。在全球用户规模下，这是一个具有显著成本影响的 bug。

---

## 3.3 Feature Flag 与死代码消除

Claude Code 使用 Bun 打包器的 `bun:bundle` 特性来做**编译期死代码消除（Dead Code Elimination, DCE）**。与运行时的 `if-else` 判断不同，DCE 在打包时就把未激活的代码分支彻底删除，最终的二进制文件中根本不包含这些代码。

在 `src/constants/prompts.ts` 文件顶部，可以看到大量这样的模式：

```typescript
// Dead code elimination: conditional imports for feature-gated modules
/* eslint-disable @typescript-eslint/no-require-imports */
const getCachedMCConfigForFRC = feature('CACHED_MICROCOMPACT')
  ? (
      require('../services/compact/cachedMCConfig.js') as typeof import('../services/compact/cachedMCConfig.js')
    ).getCachedMCConfig
  : null

const proactiveModule =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('../proactive/index.js')
    : null

const BRIEF_PROACTIVE_SECTION: string | null =
  feature('KAIROS') || feature('KAIROS_BRIEF')
    ? (
        require('../tools/BriefTool/prompt.js') as typeof import('../tools/BriefTool/prompt.js')
      ).BRIEF_PROACTIVE_SECTION
    : null

const DISCOVER_SKILLS_TOOL_NAME: string | null = feature(
  'EXPERIMENTAL_SKILL_SEARCH',
)
  ? (
      require('../tools/DiscoverSkillsTool/prompt.js') as typeof import('../tools/DiscoverSkillsTool/prompt.js')
    ).DISCOVER_SKILLS_TOOL_NAME
  : null
```

> **中文附注**：使用 `feature()` 函数（来自 `bun:bundle`）做编译期条件判断。当某个 feature 未激活时，整个 `require()` 调用及其对应的模块代码会在打包阶段被彻底删除，不会出现在最终的发布产物中。`proactiveModule`、`BRIEF_PROACTIVE_SECTION`、`DISCOVER_SKILLS_TOOL_NAME` 等变量在对应 feature 未激活时均为 `null`，相关的 prompt 内容不会出现在任何用户的 system prompt 中。

这套机制带来了两个直接收益：

**第一，产物体积精简。** 外部发布版本的 Claude Code 不包含 `PROACTIVE`、`KAIROS`、`EXPERIMENTAL_SKILL_SEARCH` 等实验性功能的代码，二进制体积更小，启动更快。

**第二，prompt 内容的版本隔离。** 不同功能配置的用户，收到的 system prompt 在内容上完全不同，而不是靠同一份 prompt 里的条件判断来区别对待。这意味着实验性功能的 prompt 从物理上就不存在于普通用户的运行环境中。

### 注意：`feature()` 是编译期概念，不是运行时开关

这里容易产生误解：`feature()` 不是一个运行时读取配置文件的函数，它是在打包阶段被 Bun 静态分析和替换的。如果 `feature('PROACTIVE')` 在打包时被评估为 `false`，那么 `proactiveModule` 就是 `null`，并且任何引用 `proactiveModule` 的代码路径也会随之被消除。这是真正的编译期优化，与 GrowthBook（运行时 feature flag）是两套完全独立的机制。

---

## 3.4 用户分层：`process.env.USER_TYPE`

Claude Code 的 prompt 在多处根据 `process.env.USER_TYPE === 'ant'` 做差异化处理。`ant` 指代 Anthropic 内部员工，这个环境变量在内部构建中被设为 `'ant'`，在外部发布版本中则为其他值（或不存在）。

这一机制影响了 prompt 的多个具体内容。以注释写作规范为例：

```typescript
// @[MODEL LAUNCH]: Update comment writing for Capybara — remove or soften once
// the model stops over-commenting by default
...(process.env.USER_TYPE === 'ant'
  ? [
      `Default to writing no comments. Only add one when the WHY is non-obvious: a hidden constraint, a subtle invariant, a workaround for a specific bug, behavior that would surprise a reader. If removing the comment wouldn't confuse a future reader, don't write it.`,
      `Don't explain WHAT the code does, since well-named identifiers already do that. Don't reference the current task, fix, or callers ("used by X", "added for the Y flow", "handles the case from issue #123"), since those belong in the PR description and rot as the codebase evolves.`,
      `Don't remove existing comments unless you're removing the code they describe or you know they're wrong. A comment that looks pointless to you may encode a constraint or a lesson from a past bug that isn't visible in the current diff.`,
      // @[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302) — un-gate once validated on external via A/B
      `Before reporting a task complete, verify it actually works: run the test, execute the script, check the output. Minimum complexity means no gold-plating, not skipping the finish line. If you can't verify (no test exists, can't run the code), say so explicitly rather than claiming success.`,
    ]
  : []),
```

> **中文附注**：内部用户（`USER_TYPE === 'ant'`）会看到四条额外的注释规范：默认不写注释、不解释代码做什么（标识符已经说明了）、不轻易删除现有注释（可能编码了历史约束）、完成任务前必须实际验证。这四条规则不出现在外部用户的 prompt 中。旁边的注释表明，这是针对特定模型版本（Capybara）过度注释倾向的专项修复，计划在问题解决后移除或软化。

以及输出效率的差异——内外两个版本可以说是完全不同的风格要求：

**内部用户（`ant`）**：

```typescript
if (process.env.USER_TYPE === 'ant') {
  return `# Communicating with the user
When sending user-facing text, you're writing for a person, not logging to a console. Assume users can't see most tool calls or thinking - only your text output. Before your first tool call, briefly state what you're about to do. While working, give short updates at key moments: when you find something load-bearing (a bug, a root cause), when changing direction, when you've made progress without an update.

When making updates, assume the person has stepped away and lost the thread. They don't know codenames, abbreviations, or shorthand you created along the way, and didn't track your process. Write so they can pick back up cold: use complete, grammatically correct sentences without unexplained jargon. Expand technical terms. Err on the side of more explanation. Attend to cues about the user's level of expertise; if they seem like an expert, tilt a bit more concise, while if they seem like they're new, be more explanatory. 

Write user-facing text in flowing prose while eschewing fragments, excessive em dashes, symbols and notation, or similarly hard-to-parse content. Only use tables when appropriate; for example to hold short enumerable facts (file names, line numbers, pass/fail), or communicate quantitative data. Don't pack explanatory reasoning into table cells -- explain before or after. Avoid semantic backtracking: structure each sentence so a person can read it linearly, building up meaning without having to re-parse what came before. 

What's most important is the reader understanding your output without mental overhead or follow-ups, not how terse you are. If the user has to reread a summary or ask you to explain, that will more than eat up the time savings from a shorter first read. Match responses to the task: a simple question gets a direct answer in prose, not headers and numbered sections. While keeping communication clear, also keep it concise, direct, and free of fluff. Avoid filler or stating the obvious. Get straight to the point. Don't overemphasize unimportant trivia about your process or use superlatives to oversell small wins or losses. Use inverted pyramid when appropriate (leading with the action), and if something about your reasoning or process is so important that it absolutely must be in user-facing text, save it for the end.

These user-facing text instructions do not apply to code or tool calls.`
}
```

> **中文附注（内部用户版）**：
>
> **# 与用户沟通**
>
> 在发送面向用户的文本时，你是在为一个真实的人写作，而不是在记录控制台日志。假设用户看不到大多数工具调用或思考过程——他们只能看到你的文字输出。在第一次工具调用之前，简要说明你打算做什么。在工作过程中，在关键时刻给出简短的更新：当你发现关键问题（一个 bug、一个根本原因）时，当你改变方向时，当你在没有更新的情况下取得进展时。
>
> 在做更新时，假设这个人已经离开并失去了上下文。他们不知道你在途中创造的代号、缩写或简写，也没有跟踪你的过程。写作时让他们能够重新接上：使用完整的、语法正确的句子，不带无解释的术语，展开技术术语，宁可多解释一些。注意用户专业程度的线索；如果他们看起来是专家，稍微简洁一点；如果他们看起来是新手，多一些解释。
>
> 以流畅的散文书写面向用户的文本，避免使用片段、过多的破折号、符号和记法或类似难以解析的内容。只在适当时使用表格：例如用于保存简短的可枚举事实（文件名、行号、通过/失败）或传达定量数据。不要把解释性推理塞进表格单元格——在前后解释。避免语义回溯：构造每个句子使人可以线性阅读，不需要重新解析之前的内容。
>
> 最重要的是读者在没有脑力开销或后续问题的情况下理解你的输出，而不是你有多简洁。如果用户不得不重新阅读摘要或要求你解释，这将远远超过更短的第一次阅读节省的时间。根据任务匹配回应：一个简单的问题得到散文形式的直接回答，而不是标题和编号部分。在保持清晰的同时，也要保持简洁、直接，没有废话。避免填充语或陈述显而易见的内容。直接切入主题。不要过度强调不重要的过程细节，也不要用最高级词语来过度渲染小的成功或失败。适当时使用倒金字塔（以行动开头），如果关于你的推理或过程的某些内容如此重要以至于绝对必须出现在面向用户的文本中，将其保留到最后。
>
> 这些面向用户的文本说明不适用于代码或工具调用。

**外部用户**：

```typescript
return `# Output efficiency

IMPORTANT: Go straight to the point. Try the simplest approach first without going in circles. Do not overdo it. Be extra concise.

Keep your text output brief and direct. Lead with the answer or action, not the reasoning. Skip filler words, preamble, and unnecessary transitions. Do not restate what the user said — just do it. When explaining, include only what is necessary for the user to understand.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan

If you can say it in one sentence, don't use three. Prefer short, direct sentences over long explanations. This does not apply to code or tool calls.`
```

> **中文附注（外部用户版）**：
>
> **# 输出效率**
>
> **重要**：直接切入主题。先尝试最简单的方法，不要绕圈子。不要过度。要格外简洁。
>
> 保持文字输出简短直接。以答案或行动开头，而非推理。省略填充词、前言和不必要的过渡。不要重述用户说的话——直接做。解释时，只包含用户理解所必需的内容。
>
> 将文字输出集中在：
> - 需要用户输入的决策
> - 自然里程碑处的高级状态更新
> - 改变计划的错误或阻碍
>
> 如果能用一句话说清楚，就不要用三句话。优先使用简短、直接的句子而非长篇解释。这不适用于代码或工具调用。

**这两段 prompt 的差异，揭示了一个重要的设计哲学**：内部用户（工程师和研究员）需要模型在每一步都清晰表达推理和上下文，以便他们能够理解模型的决策过程并发现潜在问题；外部用户则更希望直接得到结果，过多的"思考过程展示"反而是噪音。同一个模型，通过不同的输出风格 prompt，能够呈现出截然不同的沟通特征。

### `USER_TYPE` 的工程实现

代码注释中有一个关键的工程约束值得单独说明：

```typescript
// DCE: `process.env.USER_TYPE === 'ant'` is build-time --define. It MUST be
// inlined at each callsite (not hoisted to a const) so the bundler can
// constant-fold it to `false` in external builds and eliminate the branch.
```

> **中文附注**：`process.env.USER_TYPE === 'ant'` 是构建时通过 `--define` 注入的编译期常量。它**必须**在每个调用处内联（不能提升为一个 `const`），这样打包器才能在外部构建时将其常量折叠为 `false`，从而消除整个分支。

这个注释说明了一个反直觉的代码规范：**你不能用常量来简化代码**。通常的工程直觉是"如果一个表达式在多处使用，应该提取为常量"。但在这里，`process.env.USER_TYPE === 'ant'` 必须在每个 `if` 判断处原样出现，不能这样写：

```typescript
// ❌ 错误写法：打包器无法静态分析这个变量
const isAnt = process.env.USER_TYPE === 'ant'
if (isAnt) { ... }
```

而必须这样：

```typescript
// ✅ 正确写法：打包器可以识别并替换这个表达式
if (process.env.USER_TYPE === 'ant') { ... }
```

原因是 Bun 打包器只能识别直接出现的 `process.env.USER_TYPE === 'ant'` 模式，并将其替换为 `false`（在外部构建时），进而消除整个 `if` 分支。如果先赋值给变量，打包器无法追踪这个变量的来源，就无法做静态替换。这是工程约束驱动代码风格的一个典型案例。

---

## 3.5 启动时并行预取

`src/main.tsx` 文件的最顶部，在所有 `import` 语句之前，有三行在视觉上很不起眼、但工程价值显著的代码：

```typescript
// These side-effects must run before all other imports:
// 1. profileCheckpoint marks entry before heavy module evaluation begins
// 2. startMdmRawRead fires MDM subprocesses (plutil/reg query) so they run in
//    parallel with the remaining ~135ms of imports below
// 3. startKeychainPrefetch fires both macOS keychain reads (OAuth + legacy API
//    key) in parallel — isRemoteManagedSettingsEligible() otherwise reads them
//    sequentially via sync spawn inside applySafeConfigEnvironmentVariables()
//    (~65ms on every macOS startup)
import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';

// eslint-disable-next-line custom-rules/no-top-level-side-effects
profileCheckpoint('main_tsx_entry');
import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';

// eslint-disable-next-line custom-rules/no-top-level-side-effects
startMdmRawRead();
import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';

// eslint-disable-next-line custom-rules/no-top-level-side-effects
startKeychainPrefetch();
```

> **中文附注**：这三个副作用必须在所有其他 import 之前运行：
> 1. `profileCheckpoint` 在重量级模块求值开始之前标记入口
> 2. `startMdmRawRead` 启动 MDM 子进程（`plutil`/`reg query`），让它们与后续约 135ms 的 import 过程并行运行
> 3. `startKeychainPrefetch` 并行触发两次 macOS keychain 读取（OAuth + 旧版 API key）——否则 `isRemoteManagedSettingsEligible()` 会通过同步子进程调用顺序读取它们（在每次 macOS 启动时约需 65ms）

注释里的数字很关键：**135ms** 和 **65ms**。这两个操作如果顺序执行，会在模块加载期间额外增加 200ms 的等待——对于一个命令行工具来说，这是用户能明显感知到的延迟。通过在 import 开始之前就触发这两个异步操作，它们可以与后续 135ms 的模块加载并行进行，在用户看来，工具的启动时间就缩短了。

这与 prompt 设计有什么关系？

**直接关系**：system prompt 的动态部分（环境信息、记忆文件）需要从文件系统或系统 API 读取数据。如果等到需要构建 system prompt 时才开始读取，用户需要等待这些 I/O 操作完成。通过预取，这些数据在模块加载期间就已经开始准备，等到构建 system prompt 时，大概率可以直接使用已经就绪的结果。

**间接关系**：预取策略体现了整个系统的设计原则——**用户等待的每一毫秒都是可以被工程优化压缩的**。同样的原则应用于 prompt cache（减少重复计算的 token 开销）、Section 机制（精确控制哪些内容需要重算）和 DCE（减少加载不必要代码的时间）。

第一次渲染完成之后，还有一轮延迟预取：

```typescript
/**
 * Start background prefetches and housekeeping that are NOT needed before first render.
 * These are deferred from setup() to reduce event loop contention and child process
 * spawning during the critical startup path.
 * Call this after the REPL has been rendered.
 */
export function startDeferredPrefetches(): void {
  // ...
  // Process-spawning prefetches (consumed at first API call, user is still typing)
  void initUser();
  void getUserContext();
  prefetchSystemContextIfSafe();
  // ...
  void prefetchAwsCredentialsAndBedRockInfoIfSafe();
  // ...
  void prefetchGcpCredentialsIfSafe();
  void prefetchOfficialMcpUrls();
}
```

> **中文附注**：启动后台预取和维护工作，这些工作在第一次渲染之前不需要。从 `setup()` 中延迟，以减少关键启动路径上的事件循环竞争和子进程生成。在 REPL 渲染完成后调用。注释说明：这些预取在"用户还在输入时"完成（consumed at first API call, user is still typing），即利用用户输入第一条消息的时间窗口来预热数据，让第一次 API 调用时就能拿到已缓存的用户上下文、系统上下文等信息。

这个设计思路可以概括为：**把所有 I/O 操作都尽早触发，与用户的人类时间（思考、输入）重叠，而不是让用户等待机器时间**。

---

## 3.6 Prompt 随模型版本演化

本章最后一个机制，也是最能体现 Claude Code prompt 工程本质的一个：**prompt 不是一次性写好的，它随模型版本持续演化**。

代码中遍布着这样的注释：

```typescript
// @[MODEL LAUNCH]: Update comment writing for Capybara — remove or soften once
// the model stops over-commenting by default
```

```typescript
// @[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302) — un-gate once
// validated on external via A/B
```

```typescript
// @[MODEL LAUNCH]: False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)
```

```typescript
// @[MODEL LAUNCH]: Remove this section when we launch numbat.
```

> **中文附注**：这些注释是"模型发布"标记，记录了某条规则在哪个模型版本中被引入、为什么引入，以及预期在什么条件下可以撤销。例如：Capybara v8 的虚假声明率从 v4 的 16.7% 上升到 29-30%，因此加入了专项规则；Capybara 默认有过度注释的倾向，因此加入了注释规范的压制规则，计划在模型自然改善后移除。

这些注释揭示了 Anthropic 的 prompt 工程工作流：

**第一步，测量失败率。** 不是凭感觉判断"这个模型有问题"，而是有精确的数字——29-30% 的虚假声明率。这需要有完善的评估基础设施和指标体系。

**第二步，定向打补丁。** 针对特定模型的特定缺陷，加入专项的 prompt 规则。这些规则是**临时的**（注释里写明了退出条件），不是永久性的架构决策。

**第三步，A/B 测试验证。** 注释 `"un-gate once validated on external via A/B"` 说明新加的规则会先在内部用户（`USER_TYPE === 'ant'`）中测试效果，验证之后再推送到外部用户。这是 `user-level` 的灰度发布。

**第四步，随新模型迭代清理。** `"Remove this section when we launch numbat"` 表明他们知道这个规则是当前模型的补丁，新模型（numbat）上线后就可以移除。Prompt 不会无限累积临时补丁。

这个工作流让 prompt 成为一个**活的系统**，而不是一份静态文档。每次模型更新，都会有一批旧规则被移除，一批新规则被引入，以应对新模型的新特征。

---

## 3.7 小结

本章介绍了 Claude Code prompt 系统的五个工程机制：

| 机制 | 目的 | 核心手段 |
|---|---|---|
| Prompt Cache | 降低重复 token 的计算成本 | 保持 prompt 前缀稳定 |
| 全局缓存 Scope | 跨用户共享静态内容的缓存 | `scope: 'global'` + 边界标记 |
| Feature Flag DCE | 不同功能配置间的代码和 prompt 隔离 | `bun:bundle` + `feature()` |
| 用户分层 | 内外用户的 prompt 差异化 | `process.env.USER_TYPE === 'ant'` |
| 并行预取 | 降低启动延迟和首次响应延迟 | 模块加载期间触发异步 I/O |

这五个机制共同服务于一个目标：**在 prompt 足够详细（能引导正确行为）和足够经济（不产生不必要的成本和延迟）之间取得平衡**。理解了这些机制，就能理解为什么 Claude Code 的 prompt 在设计上总是在精确控制"什么内容在什么时候以什么形式出现"——每一处看似小的设计决策，背后都有可量化的成本或质量考量。

---

> **下一章**：工程机制已经完全铺垫好。从第四章开始，我们进入本书的核心内容：逐节分析主系统 prompt 的每个 Section，对每段文本做原文呈现、中文附注和设计解析。
