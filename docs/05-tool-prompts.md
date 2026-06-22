# 第五章：工具 Prompt 全解析

与系统 prompt 负责"模型是谁"不同，工具 prompt 负责"每个工具如何被正确使用"。每个工具有两层文字：短 `description`（出现在 API 工具模式里，帮助模型决定是否调用这个工具）和完整 prompt（告诉模型调用时的具体规则）。前者是导航，后者是说明书。

本章按复杂度分三档展开所有工具。第一档五个工具做完整深析，第二档工具做标准分析，第三档工具做简要引用。所有英文原文与源代码逐字一致。

> **本书贯穿的设计模式**
>
> 以下设计模式在本章中反复出现。每次出现的**最典型处**会用 `→ 原则N` 标注，完整归纳见第七章。
>
> `① 失败驱动` `② 双边约束` `③ 具体可操作` `④ 理由优于规则`
> `⑤ 决策框架` `⑥ 禁止+替代` `⑦ 认知框架` `⑧ 权限不可传递`
> `⑨ 格式即行为` `⑩ 随版本演化`

---

## 5.0 工具速览表

> 以下 `description` 字段即 API 工具模式里的 `description`，是模型看到工具名时首先读到的文字。`★` 标注第一档工具。

| 工具名 | description 字段 | 中文释义 |
|---|---|---|
| **Bash** ★ | Executes a given bash command and returns its output. | 执行 shell 命令并返回输出 |
| **Agent** ★ | Launch a new agent to handle complex, multi-step tasks autonomously. | 启动子 Agent 处理复杂多步任务 |
| **SkillTool** ★ | Execute a skill within the main conversation | 在主对话中执行 Skill |
| **EnterPlanMode** ★ | Use this tool proactively when you're about to start a non-trivial implementation task. *(外部)* / Use this tool when a task has genuine ambiguity about the right approach. *(内部)* | 进入规划模式，在实现前获取用户批准 |
| **TodoWrite** ★ | Update the todo list for the current session. To be used proactively and often to track progress and pending tasks. Make sure that at least one task is in_progress at all times. | 管理当前会话的任务列表 |
| Read | Read a file from the local filesystem. | 读取本地文件 |
| Edit | Performs exact string replacements in files. | 精确字符串替换编辑文件 |
| Write | Write a file to the local filesystem. | 向本地文件系统写入文件 |
| Glob | Fast file pattern matching tool that works with any codebase size | 快速文件模式匹配 |
| Grep | A powerful search tool built on ripgrep | 基于 ripgrep 的强大搜索工具 |
| WebFetch | Fetches content from a specified URL and processes it using an AI model | 抓取 URL 内容并用小模型处理 |
| WebSearch | Allows Claude to search the web and use the results to inform responses | 网络搜索 |
| AskUserQuestion | Asks the user multiple choice questions to gather information, clarify ambiguity, understand preferences, make decisions or offer them choices. | 向用户提出多选题 |
| ExitPlanMode | Use this tool when you are in plan mode and have finished writing your plan | 退出规划模式请求用户审批 |
| TaskCreate | Create a new task in the task list | 创建任务 |
| TaskUpdate | Update a task in the task list | 更新任务状态 |
| TaskList | List all tasks in the task list | 列出所有任务 |
| TaskGet | Get a task by ID from the task list | 按 ID 获取任务 |
| TaskStop | *(停止任务执行)* | 停止任务 |
| SendMessage | Send a message to another agent | 向另一个 Agent 发送消息 |
| TeamCreate | *(创建多 Agent 协作团队)* | 创建 Agent 团队 |
| EnterWorktree | *(创建隔离的 git worktree)* | 进入 worktree 隔离环境 |
| ExitWorktree | Exit a worktree session created by EnterWorktree | 退出 worktree 会话 |
| ConfigTool | Get or set Claude Code configuration settings. | 读写 Claude Code 配置 |
| CronCreate | Schedule a prompt to run at a future time | 计划未来执行的任务 |
| CronDelete | Cancel a scheduled cron job by ID | 取消计划任务 |
| CronList | List scheduled cron jobs | 列出计划任务 |
| BriefTool | Send a message to the user | 向用户发送消息（自主模式专用） |
| SleepTool | Wait for a specified duration | 等待指定时长 |
| LSPTool | Interact with Language Server Protocol (LSP) servers to get code intelligence features. | 与 LSP 服务器交互获取代码智能 |
| ToolSearch | *(加载延迟工具的完整 schema)* | 延迟工具 schema 加载 |
| PowerShellTool | *(Windows PowerShell 等价的 Bash)* | Windows PowerShell 执行 |
| NotebookEdit | *(Jupyter notebook 编辑)* | 编辑 Jupyter notebook |
| MCPTool | *(MCP 工具调用)* | MCP 工具 |
| RemoteTrigger | *(远程触发器)* | 远程触发 |
| ListMcpResources | *(列出 MCP 资源)* | 列出 MCP 资源 |
| ReadMcpResource | *(读取 MCP 资源)* | 读取 MCP 资源 |

---

## 5.1 第一档工具：完整深析

---

### 5.1.1 BashTool：Shell 执行与 Git 安全协议

BashTool 是整个工具体系中最复杂的一个，源码 369 行，是第二大工具（AgentTool 287 行）的 1.3 倍。它的复杂度来自三个叠加维度：工具使用规范、沙箱约束、git 操作完整协议。

---

**英文原文：核心描述与工具优先级**（来源：`src/tools/BashTool/prompt.ts`，第 275–363 行，`getSimplePrompt()` 函数）

```
Executes a given bash command and returns its output.

The working directory persists between commands, but shell state does not. The shell environment is initialized from the user's profile (bash or zsh).

IMPORTANT: Avoid using this tool to run `find`, `grep`, `cat`, `head`, `tail`, `sed`, `awk`, or `echo` commands, unless explicitly instructed or after you have verified that a dedicated tool cannot accomplish your task. Instead, use the appropriate dedicated tool as this will provide a much better experience for the user:

 - File search: Use Glob (NOT find or ls)
 - Content search: Use Grep (NOT grep or rg)
 - Read files: Use Read (NOT cat/head/tail)
 - Edit files: Use Edit (NOT sed/awk)
 - Write files: Use Write (NOT echo >/cat <<EOF)
 - Communication: Output text directly (NOT echo/printf)
While the Bash tool can do similar things, it's better to use the built-in tools as they provide a better user experience and make it easier to review tool calls and give permission.
```

> **中文附注**：
>
> 执行给定的 bash 命令并返回其输出。
>
> 工作目录在命令之间持续存在，但 shell 状态不会。shell 环境从用户的配置文件（bash 或 zsh）初始化。
>
> **重要**：避免使用此工具运行 `find`、`grep`、`cat`、`head`、`tail`、`sed`、`awk` 或 `echo` 命令，除非被明确要求或在确认专用工具无法完成任务之后。否则请使用适当的专用工具，这将为用户提供更好的体验：
>
> - 文件搜索：使用 Glob（不是 find 或 ls）
> - 内容搜索：使用 Grep（不是 grep 或 rg）
> - 读取文件：使用 Read（不是 cat/head/tail）
> - 编辑文件：使用 Edit（不是 sed/awk）
> - 写入文件：使用 Write（不是 echo >/cat <<EOF）
> - 通信：直接输出文字（不是 echo/printf）
>
> 虽然 Bash 工具可以做类似的事情，但使用内置工具更好，因为它们提供了更好的用户体验，更容易审查工具调用和授予权限。

**设计解析**

第一句话"working directory persists between commands, but shell state does not"是一个关键的技术说明——它预防了一个真实的误用：有些开发者（包括模型）会假设在一个 Bash 调用里设置的环境变量（`export FOO=bar`）能在下一次调用里继续生效，但实际上不会。每次 Bash 调用都是一个独立的 shell 进程。

工具优先级列表使用了"NOT X"的对比格式，而非简单的"use Y"。这是深思熟虑的选择：`NOT cat/head/tail`、`NOT grep or rg`——精确命名的是模型最可能自然选择的那些命令，而不是泛泛说"不要用系统命令"。模型在选择命令时会有强烈的内部关联（`read file → cat`），精确命名 `cat` 能在激活层面建立精准的抑制。

注意最后的"Communication: Output text directly (NOT echo/printf)"——这是通信手段的规范化。模型可能会用 `echo "Here is the result"` 这种方式输出信息，但这会把通信行为藏在 Bash 工具调用里，用户只能看到 Bash 结果，而非直接的模型输出。

`→ 原则⑥：禁止+替代` — "NOT cat/head/tail → use Read"：每条禁令立即跟一个可执行的替代选项，不让模型陷入"那用什么"的歧义，见第七章 6.1 节。

---

**英文原文：指令条目**（来源：`src/tools/BashTool/prompt.ts`，第 331–352 行）

