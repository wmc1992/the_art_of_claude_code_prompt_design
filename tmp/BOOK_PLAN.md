# 写书计划：Claude Code Prompt 设计艺术

> 本文档是写作计划与进度追踪表。每完成一章，在对应条目打 ✅。

---

## 书名（暂定）

**《Claude Code Prompt 设计艺术：一部工业级 AI Agent 的指令工程全解》**

副标题备选：
- 深度剖析 Anthropic 如何用 Prompt 驱动世界最复杂的 AI 编程工具
- 从源码出发，解读 Prompt Engineering 的工业实践

---

## 受众与定位

- **主要受众**：算法工程师、AI 产品技术负责人、LLM 应用开发者
- **知识前提**：熟悉 LLM 原理、了解 Prompt Engineering 基本概念、有工程系统设计经验
- **阅读目标**：读完本书，读者能深刻理解 Claude Code prompt 的每一处设计决策，并能将这些原则迁移到自己的 LLM 系统设计中
- **写作风格**：深入浅出。专业不等于晦涩，每个设计模式都配具体的原文和可类比的现实场景

---

## 源码范围

**代码库路径**：`/Users/wangmingchao/WorkProject/wmc_git_space/tmp/claude-code/src/`

涉及的核心文件：

```
constants/
  prompts.ts                  # 主系统 prompt（~900 行，最核心）
  systemPromptSections.ts     # Section 缓存机制
  cyberRiskInstruction.ts     # 安全边界指令
  outputStyles.ts             # 输出风格配置

services/compact/
  prompt.ts                   # 压缩摘要 prompt（全量 + 局部 + UP_TO 三种变体）

tools/（按规模排列）
  BashTool/prompt.ts          # 369 行
  AgentTool/prompt.ts         # 287 行
  SkillTool/prompt.ts         # 241 行
  TodoWriteTool/prompt.ts     # 184 行（即 TaskCreate 的 todo 变体）
  EnterPlanModeTool/prompt.ts # 170 行
  PowerShellTool/prompt.ts    # 145 行
  ScheduleCronTool/prompt.ts  # 135 行
  ToolSearchTool/prompt.ts    # 121 行
  TeamCreateTool/prompt.ts    # 113 行
  ConfigTool/prompt.ts        #  93 行
  TaskUpdateTool/prompt.ts    #  77 行
  TaskCreateTool/prompt.ts    #  56 行
  FileReadTool/prompt.ts      #  49 行
  SendMessageTool/prompt.ts   #  49 行
  TaskListTool/prompt.ts      #  49 行
  WebFetchTool/prompt.ts      #  46 行
  AskUserQuestionTool/prompt.ts #  44 行
  WebSearchTool/prompt.ts     #  34 行
  ExitWorktreeTool/prompt.ts  #  32 行
  EnterWorktreeTool/prompt.ts #  30 行
  ExitPlanModeTool/prompt.ts  #  29 行
  FileEditTool/prompt.ts      #  28 行
  TaskGetTool/prompt.ts       #  24 行
  BriefTool/prompt.ts         #  22 行
  LSPTool/prompt.ts           #  21 行
  ListMcpResourcesTool/prompt.ts # 20 行
  FileWriteTool/prompt.ts     #  18 行
  GrepTool/prompt.ts          #  18 行
  SleepTool/prompt.ts         #  17 行
  RemoteTriggerTool/prompt.ts #  15 行
  ReadMcpResourceTool/prompt.ts # 16 行
  TaskStopTool/prompt.ts      #   8 行
  GlobTool/prompt.ts          #   7 行
  MCPTool/prompt.ts           #   3 行
  NotebookEditTool/prompt.ts  #   3 行
```

---

## 工具分析档位

每章写作时按此档位处理工具 prompt：

### 第一档：完整深析（逐条展开，不省略）
| 工具 | 行数 | 理由 |
|---|---|---|
| BashTool | 369 | 最复杂，含 git 安全协议、沙箱约束、步骤化流程 |
| AgentTool | 287 | 子 agent 编排的语义核心，fork/非fork 双模式 |
| SkillTool | 241 | Skill 系统的调用协议 |
| EnterPlanModeTool | 170 | Plan 模式的完整状态机描述 |
| TodoWriteTool | 184 | 任务追踪 prompt 的代表，设计独特 |

### 第二档：标准分析（完整但精炼）
PowerShellTool、ScheduleCronTool、ToolSearchTool、TeamCreateTool、ConfigTool、
TaskUpdateTool、TaskCreateTool、FileReadTool、SendMessageTool、TaskListTool、
WebFetchTool、AskUserQuestionTool、WebSearchTool、ExitWorktreeTool、EnterWorktreeTool、
ExitPlanModeTool、FileEditTool、TaskGetTool

### 第三档：简要 + 交叉引用（指出复用的模式，不重复展开）
BriefTool、LSPTool、ListMcpResourcesTool、FileWriteTool、GrepTool、SleepTool、
RemoteTriggerTool、ReadMcpResourceTool、TaskStopTool、GlobTool、MCPTool、NotebookEditTool

