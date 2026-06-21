# 第一章：Claude Code 系统概览

在深入研究任何具体的 prompt 之前，我们需要先理解这个系统本身是什么，它由哪些部分组成，以及 prompt 在整个数据流中处于什么位置。没有系统观的 prompt 分析，就像在不了解人体结构的情况下研究药物配方——你或许能看懂某些成分，但无法理解它为什么有效、在哪里起作用。

---

## 1.1 Claude Code 是什么

Claude Code 是 Anthropic 开发的命令行 AI 编程工具。用户在终端中启动它，用自然语言下达任务——"帮我找出这个 bug"、"重构这个函数"、"写测试"——然后 Claude Code 自主地读取文件、执行命令、修改代码，直到完成任务。

这个描述听起来简单，但隐藏着相当高的系统复杂度。一个真正能胜任软件工程任务的 AI 工具，需要：

- **理解上下文**：知道当前工作目录、代码库结构、正在使用的技术栈
- **使用工具**：能读文件、写文件、运行 shell 命令、搜索代码
- **管理状态**：跨多轮对话保持任务的连续性，不因对话历史过长而迷失
- **控制风险**：在执行高危操作（删文件、强制推送代码）前征得用户同意
- **协调子任务**：将复杂任务分解，必要时生成子 agent 并行处理
- **与外部系统集成**：MCP 协议、语言服务器、IDE 扩展

这些需求的每一条，都在 prompt 设计上留下了清晰的印记。

---

## 1.2 系统规模

在分析 prompt 之前，有必要对这个系统的规模有一个感性认识。

**源代码体量**：

| 维度 | 数量 |
|---|---|
| 总源文件数 | 1,902 个 |
| 核心查询引擎（`QueryEngine.ts`） | 1,295 行 |
| 查询管道（`query.ts`） | 1,729 行 |
| 工具基础类型（`Tool.ts`） | 792 行 |
| 命令注册（`commands.ts`） | 754 行 |
| 工具目录数 | 43 个 |
| 主系统 prompt 文件（`constants/prompts.ts`） | ~900 行 |

这是一个远超"示例项目"规模的工程系统。1,902 个文件意味着它有专门的模块系统、抽象层次和内部 API。这一点很重要：它的 prompt 不是一个人在一个下午写出来的草稿，而是一个大型工程团队长期迭代的产物。

**主要顶层目录一览**：

```
src/
├── main.tsx              # CLI 入口，Commander.js 解析
├── QueryEngine.ts        # LLM 查询引擎核心
├── query.ts              # 查询管道与对话循环
├── Tool.ts               # 工具基础类型定义
├── commands.ts           # 斜杠命令注册
├── tools.ts              # 工具注册
│
├── tools/                # 43 个工具的具体实现
├── commands/             # 斜杠命令实现（~50 个）
├── components/           # Ink UI 组件（~140 个）
├── services/             # 外部服务集成
│   ├── api/              # Anthropic API 客户端
│   ├── mcp/              # MCP 协议
│   ├── compact/          # 上下文压缩
│   ├── oauth/            # 认证
│   └── analytics/        # Feature flag（GrowthBook）
│
├── bridge/               # IDE 扩展通信层
├── coordinator/          # 多 Agent 编排
├── skills/               # 可复用 Skill 系统
├── plugins/              # 插件系统
├── memdir/               # 持久化记忆
├── remote/               # 远程会话
├── hooks/                # React hooks + 权限系统
├── constants/            # 常量，含主系统 prompt
└── utils/                # 工具函数
```

---

## 1.3 技术栈

理解技术栈有助于理解某些 prompt 设计决策的背景——比如为什么 prompt 里要特别说明"使用 Bun 运行时"，或者为什么某些工具有特定的行为约束。

