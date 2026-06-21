# Claude Code Prompt 设计艺术

**副标题：深度剖析 Anthropic 如何用 Prompt 驱动世界最复杂的 AI 编程工具**

---

## 关于本项目

本项目是一本基于 Claude Code 源代码写成的技术书籍，全名《Claude Code Prompt 设计艺术：一部工业级 AI Agent 的指令工程全解》。

书中以 Claude Code 的真实 Prompt 源码为第一手材料，逐条分析每一处设计决策——从角色定义到任务执行规范，从 Git 安全协议到压缩摘要机制，从子 Agent 编排到十条核心设计原则——试图回答一个问题：**工业级 AI Agent 的 Prompt，究竟是怎么写出来的？**

> **⚠️ 注意**：本书完全由 Claude Code + Claude Sonnet 4.6 自动写成，未经人工逐字校验。英文 Prompt 原文已与源文件核对，但分析文字、中文翻译及设计解读均为模型输出，可能存在误读或过度推断。遇到存疑之处，请以 Claude Code 源代码为准。

---

## 书籍结构

| 部分 | 章节 | 内容 |
|---|---|---|
| 第一部分 | 第 1 章 | Claude Code 系统概览：规模、技术栈、Prompt 的位置 |
| 第一部分 | 第 2 章 | Prompt 的组织架构：Section 机制、静态与动态边界 |
| 第一部分 | 第 3 章 | Prompt 的工程机制：缓存、Feature Flag、用户分层、版本演化 |
| 第二部分 | 第 4 章 | 主系统 Prompt 全解析：13 个 Section 逐条分析 |
| 第二部分 | 第 5 章 | 工具 Prompt 全解析：30+ 工具，三档深度覆盖 |
| 第二部分 | 第 6 章 | 压缩摘要 Prompt 全解析：三层压缩架构与所有 prompt 常量 |
| 第三部分 | 第 7 章 | Prompt 设计的十条核心原则：失败驱动、双边约束、决策框架…… |
| 附录 | 附录 A | 英文原版 Prompt 完整收录（逐字，含源文件行号） |
| 附录 | 附录 B | 中文译文完整收录 |

---

## 在线阅读

> 如果你 fork 了本仓库并启用了 GitHub Pages，部署后可在此处阅读。

本书使用 [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) 构建，支持中文搜索、亮色/暗色切换、代码一键复制。

**本地运行：**

```bash
pip install mkdocs-material
mkdocs serve
```

然后访问 `http://127.0.0.1:8000`。

---

## 分析的源码范围

核心分析文件来自 Claude Code 的 `src/` 目录：

```
constants/prompts.ts                # 主系统 prompt（~900 行）
services/compact/prompt.ts          # 压缩摘要 prompt（三种变体）
tools/BashTool/prompt.ts            # Shell 执行与 Git 安全协议（369 行）
tools/AgentTool/prompt.ts           # 子 Agent 编排与 Fork 模式（287 行）
tools/SkillTool/prompt.ts           # Skill 调用协议（241 行）
tools/EnterPlanModeTool/prompt.ts   # 规划模式状态机（170 行）
tools/TodoWriteTool/prompt.ts       # 任务追踪与进度管理（184 行）
...以及另外 25+ 个工具的 prompt.ts
```

书中所有英文 Prompt 引用均标注源文件路径和行号，可与原始代码逐字比对。

---

## 十条核心设计原则（第七章摘要）

| 原则 | 核心思想 |
|---|---|
| 失败驱动 | 规则从真实失败数据推导，而非凭感觉编写 |
| 双边约束 | 同时锁定上下两侧边界，而非单侧防守 |
| 具体可操作 | 精确到命令、标志、格式，不留解读空间 |
| 理由优于规则 | 解释工程原因，让模型举一反三 |
| 决策框架 | 给出判断维度，而非穷举规则列表 |
| 禁止+替代 | 每条禁令配套可执行的替代方案 |
| 认知框架 | 用隐喻建立完整心智模型，比列表更持久 |
| 权限不可传递 | 授权是点状的，不随时间或范围传播 |
| 格式即行为 | 格式结构本身传达行为意图 |
| 随版本演化 | Prompt 跟随模型能力显式升级，不固守旧规 |

---

## 写作方式

本书使用 Claude Code 自身作为写作工具，在同一个 Claude Code 会话中完成：

- 读取源代码（`src/constants/prompts.ts` 等）
- 对照源文件逐字引用英文原文
- 生成中文注释、设计解析、原则提炼

这意味着**工具本身就是研究对象**——一个有趣的自指现象。

---

## License

本书内容（`docs/` 目录）以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，署名即可非商业转载。

Claude Code 源代码版权归 Anthropic 所有，书中引用的 Prompt 原文仅作研究分析用途。