---

## 章节结构与进度

### 前置材料
- [ ] **封面页**：书名、副标题、版本说明
- [ ] **前言**：为什么研究 Claude Code 的 prompt、本书的结构、阅读建议
  - 输出文件：`book/00-preface.md`

---

### 第一章：Claude Code 系统概览
- [ ] 完成
- **内容**：
  - Claude Code 是什么，解决什么问题
  - 系统规模：~1900 文件、51万行代码、35+ 工具、50+ 命令
  - 技术栈全景（Bun、React/Ink、Commander.js、Zod v4、MCP SDK 等）
  - Prompt 在系统中的位置：从用户输入到 API 调用的完整路径
  - 为什么 Claude Code 的 prompt 值得专门研究
- **源文件**：`src/main.tsx`（前100行）、`README.md`
- **输出文件**：`book/01-overview.md`

---

### 第二章：Prompt 的组织架构
- [ ] 完成
- **内容**：
  - System prompt 不是一段文字，而是一个组装系统
  - Section 机制：每个 section 只负责一类职责
  - 静态内容与动态内容的分界线（`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`）
  - 工具级 prompt 的设计范式：tool description vs usage prompt
  - 整体组装流程图（用 ASCII 图或列表表示）
- **源文件**：`src/constants/prompts.ts`（getSystemPrompt 函数）、`src/constants/systemPromptSections.ts`
- **输出文件**：`book/02-architecture.md`

---

### 第三章：Prompt 的工程机制
- [ ] 完成
- **内容**：
  - Prompt Cache 机制与成本模型（为什么要分静态/动态）
  - `systemPromptSection` vs `DANGEROUS_uncachedSystemPromptSection`（含原文 + 译文）
  - 全局缓存 scope（`scope: 'global'`）的含义
  - Feature Flag 死代码消除（`feature('VOICE_MODE')`等）
  - 用户分层：`process.env.USER_TYPE === 'ant'` 的内外差异
  - 启动时并行预取（`startMdmRawRead`、`startKeychainPrefetch`）
  - 自主模式（Proactive）下 prompt 的完全替换
- **源文件**：`src/constants/systemPromptSections.ts`、`src/main.tsx`（前20行）、`src/constants/prompts.ts`（Feature Flag 部分）
- **输出文件**：`book/03-engineering.md`

---

### 第四章：主系统 Prompt 全解析
- [ ] 完成
- **内容**（每节：英文原文 → 中文附注 → 设计解析）：
  1. `getSimpleIntroSection()`：角色定义
  2. `getSimpleSystemSection()`：系统规则
  3. `getSimpleDoingTasksSection()`：任务执行规范（含代码风格子规则）
  4. `getActionsSection()`：高危操作决策框架
  5. `getUsingYourToolsSection()`：工具选择逻辑
  6. `getSimpleToneAndStyleSection()`：输出风格
  7. `getOutputEfficiencySection()`：输出效率（内外两个版本对比）
  8. `getSessionSpecificGuidanceSection()`：会话级动态指导
  9. `computeSimpleEnvInfo()`：环境信息注入
  10. `getLanguageSection()`：语言偏好
  11. `getProactiveSection()`：自主模式（Proactive/Kairos）
  12. `CYBER_RISK_INSTRUCTION`：安全边界
  13. `SUMMARIZE_TOOL_RESULTS_SECTION`：工具结果处理提示
  14. `getFunctionResultClearingSection()`：函数结果清理提示
- **源文件**：`src/constants/prompts.ts`（全文）、`src/constants/cyberRiskInstruction.ts`
- **输出文件**：`book/04-system-prompt.md`

---

### 第五章：工具 Prompt 全解析
- [ ] 完成
- **结构**：
  - 5.0 工具速览表（所有工具的 description 字段一览，含中文译文）
  - 5.1 第一档工具（完整深析）
    - BashTool：Shell 执行与 Git 安全协议
    - AgentTool：子 Agent 编排与 Fork 模式
    - SkillTool：Skill 调用协议
    - EnterPlanModeTool：Plan 模式状态机
    - TodoWriteTool：任务追踪与进度管理
  - 5.2 第二档工具（标准分析）
    - 按功能分组：文件操作类、搜索类、Web 类、任务管理类、多 Agent 类、配置类、工作区类
  - 5.3 第三档工具（简要 + 交叉引用）
- **源文件**：所有 `src/tools/*/prompt.ts`
- **输出文件**：`book/05-tool-prompts.md`

---