```
# Instructions
 - If your command will create new directories or files, first use this tool to run `ls` to verify the parent directory exists and is the correct location.
 - Always quote file paths that contain spaces with double quotes in your command (e.g., cd "path with spaces/file.txt")
 - Try to maintain your current working directory throughout the session by using absolute paths and avoiding usage of `cd`. You may use `cd` if the User explicitly requests it.
 - You may specify an optional timeout in milliseconds (up to 600000ms / 10 minutes). By default, your command will timeout after 120000ms (2 minutes).
 - You can use the `run_in_background` parameter to run the command in the background. Only use this if you don't need the result immediately and are OK being notified when the command completes later. You do not need to check the output right away - you'll be notified when it finishes. You do not need to use '&' at the end of the command when using this parameter.
 - When issuing multiple commands:
   - If the commands are independent and can run in parallel, make multiple Bash tool calls in a single message. Example: if you need to run "git status" and "git diff", send a single message with two Bash tool calls in parallel.
   - If the commands depend on each other and must run sequentially, use a single Bash call with '&&' to chain them together.
   - Use ';' only when you need to run commands sequentially but don't care if earlier commands fail.
   - DO NOT use newlines to separate commands (newlines are ok in quoted strings).
 - For git commands:
   - Prefer to create a new commit rather than amending an existing commit.
   - Before running destructive operations (e.g., git reset --hard, git push --force, git checkout --), consider whether there is a safer alternative that achieves the same goal. Only use destructive operations when they are truly the best approach.
   - Never skip hooks (--no-verify) or bypass signing (--no-gpg-sign, -c commit.gpgsign=false) unless the user has explicitly asked for it. If a hook fails, investigate and fix the underlying issue.
 - Avoid unnecessary `sleep` commands:
   - Do not sleep between commands that can run immediately — just run them.
   - If your command is long running and you would like to be notified when it finishes — use `run_in_background`. No sleep needed.
   - Do not retry failing commands in a sleep loop — diagnose the root cause.
   - If waiting for a background task you started with `run_in_background`, you will be notified when it completes — do not poll.
   - If you must poll an external process, use a check command (e.g. `gh run view`) rather than sleeping first.
   - If you must sleep, keep the duration short (1-5 seconds) to avoid blocking the user.
```

> **中文附注**：
>
> **# 指令**
>
> - 如果你的命令将创建新目录或文件，首先用此工具运行 `ls` 来验证父目录存在且是正确的位置。
> - 在命令中始终用双引号引用包含空格的文件路径（例如，`cd "path with spaces/file.txt"`）
> - 尝试在整个会话中通过使用绝对路径并避免使用 `cd` 来保持你的当前工作目录。如果用户明确要求，你可以使用 `cd`。
> - 你可以指定可选的超时时间（毫秒，最多 600000ms / 10 分钟）。默认情况下，你的命令将在 120000ms（2 分钟）后超时。
> - 你可以使用 `run_in_background` 参数在后台运行命令。只有在不需要立即获得结果且可以接受稍后收到通知的情况下才使用这个。你不需要立即检查输出——完成时会收到通知。使用此参数时，你不需要在命令末尾使用 `&`。
> - 在发出多个命令时：
>   - 如果命令是独立的且可以并行运行，在一条消息中发出多个 Bash 工具调用。例如：如果你需要运行 "git status" 和 "git diff"，发送一条包含两个并行 Bash 工具调用的消息。
>   - 如果命令相互依赖且必须顺序运行，使用单个带 `&&` 的 Bash 调用链接它们。
>   - 只有在需要顺序运行但不关心之前命令是否失败时才使用 `;`。
>   - **不要**使用换行符分隔命令（在带引号的字符串中换行是可以的）。
> - 对于 git 命令：
>   - 优先创建新提交而不是修改现有提交。
>   - 在运行破坏性操作（如 `git reset --hard`、`git push --force`、`git checkout --`）之前，考虑是否有更安全的替代方案。只有在确实是最佳方法时才使用破坏性操作。
>   - 除非用户明确要求，否则不要跳过钩子（`--no-verify`）或绕过签名（`--no-gpg-sign`、`-c commit.gpgsign=false`）。如果钩子失败，调查并修复底层问题。
> - 避免不必要的 `sleep` 命令：
>   - 不要在可以立即运行的命令之间睡眠——直接运行它们。
>   - 如果你的命令是长时间运行的且想在完成时收到通知——使用 `run_in_background`。不需要睡眠。
>   - 不要在睡眠循环中重试失败的命令——诊断根本原因。
>   - 如果等待用 `run_in_background` 启动的后台任务，完成时会收到通知——不要轮询。
>   - 如果必须轮询外部进程，先使用检查命令（如 `gh run view`）而不是先睡眠。
>   - 如果必须睡眠，保持时间较短（1-5 秒）以避免阻塞用户。

**设计解析**

**多命令决策树** 是这段指令里最精彩的设计。三种情况对应三种不同的命令组合策略：

| 情况 | 策略 | 例子 |
|---|---|---|
| 独立命令，顺序无关 | 并行 Bash 工具调用 | git status + git diff → 两个并行调用 |
| 依赖命令，顺序有关 | 单个调用 + `&&` 链接 | build && test |
| 顺序但不关心前序结果 | 单个调用 + `;` 分隔 | cleanup ; run |

这个决策树解决了一个实际的执行效率问题：`git status` 和 `git diff` 完全独立，完全可以并行；但如果用一个 Bash 调用里用 `&&` 或 `;` 串联，就变成串行了。"如果你需要运行 'git status' 和 'git diff'，发送一条包含两个并行 Bash 工具调用的消息"——这个具体示例直接教会了模型正确的并行行为。

**`run_in_background` 参数的两个要点** 是并行能力的补充：
1. "不需要在命令末尾使用 `&`"——消除了模型可能有的"后台=在命令后加`&`"的错误联想
2. "不要轮询"——防止了一种常见的低效循环：启动后台任务 → sleep → 检查 → sleep → 检查

**Sleep 避免规则的细度** 值得单独分析。这一小节有 6 条规则，全部是关于"什么时候不能 sleep"，只有一条"如果必须 sleep，保持短"。这种密集的反面规范反映了一个真实的模型行为问题：模型在需要等待时，自然倾向于插入 `sleep N` 来"等一会儿"，这会：
- 阻塞整个会话（用户看到工具在执行，但什么也没发生）
- 浪费 API 调用轮次
- 掩盖根本原因（失败→sleep 重试≠理解为什么失败）

这 6 条规则几乎覆盖了 sleep 可能被滥用的所有场景。

---

**英文原文：Git Commit 安全协议**（来源：`src/tools/BashTool/prompt.ts`，第 82–125 行，仅外部用户）

```
# Committing changes with git

Only create commits when requested by the user. If unclear, ask first. When the user asks you to create a new git commit, follow these steps carefully:

You can call multiple tools in a single response. When multiple independent pieces of information are requested and all commands are likely to succeed, run multiple tool calls in parallel for optimal performance. The numbered steps below indicate which commands should be batched in parallel.

Git Safety Protocol:
- NEVER update the git config
- NEVER run destructive git commands (push --force, reset --hard, checkout ., restore ., clean -f, branch -D) unless the user explicitly requests these actions. Taking unauthorized destructive actions is unhelpful and can result in lost work, so it's best to ONLY run these commands when given direct instructions 
- NEVER skip hooks (--no-verify, --no-gpg-sign, etc) unless the user explicitly requests it
- NEVER run force push to main/master, warn the user if they request it
- CRITICAL: Always create NEW commits rather than amending, unless the user explicitly requests a git amend. When a pre-commit hook fails, the commit did NOT happen — so --amend would modify the PREVIOUS commit, which may result in destroying work or losing previous changes. Instead, after hook failure, fix the issue, re-stage, and create a NEW commit
- When staging files, prefer adding specific files by name rather than using "git add -A" or "git add .", which can accidentally include sensitive files (.env, credentials) or large binaries
- NEVER commit changes unless the user explicitly asks you to. It is VERY IMPORTANT to only commit when explicitly asked, otherwise the user will feel that you are being too proactive

1. Run the following bash commands in parallel, each using the Bash tool:
  - Run a git status command to see all untracked files. IMPORTANT: Never use the -uall flag as it can cause memory issues on large repos.
  - Run a git diff command to see both staged and unstaged changes that will be committed.
  - Run a git log command to see recent commit messages, so that you can follow this repository's commit message style.
2. Analyze all staged changes (both previously staged and newly added) and draft a commit message:
  - Summarize the nature of the changes (eg. new feature, enhancement to an existing feature, bug fix, refactoring, test, docs, etc.). Ensure the message accurately reflects the changes and their purpose (i.e. "add" means a wholly new feature, "update" means an enhancement to an existing feature, "fix" means a bug fix, etc.)
  - Do not commit files that likely contain secrets (.env, credentials.json, etc). Warn the user if they specifically request to commit those files
  - Draft a concise (1-2 sentences) commit message that focuses on the "why" rather than the "what"
  - Ensure it accurately reflects the changes and their purpose
3. Run the following commands in parallel:
   - Add relevant untracked files to the staging area.
   - Create the commit with a message ending with:
   Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
   - Run git status after the commit completes to verify success.
   Note: git status depends on the commit completing, so run it sequentially after the commit.
4. If the commit fails due to pre-commit hook: fix the issue and create a NEW commit

Important notes:
- NEVER run additional commands to read or explore code, besides git bash commands
- NEVER use the TodoWrite or Agent tools
- DO NOT push to the remote repository unless the user explicitly asks you to do so
- IMPORTANT: Never use git commands with the -i flag (like git rebase -i or git add -i) since they require interactive input which is not supported.
- IMPORTANT: Do not use --no-edit with git rebase commands, as the --no-edit flag is not a valid option for git rebase.
- If there are no changes to commit (i.e., no untracked files and no modifications), do not create an empty commit
- In order to ensure good formatting, ALWAYS pass the commit message via a HEREDOC, a la this example:
<example>
git commit -m "$(cat <<'EOF'
   Commit message here.

   Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
   EOF
   )"
</example>
```

> **中文附注**：
>
> **# 使用 git 提交更改**
>
> 只在用户请求时创建提交。如果不确定，先询问。当用户要求创建新的 git 提交时，请仔细按以下步骤操作：
>
> *(Git 安全协议七条规则 — 见正文分析)*
>
> 1. 并行运行以下 bash 命令（每个都用 Bash 工具）：
>    - 运行 git status 查看所有未追踪文件。**重要**：不要使用 -uall 标志，因为它可能导致大型仓库内存问题。
>    - 运行 git diff 查看将被提交的已暂存和未暂存更改。
>    - 运行 git log 查看最近的提交消息，以便跟随此仓库的提交消息风格。
> 2. 分析所有已暂存的更改并起草提交消息（专注于"为什么"而非"什么"）
> 3. 并行运行：添加未追踪文件 + 创建带 Co-Authored-By 署名的提交 + 验证的 git status
> 4. 如果因 pre-commit hook 失败：修复问题并创建新提交

