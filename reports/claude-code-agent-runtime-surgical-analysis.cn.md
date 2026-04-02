[返回目录](../README.cn.md) · **中文版** | [English](claude-code-agent-runtime-surgical-analysis.md)

**目录：** [0](#0-范围) · [1](#1-一句话机器定义) · [2](#2-为什么存在这部分) · [3](#3-顶层控制图) · [4](#4-具体入口点) · [5](#5-正常路径外科演练) · [6](#6-调度图) · [7](#7-状态模型) · [8](#8-数据转换链) · [9](#9-副作用图) · [10](#10-重要分支点) · [11](#11-故障和恢复逻辑) · [12](#12-真正的核心循环) · [13](#13-架构压缩) · [14](#14-设计意图推断) · [15](#15-安全手术图) · [16](#16-学习价值图) · [17](#17-重用--复制价值图) · [18](#18-心理模型包)

---

# Claude Code — 代码外科分析报告

**分析代码库：** `repos/claude-code`
**分析视角：** "Reel Creator" 构建者 — 一个 AI 原生社交媒体内容创作 Agent OS，类似于 Claude Code，但用于内容创作而非编码
**报告日期：** 2026-04-01
**完成分析轮次：** 7

---

## 0. 范围

本报告通过七个分析轮次对 Claude Code 代码库（`repos/claude-code/src/`）进行外科解剖：入口点追踪、正常路径演练、状态架构、调度机制、副作用映射、故障面编目和架构压缩。每个发现都与特定的文件和行号相关联。最终交付成果（第 16-17 节）回答了 Reel Creator 构建者的主要问题：对于每个架构组件，在第一阶段（截止日期：4 月 15 日）是 COPY-PASTE（直接复制）、USE-THE-ESSENCE（使用精髓）还是 JUST-FOR-LEARN（仅供学习）？

代码库在 `src/` 下包含约 150+ 个 TypeScript 源文件。按行数统计的关键文件：`src/main.tsx`（4,683 行）、`src/query.ts`（1,729 行）、`src/skills/loadSkillsDir.ts`（1,086 行）、`src/utils/forkedAgent.ts`（689 行）、`src/Tool.ts`（792 行）、`src/services/tools/toolOrchestration.ts`（188 行）、`src/state/store.ts`（34 行）、`src/query/deps.ts`（40 行）。

---

## 1. 一句话机器定义

Claude Code 执行一个有状态的、权限门控的、工具调用代理循环，它接收自然语言提示，迭代地分发 LLM API 调用和工具调用，直到没有工具使用块为止，并将完整的对话记录持久化为仅追加的 JSONL，同时维护多级上下文窗口压缩栈以将令牌使用量保持在模型限制内。

---

## 2. 为什么存在这部分

该系统的存在是为了解决一个特定的可靠性问题：原始的 LLM 补全无法安全地写入文件、运行 shell 命令或在真实文件系统上执行多步操作，除非有 (a) 一个确定性调度层，将每个模型请求的工具调用路由到正确的实现，(b) 一个权限门，在破坏性操作之前强制执行人工参与的同意，以及 (c) 一个在长会话中跨上下文窗口溢出存活的内存架构。如果没有这三者，纯聊天界面要么在复杂任务上静默失败，要么在没有用户意识的情况下破坏性地修改文件系统。

Reel Creator 面临着完全相同的问题域，只是转移到了内容领域：原始的 LLM 补全无法安全地发布到 TikTok/Instagram、调用资产生成 API 或管理多平台发布管道，除非有相同的三个组件。架构动机完全转移；工具完全改变。

---

## 3. 顶层控制图

```
CLI 入口点 (src/entrypoints/cli.tsx)
    │
    ▼
Commander.js 标志解析 (src/main.tsx)
    │
    ├─── 交互式 REPL 路径 ──► Ink/React UI (src/replLauncher.tsx)
    │                                    │
    │                              REPL 输入循环
    │                                    │
    └─── 无头/打印路径 ──► QueryEngine.submitMessage()
                                        │
                                        ▼
                               src/query.ts:queryLoop()   ◄─── while(true) 迭代引擎
                                        │
                          ┌─────────────┼──────────────┐
                          ▼             ▼               ▼
                   压缩检查      callModel()        runTools()
                                (LLM API)          (工具调度)
                          │             │               │
                          │      流式 SSE        权限门
                          │      tool_use 块  → tool.call()
                          │             │               │
                          └─────────────┴───────────────┘
                                        │
                                   对话记录
                                   JSONL 写入
                                  (~/.claude/projects/)
```

当模型发出零 `tool_use` 内容块的响应时，循环终止。状态同时存在于三个地方：响应式 `AppState` 存储（面向 UI）、`QueryEngine` 中的 `mutableMessages[]` 数组（对话历史）以及磁盘上的仅追加 JSONL（持久对话记录）。

---

## 4. 具体入口点

**主二进制入口点：** `src/entrypoints/cli.tsx:main()` — 读取 `process.argv`，对 `--version` 和 `--daemon-worker` 标志执行快速路径退出，然后委托给 `src/main.tsx`。

**CLI 程序定义：** `src/main.tsx` — 构造一个 Commander.js 程序，带有 `[prompt]` 位置参数和标志，包括 `--print`（`-p`）、`--model`、`--resume`、`--agent`、`--permission-mode`、`--output-format`、`--max-turns`、`--allowedTools`、`--disallowedTools`，以及子命令 `mcp`、`auth`、`plugin`、`marketplace`、`agents`、`open`。

**REPL 斜杠命令：** `src/commands.ts:COMMANDS()` — 返回启动时注册的约 80+ 个 REPL 内命令：`/compact`、`/clear`、`/help`、`/model`、`/memory`、`/config`、`/mcp`、`/skills`、`/tasks`、`/review`、`/vim`、`/permissions`、`/plan`、`/hooks`、`/agents`、`/plugin`、`/rewind`、`/cost`、`/diff`、`/files`、`/branch`、`/resume`、`/session`、`/export`、`/stats`、`/effort`、`/fast`、`/passes`。动态技能和插件命令通过第 476 行的 `getCommands()` 合并。

**SDK/无头入口点：** `src/QueryEngine.ts:184` — `QueryEngine` 类构造函数初始化 `mutableMessages[]`、`abortController` 和 `readFileState`。第 209 行的 `submitMessage()` 组装完整的系统提示，处理用户输入，并调用 `query()`。

**代理循环入口点：** `src/query.ts:queryLoop()` 在第 241 行 — 驱动所有 LLM 调用和工具执行的异步生成器。由第 219 行的 `query()` 调用，后者包装生成器。

---

## 5. 正常路径外科演练

**场景：** 用户从终端运行 `claude "fix the bug in auth.ts"`。

**步骤 1 — 二进制读取 argv：** `src/entrypoints/cli.tsx:main()` 接收 `process.argv = ["node", "claude", "fix the bug in auth.ts"]`。未检测到快速路径标志。委托给 `src/main.tsx`。

**步骤 2 — Commander 解析标志：** `src/main.tsx`（Commander 程序定义，约第 200 行）将 `"fix the bug in auth.ts"` 解析为 `[prompt]` 位置参数。因为存在提示但没有显式的 `-p` 标志，代码激活无头打印路径。通过 `src/state/store.ts:createStore()` 的 `createStore(headlessInitialState)` 构造 `headlessStore`。

**步骤 3 — QueryEngine 初始化：** `src/QueryEngine.ts:200` — 构造函数设置 `this.mutableMessages = []`，创建 `AbortController`，初始化 `readFileState`。尚未进行 API 调用。

**步骤 4 — submitMessage() 组装上下文：** `src/QueryEngine.ts:209:submitMessage()` 在第 292 行调用 `fetchSystemPromptParts()`，后者调用：
- `getSystemPrompt()` — 返回基础系统提示字符串
- `src/utils/queryContext.ts` 的 `getUserContext()` — 通过 `src/utils/claudemd.ts` 加载 CLAUDE.md 层次结构（managed → user → project → local 层，支持 `@include` 指令），追加当前日期
- `getSystemContext()` — 追加 git 状态输出

**步骤 5 — 用户输入处理：** `src/QueryEngine.ts:416:processUserInput()` 在 `src/utils/processUserInput/processUserInput.ts:85` 接收 `"fix the bug in auth.ts"`。`parseSlashCommand()` 未找到前导斜杠。使用原始文本创建 `UserMessage`。`submitMessage()` 在第 431 行推送到 `mutableMessages`，在第 451 行通过 `src/utils/sessionStorage.ts:recordTranscript()` 记录对话记录条目。

**步骤 6 — query() 委托给 queryLoop()：** `src/query.ts:219:query()` 在第 241 行调用 `queryLoop()`，传递组装的消息、系统提示、工具列表和依赖注入对象（`deps`）。

**步骤 7 — while(true) 循环开始：** `src/query.ts:307` — 循环体。压缩检查首先运行（第 1 轮全部通过）。第 659 行：`for await (const message of deps.callModel({messages, system, tools, ...}))` — `deps.callModel` 解析为 `src/services/api/claude.ts:752` 的 `queryModelWithStreaming`。

**步骤 8 — LLM 流式响应：** `src/services/api/claude.ts:752:queryModelWithStreaming()` 构造 Anthropic API 请求，通过 `@anthropic-ai/sdk` 调用 `client.beta.messages.stream()`。模型响应一个 `tool_use` 块，请求 `FileReadTool` 读取 `auth.ts`。流式事件将 `AssistantMessage` 和 `StreamEvent` 对象yield回循环。工具使用块在第 830-834 行累积到 `toolUseBlocks`。

**步骤 9 — 工具调度：** 流式完成后，`src/query.ts` 第 1380-1408 行调用 `src/services/tools/toolOrchestration.ts:runTools()` 的 `runTools(toolUseBlocks, ...)`。`runTools()` 对块进行分区：`FileReadTool` 是只读的，因此进入并发批次（最多 10 个并发）。为每个调用 `runToolUse()`。

**步骤 10 — 权限门和工具执行：** `src/services/tools/toolExecution.ts:337:runToolUse()` 通过 `findToolByName()` 按名称查找 `FileReadTool`。使用 Zod schema 验证输入 `{path: "auth.ts"}`。调用第 599 行的 `checkPermissionsAndCallTool()`，后者调用 `canUseTool()` → `src/utils/permissions/permissions.ts:hasPermissionsToUseTool()`。默认模式下只读文件操作解析为 `allow`。调用 `tool.call(parsedInput, toolUseContext)`。

**步骤 11 — 工具结果反馈：** `FileReadTool` 从磁盘读取 `auth.ts` 内容。结果作为一个带有 `tool_result` 内容块的 `UserMessage` yield。对话记录被记录。循环迭代结束，`continue` 返回步骤 7。

**步骤 12 — 包含文件内容的第二次 LLM 调用：** 循环迭代 2 使用现在包含 `auth.ts` 内容的消息调用 `deps.callModel()`。模型返回带有建议补丁的 `FileEditTool` tool_use 块。

**步骤 13 — 写权限检查中断：** `FileEditTool` 的 `checkPermissionsAndCallTool()` — 默认模式下的写操作。`hasPermissionsToUseTool()` 解析为 `ask`。`src/hooks/useCanUseTool.tsx:93` 队列确认对话框。用户看到接受/拒绝 UI。用户接受。

**步骤 14 — 编辑应用并循环终止：** `FileEditTool.call()` 将补丁应用到 `auth.ts`。工具结果 yield 成功消息。循环迭代 3：模型返回零 `tool_use` 块的响应。第 1062 行 — 停止条件满足。`queryLoop()` yield 最终 `AssistantMessage` 并返回。对话记录在 `~/.claude/projects/<hash>/<sessionId>.jsonl` 最终化。

---

## 6. 调度图

**LLM → 工具调度：** 模型发出带有 `name` 字段的 `tool_use` 内容块。没有代理端路由逻辑选择工具。`src/services/tools/toolOrchestration.ts` 的 `runTools()` 接收原始块，按只读分类分区，并通过 `src/services/tools/toolExecution.ts:337` 的 `runToolUse()` 分发每个。`src/Tool.ts` 的 `findToolByName()` 通过字符串名称匹配定位正确的工具对象。

**工具注册表：** `src/tools.ts:getAllBaseTools()` — 所有工具对象的硬编码列表：`AgentTool`、`BashTool`、`FileEditTool`、`FileReadTool`、`FileWriteTool`、`GlobTool`、`GrepTool`、`NotebookEditTool`、`TodoWriteTool`、`SkillTool`、`ToolSearchTool` 等。特性门控工具有条件地包含。`getTools()` 通过 `isEnabled()` 返回 true、拒绝规则和工具预设配置过滤列表。MCP 工具动态实例化为从连接服务器发现的 `MCPTool` 对象。

**斜杠命令调度：** `src/utils/processUserInput/processUserInput.ts:85:processUserInput()` 调用 `parseSlashCommand()` 检测前导 `/`，然后调用 `findCommand()` 从 `COMMANDS()` 定位匹配的 `Command` 对象。具有本地 `call()` 函数的命令立即在进程中执行。返回提示字符串的命令作为用户消息转发给模型。

**子代理生成：** `src/tools/AgentTool/runAgent.ts:runAgent()` 通过 `src/utils/forkedAgent.ts` 的 `createSubagentContext()` 创建隔离的执行上下文。上下文克隆父级的文件状态缓存，创建子 `AbortController`，分配新的 `agentId`，并使用新的空 `messages[]` 数组递归调用 `query()`。代理定义从带有 YAML frontmatter 的 `.claude/agents/` markdown 文件加载。

**MCP 工具加载：** `src/services/mcp/client.ts:connectToServer()` 创建传输（stdio、SSE、HTTP 或 WebSocket，基于配置），调用 `client.listTools()` 发现可用工具，并为每个实例化 `MCPTool` 对象。MCP 提示成为 `PromptCommand` 技能对象。资源可通过 `ListMcpResourcesTool` 和 `ReadMcpResourceTool` 获得。

**技能加载和调用：** `src/skills/loadSkillsDir.ts` 扫描 `~/.claude/skills/`、`.claude/skills/` 和托管路径。每个带有有效 YAML frontmatter（name、description、whenToUse、tools、model）的 `.md` 文件成为一个 `PromptCommand`。捆绑技能在 `src/skills/bundled/` 中发布。当模型在 `src/tools/SkillTool/SkillTool.ts` 调用 `SkillTool` 时，它调用 `getPromptForCommand()` 并通过 `runAgent()` 将技能作为 forked 代理子查询运行。

**延迟工具发现：** `src/tools/ToolSearchTool/ToolSearchTool.ts` — 不在初始工具列表中的工具在运行时可被发现。当模型用查询调用 `ToolSearch` 时，它返回匹配工具的完整 JSON schema，允许模型在后续轮次中调用它们。

---

## 7. 状态模型

**AppState**（`src/state/AppStateStore.ts:89`，类型为 `DeepImmutable`）：所有面向 UI 和会话级状态的主要响应式存储。字段包括：`settings`（从所有配置层合并）、`verbose`、`mainLoopModel`、`statusLineText`、`toolPermissionContext`（权限模式 + 按源键控的 allow/deny/ask 规则集）、`mcp`（连接的客户端、工具、命令、资源）、`plugins`、`tasks`（`taskId → TaskState` 的映射）、`agentNameRegistry`、`foregroundedTaskId`、`speculation`、`fileHistory`、`attribution`、`denialTracking`、`fastMode`、`effortValue`、`advisorModel`、`notifications`、`elicitationQueue`、`todoLists`、`bridge`、`remote`、`theme` 以及所有 Ink UI 显示字段。

**查询循环状态**（`src/query.ts:204`）：`queryLoop()` 内部的本地可变状态 — `messages[]`、`toolUseContext`、`autoCompactTracking`、`maxOutputTokensRecoveryCount`、`hasAttemptedReactiveCompact`、`maxOutputTokensOverride`、`pendingToolUseSummary`、`stopHookActive`、`turnCount`、`transition`。仅在一个 `queryLoop()` 调用的一生中存在。

**引导状态**（`src/bootstrap/state.ts`）：模块级全局可变变量 — `originalCwd`、`sessionId`、`totalCostUSD`、`modelUsage`、`mainLoopModelOverride`、`isInteractive`、`registeredHooks`、`invokedSkills`、`agentColorMap`。第 32 行的注释写着"DO NOT ADD MORE STATE HERE"，表明已知的技术债务。

**对话历史**：双重表示 — 在内存中作为 `QueryEngine`（第 186 行）中的 `mutableMessages: Message[]`，以及查询循环内作为 `state.messages`，再加上磁盘上 `~/.claude/projects/<hash>/<sessionId>.jsonl` 的仅追加 JSONL，由 `src/utils/sessionStorage.ts:recordTranscript()` 写入。

**工具权限状态**（`src/Tool.ts:123` — `ToolPermissionContext`）：`mode`（`default` / `plan` / `bypassPermissions` / `auto` 之一）、`alwaysAllowRules`、`alwaysDenyRules`、`alwaysAskRules` 每个按源键控（`localSettings`、`userSettings`、`projectSettings`、`policySettings`、`cliArg`、`command`、`session`），加上 `additionalWorkingDirectories`。存储在 `AppState.toolPermissionContext` 内部。

**上下文窗口组装**（每轮迭代组装）：`getMessagesAfterCompactBoundary()` → `applyToolResultBudget()` → 截断 → 微压缩 → 上下文折叠 → 自动压缩 → `prependUserContext(CLAUDE.md 内容)` → `appendSystemContext(git 状态)` → 完整系统提示拼接。

---

## 8. 数据转换链

原始用户字符串 `"fix the bug in auth.ts"` 通过以下管道转换：

1. **字符串 → Commander ParsedArgs**（`src/main.tsx`）：Commander.js 将 argv 解析为 `{prompt: "fix the bug in auth.ts", model: undefined, print: false, ...}`。

2. **ParsedArgs → UserMessage**（`src/utils/processUserInput/processUserInput.ts:85`）：`processUserInput()` 将字符串包装在类型化的 `UserMessage` 对象中，带有 `role: "user"` 和 `content: [{type: "text", text: "fix the bug in auth.ts"}]`。

3. **UserMessage → API 请求**（`src/QueryEngine.ts:209`）：`submitMessage()` 组装 `{system: [systemPrompt, claudeMdContent, gitStatus], messages: [UserMessage], tools: [FileReadTool schema, FileEditTool schema, ...], model: "claude-opus-4-5", max_tokens: 8192, ...}`。

4. **API 请求 → 流式事件**（`src/services/api/claude.ts:752`）：`@anthropic-ai/sdk` 流式传输 `message_start`、`content_block_start`、`content_block_delta`、`message_delta` 事件。组装成类型化的 `AssistantMessage`。

5. **AssistantMessage → ToolUseBlock[]**（`src/query.ts:830`）：循环收集 `type === "tool_use"` 的 `content` 块到 `toolUseBlocks[]`。

6. **ToolUseBlock → Zod 验证的输入**（`src/services/tools/toolExecution.ts:617`）：每个块的 `input` 字段通过工具的 Zod schema。无效输入在接触任何文件系统之前用 `InputValidationError` 拒绝。

7. **验证输入 → 工具结果**（`tool.call(parsedInput, ctx)`）：工具执行，返回 `{type: "tool_result", tool_use_id, content: string | ContentBlock[]}`。

8. **工具结果 → UserMessage**（`src/query.ts`）：包装为带有 `role: "user"` 和 `content: [tool_result 块]` 的 `UserMessage`，追加到 `messages[]` 用于下次迭代。

9. **最终 AssistantMessage → JSONL**（`src/utils/sessionStorage.ts:recordTranscript()`）：每条消息作为类型化的 JSONL 条目追加到 `~/.claude/projects/<hash>/<sessionId>.jsonl`。

---

## 9. 副作用图

**文件系统写入：**
- `src/utils/sessionStorage.ts:recordTranscript()` — 追加 JSONL 到 `~/.claude/projects/<hash>/<sessionId>.jsonl`
- `FileWriteTool.call()` — 在工作目录中创建/覆盖文件
- `FileEditTool.call()` — 将补丁应用到现有文件
- `NotebookEditTool.call()` — 变更 Jupyter 笔记本单元格
- `TodoWriteTool.call()` — 将待办状态写入磁盘
- `src/utils/config.ts:saveGlobalConfig()` — 将合并的配置持久化到 `~/.claude.json`
- `src/memdir/` 内存写入 — 创建/更新 `~/.claude/memory/MEMORY.md` 和主题文件
- `src/utils/fileHistory.ts` — 写入撤销快照以实现可逆性

**LLM API 调用**（`src/services/api/claude.ts:752`）：每轮循环迭代的主模型调用，加上：压缩摘要（自动压缩 forked 代理）、内存提取（`src/services/extractMemories/`）、代理摘要（`src/services/AgentSummary/`）、提示建议（`src/services/PromptSuggestion/`）、工具使用摘要、通过 `SideQuestionTool` 的附带问题、自动模式权限分类器。

**子进程生成：**
- `BashTool.call()` — 使用 macOS sandbox-exec 包装器（`src/utils/sandbox/sandbox-adapter.ts:SandboxManager`）的 `child_process.spawn()`
- `PowerShellTool.call()` — Windows 子进程等价物
- `LocalShellTask` — 长时间运行的后台任务进程
- MCP stdio 服务器 — 当 MCP 配置引用可执行路径时通过 `StdioClientTransport` 生成

**MCP 网络调用**（`src/services/mcp/client.ts`）：通过 stdio、SSE、HTTP 或 WebSocket 传输的 `client.callTool()`、`client.listTools()`、`client.listPrompts()`、`client.listResources()`、`client.readResource()`。

**持久化路径清单：**
- `~/.claude/projects/<hash>/<sessionId>.jsonl` — 会话对话记录
- `~/.claude/projects/<hash>/<sessionId>/agents/<agentId>.jsonl` — 子代理对话记录
- `~/.claude.json` — 全局配置（模型、权限、API 密钥）
- `.claude/settings.json` — 项目级设置
- `~/.claude/settings.json` — 用户级设置
- `~/.claude/memory/MEMORY.md` — 持久化内存索引
- `.claude/memory/` — 项目范围的内存文件
- `~/.claude/sessions/` — 会话元数据索引

---

## 10. 重要分支点

**权限模式分支**（`src/utils/permissions/permissions.ts:hasPermissionsToUseTool()`）：首先评估所有源的拒绝规则（如果任何拒绝规则匹配 → `deny`），然后是允许规则，然后是模式默认回退。模式：`default`（危险操作 → ask）、`plan`（所有写入 → deny）、`bypassPermissions`（一切 → allow）、`auto`（LLM 分类器决定）。规则源优先级强制为：`policySettings > projectSettings > localSettings > userSettings > cliArg > command > session`。

**交互式 vs 无头分支**（`src/main.tsx`）：`--print`/`-p` 标志的存在或没有 TTY 的位置提示参数路由到 `QueryEngine` 无头路径。没有初始提示的交互式终端路由到 `src/replLauncher.tsx` Ink/React REPL。

**工具批处理分支**（`src/services/tools/toolOrchestration.ts:runTools()`）：将 `toolUseBlocks[]` 分区为只读（并发，最多 10 个同时）和非只读（串行，一次一个）。工具的 `isReadOnly()` 方法决定分区。

**上下文窗口溢出分支**（`src/query.ts:628`）：在每次 API 调用之前，测量组装的消息令牌计数与模型限制。按照顺序触发压缩栈：Snip → Microcompact → Context Collapse → Autocompact → Reactive Compact（413 响应时的紧急情况）。每个机制仅在前一个不足时触发。

**子代理 vs 内联执行分支**（`src/tools/AgentTool/runAgent.ts`）：`AgentTool` 生成隔离的 `query()` 递归与克隆的上下文。`SkillTool` 类似地通过 `runAgent()` 分叉。所有其他工具在父循环内内联执行。

**回退模型分支**（`src/query.ts:894`）：`FallbackTriggeredError` 将 `mainLoopModel` 切换到配置的回退并重试当前迭代而不丢失消息历史。

**MCP 工具发现分支**（`src/services/mcp/client.ts`）：在会话启动时，每个配置的 MCP 服务器连接并发现工具。发现的工具合并到会话的活动工具集中。一个服务器连接失败不会阻止其他服务器。

---

## 11. 故障和恢复逻辑

**权限拒绝：** `src/services/tools/toolExecution.ts:599:checkPermissionsAndCallTool()` 门控每个工具调用。拒绝产生带有 `is_error: true` 的 `tool_result` 和解释权限拒绝的人类可读消息。模型接收此结果并可能要求用户更改权限或建议替代方法。

**输入验证失败：** `src/services/tools/toolExecution.ts:617` — Zod 解析 `tool_use.input` 失败 → 抛出 `InputValidationError` → 在第 469 行捕获 → 返回带有 `is_error: true` 的 tool_result 给模型。模型用修正的输入重新尝试。

**运行时工具错误：** `src/services/tools/toolExecution.ts:469` — 任何来自 `tool.call()` 的异常被捕获，包装为带有 `is_error: true` 的 tool_result，并反馈给模型。通过 `runPostToolUseFailureHooks()` 运行故障后钩子。模型重试或放弃操作。

**LLM API 速率限制：** `src/services/api/withRetry.ts` — 在 429 响应上具有抖动的指数退避。可配置的重试计数和延迟乘数。

**回退模型：** `src/query.ts:894–951` — 循环体中捕获的 `FallbackTriggeredError`。将 `mainLoopModel` 切换到配置中的 `fallbackModel`，重构 API 调用，重试当前轮次。

**最大输出令牌恢复：** `src/query.ts:1188–1256` — 在 `max_tokens` 停止原因上，通过追加延续元消息最多重试 3 次。`maxOutputTokensRecoveryCount` 跟踪计数器。

**上下文窗口溢出栈**（五种机制，每个在越来越高严重性下触发）：
1. **Snip** — 从历史中移除最老的非必要消息
2. **Microcompact** — 截断过大的单个工具结果，用文件引用替换内容（`src/utils/sessionStorage.ts` 持久化替换记录用于 `--resume` 重构）
3. **Context Collapse** — 将整个较早的轮次折叠为一个摘要块
4. **Autocompact** — 分叉一个新的代理来摘要整个对话，用摘要替换消息，在 JSONL 中记录"上下文折叠提交"
5. **Reactive Compact** — 通过从 API 接收 413 `prompt_too_long` 错误触发的紧急自动压缩（`src/query.ts:1062–1183`，`hasAttemptedReactiveCompact` 标志防止无限重试）

**不确定性权限的人工参与**（`src/hooks/useCanUseTool.tsx:93–167`）：当权限解析为 `ask` 时，协调器通过 swarm worker → 推测分类器检查 → `handleInteractivePermission()` 路由。这通过 `setToolUseConfirmQueue` 在 Ink/React UI 中队列确认对话框。用户看到接受/拒绝/始终允许选项。每个工具的对话框组件位于 `src/components/permissions/`。

## 12. 真正的核心循环

负载的核心由七个相互依存的部分组成。移除任何一个，系统就会停止运转：

```
queryLoop() ──calls──► queryModelWithStreaming()
     │                         │
     │                 streams AssistantMessage
     │                         │
     ◄────── tool_use blocks ───┘
     │
     ▼
runTools() ──partitions──► runToolUse()
                                │
                    checkPermissionsAndCallTool()
                                │
                    hasPermissionsToUseTool()
                                │
                          tool.call()
                                │
                    tool_result → next iteration
```

**七个负载文件：**
1. `src/query.ts:queryLoop()` — 迭代引擎、压缩编排、停止检测
2. `src/services/api/claude.ts:queryModelWithStreaming()` — LLM 调用、流式组装、提供商路由（第一方 / Bedrock / Vertex）
3. `src/services/tools/toolExecution.ts:checkPermissionsAndCallTool()` — 工具调度 + 权限门
4. `src/Tool.ts` — 工具类型系统、`findToolByName()`、`ToolPermissionContext` 定义
5. `src/tools.ts:getAllBaseTools()` — 工具注册表、`getTools()` 过滤
6. `src/state/store.ts:createStore()` — 响应式状态容器（34 行，纯净）
7. `src/utils/permissions/permissions.ts:hasPermissionsToUseTool()` — 具有源优先级排序的权限评估

其他一切 — 技能、MCP、子代理、内存、压缩、UI — 都层叠在这七个之上。Reel Creator 需要所有七个的等效实现，具有不同的工具和不同的 LLM 配置。

---

## 13. 架构压缩

**权限模型** 以四种模式运作（`default` / `plan` / `bypassPermissions` / `auto`），拒绝规则在允许规则之前跨七个源层评估。拒绝是保守的：任何源的任何拒绝规则都阻止。允许需要最高优先级源的明确许可。`auto` 模式添加了一个 LLM 分类器，预测当前工具调用是否匹配会话的原始意图，使用拒绝跟踪来建立先验。

**内存架构** 以四个独立的时间范围运作：
- **CLAUDE.md**（`src/utils/claudemd.ts`）：在会话开始时注入的静态指令，4 层层次结构（managed → user → project → local），支持 `@include` 指令。在进程重启后保持不变。
- **MEMORY.md / 主题文件**（`src/memdir/`）：模型可写的持久化内存，指向主题文件的索引文件。在会话之间保持。模型通过标准 `FileWriteTool` 写入。
- **会话对话记录**（`src/utils/sessionStorage.ts`）：仅追加 JSONL，可通过 `--resume` 恢复。记录工具结果、文件历史快照、上下文折叠提交、内容替换记录。
- **调用的技能缓存**（`src/bootstrap/state.ts:invokedSkills`）：跟踪当前会话中调用的技能，在压缩后保持，以便模型不会重新调用已使用的技能。

**技能系统** 使用 markdown 文件与 YAML frontmatter 作为技能定义格式。发现在五个位置按优先级顺序运行。技能与子代理组合：`SkillTool` 通过用技能提示作为系统上下文分叉新的 `query()` 调用来调用技能。这意味着技能是一等公民代理，而不仅仅是提示模板。

**最复杂的机制**（按复杂度排序）：
1. **五级压缩栈** — Snip → Microcompact → Context Collapse → Autocompact → Reactive Compact。每个在不同的阈值触发。它们共同允许在有限上下文窗口内无限制的会话长度。
2. **流式工具执行器**（`src/services/tools/StreamingToolExecutor.ts`）— 在 tool_use 块流入时开始执行只读工具，在完整模型响应完成之前。减少多工具响应上的延迟。
3. **权限分类器** — `auto` 模式中预测 approve/deny 的 LLM 调用，具有置信度级别和拒绝跟踪作为先验。
4. **提示缓存共享**（`src/utils/forkedAgent.ts:CacheSafeParams`）— 子代理查询通过发送相同的系统提示前缀重用父级的 API 端提示缓存。减少压缩摘要和内存提取的成本。
5. **技能预取** — 当模型流式传输时异步发现相关技能，使用 signal/settle 模式在当前轮次完成之前呈现结果。
6. **工具结果预算管理** — 跟踪每条消息的大小，用文件引用替换过大的工具结果到 JSONL，持久化替换记录以便 `--resume` 可以重构完整内容。

**干净且隔离良好的文件**（可独立阅读和适配）：
- `src/state/store.ts` — 34 行，纯净响应式存储，无外部依赖
- `src/services/tools/toolOrchestration.ts` — 188 行，清晰只读/串行批处理逻辑
- `src/query/deps.ts` — 40 行，查询循环的清晰依赖注入接口
- `src/utils/forkedAgent.ts` — 具有明确注释的文档完善的缓存共享语义
- `src/skills/loadSkillsDir.ts` — 清晰 frontmatter 解析和多路径发现
- `src/Task.ts` — 无副作用的清晰类型定义

**混乱的文件**（谨慎阅读，不要全盘复制）：
- `src/main.tsx` — 4,683 行：CLI 解析 + 信任对话框 + REPL 启动 + 无头执行 + MCP 初始化 + 插件初始化 + 迁移 + 遥测。一切都相互交织。
- `src/query.ts` — 1,729 行：`while(true)` 体内深度嵌套的恢复逻辑；压缩、回退、流式传输和工具调度都交织在一起。
- `src/bootstrap/state.ts` — 全局可变单例，具有 200+ 字段；在其自己的注释中被承认为反模式。
- `src/Tool.ts` — 792 行混合工具定义、权限上下文类型、进度回调类型和 UI 交互钩子。

---

## 14. 设计意图推断

**循环即生成器模式**（`async function* queryLoop()`）表明架构师针对流式 UI 更新进行了优化：每个 `yield` 在不阻塞循环的情况下将消息片段推送到 React/Ink UI。生成器模式还使循环可暂停 — `--resume` 标志从 JSONL 重构消息数组并在正确的轮次重新进入新的生成器。

**依赖注入对象**（`src/query/deps.ts`）包装 `callModel`、`runTools` 和 `recordTranscript` 表明循环是为可测试性和交换 LLM 后端而设计的。Reel Creator 应采用相同的模式：`deps.callModel` 成为可交换的接口，无论调用 Claude、GPT-4o 还是本地模型。

**YAML frontmatter 技能格式** 表明技能系统是为非工程师创作而设计的。技能是文档优先的：`whenToUse` 字段是人类可读 prose，同时告知模型的工具选择和技能预取系统。Reel Creator 的工作流定义应采用相同的模式 — markdown 优先，结构化 frontmatter 供机器消费。

**四源权限层次结构**（`policySettings > projectSettings > localSettings > userSettings > cliArg > command > session`）表明有意在企业环境中运营，其中 IT 策略覆盖用户偏好。Reel Creator 的等价物是平台策略覆盖创作者偏好（例如，品牌账户无法覆盖品牌的内容策略）。

**`DeepImmutable<AppState>` 类型** 在响应式存储上表明 UI 组件被禁止直接修改状态。所有变更通过存储的调度机制路由。这防止了一类 React 组件副作用代理循环状态的 bug。Reel Creator 必须强制执行相同的边界。

**`src/bootstrap/state.ts:32` 的"DO NOT ADD MORE STATE HERE"注释** 是一种架构坦白：引导单例有机增长并成为跨切面问题的倾倒场。意图始终是保持其最小化。Reel Creator 应在编写任何代码之前预先强制执行此约束。

**五级压缩栈** 表明上下文窗口管理不是前置设计的 — 它随着用户遇到更长会话而增量增长。每个机制当前一个证明不足时被添加。Reel Creator 应从第一天起就将上下文管理作为一等 concern 来对待，而不是事后改造。

---

## 15. 安全手术图

以下更改可以安全地进行而不会导致级联故障，从最安全到最不安全排序：

**可以安全添加**（没有现有代码路径破坏）：
- 实现 `Tool` 接口（`src/Tool.ts`）的新工具类，在 `src/tools.ts` 的 `getAllBaseTools()` 中注册
- `.claude/skills/` 中的新技能 `.md` 文件 — 自动被 `loadSkillsDir.ts` 拾取
- `src/commands.ts` 中 `COMMANDS()` 的新斜杠命令
- `.claude/mcp.json` 中的新 MCP 服务器 — 通过 `src/services/mcp/client.ts` 在会话开始时加载
- 新权限规则源 — 在 `src/utils/permissions/permissions.ts` 扩展源优先级列表

**可以安全替换**（隔离良好，单一调用者）：
- `src/state/store.ts:createStore()` — 34 行，纯函数，接口清晰
- `src/services/tools/toolOrchestration.ts:runTools()` — 188 行，查询循环中的单一调用者，定义良好的输入/输出类型
- `src/query/deps.ts` — 40 行，依赖注入接口；交换 `callModel` 实现以指向不同的 LLM 提供商

**可以安全扩展**（仅添加性）：
- `src/utils/claudemd.ts` — 添加新层次结构层而不破坏现有四层
- `src/utils/sessionStorage.ts` — 添加新 JSONL 条目类型（现有读取器忽略未知类型）
- `src/bootstrap/state.ts` — 仅在绝对跨切面时添加字段；优先使用 AppState

**高风险操作**（高耦合）：
- `src/query.ts` — 修改 `while(true)` 体有破坏五种压缩机制、回退模型切换或流式协调的风险
- `src/main.tsx` — 上帝文件；初始化顺序的更改破坏 MCP 设置、插件加载或信任对话框排序
- `src/services/api/claude.ts:queryModelWithStreaming()` — 流式组装与 SDK 的事件形状紧密耦合；更改破坏 tool_use 块累积逻辑
- `src/utils/permissions/permissions.ts` — 权限评估顺序是安全关键的；错误的优先级排序创建权限提升

---

## 16. 学习价值图

以下 Claude Code 中的机制对 Reel Creator 构建者具有最高的学习价值，与代码是否可重用无关：

**深入研究：**

1. **`queryLoop()` / `deps.callModel()` / `runTools()` 三角**（`src/query.ts:307`、`src/query/deps.ts`、`src/services/tools/toolOrchestration.ts`）：揭示如何构建核心代理循环以保持可测试性，尽管它是一个无限生成器。依赖注入模式特别教授如何在不触及循环逻辑的情况下交换 LLM 后端。可直接应用于 Reel Creator 的内容生成循环。

2. **CLAUDE.md 4 层层次结构**（`src/utils/claudemd.ts`）：managed → user → project → local 的分层与 `@include` 指令支持，教授如何构建灵活的指令注入系统，适用于个人创作者和企业品牌账户。"managed"层（企业 IT 策略）直接映射到 Reel Creator 的"品牌内容策略"层。

3. **技能 frontmatter 格式**（`src/skills/loadSkillsDir.ts`）：YAML frontmatter 字段（`name`、`description`、`whenToUse`、`tools`、`model`）教授如何构建既人类可读又机器可消费的工作流定义。Reel Creator 的"内容工作流"定义应从此 schema 派生。

4. **权限模式系统**（`src/utils/permissions/permissions.ts:hasPermissionsToUseTool()`）：四种模式设计（default / plan / bypassPermissions / auto）教授如何向用户展示渐进式安全模型。deny-before-allow 评估顺序教授保守默认，这对内容发布是正确的（不要意外发布）。

5. **响应式存储模式**（`src/state/store.ts`）：34 行展示了一个最小的、框架无关的响应式容器。研究以理解 `DeepImmutable` 契约执行模式。

6. **子代理上下文隔离**（`src/utils/forkedAgent.ts`）：教授如何运行并行代理工作流（例如，同时生成字幕 + 创建缩略图）而不交叉污染消息历史或文件状态缓存。

7. **工具结果预算 + 内容替换记录**（`src/utils/sessionStorage.ts`）：教授如何处理生成的资产（图像、视频）太大而无法保持在消息历史内联的情况。用文件引用替换内容并持久化替换记录以供重构的模式直接适用于 Reel Creator 处理生成的媒体。

8. **流式工具执行器**（`src/services/tools/StreamingToolExecutor.ts`）：教授如何在工具从模型响应流入时开始执行，减少并行工具调用上的延迟。适用于 Reel Creator 的并行资产生成管道。

**为未来阶段研究：**

9. **五级压缩栈**（`src/query.ts`）：对于第一阶段为时过早，但教授在规模时出现哪些故障模式。按顺序和触发条件研究以为 Reel Creator 的令牌预算设计提供信息。

10. **通过 `CacheSafeParams` 的提示缓存共享**（`src/utils/forkedAgent.ts`）：子代理查询的成本优化模式。在 Reel Creator 达到生产流量后研究。

---

## 17. 重用 / 复制价值图

### A. COPY-PASTE（直接使用，假设许可证 OK）

这些组件体积小、隔离、正确，只需极少或零适配即可应用于 Reel Creator：

**1. 响应式存储 — `src/state/store.ts:createStore()`（34 行）**
一个纯函数，将值包装在带有 `getState()` 和 `setState()` 的响应式容器中。零 Claude Code 特定依赖。Reel Creator 正好需要此模式来分离代理循环状态和 UI 状态。复制 34 行实现，重命名类型，完成。

**2. 工具编排批处理逻辑 — `src/services/tools/toolOrchestration.ts:runTools()`（188 行）**
将工具调用分区为只读并发（最多 N 个）和非只读串行批次。分区逻辑是工具无关的 — 它调用 `tool.isReadOnly()` 并通过传入的 `runToolUse` 函数分发。Reel Creator 需要相同的逻辑：资产读取工具（获取图像、读取字幕草稿）并发运行；发布工具（发布到 TikTok、写入数据库）串行运行。复制批处理逻辑，交换工具实现。

**3. 依赖注入接口 — `src/query/deps.ts`（40 行）**
定义将代理循环与其外部依赖（`callModel`、`runTools`、`recordTranscript`）解耦的 `QueryDeps` 接口。复制此接口模式。在 Reel Creator 中，`callModel` 成为可交换的 LLM 接口，`runTools` 成为内容动作分发器，`recordTranscript` 成为生产会话记录器。

**4. JSONL 对话记录 schema 和 `recordTranscript()` — `src/utils/sessionStorage.ts`（关键类型化条目定义）**
带有类型化条目（`transcript`、`fileHistorySnapshot`、`attribution`、`contentReplacement`、`contextCollapseCommit`）的仅追加 JSONL，解决了 Reel Creator 面临的完全相同的问题：持久化具有大型生成资产的长时间内容创建会话。复制 JSONL 模式和类型化条目 discriminated union。用媒体中心的等效项替换文件中心的条目（`generatedAsset`、`publishRecord`、`revisionHistory`）。

**5. 技能定义的 frontmatter 解析 — `src/skills/loadSkillsDir.ts`（frontmatter 解析 + PromptCommand 组装）**
markdown + YAML frontmatter 技能发现模式（扫描目录 → 解析 frontmatter → 组装 PromptCommand）直接适用于 Reel Creator 的"内容工作流"定义。一个"TikTok 病毒短视频"工作流在结构上与 Claude Code 技能相同。复制发现和解析逻辑；用 Reel Creator 的内容执行引擎替换技能执行后端。

**6. 工具输入边界的 Zod 验证 — `src/services/tools/toolExecution.ts:617`（模式）**
在任何工具执行之前 `tool.inputSchema.parse(rawInput)` 的 3 行模式，失败时 `InputValidationError`，防止格式错误的 LLM 输出到达有副作用的代码。此模式原样复制到 Reel Creator 的工具调度层。每个内容动作工具在接触任何外部 API 之前验证其输入。

**7. 任务类型定义 — `src/Task.ts`**
任务状态、进度和状态的清晰 TypeScript 类型定义。零 Claude Code 特定内容。作为 Reel Creator 内容任务系统的类型基础复制。将 `Task` 重命名为 `ContentTask`，调整状态枚举值以匹配内容创建状态（`drafting`、`generating`、`reviewing`、`publishing`、`published`）。

---

### B. USE-THE-ESSENCE（适配模式，不复制代码）

这些设计在架构上适用但需要为 Reel Creator 的领域重建：

**1. 代理循环 — `src/query.ts:queryLoop()` → Reel Creator: `contentLoop()`**
*要提取的模式：* 调用 LLM、收集动作块、分发动作、将结果反馈、当没有动作剩余时停止的 `while(true)` 异步生成器。
*为何不可移植：* 1,729 行实现与 Claude Code 的压缩栈、文件工具语义、特定代码的上下文组装（git 状态、CLAUDE.md）和代码审查器 UI 组件深度纠缠。核心循环逻辑是正确的；周围的机制对内容创建是错误的。
*要重建的内容：* 具有相同 while/call/dispatch/result/stop 结构的 `contentLoop()` 异步生成器，但具有内容上下文组装（平台规格、品牌指南、热门音频）、内容特定压缩（总结生成历史而非代码差异）和内容特定停止条件（资产批准、帖子排期）。

**2. CLAUDE.md 4 层上下文注入 — `src/utils/claudemd.ts` → Reel Creator: 品牌上下文系统**
*要提取的模式：* 在会话开始时分层指令注入 — managed（平台）> user（创作者偏好）> project（活动）> local（会话覆盖）。支持 `@include` 组合指令片段。
*为何不可移植：* CLAUDE.md 字面上以 Claude 命名，结构围绕编码工作流（项目架构、编码风格）。`@include` 语法和层发现路径硬编码到 `.claude/` 目录。
*要重建的内容：* 具有相同 `@include` 指令语法但不同发现路径的 `BrandContext` 加载器，层为：platform-policy（Reel Creator 的 TOS 强制限制）> brand-guide（来自品牌的 `.reel/brand.md`）> campaign-brief（来自 `.reel/campaign.md`）> session-override（创作者当前会话笔记）。

**3. 权限门系统 — `src/utils/permissions/permissions.ts` → Reel Creator: 发布门**
*要提取的模式：* 具有源优先级排序的 deny-before-allow 规则评估。四种从保守到宽松的模式。每个动作的 allow/deny/ask 规则按源存储。
*为何不可移植：* 规则按键控到文件系统路径和 bash 命令模式。Zod schema 和规则匹配逻辑特定于文件操作和 shell 命令。
*要重建的内容：* 具有相同四种模式结构（`draft` / `review` / `scheduled` / `live`）和相同 deny-before-allow 评估的 `PublishingGate`。规则按键控到平台动作（`post_to_tiktok`、`publish_instagram_reel`、`send_to_brand_review`）而非文件路径。`policySettings > brandSettings > creatorSettings > sessionOverride` 优先级顺序完全镜像 Claude Code 的源层次结构。

**4. 带隔离上下文的子代理生成 — `src/utils/forkedAgent.ts` → Reel Creator: 并行资产代理**
*要提取的模式：* 用克隆的不可变状态、独立的 abort 控制器、单独的消息历史和将结果报告回父级的能力分叉新代理。
*为何不可移植：* `createSubagentContext()` 克隆 `fileStateCache`（本地文件的读取缓存）并继承特定代码的工作目录模型。Reel Creator 的"资产状态"是远程的（S3/CDN URL、平台草稿 ID）而非本地文件路径。
*要重建的内容：* 克隆当前活动上下文（品牌指南、平台规格、当前草稿 URL）的 `forkContentAgent()` 函数，创建隔离的 `AbortController`，并运行子 `contentLoop()`。用于并行化：生成缩略图时生成字幕、A/B 变体生成、主体内容生成时进行合规审查。

**5. 内存架构 — `src/memdir/` + `src/utils/claudemd.ts` → Reel Creator: 创作者内存**
*要提取的模式：* 两层内存 — 静态指令（在会话开始时注入的 CLAUDE.md 等效物）加上动态模型可写内存（在会话之间持久化的 MEMORY.md 等效物）。主题文件索引模式用于组织持久化内存。
*为何不可移植：* 内存文件位于具有代码项目语义的 `.claude/` 目录中。内存提取提示针对提取编码偏好和项目知识进行了调整。
*要重建的内容：* 具有 `.reel/brand-memory.md`（品牌持久化事实：语调、重复 hashtag、平台性能模式）和 `.reel/session-memory.md`（会话持久化上下文：当前活动、受众笔记）的 `CreatorMemory` 系统。针对内容创建偏好调整的内存提取提示。

**6. 钩子系统 — `src/hooks/`（执行前/后工具钩子）→ Reel Creator: 内容事件钩子**
*要提取的模式：* 注册在特定工具调用之前/之后触发的函数。预钩子可以中止动作。后钩子接收结果用于副作用（日志、通知、分析）。
*为何不可移植：* 钩子针对文件系统 和 shell 工具名称（`FileWriteTool`、`BashTool`）注册。React/Ink 钩子架构特定于终端 UI。
*要重建的内容：* 在内容动作之前/之后触发的 `ContentHookRegistry`（`BeforePublish`、`AfterGenerate`、`OnBrandViolation`）。预发布钩子检查合规性。生成后钩子触发缩略图生成。OnBrandViolation 钩子队列人工审查。

---

### C. JUST-FOR-LEARN（研究，第一阶段不应用）

这些机制具有教育价值但对于 Reel Creator 4 月 15 日的第一阶段截止日期为时过早：

**1. 五级压缩栈（`src/query.ts` — Snip → Microcompact → Context Collapse → Autocompact → Reactive Compact）**
*学习内容：* 上下文窗口在规模时溢出。五种机制代表五种独立发现的故障模式，每种需要不同的缓解。Autocompact 机制（分叉一个代理来总结完整对话，用摘要替换消息）是最可转移的洞察：将上下文总结视为一等代理任务，而非字符串截断。
*为何不在第一阶段：* Reel Creator 第一阶段的会话将是短暂的（一个活动、一篇帖子）。上下文溢出在 Phase 2+ 引入多会话活动和多平台工作流时才会出现。在观察到故障之前实现所有五种机制是过早优化。

**2. 流式工具执行器 — `src/services/tools/StreamingToolExecutor.ts`**
*学习内容：* 在工具从模型响应流出时开始执行，而非等待完整响应完成。用于协调部分接收的 tool_use 块与执行管道的 signal/settle 模式。
*为何不在第一阶段：* 第一阶段 Reel Creator 内容动作（生成字幕、创建缩略图、排期帖子）具有延迟容忍且是顺序的。在 Phase 2 需要并行高吞吐量资产管道之前，流式执行的复杂性是不合理的。

**3. 通过 CacheSafeParams 的提示缓存共享 — `src/utils/forkedAgent.ts`**
*学习内容：* 具有与父级相同系统提示前缀的子代理查询重用 API 提供商的缓存 KV-store，减少压缩摘要、内存提取和代理摘要的延迟和令牌成本。
*为何不在第一阶段：* Reel Creator 第一阶段不会有足够流量使提示缓存优化成为有意义的关注。在流量数据揭示子代理调用的成本概况之后实施。

**4. 权限分类器（基于 LLM 的 auto 模式）— `src/services/api/claude.ts` auto 模式调用**
*学习内容：* 分类器使用当前工具调用和会话的原始意图调用快速模型，问："此动作是否与用户开始此会话要做的匹配？"拒绝跟踪在会话内建立使分类器更准确的先验。
*为何不在第一阶段：* 第一阶段 Reel Creator 将在明确的 `review` 模式（所有发布动作需要确认）下运营。自动分类器只在 Phase 3 当 Reel Creator 运行计划性自主发布活动而没有在线人类时才有价值。

**5. ToolSearchTool 延迟工具加载 — `src/tools/ToolSearchTool/ToolSearchTool.ts`**
*学习内容：* 并非所有工具都需要在初始上下文窗口中。返回完整 JSON schema 的搜索工具允许模型发现和使用不在其初始工具集中的工具。这为常见任务保持初始提示较短。
*为何不在第一阶段：* 第一阶段 Reel Creator 将有少于 20 个工具。延迟加载只在工具集超过上下文预算时值得 — 可能 Phase 3+ 当 Reel Creator 支持 50+ 平台集成时。

**6. 带 signal/settle 的技能预取 — `src/skills/loadSkillsDir.ts` + 异步发现**
*学习内容：* 当模型流式传输其响应时，使用部分响应文本上的关键字匹配异步发现当前轮次相关的技能。在模型完成之前呈现结果，以便相关技能立即可用。
*为何不在第一阶段：* 第一阶段 Reel Creator 将有一个小的已知工作流库。在工作流库增长到需要智能发现的 30+ 工作流之前，异步预取的复杂性是不合理的。

**7. 多提供商 API 路由 — `src/services/api/claude.ts`（Bedrock / Vertex / 第一方分支逻辑）**
*学习内容：* 模型调用层在单个 `queryModelWithStreaming()` 接口后抽象三个 API 表面（Anthropic 第一方、AWS Bedrock、Google Vertex）。每个提供商的认证、端点 URL 和请求形状不同。
*为何不在第一阶段：* Reel Creator 第一阶段针对单个 LLM 提供商。企业多云路由在为需要特定云区域数据驻留的品牌账户服务时才相关。

---

## 18. 心理模型包

- `queryLoop()` 迭代：调用 LLM，收集 tool_use 块，执行工具，将结果反馈，当剩余零块时停止。
- 每个工具调用在执行前通过 `hasPermissionsToUseTool()`；拒绝规则在允许规则之前跨七个源层评估。
- 响应式存储（`createStore()`）强制执行硬边界：代理循环变更状态，UI 仅读取它。
- CLAUDE.md 在会话开始时注入四层静态上下文；MEMORY.md 提供模型可写的持久化跨会话内存。
- 技能是带有 YAML frontmatter 的 markdown 文件，模型像工具一样调用；每个技能生成一个隔离的 forked `query()` 调用。
- 会话状态同时存在于三个地方：内存中的 `mutableMessages[]`、响应式 `AppState` 和磁盘上的仅追加 JSONL。
- 五级压缩栈存在是因为上下文溢出逐渐表现出来；每个级别在不同的阈值触发以避免不必要的摘要。
- `src/main.tsx` 是一个 4,683 行的上帝文件；其他一切相对干净，可外科替换。
