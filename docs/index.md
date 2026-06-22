# 目录

## 《Claude Code Prompt 设计艺术》

> 深度剖析 Anthropic 如何用 Prompt 驱动世界最复杂的 AI 编程工具

!!! warning "关于本书的生成方式"

    本书**完全由 [Claude Code](https://claude.ai/code) + Claude Sonnet 4.6 自动写成**，未经人工逐字校验。

    具体而言：所有章节内容由 AI 在阅读 Claude Code 源代码后自动生成；英文 Prompt 原文已与源文件逐字核对，但分析文字、中文翻译及设计解读均为模型输出，可能存在误读、遗漏或过度推断。

    **阅读建议**：将本书视为"有经验的读者对源码的第一遍注解"，而非权威文档。遇到存疑之处，请以 Claude Code 源代码为准。

---

## [前言](00-preface.md)

- 一个罕见的机会
- 为什么要写这本书
- 这本书是什么
- 关于源材料的说明
- 本书的结构
- 如何阅读这本书
- 一点个人的观察

---

## 第一部分：认识系统（第 1–3 章）

---

### [第一章　Claude Code 系统概览](01-overview.md)

- 1.1　Claude Code 是什么
- 1.2　系统规模
- 1.3　技术栈
- 1.4　Prompt 在系统中的位置
- 1.5　工具系统的结构
- 1.6　斜杠命令系统
- 1.7　自主模式（Proactive/Kairos）
- 1.8　小结：为什么这个系统的 Prompt 值得专门研究

---

### [第二章　Prompt 的组织架构](02-architecture.md)

- 2.1　Section：职责隔离的基本单元
- 2.2　两种 Section：缓存与易失
- 2.3　Section 的解析：缓存优先
- 2.4　静态与动态的边界
- 2.5　最终组装：`getSystemPrompt` 的返回结构
- 2.6　两种路径：普通模式与自主模式
- 2.7　工具级 Prompt 的双层结构
  - 第一层：`description` 字段
  - 第二层：详细使用说明（`prompt.ts`）
- 2.8　整体结构图
- 2.9　小结

---

### [第三章　Prompt 的工程机制](03-engineering.md)

- 3.1　Prompt Cache 与成本模型
  - Prompt Cache 的基本原理
  - 缓存失效的代价
- 3.2　全局缓存 Scope：跨用户共享的静态内容
- 3.3　Feature Flag 与死代码消除
  - `feature()` 是编译期概念，不是运行时开关
- 3.4　用户分层：`process.env.USER_TYPE`
  - `USER_TYPE` 的工程实现
- 3.5　启动时并行预取
- 3.6　Prompt 随模型版本演化
- 3.7　小结

---

## 第二部分：逐条分析（第 4–6 章）

---

### [第四章　主系统 Prompt 全解析](04-system-prompt.md)

**第一部分：静态 Section（边界之前）**

- 4.1　`getSimpleIntroSection()`：角色定义
- 4.2　`getSimpleSystemSection()`：系统规则
- 4.3　`getSimpleDoingTasksSection()`：任务执行规范
- 4.4　`getActionsSection()`：高危操作决策框架
- 4.5　`getUsingYourToolsSection()`：工具选择逻辑
- 4.6　`getSimpleToneAndStyleSection()`：输出风格
- 4.7　`getOutputEfficiencySection()`：输出效率

**第二部分：动态 Section（边界之后）**

- 4.8　`getSessionSpecificGuidanceSection()`：会话级动态指导
- 4.9　`computeSimpleEnvInfo()`：环境信息注入
- 4.10　`getLanguageSection()`：语言偏好
- 4.11　`SUMMARIZE_TOOL_RESULTS_SECTION`：工具结果摘记提示
- 4.12　`getFunctionResultClearingSection()`：函数结果清理提示
- 4.13　`getProactiveSection()`：自主模式行为规范
- 4.14　小结

---

### [第五章　工具 Prompt 全解析](05-tool-prompts.md)

**5.0　工具速览表**

**5.1　第一档工具：完整深析**

- 5.1.1　BashTool：Shell 执行与 Git 安全协议
- 5.1.2　AgentTool：子 Agent 编排与 Fork 模式
- 5.1.3　SkillTool：Skill 调用协议
- 5.1.4　EnterPlanModeTool：规划模式状态机
- 5.1.5　TodoWriteTool：任务追踪与进度管理

**5.2　第二档工具：标准分析**

- 5.2.1　文件操作类（Read、Edit）
- 5.2.2　搜索类（Glob、Grep）
- 5.2.3　Web 类（WebFetch、WebSearch）
- 5.2.4　用户交互类（AskUserQuestion、ExitPlanMode）
- 5.2.5　多 Agent 类（SendMessage、TeamCreate）
- 5.2.6　计划调度类（CronCreate）
- 5.2.7　配置与工作区类（ConfigTool、EnterWorktree / ExitWorktree）

**5.3　第三档工具：简要 + 模式引用**

**5.4　跨工具设计模式总结**

---

### [第六章　压缩摘要 Prompt 全解析](06-compact-prompts.md)

- 6.1　三层压缩架构概览
- 6.2　`NO_TOOLS_PREAMBLE`：关键约束的双重封锁
- 6.3　`DETAILED_ANALYSIS_INSTRUCTION_BASE`：引导式分析模板
- 6.4　`BASE_COMPACT_PROMPT`：全量压缩的 9 个 Section
- 6.5　`PARTIAL_COMPACT_PROMPT`：局部压缩变体
- 6.6　`PARTIAL_COMPACT_UP_TO_PROMPT`：反向压缩变体
- 6.7　`NO_TOOLS_TRAILER`：尾部重复约束
- 6.8　`getCompactUserSummaryMessage()`：摘要注入消息
- 6.9　`formatCompactSummary()`：Scratchpad 剥离机制
- 6.10　微压缩的工作原理
  - `TIME_BASED_MC_CLEARED_MESSAGE`
  - 时间触发型微压缩
  - 缓存编辑型微压缩（CACHED_MICROCOMPACT）
- 6.11　会话记忆压缩：第三层架构
- 6.12　压缩 Prompt 设计总结

---

## 第三部分：设计提炼（第 7 章）

---

### [第七章　Prompt 设计的十条核心原则](07-design-principles.md)

| 编号 | 原则名 | 核心思想 |
|---|---|---|
| 原则一 | 失败驱动 | 规则从真实失败数据推导，而非凭感觉编写 |
| 原则二 | 双边约束 | 同时锁定上下两侧边界，而非单侧防守 |
| 原则三 | 具体可操作 | 精确到命令、标志、格式，不留解读空间 |
| 原则四 | 理由优于规则 | 解释工程原因，让模型举一反三 |
| 原则五 | 决策框架 | 给出判断维度，而非穷举规则列表 |
| 原则六 | 禁止+替代 | 每条禁令配套可执行的替代方案 |
| 原则七 | 认知框架 | 用隐喻建立完整心智模型，比列表更持久 |
| 原则八 | 权限不可传递 | 授权是点状的，不随时间或范围传播 |
| 原则九 | 格式即行为 | 格式结构本身传达行为意图 |
| 原则十 | 随版本演化 | Prompt 跟随模型能力显式升级，不固守旧规 |
| 原则十一 | 测量替代预算 | 用工具输出 cap + 压缩缓冲替代 Section 预分配 |

每条原则均包含：定义 · Claude Code 中的体现 · 为什么有效 · 如何迁移

- 十一条原则的关系
- 小结：工业级 Prompt 的特征

---

## 附录

---

### [附录 A　英文原版 Prompt 完整收录](appendix-a-english.md)

- A.1　主系统 Prompt 各 Section（A.1.1–A.1.16）
- A.2　工具 Prompt
  - A.2.1　BashTool（第一档，完整收录）
  - A.2.2　AgentTool（第一档，完整收录）
  - A.2.3　EnterPlanModeTool（第一档，完整收录）
  - A.2.4　TodoWriteTool（第一档，完整收录）
  - A.2.5　第二档工具 Prompt 节选
  - A.2.6　第三档工具 Prompt 索引
- A.3　压缩摘要 Prompt（A.3.1–A.3.8）
- A.4　支撑性 Prompt 常量（A.4.1–A.4.3）

---

### [附录 B　中文译文完整收录](appendix-b-chinese.md)

- B.1　主系统 Prompt 各 Section（B.1.1–B.1.16）
- B.2　工具 Prompt（B.2.1–B.2.4）
- B.3　压缩摘要 Prompt（B.3.1–B.3.8）
- B.4　支撑性 Prompt 常量（B.4.1–B.4.3）

> 附录 B 与附录 A 一一对应；技术术语、工具名、命令保留英文原文。如有出入，以附录 A 为准。