**设计解析**

**Git Safety Protocol 是一个正式的安全协议，而非建议列表。** 7 条规则使用 `NEVER`/`CRITICAL` 等强调语级别的修饰词，结构上类似于安全系统的操作规程（SOP），而不是普通的提示。这种语气选择是有意的：git 操作的后果（丢失工作、破坏共享分支）是不可逆的，需要特别高的约束强度。

**"hook 失败→创建新提交"的 CRITICAL 规则** 解决了一个真实的、严重的失败模式：

```
CRITICAL: Always create NEW commits rather than amending, unless the user explicitly
requests a git amend. When a pre-commit hook fails, the commit did NOT happen — so
--amend would modify the PREVIOUS commit, which may result in destroying work or losing
previous changes.
```

这条规则的逻辑非常精确：`--amend` 在 hook 失败时（提交没有完成）会修改的是**前一个**提交，而不是失败的那个。如果没有这条规则，模型可能会推断"提交失败→修复→`git commit --amend`"，结果是悄悄修改了一个与当前任务完全无关的历史提交。

**HEREDOC 要求的工程原因** 不只是格式问题：

```
In order to ensure good formatting, ALWAYS pass the commit message via a HEREDOC
```

提交消息里的 Co-Authored-By 署名包含特定的格式（空行+具体格式），在 shell 里直接用 `-m "..."` 传递多行字符串时，不同 shell 和转义规则差异可能导致格式错误。HEREDOC 绕过了这个问题，确保提交消息的格式完全可预测。

**"-uall 标志"的具体禁止** 是细节的极致：

```
IMPORTANT: Never use the -uall flag as it can cause memory issues on large repos.
```

`git status -uall` 会递归列出所有未追踪文件，在有大量未追踪文件的仓库（如包含 `node_modules` 或 `build` 目录的仓库）中会导致内存爆炸。这条规则来自于真实的生产事故——在大型代码库上跑这个命令会导致 Claude Code 进程 OOM。

**"步骤式协议"的设计意图** 值得专门分析。整个 git commit 过程被分解为 4 个明确标号的步骤，每步内部再细分为并行/顺序子操作。这是"SOP 化"的 prompt 设计：把原本隐式的操作序列变成显式的、有顺序约束的协议。模型不需要推断"先读还是先写"，一切都已经规定好了。步骤 1 和步骤 3 的"并行"明确标注，让模型能最大化利用并发能力。

`→ 原则③：具体可操作` — "NEVER `--no-verify`""NEVER force push to main/master" 等逐字枚举 + HEREDOC 要求，规则具体到命令级别，不可有歧义，见第七章 3.1 节。

---

**英文原文：PR 创建协议**（来源：`src/tools/BashTool/prompt.ts`，第 127–160 行）

```
# Creating pull requests
Use the gh command via the Bash tool for ALL GitHub-related tasks including working with issues, pull requests, checks, and releases. If given a Github URL use the gh command to get the information needed.

IMPORTANT: When the user asks you to create a pull request, follow these steps carefully:

1. Run the following bash commands in parallel using the Bash tool, in order to understand the current state of the branch since it diverged from the main branch:
   - Run a git status command to see all untracked files (never use -uall flag)
   - Run a git diff command to see both staged and unstaged changes that will be committed
   - Check if the current branch tracks a remote branch and is up to date with the remote, so you know if you need to push to the remote
   - Run a git log command and `git diff [base-branch]...HEAD` to understand the full commit history for the current branch (from the time it diverged from the base branch)
2. Analyze all changes that will be included in the pull request, making sure to look at all relevant commits (NOT just the latest commit, but ALL commits that will be included in the pull request!!!), and draft a pull request title and summary:
   - Keep the PR title short (under 70 characters)
   - Use the description/body for details, not the title
3. Run the following commands in parallel:
   - Create new branch if needed
   - Push to remote with -u flag if needed
   - Create PR using gh pr create with the format below. Use a HEREDOC to pass the body to ensure correct formatting.
<example>
gh pr create --title "the pr title" --body "$(cat <<'EOF'
## Summary
<1-3 bullet points>

## Test plan
[Bulleted markdown checklist of TODOs for testing the pull request...]

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
</example>

Important:
- DO NOT use the TodoWrite or Agent tools
- Return the PR URL when you're done, so the user can see it

# Other common operations
- View comments on a Github PR: gh api repos/foo/bar/pulls/123/comments
```

> **中文附注**：
>
> **# 创建 pull request**
>
> 对于所有 GitHub 相关任务，使用 Bash 工具通过 `gh` 命令处理。给定 GitHub URL 时使用 `gh` 命令获取所需信息。
>
> **重要**：当用户要求创建 pull request 时，请仔细按以下步骤操作：
>
> 1. 并行运行 4 个命令，理解分支自分叉以来的完整状态：git status、git diff、远程追踪状态、git log + `git diff [base-branch]...HEAD`
> 2. 分析所有将包含在 PR 中的更改（**不只是最新提交，而是所有提交!!!**），起草标题和摘要
> 3. 并行运行：创建分支（如需）、推送（如需）、用 HEREDOC 格式的 gh pr create 创建 PR

**设计解析**

PR 创建协议有一处设计非常值得注意：步骤 2 里的三个感叹号——

```
making sure to look at all relevant commits (NOT just the latest commit, but ALL commits
that will be included in the pull request!!!)
```

三个感叹号在 Anthropic 的 prompt 里极其罕见。它们的出现标志着这里存在一个被反复犯的错误：模型在起草 PR 描述时，倾向于只分析 `git diff HEAD~1` 的内容（最新一次提交），而不是整个 feature 分支相对于 base 分支的全部变更。这导致 PR 描述只覆盖了最后一个提交，而遗漏了整个 PR 的实际内容。三个感叹号是"这个错误发生过很多次"的直接证据。

---

### 5.1.2 AgentTool：子 Agent 编排与 Fork 模式

**英文原文：共享核心描述**（来源：`src/tools/AgentTool/prompt.ts`，第 202–212 行）

```
Launch a new agent to handle complex, multi-step tasks autonomously.

The Agent tool launches specialized agents (subprocesses) that autonomously handle complex tasks. Each agent type has specific capabilities and tools available to it.

Available agent types are listed in <system-reminder> messages in the conversation.

When using the Agent tool, specify a subagent_type to use a specialized agent, or omit it to fork yourself — a fork inherits your full conversation context.
```

> **中文附注**：
>
> 启动一个新 Agent 来自主处理复杂的多步骤任务。
>
> Agent 工具启动专门的 Agent（子进程），自主处理复杂任务。每种 Agent 类型都有特定的能力和可用工具。
>
> 可用的 Agent 类型列在对话中的 `<system-reminder>` 消息里。
>
> 使用 Agent 工具时，指定 `subagent_type` 以使用专门的 Agent，或省略它来 fork 自己——fork 继承你的完整对话上下文。

**设计解析**

注意代理列表的这段话：`Available agent types are listed in <system-reminder> messages in the conversation.`

这是第三章介绍的"agent list 注入附件"机制在工具 prompt 里的体现。当 `shouldInjectAgentListInMessages()` 返回 true（默认行为），agent 列表不再嵌入在工具的 description 字段里，而是通过独立的消息附件注入。这样做的原因代码注释说得很清楚：

```typescript
// The dynamic agent list was ~10.2% of fleet cache_creation tokens:
// MCP async connect, /reload-plugins, or permission-mode changes mutate the list →
// description changes → full tool-schema cache bust.
```

agent 列表动态变化（MCP 服务器加载、权限变化）会导致工具 schema 缓存全部失效，代价是 10.2% 的 cache_creation token。把列表移到消息附件后，工具 description 保持静态，缓存稳定。一个技术优化决策直接重塑了 prompt 的结构。

---

**英文原文："当不该用 Agent"规则**（来源：`src/tools/AgentTool/prompt.ts`，第 232–240 行）

```
When NOT to use the Agent tool:
- If you want to read a specific file path, use the Read tool or the Glob tool instead of the Agent tool, to find the match more quickly
- If you are searching for a specific class definition like "class Foo", use the Glob tool instead, to find the match more quickly
- If you are searching for code within a specific file or set of 2-3 files, use the Read tool instead of the Agent tool, to find the match more quickly
- Other tasks that are not related to the agent descriptions above
```

> **中文附注**：
>
> **何时不使用 Agent 工具**：
>
> - 如果你想读取特定文件路径，使用 Read 工具或 Glob 工具，而不是 Agent 工具，以更快找到匹配
> - 如果你在搜索特定的类定义如"class Foo"，使用 Glob 工具而不是 Agent 工具，以更快找到匹配
> - 如果你在特定文件或 2-3 个文件集合中搜索代码，使用 Read 工具而不是 Agent 工具，以更快找到匹配
> - 其他与上述 Agent 描述不相关的任务

**设计解析**

"当不使用 Agent"的规则是工具 prompt 中最典型的"过度使用防止"设计。Agent 工具是昂贵且慢的（需要启动子进程、独立的 API 调用链），而模型有一种自然倾向：一旦知道 Agent 强大，就会倾向于用 Agent 解决一切问题。

每个"不使用"的条目都附有具体的替代方案和理由（"更快找到匹配"），而非抽象的"Agent 太重"。这是"禁止+理由+替代方案"三元组的体现：不只禁止，还解释为什么，并且给出出路。

---

**英文原文：Fork 模式专属指导**（来源：`src/tools/AgentTool/prompt.ts`，第 81–96 行）