| 层次 | 技术 | 作用 |
|---|---|---|
| **运行时** | [Bun](https://bun.sh) | 替代 Node.js，启动更快，原生支持 TypeScript |
| **语言** | TypeScript（strict 模式） | 全量类型覆盖 |
| **终端 UI** | React + [Ink](https://github.com/vadimdemedes/ink) | 在终端中渲染 React 组件 |
| **CLI 解析** | Commander.js（`@commander-js/extra-typings`） | 子命令、选项解析 |
| **Schema 验证** | Zod v4 | 运行时类型检查和 JSON Schema 生成 |
| **代码搜索** | ripgrep | GrepTool 的底层实现 |
| **AI 协议** | MCP SDK | 与外部 MCP 服务器通信 |
| **语言服务** | LSP（Language Server Protocol） | 代码补全、跳转定义等 |
| **API 客户端** | Anthropic SDK（`@anthropic-ai/sdk`） | 调用 Claude API |
| **认证** | OAuth 2.0、JWT、macOS Keychain | 多种认证方式 |
| **Feature Flag** | GrowthBook | A/B 测试和功能开关 |
| **遥测** | OpenTelemetry + gRPC | 性能监控 |
| **打包** | Bun bundler（含 `bun:bundle`） | 死代码消除，Feature Flag 静态分析 |

值得单独说明的是 **Bun 的 `bun:bundle` 特性**。Claude Code 大量使用了这个机制来做编译期的死代码消除：

```typescript
import { feature } from 'bun:bundle'

// 当 VOICE_MODE 未激活时，整个 require() 在打包时被完全删除
const voiceModule = feature('VOICE_MODE') ? require('./voice/index.js') : null
```

这意味着不同功能配置下，用户实际下载和运行的代码是不同的二进制。这一机制也影响了 prompt 的组织方式——某些 prompt 文本只在特定 feature 激活时才会出现。本书后续章节会在涉及具体 prompt 时标注它们所依赖的 feature flag。

---

## 1.4 Prompt 在系统中的位置

要理解 prompt 的设计，必须先理解它在数据流中处于哪个位置。以下是从用户输入到 API 响应的完整路径：

```
用户在终端输入一条指令
         │
         ▼
┌─────────────────────┐
│   Ink/React UI      │  终端界面，捕获用户输入
│   (components/)     │
└─────────┬───────────┘
          │ 用户消息
          ▼
┌─────────────────────┐
│   QueryEngine.ts    │  对话循环的主控制器
│                     │  管理消息历史、工具执行、重试
└─────────┬───────────┘
          │ 准备 API 请求
          ▼
┌─────────────────────────────────────────────────┐
│   fetchSystemPromptParts()                      │
│   (utils/queryContext.ts)                       │
│                                                 │
│   ┌─────────────────────────────────────────┐   │
│   │ getSystemPrompt()                       │   │
│   │ (constants/prompts.ts)                  │   │
│   │                                         │   │
│   │  静态 sections（可缓存）：               │   │
│   │  · 角色定义                             │   │
│   │  · 系统规则                             │   │
│   │  · 任务执行规范                         │   │
│   │  · 高危操作决策                         │   │
│   │  · 工具使用指南                         │   │
│   │  · 输出风格                             │   │
│   │  ─────────────── BOUNDARY ──────────────│   │
│   │  动态 sections（每次重算）：             │   │
│   │  · 环境信息（cwd、OS、git状态）         │   │
│   │  · 用户记忆                             │   │
│   │  · MCP 服务器指令                       │   │
│   │  · 语言偏好                             │   │
│   └─────────────────────────────────────────┘   │
│                                                 │
│   + getUserContext()   ← 用户上下文             │
│   + getSystemContext() ← 系统上下文             │
└─────────┬───────────────────────────────────────┘
          │ system prompt + 对话历史
          ▼
┌─────────────────────┐
│   Anthropic API     │  Claude 模型
│   (claude.ts)       │
└─────────┬───────────┘
          │ 流式响应
          ▼
┌─────────────────────┐
│   工具执行层         │  解析 tool_call，执行工具
│   (query.ts)        │  结果追加到消息历史
└─────────┬───────────┘
          │ 继续对话 or 结束
          ▼
     返回结果给用户
```

这张图揭示了一个关键事实：**system prompt 是每次 API 调用的固定前缀，它与整个对话历史一起构成模型的完整输入**。

这带来两个重要推论：

**推论一：system prompt 的设计质量，直接决定了模型在每一轮对话中的行为基线。** 它不是一次性的"初始化"，而是每轮都在场的"行为规范"。

**推论二：system prompt 的长度有直接的成本代价。** 每次 API 调用都要发送完整的 system prompt，加上全部对话历史，再加上工具描述。Claude Code 的 system prompt 展开后约 3,000-4,000 个 token，在高频使用场景下，这是一笔不小的开销。这解释了为什么 Anthropic 在 prompt 的缓存策略上投入了大量工程资源——我们将在第三章详细分析。

---

## 1.5 工具系统的结构

除了 system prompt，Claude Code 的另一个关键 prompt 来源是**工具描述**。每当模型需要决定调用哪个工具时，它依赖的是每个工具在 API 请求中携带的描述信息。

Claude Code 共有 43 个工具目录，覆盖的能力如下：

**文件操作类**

| 工具 | 功能 |
|---|---|
| `FileReadTool` | 读取文件（支持图片、PDF、Jupyter Notebook） |
| `FileWriteTool` | 创建或覆写文件 |
| `FileEditTool` | 精确字符串替换（局部修改） |
| `NotebookEditTool` | Jupyter Notebook 单元格编辑 |

**代码搜索类**

| 工具 | 功能 |
|---|---|
| `GlobTool` | 文件模式匹配搜索 |
| `GrepTool` | 基于 ripgrep 的内容搜索 |
| `LSPTool` | Language Server Protocol 查询 |

**执行类**

| 工具 | 功能 |
|---|---|
| `BashTool` | Shell 命令执行（跨平台主工具） |
| `PowerShellTool` | Windows PowerShell 执行 |
| `REPLTool` | 交互式 REPL 执行环境 |

**Web 类**

| 工具 | 功能 |
|---|---|
| `WebFetchTool` | 抓取 URL 内容 |
| `WebSearchTool` | 网络搜索 |

**多 Agent 类**

| 工具 | 功能 |
|---|---|
| `AgentTool` | 生成子 Agent（fork 或全新） |
| `SendMessageTool` | 向其他 Agent 发送消息 |
| `TeamCreateTool` | 创建 Agent 团队（并行工作） |
| `TeamDeleteTool` | 解散 Agent 团队 |

**任务管理类**

| 工具 | 功能 |
|---|---|
| `TaskCreateTool` | 创建任务 |
| `TaskUpdateTool` | 更新任务状态 |
| `TaskGetTool` | 查询任务详情 |
| `TaskListTool` | 列出所有任务 |
| `TaskStopTool` | 停止任务 |
| `TodoWriteTool` | 写入 todo 列表（任务追踪的另一种形态） |

**工作流控制类**

| 工具 | 功能 |
|---|---|
| `EnterPlanModeTool` | 进入计划模式 |
| `ExitPlanModeTool` | 退出计划模式（呈现计划供用户审批） |
| `EnterWorktreeTool` | 进入 Git worktree 隔离环境 |
| `ExitWorktreeTool` | 退出 Git worktree |
| `SkillTool` | 执行 Skill（可复用工作流） |
| `SleepTool` | 自主模式下等待 |
| `ScheduleCronTool` | 创建定时触发器 |
| `RemoteTriggerTool` | 远程触发器 |

**配置与发现类**

| 工具 | 功能 |
|---|---|
| `ToolSearchTool` | 发现并加载延迟工具（Deferred Tools） |
| `ConfigTool` | 读写 Claude Code 配置 |
| `AskUserQuestionTool` | 向用户提出多选题 |
| `BriefTool` | 自主模式下的状态简报 |

**MCP 类**

| 工具 | 功能 |
|---|---|
| `MCPTool` | 调用 MCP 服务器的工具 |
| `ListMcpResourcesTool` | 列出 MCP 资源 |
| `ReadMcpResourceTool` | 读取 MCP 资源 |

**其他**

| 工具 | 功能 |
|---|---|
| `SyntheticOutputTool` | 结构化输出生成 |
| `TaskOutputTool` | 任务输出记录 |
| `McpAuthTool` | MCP 认证 |

这 43 个工具的每一个，都有对应的 prompt——既有传入 API 的简短描述（`description` 字段），也有详细的行为规范（各工具的 `prompt.ts` 文件）。第五章将逐一分析其中设计最丰富的工具。

---

## 1.6 斜杠命令系统

除了工具，Claude Code 还有一套面向用户的**斜杠命令**（slash commands），通过 `/` 前缀在对话输入框中触发。这些命令不经过 LLM，而是直接在本地执行。

典型的斜杠命令包括：

| 命令 | 功能 |
|---|---|
| `/compact` | 手动触发对话压缩 |
| `/clear` | 清空对话历史 |
| `/config` | 管理设置 |
| `/memory` | 查看/编辑持久化记忆 |
| `/cost` | 查看当前会话的 token 消耗 |
| `/doctor` | 运行环境诊断 |
| `/login` / `/logout` | 认证管理 |
| `/vim` | 切换 Vim 输入模式 |
| `/review` | 代码审查 |
| `/commit` | 创建 git 提交 |
| `/diff` | 查看当前变更 |
| `/share` | 上传会话转录 |
| `/resume` | 恢复历史会话 |
| `/theme` | 切换主题 |
| `/mcp` | 管理 MCP 服务器 |
| `/skills` | 管理 Skill |

斜杠命令本身通常不涉及 prompt 设计（它们是本地执行的），但有一个例外：**Skill**。Skill 是用户定义的可复用工作流，通过 `/skill-name` 触发后，会将预先定义好的 prompt 注入到当前对话中。这是用户层面可以直接定义和使用 prompt 的机制，将在第五章的 SkillTool 分析中详细介绍。

---

## 1.7 自主模式（Proactive/Kairos）

Claude Code 还有一种特殊的运行模式，代码中称为 **Proactive** 或 **Kairos**（不同的 feature flag 对应不同的激活方式）。

在这种模式下，Claude Code 不再被动等待用户输入，而是作为一个持续运行的后台 Agent，自主地规划工作、执行任务、在需要时唤醒自己。这个模式下的 prompt 与普通交互模式**完全不同**——系统 prompt 会被完全替换为自主模式专用的版本，行为约束从"谨慎确认"转变为"偏向行动"。

这一点在架构上非常重要：**Claude Code 的 prompt 不是一套固定的文本，而是一个根据运行模式、用户身份、当前工具集动态组装的系统**。同一个 Claude 模型，在不同配置下收到的 prompt 可能有显著差异，表现出来的行为也因此不同。

---

## 1.8 小结：为什么这个系统的 prompt 值得专门研究

读完本章，你可能注意到 Claude Code 的复杂程度远超一般的 LLM 应用。它不只是"把用户消息发给 API 然后显示回答"，而是一个有完整工具系统、权限系统、状态管理、多 Agent 编排、上下文压缩的工程系统。

这种复杂性对 prompt 设计提出了罕见的高要求：

**一、行为必须高度精确。** 模型需要知道什么时候可以自主执行，什么时候必须先征得用户同意；什么工具该用，什么操作绝对不能做。模糊的指令会造成真实的损失（代码被删、提交被污染）。

**二、成本必须可控。** 每次 API 调用都携带完整的 system prompt。在一个被大量开发者高频使用的商业产品中，prompt 的 token 效率直接影响运营成本。

**三、行为必须随模型版本演化。** 不同的 Claude 模型有不同的默认行为倾向——有的会过度解释，有的容易做出虚假声明，有的会在不该调用工具时调用工具。Prompt 需要针对每个模型版本的具体缺陷做出调整。

**四、必须在多种运行模式下保持一致性。** 同一套工具和规则，需要在交互模式、自主模式、IDE 集成模式下都表现合理，而这些模式下用户的期望和容忍度是完全不同的。

正是这些高要求，造就了本书研究的对象：一份在工业压力下被反复锤炼、极其精细的 prompt 工程实践。

---

> **下一章**：我们将深入 prompt 的内部结构，理解它是如何被分解为独立的 section、动态组装，以及静态内容和动态内容为什么要被严格分开。