### 第六章：Compact 压缩摘要 Prompt 全解析
- [ ] 完成
- **内容**（每节：英文原文 → 中文附注 → 设计解析）：
  1. 三层压缩架构概览（微压缩、全量压缩、会话记忆压缩）
  2. `NO_TOOLS_PREAMBLE`：关键约束的双重封锁
  3. `DETAILED_ANALYSIS_INSTRUCTION_BASE`：引导式分析模板
  4. `BASE_COMPACT_PROMPT`：全量压缩 prompt 的 9 个 section
  5. `PARTIAL_COMPACT_PROMPT`：局部压缩变体
  6. `PARTIAL_COMPACT_UP_TO_PROMPT`：反向压缩变体（最特殊的一个）
  7. `NO_TOOLS_TRAILER`：尾部重复约束
  8. `getCompactUserSummaryMessage()`：摘要注入消息的三种模式
  9. `formatCompactSummary()`：Scratchpad 剥离机制
  10. 微压缩的工作原理（`TIME_BASED_MC_CLEARED_MESSAGE`）
- **源文件**：`src/services/compact/prompt.ts`、`src/services/compact/microCompact.ts`、`src/services/compact/sessionMemoryCompact.ts`
- **输出文件**：`book/06-compact-prompts.md`

---

### 第七章：Prompt 有效性的设计原则
- [ ] 完成
- **内容**：
  对前四章分析出现的所有设计模式做系统性归纳，每个原则：
  - 命名与定义
  - 在 Claude Code 中的具体体现（多处引用，不省略）
  - 为什么有效（认知机制或工程原因）
  - 如何迁移到你自己的 prompt 设计
  
  十个核心原则：
  1. 失败驱动：针对失败模式写 prompt，而非描述理想行为
  2. 双边约束：同时防止不足与过度
  3. 具体可操作：命名 > 抽象，命令名 > 类别描述
  4. 理由优于规则：带 why 的规则才能泛化
  5. 决策框架优于规则列表：给判断逻辑，不穷举情况
  6. 禁止与替代成对：每个约束都要提供出口
  7. 认知框架：给模型思维工具，不只给外部约束
  8. 权限不可传递：授权范围不能被模型自行扩展
  9. 格式即行为：输出格式要求直接影响思维质量
  10. Prompt 随模型版本演化：观察-修正的迭代工程
- **输出文件**：`book/07-design-principles.md`

---

### 附录
- [ ] **附录 A：英文原版 Prompt 完整收录**
  - 纯文本，逐节收录，供快速查阅
  - 输出文件：`book/appendix-a-english.md`

- [ ] **附录 B：中文译文完整收录**
  - 对应附录 A 的中文版本
  - 输出文件：`book/appendix-b-chinese.md`

---

## 写作规范

### 英文原文要求（最高优先级）
- 所有英文 prompt 必须与源代码**逐字一致**，不得有任何修改
- 每章完成后执行一次校验：对照源文件核查所有引用的英文 prompt
- 格式：用代码块（\`\`\`）包裹，注明来源文件和行号

### 每段 prompt 的呈现格式
```
#### [Section 名称]

**英文原文**（来源：`src/...`，第 X-Y 行）
\`\`\`
[原文 verbatim]
\`\`\`

**中文译文**
> [对应的中文翻译]

**设计解析**
[分析文字]
```

### 设计解析的写法
- 每处设计都要分析，即使模式重复，也要指出"此处再次运用了 [模式名]，在此处的特殊语境是..."
- 不使用术语堆砌，每个概念配一句话的直白解释
- 分析长度与 prompt 复杂度成比例：简短的 prompt 配简洁的分析

### 章节文件命名
```
book/
  00-preface.md
  01-overview.md
  02-architecture.md
  03-engineering.md
  04-system-prompt.md
  05-tool-prompts.md
  06-compact-prompts.md
  07-design-principles.md
  appendix-a-english.md
  appendix-b-chinese.md
```

---

## 进度记录

| 章节 | 状态 | 完成日期 | 备注 |
|---|---|---|---|
| 前言 | ✅ 完成 | 2026-06-21 | - |
| 第一章 | ✅ 完成 | 2026-06-21 | - |
| 第二章 | ✅ 完成 | 2026-06-21 | - |
| 第三章 | ✅ 完成 | 2026-06-21 | - |
| 第四章 | ✅ 完成 | 2026-06-21 | 一次交付完成，覆盖全部14个Section |
| 第五章 | ✅ 完成 | 2026-06-21 | 一次交付完成，覆盖全部35+工具，分三档深度 |
| 第六章 | ✅ 完成 | 2026-06-21 | 一次交付完成，覆盖三层压缩架构 |
| 第七章 | ✅ 完成 | 2026-06-21 | 一次交付完成，十条核心原则，含 Claude Code 多章节交叉引用 |
| 附录 A | ✅ 完成 | 2026-06-21 | 完整收录，A.1系统Prompt全16节、A.2工具4个第一档完整+第二三档索引、A.3压缩Prompt全8节、A.4支撑常量 |
| 附录 B | ✅ 完成 | 2026-06-21 | 完整中文译文，与附录 A 结构对应，技术术语保留原文 |

---

## 待确认事项

- [ ] 书名最终版本
- [ ] mkdocs 的 nav 配置是否需要我在计划中列出