```
## When to fork

Fork yourself (omit `subagent_type`) when the intermediate tool output isn't worth keeping in your context. The criterion is qualitative — "will I need this output again" — not task size.
- **Research**: fork open-ended questions. If research can be broken into independent questions, launch parallel forks in one message. A fork beats a fresh subagent for this — it inherits context and shares your cache.
- **Implementation**: prefer to fork implementation work that requires more than a couple of edits. Do research before jumping to implementation.

Forks are cheap because they share your prompt cache. Don't set `model` on a fork — a different model can't reuse the parent's cache. Pass a short `name` (one or two words, lowercase) so the user can see the fork in the teams panel and steer it mid-run.

**Don't peek.** The tool result includes an `output_file` path — do not Read or tail it unless the user explicitly asks for a progress check. You get a completion notification; trust it. Reading the transcript mid-flight pulls the fork's tool noise into your context, which defeats the point of forking.

**Don't race.** After launching, you know nothing about what the fork found. Never fabricate or predict fork results in any format — not as prose, summary, or structured output. The notification arrives as a user-role message in a later turn; it is never something you write yourself. If the user asks a follow-up before the notification lands, tell them the fork is still running — give status, not a guess.

**Writing a fork prompt.** Since the fork inherits your context, the prompt is a *directive* — what to do, not what the situation is. Be specific about scope: what's in, what's out, what another agent is handling. Don't re-explain background.
```

> **中文附注**：
>
> **## 何时 fork**
>
> 当中间工具输出不值得保留在你的上下文中时，fork 自己（省略 `subagent_type`）。判断标准是定性的——"我还会再需要这个输出吗"——而不是任务大小。
>
> - **研究**：fork 开放式问题。如果研究可以分解为独立问题，在一条消息中启动并行 fork。fork 比新 subagent 更适合这种情况——它继承上下文并共享缓存。
> - **实现**：优先 fork 需要超过几次编辑的实现工作。在跳到实现之前先做研究。
>
> fork 是廉价的，因为它们共享 prompt 缓存。不要在 fork 上设置 `model`——不同的模型无法重用父缓存。传递一个短 `name`（一两个词，小写），以便用户可以在 teams 面板中看到 fork 并在运行中引导它。
>
> **不要偷看。** 工具结果包含一个 `output_file` 路径——除非用户明确要求进度检查，否则不要 Read 或 tail 它。你会收到完成通知；相信它。在运行中读取记录会将 fork 的工具噪音拉入你的上下文，这违背了 fork 的目的。
>
> **不要抢跑。** 启动后，你对 fork 找到了什么一无所知。永远不要以任何格式伪造或预测 fork 结果——不是散文、摘要或结构化输出。通知以用户角色消息的形式出现在后续轮次中；它永远不是你自己写的东西。如果用户在通知到来之前询问后续问题，告诉他们 fork 仍在运行——给出状态，而不是猜测。
>
> **写 fork prompt。** 由于 fork 继承你的上下文，prompt 是一个**指令**——做什么，而不是情况是什么。具体说明范围：什么在范围内，什么在范围外，另一个 Agent 在处理什么。不要重新解释背景。

**设计解析**

Fork 模式的三条粗体规则——"**Don't peek**"、"**Don't race**"、"**Writing a fork prompt**"——各自对应一种真实的 fork 误用模式：

**Don't peek** 防止的是：模型在 fork 运行时好奇地读取 fork 的输出文件来"检查进度"。这个行为会将 fork 的所有工具调用输出拉入父 agent 的上下文——正是 fork 的目的就是**不**让这些输出进入父上下文（因为它们通常是大量的中间探索输出，以后不需要）。

**Don't race** 防止的是：模型在 fork 还未完成时，基于推测提前说"fork 找到了 X"。这是一种幻觉行为——模型用自己的内部推断来填充 fork 结果的空白。规则明确要求："notified arrives as a user-role message in a later turn; it is never something you write yourself."——模型的职责是等待，不是预测。

**Writing a fork prompt** 解决的是：fork 因为继承了父上下文，所以 prompt 不需要是"briefing"（背景介绍），而应该是"directive"（指令）。这条规则配合"Don't re-explain background"——因为背景已经在上下文里了，重复解释是浪费 token。

---

**英文原文：写 Agent Prompt 的指导**（来源：`src/tools/AgentTool/prompt.ts`，第 99–113 行）

```
## Writing the prompt

When spawning a fresh agent (with a `subagent_type`), it starts with zero context. Brief the agent like a smart colleague who just walked into the room — it hasn't seen this conversation, doesn't know what you've tried, doesn't understand why this task matters.
- Explain what you're trying to accomplish and why.
- Describe what you've already learned or ruled out.
- Give enough context about the surrounding problem that the agent can make judgment calls rather than just following a narrow instruction.
- If you need a short response, say so ("report in under 200 words").
- Lookups: hand over the exact command. Investigations: hand over the question — prescribed steps become dead weight when the premise is wrong.

For fresh agents, terse command-style prompts produce shallow, generic work.

**Never delegate understanding.** Don't write "based on your findings, fix the bug" or "based on the research, implement it." Those phrases push synthesis onto the agent instead of doing it yourself. Write prompts that prove you understood: include file paths, line numbers, what specifically to change.
```

> **中文附注**：
>
> **## 写 prompt**
>
> 当生成新 Agent（带 `subagent_type`）时，它从零上下文开始。像对待一个刚走进房间的聪明同事一样简报 Agent——它没有看过这次对话，不知道你尝试了什么，不理解为什么这个任务很重要。
>
> - 解释你想完成什么以及为什么。
> - 描述你已经学到或排除的内容。
> - 提供足够的关于周围问题的上下文，使 Agent 能够做出判断，而不只是遵循狭窄的指令。
> - 如果你需要简短的回应，说出来（"报告在 200 字以内"）。
> - 查找：交出确切的命令。调查：交出问题——当前提有误时，规定的步骤成了累赘。
>
> 对于新 Agent，简洁的命令式 prompt 会产生浅薄、通用的工作。
>
> **永远不要委托理解。** 不要写"根据你的发现，修复 bug"或"根据研究，实现它。"这些短语将综合的工作推给 Agent，而不是你自己来做。写能证明你理解的 prompt：包含文件路径、行号、具体要更改什么。

**设计解析**

"像对待一个刚走进房间的聪明同事一样简报 Agent"——这是全书中最具指导性的 prompt 写作隐喻。它在一句话里传达了关于 sub-agent prompt 质量的全部要求：
- 聪明的→不需要解释基础概念，不要过度指定步骤
- 刚走进房间→不了解背景，需要完整说明
- 同事→可以做判断，不只是执行命令

**"Never delegate understanding"** 是这一节最重要的设计原则。它精确命名了 agent 编排中一种常见的架构失败：协调者（coordinator）收到子 agent 的研究报告后，写一条消息给另一个子 agent："根据上面的研究，实现它。"这个做法看起来合理，实际上是把综合理解的工作推给了子 agent——而那个子 agent 没有足够的上下文来正确地综合，结果是错误的或通用的实现。

"Write prompts that prove you understood: include file paths, line numbers, what specifically to change"——这三个具体要求（文件路径、行号、具体更改）定义了"理解过"的prompt的最低标准。

`→ 原则⑦：认知框架` — "刚走进房间的聪明同事"用一句话传达了三个维度的 sub-agent prompt 质量要求，超过任何规则列表的记忆效率，见第七章 7.1 节。

---

### 5.1.3 SkillTool：Skill 调用协议

**英文原文**（来源：`src/tools/SkillTool/prompt.ts`，第 173–196 行）

```
Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills match. Skills provide specialized capabilities and domain knowledge.

When users reference a "slash command" or "/<something>" (e.g., "/commit", "/review-pr"), they are referring to a skill. Use this tool to invoke it.

How to invoke:
- Use this tool with the skill name and optional arguments
- Examples:
  - `skill: "pdf"` - invoke the pdf skill
  - `skill: "commit", args: "-m 'Fix bug'"` - invoke with arguments
  - `skill: "review-pr", args: "123"` - invoke with arguments
  - `skill: "ms-office-suite:pdf"` - invoke using fully qualified name

Important:
- Available skills are listed in system-reminder messages in the conversation
- When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response about the task
- NEVER mention a skill without actually calling this tool
- Do not invoke a skill that is already running
- Do not use this tool for built-in CLI commands (like /help, /clear, etc.)
- If you see a <command-name> tag in the current conversation turn, the skill has ALREADY been loaded - follow the instructions directly instead of calling this tool again
```

> **中文附注**：
>
> 在主对话中执行 Skill
>
> 当用户要求执行任务时，检查是否有任何可用的 Skill 匹配。Skill 提供专门的能力和领域知识。
>
> 当用户引用"斜线命令"或"/<某物>"（如 "/commit"、"/review-pr"）时，他们指的是一个 Skill。使用此工具来调用它。
>
> **重要**：
>
> - 可用的 Skill 列在对话中的 system-reminder 消息里
> - 当 Skill 匹配用户的请求时，这是一个**阻塞要求**：在对任务生成任何其他响应之前调用相关的 Skill 工具
> - **永远不要提及 Skill 而不实际调用此工具**
> - 不要调用已经在运行的 Skill
> - 不要将此工具用于内置 CLI 命令（如 /help、/clear 等）
> - 如果在当前对话轮次中看到 `<command-name>` 标签，Skill 已经被加载——直接按照指令操作，而不是再次调用此工具

**设计解析**

**"BLOCKING REQUIREMENT"** 是整个工具 prompt 体系中唯一使用这个词组的地方。普通的限制用 "IMPORTANT" 或 "NEVER"，而 Skill 的调用要求用的是 "BLOCKING"——这个词来自并发编程，意味着"在这个条件满足之前，不能做任何其他事情"。

这种强度的选择是有原因的：模型在识别到用户意图后，有时会"先解释"——"我看到你想用 /commit，这个命令会......"——然后才调用工具。这种行为在 Skill 系统里是有害的，因为 Skill 可能会动态替换整个响应框架，提前的解释会成为多余的噪音，甚至与 Skill 生成的内容冲突。BLOCKING REQUIREMENT 直接禁止了这种"先说话再行动"的顺序。

配合后面的"NEVER mention a skill without actually calling this tool"，这两条规则形成了一个完整的闭环：不要提及 → 调用工具 → Skill 生成响应。

**`<command-name>` 标签的检测** 是防止重复调用的机制。当 Skill 被调用后，其内容会用 `<command-name>` 标签注入到当前上下文中。模型如果检测到这个标签，就知道"这个 Skill 已经在这一轮被调用过了"，直接按照 Skill 的指令操作，而不是再次触发调用——防止了调用循环。

**Budget 管理的工程设计** 值得单独一提。Skill 列表使用了一个精密的预算系统：

```typescript
// Skill listing gets 1% of the context window (in characters)
export const SKILL_BUDGET_CONTEXT_PERCENT = 0.01
```

当 Skill 数量过多时，系统会按优先级截断：bundled（内置）Skill 始终保留完整描述；用户安装的 Skill 按预算比例截断描述；极端情况下退化到纯名称列表。这是在"模型能看到足够信息做决策"和"不把上下文窗口的 1% 以上用于工具目录"之间的精确权衡。

---

### 5.1.4 EnterPlanModeTool：规划模式状态机

EnterPlanModeTool 的设计特点是内外两套完全不同的 prompt——不是在措辞上的调整，而是在**触发条件**和**默认倾向**上的根本性差异。

**英文原文（外部用户版）**（来源：`src/tools/EnterPlanModeTool/prompt.ts`，第 22–98 行）

```
Use this tool proactively when you're about to start a non-trivial implementation task. Getting user sign-off on your approach before writing code prevents wasted effort and ensures alignment. This tool transitions you into plan mode where you can explore the codebase and design an implementation approach for user approval.

## When to Use This Tool

**Prefer using EnterPlanMode** for implementation tasks unless they're simple. Use it when ANY of these conditions apply:

1. **New Feature Implementation**: Adding meaningful new functionality
2. **Multiple Valid Approaches**: The task can be solved in several different ways
3. **Code Modifications**: Changes that affect existing behavior or structure
4. **Architectural Decisions**: The task requires choosing between patterns or technologies
5. **Multi-File Changes**: The task will likely touch more than 2-3 files
6. **Unclear Requirements**: You need to explore before understanding the full scope
7. **User Preferences Matter**: The implementation could reasonably go multiple ways
   - If you would use AskUserQuestion to clarify the approach, use EnterPlanMode instead
   - Plan mode lets you explore first, then present options with context

## When NOT to Use This Tool

Only skip EnterPlanMode for simple tasks:
- Single-line or few-line fixes (typos, obvious bugs, small tweaks)
- Adding a single function with clear requirements
- Tasks where the user has given very specific, detailed instructions
- Pure research/exploration tasks (use the Agent tool with explore agent instead)
```

> **中文附注（外部用户）**：
>
> 当你即将开始非平凡的实现任务时，**主动**使用这个工具。在写代码之前获取用户对方法的批准，可以防止浪费精力并确保一致性。此工具将你转换到规划模式，在那里你可以探索代码库并为用户批准设计实现方法。
>
> **优先使用 EnterPlanMode**，除非任务简单。在以下**任意一个**条件适用时使用：(1) 新功能实现 (2) 多种有效方法 (3) 代码修改 (4) 架构决策 (5) 多文件更改 (6) 需求不明确 (7) 用户偏好会影响实现
>
> 只有对简单任务才跳过 EnterPlanMode：单行修复、带清晰需求的单个函数、用户给出了非常具体详细指令的任务、纯研究探索任务。

**英文原文（内部用户版）**（来源：`src/tools/EnterPlanModeTool/prompt.ts`，第 108–163 行）

```
Use this tool when a task has genuine ambiguity about the right approach and getting user input before coding would prevent significant rework. This tool transitions you into plan mode where you can explore the codebase and design an implementation approach for user approval.

## When to Use This Tool

Plan mode is valuable when the implementation approach is genuinely unclear. Use it when:

1. **Significant Architectural Ambiguity**: Multiple reasonable approaches exist and the choice meaningfully affects the codebase
2. **Unclear Requirements**: You need to explore and clarify before you can make progress
3. **High-Impact Restructuring**: The task will significantly restructure existing code and getting buy-in first reduces risk

## When NOT to Use This Tool

Skip plan mode when you can reasonably infer the right approach:
- The task is straightforward even if it touches multiple files
- The user's request is specific enough that the implementation path is clear
- You're adding a feature with an obvious implementation pattern (e.g., adding a button, a new endpoint following existing conventions)
- Bug fixes where the fix is clear once you understand the bug
- Research/exploration tasks (use the Agent tool instead)
- The user says something like "can we work on X" or "let's do X" — just get started

When in doubt, prefer starting work and using AskUserQuestion for specific questions over entering a full planning phase.
```

> **中文附注（内部用户）**：
>
> 当任务对正确方法有**真实的**歧义，且在编码前获取用户意见能防止重大返工时，使用此工具。
>
> 规划模式在实现方法真正不清晰时有价值。在以下情况使用：(1) 重大架构歧义 (2) 需求不明确 (3) 高影响的重组
>
> 当你能合理推断正确方法时跳过规划模式。有疑问时，优先开始工作并使用 AskUserQuestion 询问具体问题，而不是进入完整的规划阶段。

**设计解析**

**内外差异揭示了不同的用户期望模型。**

外部用户版的默认倾向是：**进入规划模式**，除非任务简单。它有 7 个触发条件，其中包括"多文件更改（>2-3 个文件）"、"代码修改"这类非常低门槛的条件。这种设计假设外部用户希望：在 Claude Code 做任何大的改动之前都能看到计划，因为他们对工具的信任度不如内部用户，也不了解代码库的全貌。

内部用户版的默认倾向是：**直接开始工作**，只有在"真实的歧义"时才进入规划模式。它有 3 个触发条件，全部是高门槛的（"重大架构歧义"、"高影响重组"）。"当有疑问时，优先开始工作并使用 AskUserQuestion 询问具体问题"——这条规则直接翻转了外部用户版的逻辑。

这种差异背后是对两类用户的不同认知：
- **外部用户**：对 AI 代理的能力和局限不熟悉，倾向于希望有"审批关卡"
- **内部用户（Anthropic 工程师）**：对工具熟悉，追求效率，宁愿在出错后修复而不是提前规划

**"如果你会用 AskUserQuestion，就改用 EnterPlanMode"** 是外部版里一个非常精妙的替换规则。它解决了一个边界问题：什么时候该问问题，什么时候该进入规划模式？规则给出了判断标准：如果问题的性质是"关于方法的选择"，那就不是简单的一问一答能解决的，应该进入规划模式，让模型先探索再带着充分信息来提问。

---

### 5.1.5 TodoWriteTool：任务追踪与进度管理

**英文原文：DESCRIPTION 字段**（来源：`src/tools/TodoWriteTool/prompt.ts`，第 183–184 行）

```
Update the todo list for the current session. To be used proactively and often to track progress and pending tasks. Make sure that at least one task is in_progress at all times. Always provide both content (imperative) and activeForm (present continuous) for each task.
```

> **中文附注**：
>
> 更新当前会话的待办事项列表。要主动且频繁地使用，以跟踪进度和待处理任务。确保始终至少有一个任务处于 in_progress 状态。始终为每个任务提供 content（祈使形式）和 activeForm（现在进行时形式）。

**英文原文：使用规范**（来源：`src/tools/TodoWriteTool/prompt.ts`，第 3–25 行，节选）

```
## When to Use This Tool
Use this tool proactively in these scenarios:

1. Complex multi-step tasks - When a task requires 3 or more distinct steps or actions
2. Non-trivial and complex tasks - Tasks that require careful planning or multiple operations
3. User explicitly requests todo list - When the user directly asks you to use the todo list
4. User provides multiple tasks - When users provide a list of things to be done (numbered or comma-separated)
5. After receiving new instructions - Immediately capture user requirements as todos
6. When you start working on a task - Mark it as in_progress BEFORE beginning work. Ideally you should only have one todo as in_progress at a time
7. After completing a task - Mark it as completed and add any new follow-up tasks discovered during implementation

## When NOT to Use This Tool

Skip using this tool when:
1. There is only a single, straightforward task
2. The task is trivial and tracking it provides no organizational benefit
3. The task can be completed in less than 3 trivial steps
4. The task is purely conversational or informational

NOTE that you should not use this tool if there is only one trivial task to do. In this case you are better off just doing the task directly.
```

> **中文附注**：
>
> **何时使用此工具**：主动在以下场景使用：
> 1. 复杂多步骤任务（3 个或更多步骤）
> 2. 非平凡和复杂任务
> 3. 用户明确要求待办事项列表
> 4. 用户提供多个任务
> 5. 收到新指令后——立即将用户需求捕获为待办事项
> 6. 开始任务时——在开始工作**之前**标记为 in_progress
> 7. 完成任务后——标记为已完成，并添加实现中发现的新后续任务
>
> **何时不使用**：只有一个直接的任务、任务是平凡的、不到 3 步可完成、纯对话或信息性任务

**英文原文：任务状态管理**（来源：`src/tools/TodoWriteTool/prompt.ts`，第 145–180 行）

```
## Task States and Management

1. **Task States**: Use these states to track progress:
   - pending: Task not yet started
   - in_progress: Currently working on (limit to ONE task at a time)
   - completed: Task finished successfully

   **IMPORTANT**: Task descriptions must have two forms:
   - content: The imperative form describing what needs to be done (e.g., "Run tests", "Build the project")
   - activeForm: The present continuous form shown during execution (e.g., "Running tests", "Building the project")

2. **Task Management**:
   - Update task status in real-time as you work
   - Mark tasks complete IMMEDIATELY after finishing (don't batch completions)
   - Exactly ONE task must be in_progress at any time (not less, not more)
   - Complete current tasks before starting new ones
   - Remove tasks that are no longer relevant from the list entirely

3. **Task Completion Requirements**:
   - ONLY mark a task as completed when you have FULLY accomplished it
   - If you encounter errors, blockers, or cannot finish, keep the task as in_progress
   - When blocked, create a new task describing what needs to be resolved
   - Never mark a task as completed if:
     - Tests are failing
     - Implementation is partial
     - You encountered unresolved errors
     - You couldn't find necessary files or dependencies
```

> **中文附注**：
>
> **任务状态**：pending（未开始）、in_progress（当前进行，**限一个**）、completed（完成）
>
> **重要**：任务描述必须有两种形式：
>
> - content：祈使形式（"Run tests"、"Build the project"）
> - activeForm：现在进行时形式（"Running tests"、"Building the project"）
>
> **任务管理**：完成后立即标记（不要批量）；同一时间**恰好一个** in_progress；不再相关的任务从列表中完全删除
>
> **完成要求**：只有**完全完成**时才标记为 completed；遇到错误时保持 in_progress；以下情况永远不标记为完成：测试失败、实现不完整、遇到未解决错误、找不到必要文件

**设计解析**

**"Exactly ONE task must be in_progress at any time (not less, not more)"** 是整个任务系统最严格的约束。"not less, not more"——不只是"最多一个"，而是"恰好一个"。"not less"意味着任何时候至少有一个任务在进行（你不能处于"全部 pending"状态而开始工作）；"not more"意味着不能同时进行多个任务。这个单任务约束的目的是：用可见的 in_progress 状态精确追踪模型"当前在做什么"。

**DESCRIPTION 和 PROMPT 是两个独立常量** 这个设计值得注意。`DESCRIPTION`（第 183 行）是一行简短的 schema 描述，给工具选择用；`PROMPT`（第 3 行开始）是完整的使用规范，给模型实际操作用。两者独立维护，前者强调"主动且频繁"，后者给出精确的边界条件。

**content + activeForm 双形式要求** 是一个 UI/UX 需求直接传导为 prompt 规范的例子。任务在不同状态下显示的是不同的文字：pending 显示 content（"Run tests"），in_progress 显示 activeForm（"Running tests"）。这个双形式要求确保了 UI 的自然语言一致性——进行中的任务用现在进行时。

---

## 5.2 第二档工具：标准分析

---

### 5.2.1 文件操作类

#### Read（FileReadTool）

**英文原文**（来源：`src/tools/FileReadTool/prompt.ts`，第 32–48 行）

```
Reads a file from the local filesystem. You can access any file directly by using this tool.
Assume this tool is able to read all files on the machine. If the User provides a path to a file assume that path is valid. It is okay to read a file that does not exist; an error will be returned.

Usage:
- The file_path parameter must be an absolute path, not a relative path
- By default, it reads up to 2000 lines starting from the beginning of the file
- When you already know which part of the file you need, only read that part. This can be important for larger files.
- Results are returned using cat -n format, with line numbers starting at 1
- This tool allows Claude Code to read images (eg PNG, JPG, etc). When reading an image file the contents are presented visually as Claude Code is a multimodal LLM.
- This tool can read PDF files (.pdf). For large PDFs (more than 10 pages), you MUST provide the pages parameter to read specific page ranges (e.g., pages: "1-5"). Reading a large PDF without the pages parameter will fail. Maximum 20 pages per request.
- This tool can read Jupyter notebooks (.ipynb files) and returns all cells with their outputs, combining code, text, and visualizations.
- This tool can only read files, not directories. To read a directory, use an ls command via the Bash tool.
- You will regularly be asked to read screenshots. If the user provides a path to a screenshot, ALWAYS use this tool to view the file at the path. This tool will work with all temporary file paths.
- If you read a file that exists but has empty contents you will receive a system reminder warning in place of file contents.
```

> **中文附注**：
>
> 从本地文件系统读取文件。假设此工具能读取机器上的所有文件。如果用户提供了文件路径，假设该路径有效。读取不存在的文件是可以的；会返回错误。
>
> 使用要求：绝对路径、默认最多 2000 行、大文件只读需要的部分、结果使用 cat -n 格式（带行号）、支持图片（多模态）、PDF 支持（>10 页必须指定页范围）、Jupyter notebook 支持、不能读取目录。

**设计解析**

"Assume this tool is able to read all files on the machine. If the User provides a path to a file assume that path is valid."——这是一个"信任前提"的设定，告诉模型不要犹豫、不要先验证，直接尝试读取。如果没有这个设定，模型可能会在读取敏感路径（如 `/etc/hosts`）时添加额外的警告或确认，而实际上这些都是合理的操作。

**大 PDF 必须指定页范围** 是一个容量约束的透明化：`Reading a large PDF without the pages parameter will fail`——告诉模型工具的硬限制，让模型在调用时自己做判断，而不是让工具失败后再重试。

**FILE_UNCHANGED_STUB 机制**（代码第 8 行）：

```typescript
export const FILE_UNCHANGED_STUB = 'File unchanged since last read. The content from the
earlier Read tool_result in this conversation is still current — refer to that instead
of re-reading.'
```

这是一个缓存优化：如果同一个文件在对话中已经读过，且之后没有被修改，那么再次读取时会返回这个 stub 而不是完整内容。这样既节省了 token（避免重复的大文件内容），又通知了模型可以使用之前的结果。这个 stub 文字是模型友好的——`refer to that instead of re-reading` 明确告诉模型该怎么做。

---

#### Edit（FileEditTool）

**英文原文**（来源：`src/tools/FileEditTool/prompt.ts`，第 20–27 行）

```
Performs exact string replacements in files.

Usage:
- You must use your `Read` tool at least once in the conversation before editing. This tool will error if you attempt an edit without reading the file. 
- When editing text from Read tool output, ensure you preserve the exact indentation (tabs/spaces) as it appears AFTER the line number prefix. The line number prefix format is: line number + tab. Everything after that is the actual file content to match. Never include any part of the line number prefix in the old_string or new_string.
- ALWAYS prefer editing existing files in the codebase. NEVER write new files unless explicitly required.
- Only use emojis if the user explicitly requests it. Avoid adding emojis to files unless asked.
- The edit will FAIL if `old_string` is not unique in the file. Either provide a larger string with more surrounding context to make it unique or use `replace_all` to change every instance of `old_string`.
- Use `replace_all` for replacing and renaming strings across the file. This parameter is useful if you want to rename a variable for instance.
```

> **中文附注**：
>
> 在文件中执行精确的字符串替换。使用要求：必须先读取才能编辑；保留精确缩进；优先编辑现有文件；`old_string` 不唯一时编辑失败（需要更多上下文或使用 `replace_all`）；`replace_all` 用于全文替换/重命名。

**设计解析**

**"必须先 Read 才能 Edit"** 是一个工具间约束——一个工具的使用前提是先调用了另一个工具。这个约束有两个目的：
1. 确保模型看到了文件的实际内容（包括缩进），能够准确提供 `old_string`
2. 确保缩进格式正确（`cat -n` 格式会添加行号前缀，必须在提取 `old_string` 时剥离）

**"old_string 不唯一"的设计** 反映了精确字符串替换的根本限制：如果要替换的字符串在文件里出现了多次，工具无法知道该替换哪一个。这时有两个解决方案：提供更多上下文（扩大 old_string 到足够唯一的范围），或使用 `replace_all`（替换所有出现）。这两个选项针对两种不同的场景：前者适合"只改一处"，后者适合"重命名"。

内部用户版还有一条额外规则（`minimalUniquenessHint`）：

```
Use the smallest old_string that's clearly unique — usually 2-4 adjacent lines is sufficient.
Avoid including 10+ lines of context when less uniquely identifies the target.
```

这是一个效率规则：old_string 应该足够唯一，但不要过长——10+ 行的上下文在语义上是一种 token 浪费。

---

### 5.2.2 搜索类

#### Glob

**英文原文**（来源：`src/tools/GlobTool/prompt.ts`，第 3–7 行，完整文件）

```
- Fast file pattern matching tool that works with any codebase size
- Supports glob patterns like "**/*.js" or "src/**/*.ts"
- Returns matching file paths sorted by modification time
- Use this tool when you need to find files by name patterns
- When you are doing an open ended search that may require multiple rounds of globbing and grepping, use the Agent tool instead
```

> **中文附注**：快速文件模式匹配工具，适用于任何规模的代码库。支持 glob 模式。返回按修改时间排序的匹配路径。当需要多轮 globbing 和 grepping 的开放式搜索时，改用 Agent 工具。

**设计解析**

Glob 的 prompt 是整个工具体系中最短的之一（5 行），但最后一条非常关键："开放式搜索改用 Agent"——这是在工具自身的 prompt 里主动引导用户到另一个工具的设计。这种"自我限制"（self-limiting）是工具设计成熟度的体现：Glob 清楚地知道自己适合什么（单轮精确查找），不适合什么（多轮探索性搜索）。

---

#### Grep

**英文原文**（来源：`src/tools/GrepTool/prompt.ts`，第 7–17 行）

```
A powerful search tool built on ripgrep

  Usage:
  - ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command. The Grep tool has been optimized for correct permissions and access.
  - Supports full regex syntax (e.g., "log.*Error", "function\s+\w+")
  - Filter files with glob parameter (e.g., "*.js", "**/*.tsx") or type parameter (e.g., "js", "py", "rust")
  - Output modes: "content" shows matching lines, "files_with_matches" shows only file paths (default), "count" shows match counts
  - Use Agent tool for open-ended searches requiring multiple rounds
  - Pattern syntax: Uses ripgrep (not grep) - literal braces need escaping (use `interface\{\}` to find `interface{}` in Go code)
  - Multiline matching: By default patterns match within single lines only. For cross-line patterns like `struct \{[\s\S]*?field`, use `multiline: true`
```

> **中文附注**：基于 ripgrep 的强大搜索工具。始终用 Grep 做搜索任务，不要用 Bash 调用 `grep` 或 `rg`。支持完整正则、文件类型过滤、多种输出模式。注意：ripgrep 的字面花括号需要转义。跨行匹配需要 `multiline: true`。

**设计解析**

Grep 的 prompt 包含了两个工程细节，值得分析：

**"The Grep tool has been optimized for correct permissions and access"** 是工具优先的理由——不只是"用专用工具更好"的抽象说法，而是告诉模型具体原因：权限和访问被专门优化过。模型知道这个理由后，在遇到边界情况（如需要访问特殊权限文件）时，会更自然地选择 Grep 而不是 `grep`。

**ripgrep 的转义差异注意** 是罕见的工具特性差异说明：`interface\{\}` 来查找 Go 的 `interface{}`。模型在训练数据里大量见到的是 POSIX grep，花括号不需要转义；但 ripgrep 的正则引擎对花括号有不同的处理。在 prompt 里直接给出具体例子，是防止"工具切换导致正则失效"的最直接手段。

---

### 5.2.3 Web 类

#### WebFetch

**英文原文**（来源：`src/tools/WebFetchTool/prompt.ts`，第 3–21 行）

```
- Fetches content from a specified URL and processes it using an AI model
- Takes a URL and a prompt as input
- Fetches the URL content, converts HTML to markdown
- Processes the content with the prompt using a small, fast model
- Returns the model's response about the content
- Use this tool when you need to retrieve and analyze web content

Usage notes:
  - IMPORTANT: If an MCP-provided web fetch tool is available, prefer using that tool instead of this one, as it may have fewer restrictions.
  - The URL must be a fully-formed valid URL
  - HTTP URLs will be automatically upgraded to HTTPS
  - The prompt should describe what information you want to extract from the page
  - This tool is read-only and does not modify any files
  - Results may be summarized if the content is very large
  - Includes a self-cleaning 15-minute cache for faster responses when repeatedly accessing the same URL
  - When a URL redirects to a different host, the tool will inform you and provide the redirect URL in a special format. You should then make a new WebFetch request with the redirect URL to fetch the content.
  - For GitHub URLs, prefer using the gh CLI via Bash instead (e.g., gh pr view, gh issue view, gh api).
```

> **中文附注**：抓取指定 URL 的内容并用小模型处理。如果有 MCP 提供的 web fetch 工具，优先使用它（可能限制更少）。HTTP 自动升级为 HTTPS。有 15 分钟缓存。重定向时会通知并提供新 URL。GitHub URL 优先用 `gh` CLI。

**设计解析**

**"如果有 MCP 提供的 web fetch 工具，优先使用它"** 是一个罕见的设计：工具在自身的 prompt 里建议使用其他工具。背景是 MCP（Model Context Protocol）可能提供了更强大、限制更少的 web fetch 能力。这种"自我降级"的设计确保了用户在安装了更好工具的情况下，模型会自动选择最优路径。

**二级模型处理机制** 值得注意：WebFetch 不是直接把 HTML 内容塞给主对话，而是用"小而快的模型"先处理，再把结果返回。这是一个嵌套的 prompt 链：用户的需求 → 主模型决定调用 WebFetch → WebFetch 调用小模型提取 → 结果返回主模型。这个架构让"对大型网页的问答"变得可行（小模型更快更便宜），同时也带来了 `makeSecondaryModelPrompt()` 里的引用限制（125 字符）和版权保护规则。

---

#### WebSearch

**英文原文**（来源：`src/tools/WebSearchTool/prompt.ts`，第 7–33 行）

```
- Allows Claude to search the web and use the results to inform responses
- Provides up-to-date information for current events and recent data
- Returns search result information formatted as search result blocks, including links as markdown hyperlinks
- Use this tool for accessing information beyond Claude's knowledge cutoff
- Searches are performed automatically within a single API call

CRITICAL REQUIREMENT - You MUST follow this:
  - After answering the user's question, you MUST include a "Sources:" section at the end of your response
  - In the Sources section, list all relevant URLs from the search results as markdown hyperlinks: [Title](URL)
  - This is MANDATORY - never skip including sources in your response

IMPORTANT - Use the correct year in search queries:
  - The current month is [computed at runtime]. You MUST use this year when searching for recent information, documentation, or current events.
  - Example: If the user asks for "latest React docs", search for "React documentation" with the current year, NOT last year
```

> **中文附注**：允许 Claude 搜索网络。必须在回应后附"Sources:"部分列出所有相关 URL（**强制要求，不可跳过**）。搜索查询必须使用正确的当前年份。

**设计解析**

**Sources 部分的 CRITICAL + MANDATORY 双重强调** 是版权和信息归因合规的体现。在搜索结果上，未标注来源的内容引用有潜在的版权和误导性风险。用两个最高强度的约束词（CRITICAL + MANDATORY）确保模型不会遗漏这个步骤。

**年份注入的动态机制** 是 WebSearchTool prompt 里最有趣的工程细节：

```typescript
export function getWebSearchPrompt(): string {
  const currentMonthYear = getLocalMonthYear()
  return `...The current month is ${currentMonthYear}...`
}
```

这是一个函数而不是常量，因为它需要在运行时插入当前的月份年份。模型的训练截止日期与当前时间的差异会导致搜索查询使用错误的年份（如搜索"React 2023 docs"而不是"React 2026 docs"）。动态注入当前时间是解决这个"知识截止日期偏差"问题的最直接方法。

---

### 5.2.4 用户交互类

#### AskUserQuestion

**英文原文**（来源：`src/tools/AskUserQuestionTool/prompt.ts`，第 32–44 行）

```
Use this tool when you need to ask the user questions during execution. This allows you to:
1. Gather user preferences or requirements
2. Clarify ambiguous instructions
3. Get decisions on implementation choices as you work
4. Offer choices to the user about what direction to take.

Usage notes:
- Users will always be able to select "Other" to provide custom text input
- Use multiSelect: true to allow multiple answers to be selected for a question
- If you recommend a specific option, make that the first option in the list and add "(Recommended)" at the end of the label

Plan mode note: In plan mode, use this tool to clarify requirements or choose between approaches BEFORE finalizing your plan. Do NOT use this tool to ask "Is my plan ready?" or "Should I proceed?" - use ExitPlanMode for plan approval. IMPORTANT: Do not reference "the plan" in your questions (e.g., "Do you have feedback about the plan?", "Does the plan look good?") because the user cannot see the plan in the UI until you call ExitPlanMode.
```

> **中文附注**：在执行期间询问用户问题时使用此工具。用户始终能够选择"Other"来提供自定义文字输入。推荐选项放第一位并添加"(Recommended)"标签。规划模式特别注意：**不要**问"计划好了吗？"或"该继续吗？"——这是 ExitPlanMode 的职责；**不要**在问题中引用"计划"，因为用户在 ExitPlanMode 之前看不到计划。

**设计解析**

**规划模式注意事项** 是跨工具协作语义的最清晰案例。AskUserQuestion 和 ExitPlanMode 在规划模式下有严格的职责分工：
- AskUserQuestion：在规划完成前问"要做哪种实现方式"
- ExitPlanMode：请求对完成的计划进行批准

"Do NOT use this tool to ask 'Is my plan ready?'"——这条规则防止了功能混用：如果模型用 AskUserQuestion 来问"计划好了吗"，用户看到的是普通的多选题，而不是真正的计划审批界面（ExitPlanMode 提供的 UI 更适合审批）。

**"用户看不到计划"的说明** 揭示了规划模式的 UI 状态：在 ExitPlanMode 被调用之前，计划内容是不可见的。在问题里引用"计划"会产生对话层面的语义混乱——用户收到问题"计划好了吗"，但他们根本看不到任何计划。这个边界条件说明是 UI 状态机设计直接渗透到 prompt 里的典型例子。

---

#### ExitPlanMode

**英文原文**（来源：`src/tools/ExitPlanModeTool/prompt.ts`，第 6–29 行）

```
Use this tool when you are in plan mode and have finished writing your plan to the plan file and are ready for user approval.

## How This Tool Works
- You should have already written your plan to the plan file specified in the plan mode system message
- This tool does NOT take the plan content as a parameter - it will read the plan from the file you wrote
- This tool simply signals that you're done planning and ready for the user to review and approve
- The user will see the contents of your plan file when they review it

## When to Use This Tool
IMPORTANT: Only use this tool when the task requires planning the implementation steps of a task that requires writing code.

## Before Using This Tool
Ensure your plan is complete and unambiguous:
- If you have unresolved questions about requirements or approach, use AskUserQuestion first (in earlier phases)
- Once your plan is finalized, use THIS tool to request approval

**Important:** Do NOT use AskUserQuestion to ask "Is this plan okay?" or "Should I proceed?" - that's exactly what THIS tool does.
```

> **中文附注**：在规划模式中完成计划文件后使用此工具请求用户批准。此工具不接受计划内容作为参数——它从你已写入的文件中读取计划。**重要**：不要用 AskUserQuestion 来问"计划可以吗？"——这正是本工具的职责。

**设计解析**

**"不接受计划内容作为参数"** 是这个工具最特殊的设计。与大多数工具"接受数据→执行操作"的模式不同，ExitPlanMode 是一个"状态信号"——它告诉系统"规划阶段结束，进入审批阶段"，而计划内容从文件里读取，而不是从工具参数传入。这种设计的好处是计划内容的持久性：即使工具调用失败，计划文件依然存在，可以重试。

**EnterPlanMode / AskUserQuestion / ExitPlanMode 的三角关系** 通过三个工具各自的 prompt 完整地构建了规划模式的状态机：
- Enter：请求进入规划状态
- AskUserQuestion：在规划中澄清
- Exit：完成规划请求审批

三个工具互相知道对方的存在和职责，在各自的 prompt 里明确引用对方，形成了一个完整的协作协议。

---

### 5.2.5 多 Agent 类

#### SendMessage

**英文原文（节选）**（来源：`src/tools/SendMessageTool/prompt.ts`）

```
Send a message to another agent.

Use this tool to communicate with agents spawned using the Agent tool. Once an agent is created, you can send it messages to continue the conversation or provide updates.
```

> **中文附注**：向另一个 Agent 发送消息。用于与用 Agent 工具生成的 Agent 通信。一旦创建了 Agent，可以发送消息继续对话或提供更新。

---

#### TeamCreate

**英文原文（核心部分节选）**（来源：`src/tools/TeamCreateTool/prompt.ts`，第 6–22 行）

```
Use this tool proactively whenever:
- The user explicitly asks to use a team, swarm, or group of agents
- The user mentions wanting agents to work together, coordinate, or collaborate
- A task is complex enough that it would benefit from parallel work by multiple agents

When in doubt about whether a task warrants a team, prefer spawning a team.

## Choosing Agent Types for Teammates

- **Read-only agents** (e.g., Explore, Plan) cannot edit or write files. Only assign them research, search, or planning tasks. Never assign them implementation work.
- **Full-capability agents** (e.g., general-purpose) have access to all tools including file editing, writing, and bash. Use these for tasks that require making changes.
- **Custom agents** defined in `.claude/agents/` may have their own tool restrictions.
```

**设计解析**

**"When in doubt, prefer spawning a team"** 是一个高度乐观的默认倾向，与 EnterPlanMode 内部用户版的"有疑问时直接开始工作"形成鲜明对比。TeamCreate 的使用场景（多 Agent 并行协作）在任何时候启动都比不启动的代价小——团队可以被后来关闭，但错过了使用团队的机会就失去了并行加速。

**Agent 类型约束说明**（读写能力）直接嵌入在 TeamCreate 的 prompt 里，而不是让用户去查 AgentTool 的 prompt。这体现了"信息就近原则"：在做任务分配决策的工具（TeamCreate）里，直接提供做决策所需的信息（哪种 Agent 能读写文件）。

**Teammate Idle State 的细致说明**（第 66–72 行）解决了一个多 Agent 系统的常见误解：Agent 在每轮结束后自动进入 idle 状态，这是正常行为，不是 Agent 出错或结束了工作。"Do not treat idle as an error"——把正常的系统行为明确定义为"不是错误"，防止了协调者 Agent 错误地重新启动已经只是在等待回复的 Agent。

---

### 5.2.6 计划调度类

#### CronCreate

**英文原文（核心部分）**（来源：`src/tools/ScheduleCronTool/prompt.ts`，第 88–118 行）

```
Schedule a prompt to be enqueued at a future time. Use for both recurring schedules and one-shot reminders.

Uses standard 5-field cron in the user's local timezone: minute hour day-of-month month day-of-week. "0 9 * * *" means 9am local — no timezone conversion needed.

## Avoid the :00 and :30 minute marks when the task allows it

Every user who asks for "9am" gets `0 9`, and every user who asks for "hourly" gets `0 *` — which means requests from across the planet land on the API at the same instant. When the user's request is approximate, pick a minute that is NOT 0 or 30:
  "every morning around 9" → "57 8 * * *" or "3 9 * * *" (not "0 9 * * *")
  "hourly" → "7 * * * *" (not "0 * * * *")
  "in an hour or so, remind me to..." → pick whatever minute you land on, don't round

Only use minute 0 or 30 when the user names that exact time and clearly means it.
```

> **中文附注**：调度未来执行的 prompt。**避免整点和半点**：用户说"大概 9 点"应该生成 `57 8` 或 `3 9`，而非 `0 9`——因为全球所有用"9am"的用户都会在同一时刻向 API 发请求，造成流量洪峰。只有用户明确指定整点或半点时才使用 0 或 30 分。

**设计解析**

**"避免整点和半点"的流量均衡设计** 是整个工具体系中唯一一处明确考虑了**服务端负载**的 prompt 设计。这条规则不是为了用户体验，而是为了 Anthropic 的 API 基础设施——全球用户都设置"每天 9am"的 cron，如果都用 `0 9`，就会在每天 9 点产生一个巨大的流量洪峰。

这个设计把一个基础设施问题变成了 prompt 约束，让模型成为流量均衡的执行者。模型生成的 cron 时间是分散的，全局流量自然平滑。这是"AI 系统作为分布式基础设施参与者"的一个具体案例。

`→ 原则④：理由优于规则` — "因为全球用户都会在整点发请求造成流量洪峰"：给出了工程理由，模型可以举一反三应对类似情形，见第七章 4.1 节。

---

### 5.2.7 配置与工作区类

#### ConfigTool

**description**: `Get or set Claude Code configuration settings.`

配置工具用于读写 Claude Code 的各项设置，没有特别复杂的 prompt。

#### EnterWorktree / ExitWorktree

**英文原文（EnterWorktree 节选）**（来源：`src/tools/EnterWorktreeTool/prompt.ts`）

```
Use this tool ONLY when the user explicitly asks to work in a worktree. This tool creates an isolated git worktree and switches the current session into it.
```

**设计解析**

"ONLY when the user explicitly asks"——worktree 是一个高代价操作（创建文件系统隔离），必须由用户主动要求，不能模型自行判断使用。这与 EnterPlanMode 的"主动使用"形成对比——Plan 模式对用户透明且低风险，可以主动使用；worktree 创建对文件系统有实际影响，需要明确授权。

---

## 5.3 第三档工具：简要 + 模式引用

以下工具 prompt 较短（3–22 行），主要复用了前面分析过的设计模式。

| 工具 | 行数 | 关键设计 | 引用模式 |
|---|---|---|---|
| **BriefTool** | 22 | 自主模式下向用户发送简短消息；与 BashTool 的"通信直接输出"共享语义 | 通信渠道分离 |
| **SleepTool** | 17 | 等待指定时长；与 BashTool 的 sleep 避免规则配套 | 配套约束 |
| **LSPTool** | 21 | LSP 服务器交互（定义查找、引用查找、诊断）；多模态能力扩展 | 工具扩展 |
| **ToolSearch** | - | 加载延迟工具的完整 schema；查询 → 返回 JSON schema | 延迟加载 |
| **PowerShellTool** | 145 | Windows 的 BashTool 等价物；包含相同的 sleep 避免规则、后台任务规则、git 操作规则 | 模式复制 |
| **NotebookEdit** | 3 | Jupyter notebook 编辑；最短的工具 prompt | 最小化描述 |
| **MCPTool** | 3 | MCP 工具调用入口；同上 | 最小化描述 |
| **RemoteTrigger** | 15 | 远程触发器；用于 Agent 远程执行 | Agent 扩展 |
| **ListMcpResources** | 20 | 列出 MCP 服务器的可用资源 | MCP 扩展 |
| **ReadMcpResource** | 16 | 读取 MCP 资源内容 | MCP 扩展 |
| **TaskStop** | 8 | 停止正在运行的任务 | Task 系统 |

**PowerShellTool 的"模式复制"值得单独说明。** 这个工具的 prompt 几乎是 BashTool 的 Windows 等价版本，包含相同结构的：sleep 避免规则（用 `Start-Sleep` 而非 `sleep`）、后台任务规则、git 操作规则。这不是复制粘贴的懒惰，而是有意为之的"行为一致性"——用户在 Windows 和 macOS/Linux 上应该获得同样的 Claude Code 行为，即使底层工具不同。

---

## 5.4 跨工具设计模式总结

分析 35+ 个工具的 prompt 后，可以归纳出以下跨工具的设计规律：

**模式一：工具优先级明确化**

最具代表性：BashTool 列出了 6 个"NOT Bash，用 X"的具体命令替换。这种"明确的竞争工具禁止"在 Grep（"NEVER invoke grep or rg as a Bash command"）、WebFetch（"If an MCP-provided tool is available, prefer that"）中也有体现。每个工具知道自己在工具体系里的位置，主动引导到最优的工具选择。

**模式二：过度使用防止**

AgentTool 的"When NOT to use"、TodoWriteTool 的"When NOT to Use"、EnterPlanModeTool 的"When NOT to Use"——几乎所有高代价工具都有明确的"不使用"条件。这种双向约束（什么时候用 + 什么时候不用）是成熟工具设计的标志。

**模式三：状态机显式化**

EnterPlanMode / ExitPlanMode / AskUserQuestion 的三角关系，TeamCreate / TeamShutdown / SendMessage 的团队生命周期——这些工具的 prompt 共同定义了一个明确的状态机，每个工具在状态转换图中有精确的位置。

**模式四：动态内容与静态内容分离**

WebSearch 的实时年份注入、AgentTool 的 agent 列表附件注入、ScheduleCronTool 的 jitter 说明——动态部分（随运行时变化的内容）被精确识别并隔离，保持其余部分的缓存稳定性。

**模式五：工具间协议的 prompt 层面编码**

Read-before-Edit、fork 不要 re-delegate、skill 调用后检测 `<command-name>` 标签——跨工具的使用协议被直接写进每个工具自身的 prompt，而不是集中在一个"工具使用指南"里。这种分散但精确的约束，确保了无论模型当前"看"的是哪个工具，相关的协议都在视野内。

---

> **下一章**：压缩摘要 Prompt。当对话变长时，Claude Code 会用专门设计的 prompt 对历史进行摘要——这套摘要 prompt 有着独特的设计逻辑，是理解 Claude Code 长会话能力的关键。
