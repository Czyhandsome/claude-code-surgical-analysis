[返回目录](../README.cn.md) · **中文版** | [English](openclaw-core-agent-runtime-surgical-analysis.md)

**目录：** [0](#0-范围) · [1](#1-一句话机器定义) · [2](#2-为什么存在这部分) · [3](#3-顶层控制图) · [4](#4-具体入口点) · [5](#5-正常路径外科演练) · [6](#6-调度图) · [7](#7-状态模型) · [8](#8-数据转换链) · [9](#9-副作用图) · [10](#10-重要分支点) · [11](#11-故障和恢复逻辑) · [12](#12-真正的核心循环) · [13](#13-架构压缩) · [14](#14-设计意图推断) · [15](#15-安全手术图) · [16](#16-学习价值图) · [17](#17-重用复制价值图) · [18](#18-心理模型包)

---

# OpenClaw 核心 Agent 运行时 — 代码外科分析报告

**分析对象：** `repos/openclaw` 中与 AgentOS 风格内容创作 shell 最相关的核心 agent 运行时
**日期：** 2026-04-01
**分析问题：** 当一个 AgentOS 团队构建一个先 local-first、再走向 SaaS 的 agent shell 时，OpenClaw 中哪些机制适合直接复制、深度内化，或暂时只作为学习对象？

---

## 0. 范围

**本报告分析什么：** OpenClaw 从 CLI / gateway 入口进入单个 agent turn 的执行脊柱，重点覆盖 `src/entry.ts`、`src/cli/run-main.ts`、`src/cli/program/register.agent.ts`、`src/commands/agent-via-gateway.ts`、`src/agents/agent-command.ts`、`src/agents/command/attempt-execution.ts`、`src/agents/pi-embedded-runner/run.ts`、`src/agents/pi-embedded-runner/run/attempt.ts`、`src/context-engine/*`、`src/agents/pi-tools.ts`、`src/agents/openclaw-tools.ts`、`src/agents/skills/*`、`src/plugins/loader.ts`，以及 `src/cron/isolated-agent/run.ts` 的 cron 隔离执行路径。

**本报告回答什么：** OpenClaw 实际上是如何执行一个 agent turn 的，真正的控制中心在哪里，以及当你构建一个新的 AgentOS shell 时，哪些部分值得优先吸收，哪些部分又过于产品化、不适合过早照搬。

**有意不分析什么：** 浏览器扩展内部实现、macOS App UI、大部分媒体流水线、provider 侧插件广度，以及超出调度边界之外的渠道特化实现。这些内容会扩大产品面，但不定义核心 agent machine。

---

## 1. 一句话机器定义

> 这个子系统本质上是一个“有会话的 agent turn machine”：它把经过路由的 prompt 与 workspace / session 状态，转化为一个带工具、可持久化的回复，并受到 sandbox 策略、模型与鉴权 failover、插件加载的能力面，以及 transcript 一致性的约束。

---

## 2. 为什么存在这部分

**它解决的问题：** 现代 agent shell 不能只是把文本发给模型。它必须知道是谁在发起请求、当前 turn 属于哪个 agent / workspace / session、哪些工具和技能是合法的、transcript 如何维持一致、重试 / failover / compaction 如何恢复，以及同一套核心运行时如何同时支撑 CLI、gateway、cron 和渠道投递。

**为什么逻辑在这里：** OpenClaw 尽量让产品传输层保持薄，把持久而关键的执行语义放进 agent runtime。像 `src/entry.ts:runMainOrRootHelp`、`src/commands/agent-via-gateway.ts:agentCliCommand`、`src/gateway/server-methods/agent.ts:dispatchAgentRunFromGateway` 更多是在做路由；真正决定机器如何运行的是 `src/agents/agent-command.ts:agentCommandInternal` 和 `src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`。

**它拥有的边界：** 它拥有“一次 session 中的一个 agent turn”这个工作单元。在 turn 开始前解析 session / workspace / model / tool / context 状态，在 turn 进行中驱动 live model/tool loop，在 turn 结束后持久化或修复状态。

---

## 3. 顶层控制图

```text
[CLI / Gateway / Cron 入口]
  -> [agent session + model 解析]
  -> [runAgentAttempt 调度]
       -> [CLI backend runner] -----------------> [子 CLI 进程结果]
       -> [embedded runner]
            -> [plugins + skills + sandbox + tools]
            -> [SessionManager + context engine]
            -> [live agent session prompt/tool loop]
            -> [afterTurn / compaction / persistence]
  -> [payload 归一化 + 可选投递]
```

**如何阅读这张图：** 从左到右表示一个 turn 的流动。每个箭头都意味着“把当前 turn 的状态传给下一个 reducer / dispatcher”。真正重要的分叉不是 CLI 对 gateway，而是 CLI backend 执行和 embedded 执行之间的分裂。

---

## 4. 具体入口点

### 打包后的 CLI 启动入口

- **文件 / 函数 / 类：** `repos/openclaw/src/entry.ts:runMainOrRootHelp`
- **输入类型：** `process.argv` 加环境变量
- **第一个有意义的下游跳转：** `repos/openclaw/src/cli/run-main.ts:runCli`
- **同步 / 异步：** 异步 bootstrap

### `openclaw agent ...` 命令

- **文件 / 函数 / 类：** `repos/openclaw/src/cli/program/register.agent.ts:registerAgentCommands`
- **输入类型：** Commander 解析后的 CLI 参数
- **第一个有意义的下游跳转：** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **同步 / 异步：** 异步请求-响应

### 受信任的本地 agent 运行

- **文件 / 函数 / 类：** `repos/openclaw/src/agents/agent-command.ts:agentCommand`
- **输入类型：** 来自 CLI 或内部调用方的 `AgentCommandOpts`
- **第一个有意义的下游跳转：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **同步 / 异步：** 异步请求-响应

### 面向网络入口的 agent 运行

- **文件 / 函数 / 类：** `repos/openclaw/src/agents/agent-command.ts:agentCommandFromIngress`
- **输入类型：** 带显式信任标志（如 `senderIsOwner`、`allowModelOverride`）的 ingress 参数
- **第一个有意义的下游跳转：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **同步 / 异步：** 异步请求-响应

### Gateway RPC `agent`

- **文件 / 函数 / 类：** `repos/openclaw/src/gateway/server-methods/agent.ts:dispatchAgentRunFromGateway`
- **输入类型：** gateway RPC 请求，以及请求级客户端身份 / 能力信息
- **第一个有意义的下游跳转：** `repos/openclaw/src/agents/agent-command.ts:agentCommandFromIngress`
- **同步 / 异步：** 异步请求-响应，可带后台完成回调

### 定时隔离运行

- **文件 / 函数 / 类：** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **输入类型：** cron job payload，以及解析后的 session key
- **第一个有意义的下游跳转：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent` 或 `repos/openclaw/src/agents/cli-runner.ts:runCliAgent`
- **同步 / 异步：** 异步定时任务

---

## 5. 正常路径外科演练

**选定场景：** 用户执行 `openclaw agent --local --agent ops --message "Summarize logs"`，OpenClaw 以启用工具的 embedded 模式完成一次 turn。

**为什么选这个场景：** 这是最能暴露 AgentOS 价值的路径：本地 shell 入口、显式 session / workspace / model 解析、tool / skill / context 组装、live tool loop，以及 transcript 持久化，而不会被 gateway 投递层噪音淹没。

### Step 1 — 解析 `agent` 命令

- **位置：** `repos/openclaw/src/cli/program/register.agent.ts:registerAgentCommands`
- **输入：** `message`、`agent`、`thinking`、`local`、`deliver` 等 Commander 选项
- **前置假设：** Node/runtime bootstrap 已完成 env 和 argv 的标准化
- **本步骤做什么：** 把 CLI 子命令派发到 `agentCliCommand`
- **输出：** 标准化后的 `opts`
- **重要性：** 这里决定同一个用户命令究竟走 gateway 还是 embedded 路径

### Step 2 — 选择 gateway 或 local 执行

- **位置：** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **输入：** 解析后的 CLI 参数和 runtime logger
- **前置假设：** 命令级校验已经保证 message 存在
- **本步骤做什么：** `--local` 为真时直接走 embedded；否则先尝试 gateway，失败再回落到 embedded
- **输出：** 对 `agentCommand` 的调用
- **重要性：** OpenClaw 将默认产品传输层和核心 agent runtime 分离，这个分层对未来的 CLI + SaaS AgentOS 很有价值

### Step 3 — 解析 session、workspace 与带策略的默认值

- **位置：** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution`
- **输入：** `AgentCommandOpts` 和 runtime env
- **前置假设：** 当前是本地可信调用，因此 `senderIsOwner` 已由 `agentCommand` 设置
- **本步骤做什么：** 校验 message 和 override，加载配置，经 `resolveSession` 解析 session 身份，解析 agent workspace / agent dir，计算 timeout，并导出 outbound context
- **输出：** 包含 `sessionId`、`sessionKey`、`sessionEntry`、`workspaceDir`、`agentDir`、`timeoutMs` 和持久化 override 的准备好执行的 bundle
- **重要性：** 这是把原始用户请求变成“有会话的工作单元”的第一处

### Step 4 — 冻结技能快照并持久化 session 侧运行状态

- **位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **输入：** 准备好的执行 bundle 和当前 `SessionEntry`
- **前置假设：** session / workspace 已解析，workspace bootstrap 文件已存在
- **本步骤做什么：** 对新 session 或缺失快照的 session 构建 `skillsSnapshot`，持久化 thinking / verbose override，选定生效模型 / provider，并通过 `resolveSessionTranscriptFile` 解析 transcript 文件
- **输出：** 完整 session 状态和 `sessionFile`
- **重要性：** 在任何 model loop 开始之前，OpenClaw 先固定技能可见性与 transcript 位置

### Step 5 — 把尝试分派给 embedded runner

- **位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal` 经由 `repos/openclaw/src/agents/model-fallback.ts:runWithModelFallback` 进入 `repos/openclaw/src/agents/command/attempt-execution.ts:runAgentAttempt`
- **输入：** provider / model 选择、session 标识、transcript 路径、prompt 文本、skillsSnapshot、tool/runtime 参数
- **前置假设：** fallback 策略和 auth profile 兼容性已经足够明确，可以开始一次 attempt
- **本步骤做什么：** 因当前 provider 不是 CLI backend，所以选择 embedded 路径并调用 `runEmbeddedPiAgent`
- **输出：** 一个正在运行的 embedded attempt
- **重要性：** 这是“驱动外部 coding CLI”和“驱动 OpenClaw 自己的 embedded loop”之间最核心的调度缝

### Step 6 — 串行化 turn，并加载 runtime plugins

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **输入：** 完整的 embedded run 参数 bundle
- **前置假设：** session 状态与 transcript 路径已存在
- **本步骤做什么：** 把 turn 排入 session lane 和 global lane，解析 workspace，通过 `ensureRuntimePluginsLoaded` 加载 runtime plugins，解析 auth / model 状态，单次解析 context engine，并进入 retry / failover loop
- **输出：** 一个具体的 `runEmbeddedAttempt` 调用
- **重要性：** OpenClaw 把一次 turn 当成被串行化管理的操作工作，而不是一个自由漂浮的异步函数调用

### Step 7 — 构建 sandbox、skills、tools 与运行时 system prompt

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **输入：** 当前模型配置、workspace、session id、skillsSnapshot、prompt 文本和 runtime context
- **前置假设：** plugin registry 已激活，context engine 实例也已就绪
- **本步骤做什么：** 解析 sandbox context，恢复 skill env override，加载 workspace / bootstrap 文件，构建 skills prompt，通过 `createOpenClawCodingTools` 构造 tools，挂接 MCP / LSP tools，计算 runtime capabilities，并通过 `buildEmbeddedSystemPrompt` 生成最终 system prompt
- **输出：** `effectiveTools`、`systemPromptText` 和运行时元数据
- **重要性：** 这里把 OpenClaw 的“AgentOS 策略”物化成模型真正能看到的 prompt / tool surface

### Step 8 — 获取 transcript 所有权并重建 session 状态

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:acquireSessionWriteLock`、`SessionManager.open`、`prepareSessionManagerForRun`、`createAgentSession`
- **输入：** `sessionFile`、`effectiveTools`、`systemPromptText`、auth / model registry
- **前置假设：** workspace 与 sandbox 目录都已存在
- **本步骤做什么：** 获取 transcript 写锁，修复 transcript 损伤，打开 `SessionManager`，准备 settings / resource loader，拆分 built-in 与 custom tools，创建 live agent session 对象
- **输出：** `activeSession`
- **重要性：** live model/tool loop 不是在原始文件上运行，而是在一个受保护、从 transcript 重建出来的内存 session 上运行

### Step 9 — 清洗历史并组装 context engine 输出

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:sanitizeSessionHistory` 和 `assembleAttemptContextEngine`
- **输入：** `activeSession.messages`、transcript 策略、token budget、prompt 文本
- **前置假设：** session 对象已处于 live 状态，tool set 已固定
- **本步骤做什么：** 去掉无效历史模式，限制 turn 数，修复 tool-use/result 配对，再让 context engine 改写 messages 并追加 `systemPromptAddition`
- **输出：** 模型调用实际消费的历史与 system prompt
- **重要性：** OpenClaw 明确区分“存储层 transcript 真相”和“模型上下文真相”

### Step 10 — Prompt live agent session 并观察 tool loop

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:activeSession.prompt`
- **输入：** 生效后的 prompt 文本，以及可选的 prompt-local images
- **前置假设：** session history、tool schema 和 stream wrapper 已经就位
- **本步骤做什么：** 订阅 assistant / tool 事件，对外转发工具输出和 delta，执行模型 prompt，让底层 agent 执行 tool call，并在需要时等待 compaction retry
- **输出：** 被修改后的 session messages、assistant 文本、tool metadata、usage totals，以及可能的 prompt/tool errors
- **重要性：** 这才是真正的 inner agent loop，尽管底层细节委托给了 `@mariozechner/pi-coding-agent`

### Step 11 — 完成 turn 后状态收尾

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:finalizeAttemptContextEngineTurn`
- **输入：** message snapshot、prompt error 状态、token budget、runtime context
- **前置假设：** attempt 已经完成、终止或超时
- **本步骤做什么：** 让 context engine 在 turn 后 ingest / maintain / compact，在需要时写入 prompt-error 条目，计算最终 attempt 结果，并在安全 flush 工具结果后销毁 live session
- **输出：** `EmbeddedRunAttemptResult`
- **重要性：** OpenClaw 不把 post-turn maintenance 当作可选清理，而是把它视为影响未来 turn 质量的核心行为

### Step 12 — 归一化 payload 并返回调用方

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent` 回到 `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **输入：** attempt 结果与 retry / failover 状态
- **前置假设：** session transcript 和 session store 已经被更新到可继续未来 turn 的程度
- **本步骤做什么：** 对可恢复失败执行重试，构建最终 payload 和 agent metadata，按需走渠道投递，然后返回原始 CLI 调用方
- **输出：** 用户可见回复以及运行元数据
- **重要性：** 这里把嘈杂的运行时 attempt 收敛成稳定 turn 结果

---

## 6. 调度图

### Gateway-first 与 local-first 命令调度

- **选择条件：** `--local` 标志以及 gateway 是否失败
- **如何选择：** `agentCliCommand` 直接走本地，或先走 `agentViaGatewayCommand`
- **解析器位置：** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **引入的风险：** 调用方容易误以为 gateway 模式就是核心运行时，但它其实只是传输包装层

### 可信调用与 ingress 调用的执行策略

- **选择条件：** 本地调用路径还是网络入口路径
- **如何选择：** 通过不同导出函数注入不同的信任默认值
- **解析器位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommand` 和 `repos/openclaw/src/agents/agent-command.ts:agentCommandFromIngress`
- **引入的风险：** 如果只复制本地路径，很容易在 SaaS 阶段丢掉清晰的信任边界

### CLI backend 与 embedded backend

- **选择条件：** 选中的 provider 是否是 CLI backend
- **如何选择：** `runAgentAttempt` 中的 `isCliProvider`
- **解析器位置：** `repos/openclaw/src/agents/command/attempt-execution.ts:runAgentAttempt`
- **引入的风险：** CLI 子进程与 embedded session 的 session 复用语义差异极大

### 模型 / 鉴权 / fallback 调度

- **选择条件：** 持久化 override、显式 override、auth profile 顺序、failover 原因、配置中的 fallback
- **如何选择：** `agentCommandInternal` 确定初始模型；`runWithModelFallback` 与 `runEmbeddedPiAgent` 在模型 / profile / thinking 层之间轮转
- **解析器位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal` 和 `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **引入的风险：** 选择状态分散在 session store、配置、运行时 auth 状态和 retry loop 本地状态中

### Context engine slot 调度

- **选择条件：** `config.plugins.slots.contextEngine` 或默认 slot id
- **如何选择：** 按 slot 值进行 registry lookup
- **解析器位置：** `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine`
- **引入的风险：** 接缝本身很干净，但默认引擎是兼容壳而不是强力实现；错误替换会静默降低质量

### Plugin registry 调度

- **选择条件：** plugin config、workspace 路径、load path、runtime subagent mode、显式 plugin scope
- **如何选择：** `resolveRuntimePluginRegistry` 中基于 cache key 的 registry 解析
- **解析器位置：** `repos/openclaw/src/plugins/loader.ts:resolveRuntimePluginRegistry`
- **引入的风险：** 运行时行为依赖进程级 plugin 状态和缓存兼容性，而不只是局部函数参数

### Tool surface 调度

- **选择条件：** sandbox 状态、模型 / tool 支持、agent policy、group policy、owner 状态、subagent 深度、message channel、plugin tool metadata
- **如何选择：** `createOpenClawCodingTools` 先组合原始 tools，再通过 `applyToolPolicyPipeline` 过滤
- **解析器位置：** `repos/openclaw/src/agents/pi-tools.ts:createOpenClawCodingTools`
- **引入的风险：** 最终 tool list 是多层策略的涌现结果；调试“为什么某个工具消失了”必须理解整条策略管线

### Skill source 调度

- **选择条件：** workspace skills、bundled skills、plugin skill dirs、config allow / deny、agent skill filter、runtime eligibility
- **如何选择：** 文件系统扫描加 prompt 级过滤
- **解析器位置：** `repos/openclaw/src/agents/skills/workspace.ts:resolveSkillsPromptForRun`
- **引入的风险：** 如果快照 / 过滤逻辑理解错误，prompt 可见能力和实际安装能力会分叉

---

## 7. 状态模型

### `SessionEntry`

- **类型：** session-store record
- **权威性：** 对路由、override、delivery context、skill snapshot、model/runtime metadata 来说是权威状态
- **持久还是瞬时：** 持久，存于 session store
- **创建位置：** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution` 经由 `resolveSession`
- **变更位置：** `agentCommandInternal`、`persistSessionEntry`、cron 隔离路径、gateway session 更新
- **读取方：** 模型选择、delivery logic、skill snapshot 复用、CLI session binding 复用
- **关键不变量：** `sessionKey` 是持久 bucket；`sessionId` 可以轮转；override 必须与有效 provider / model / auth 状态一致

### Transcript JSONL session file

- **类型：** 由 `SessionManager` 拥有的 transcript 文件
- **权威性：** 对 turn history 来说是权威真相
- **持久还是瞬时：** 持久，落盘
- **创建位置：** `resolveSessionTranscriptFile`
- **变更位置：** live agent session、transcript repair、prompt-error 持久化、compaction、ACP transcript 持久化
- **读取方：** `SessionManager.open`、history sanitation、context assembly、compaction、session-history tools
- **关键不变量：** 单写者、有效 session header、合理的 tool-use/result 顺序、无孤儿 trailing user turn

### `skillsSnapshot`

- **类型：** 序列化的 skill prompt / metadata snapshot
- **权威性：** 它源于文件系统技能状态，但在该 session 后续 turn 中又成为权威快照
- **持久还是瞬时：** 持久，存进 `SessionEntry`
- **创建位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **变更位置：** 新 session 或 refresh 时点
- **读取方：** `runEmbeddedAttempt` 与 cron isolated-agent 运行
- **关键不变量：** 快照必须与当前 turn 所依赖的 workspace / config 假设一致

### Live embedded attempt state

- **类型：** `runEmbeddedPiAgent` 内 retry-loop 局部状态
- **权威性：** 只对当前这一轮 run 权威
- **持久还是瞬时：** 瞬时
- **创建位置：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **变更位置：** auth rotation、fallback selection、compaction counts、usage accumulation、timeout bookkeeping
- **读取方：** retry 逻辑与最终结果整形
- **关键不变量：** provider / model / auth / think-level 的过渡必须在重试间保持内部一致

### Active plugin registry

- **类型：** 进程全局 plugin registry snapshot
- **权威性：** 对运行时可发现的 plugin tools / channels / routes 是权威
- **持久还是瞬时：** 进程期内瞬时，带缓存
- **创建位置：** `repos/openclaw/src/plugins/loader.ts:loadOpenClawPlugins`
- **变更位置：** `setActivePluginRegistry`
- **读取方：** channel registry、tool resolution、provider capability / runtime helper
- **关键不变量：** 当前 active registry 必须匹配本次 runtime 的 cache key 假设

### Context engine instance

- **类型：** `ContextEngine`
- **权威性：** 一旦被选中，它就是 context assembly / maintenance 的权威主体
- **持久还是瞬时：** 进程期内瞬时，按每次 run 解析
- **创建位置：** `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine`
- **变更位置：** 引擎自己拥有的 `bootstrap`、`assemble`、`afterTurn`、`compact` 等方法
- **读取方：** `runEmbeddedPiAgent` 和 `runEmbeddedAttempt`
- **关键不变量：** 它必须遵守运行时关于 transcript ownership 与 compaction ownership 的契约

---

## 8. 数据转换链

1. CLI / gateway / cron 输入以传输特定的参数或 RPC payload 形式进入。
2. `prepareAgentCommandExecution` 将这些传输层输入归一化为 session-scoped execution input。
3. `agentCommandInternal` 把持久化 session 状态、skills snapshot 状态和模型 override 状态融合成一个 run-ready bundle。
4. `runAgentAttempt` 再把 bundle 分流成 CLI-backend 参数或 embedded-run 参数。
5. `runEmbeddedPiAgent` 将 run 参数转成一个带 retry 管理、并且已解析 auth / model / context-engine 的 attempt context。
6. `runEmbeddedAttempt` 把 workspace / config / session 状态转换成模型可见物：tools、skills prompt、bootstrap context、runtime system prompt、清洗后的 history。
7. `activeSession.prompt(...)` 再把 prompt、history 和 tool schema 变成 assistant messages 与 tool-call cycles。
8. post-turn hooks 与 context-engine 方法把原始 messages 变成被维护过的 transcript / context 状态。
9. `runEmbeddedPiAgent` 最后把嘈杂的 attempt outcome 变成稳定 payload、usage meta 和可恢复失败路径。

---

## 9. 副作用图

### Session store 持久化

- **触发位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`、`repos/openclaw/src/agents/command/attempt-execution.ts:persistSessionEntry`
- **保护条件：** session key 和 store path 必须存在
- **是否可重试：** 有条件可重试
- **是否幂等：** 对同一合并写入基本是幂等的
- **失败后果：** 后续 turn 失去 override、skills snapshot 复用或 CLI session binding

### Transcript 文件变更

- **触发位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts` 中的 `SessionManager.open` 路径，以及 `persistAcpTurnTranscript`
- **保护条件：** 已获取 session lock，session file 已修复 / 准备完成
- **是否可重试：** 有条件可重试
- **是否幂等：** 否
- **失败后果：** 历史损坏，或出现重复 / 孤儿 turn 状态

### Plugin 加载

- **触发位置：** `repos/openclaw/src/agents/runtime-plugins.ts:ensureRuntimePluginsLoaded`
- **保护条件：** config / workspace / cache context 兼容
- **是否可重试：** 是
- **是否幂等：** 对同一 cache key 基本是幂等的
- **失败后果：** tool、channel、provider、hook 或 route 从运行时表面静默消失

### Skill env 注入

- **触发位置：** `repos/openclaw/src/agents/skills/env-overrides.ts:applySkillEnvOverrides`
- **保护条件：** skill 配置、env 清洗、敏感 key 白名单
- **是否可重试：** 是
- **是否幂等：** 有条件成立，依赖引用计数
- **失败后果：** skill 拿不到凭据，或不安全 env var 泄露给子进程

### Child CLI 进程执行

- **触发位置：** `repos/openclaw/src/agents/cli-runner/execute.ts:executePreparedCliRun`
- **保护条件：** provider 必须解析为 CLI backend，进程 supervisor 必须成功 spawn
- **是否可重试：** 有条件可重试
- **是否幂等：** 否
- **失败后果：** 拿不到 turn 结果，且子 CLI 可能已经产生部分外部副作用

### Model / API 流式调用

- **触发位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:activeSession.prompt`
- **保护条件：** auth 已解析，stream wrapper 已安装，timeout / abort signal 已激活
- **是否可重试：** 是，在 failover / compaction / thinking downgrade 策略下可重试
- **是否幂等：** 否
- **失败后果：** assistant 输出部分完成，产生 prompt error，并触发重试 / failover 升级

### Tool 执行

- **触发位置：** live agent session 内部，在 tool schema 注册之后
- **保护条件：** tool policy pipeline、sandbox policy、owner policy、provider/tool support
- **是否可重试：** 取决于具体工具
- **是否幂等：** 取决于具体工具
- **失败后果：** prompt continuation 失败、副作用重复，或 tool-loop 升级

### Cron 投递

- **触发位置：** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **保护条件：** delivery target 已解析，cron delivery contract 满足
- **是否可重试：** 有条件可重试
- **是否幂等：** 否
- **失败后果：** 定时工作完成了，但结果没有被送达目标渠道

---

## 10. 重要分支点

### Gateway-first vs local-first 命令路径

- **条件：** `opts.local === true`
- **检查位置：** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **替代路径：** 立即走本地 embedded / CLI runtime；或先走 gateway RPC，失败再退回 embedded
- **重要性：** 产品拓扑在这里变化，但核心 turn machine 不变

### ACP 与普通 agent runtime

- **条件：** ACP session resolution 返回 `ready`
- **检查位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **替代路径：** ACP control-plane turn；或普通 CLI-backend / embedded turn
- **重要性：** ACP 绕过了 embedded runtime 的大部分路径，并采用不同的 session 语义

### CLI backend 与 embedded backend

- **条件：** `isCliProvider(providerOverride, cfg)`
- **检查位置：** `repos/openclaw/src/agents/command/attempt-execution.ts:runAgentAttempt`
- **替代路径：** 子进程 CLI 运行；embedded live session 运行
- **重要性：** 这是仓库中最有杠杆的运行时接缝

### 新 session 与复用 session

- **条件：** session 新鲜度 / reset 结果，以及已有的 `skillsSnapshot`
- **检查位置：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **替代路径：** 创建快照和 transcript 路径；或复用既有快照和 transcript 血缘
- **重要性：** 决定本次 turn 是否复用既有操作记忆

### 启用 sandbox 与直接使用 host workspace

- **条件：** sandbox context 是否解析为 enabled
- **检查位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **替代路径：** 使用 sandbox workspace / fs bridge / browser bridge；或直接使用 host workspace
- **重要性：** 这里会改变 tool set、文件系统防护和 spawned subagent 的 workspace 继承方式

### Context-engine-owned compaction 与 runtime-owned compaction

- **条件：** `contextEngine.info.ownsCompaction === true`
- **检查位置：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **替代路径：** 由 engine 自己执行 compaction 生命周期；或由 runtime 驱动 legacy compaction 流程
- **重要性：** transcript reduction 的所有权在这里跨越 plugin seam 发生转移

### Compaction retry 与 failover rotation

- **条件：** retry loop 中出现 timeout / overflow / tool-result-size 信号
- **检查位置：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **替代路径：** 对同一模型做 compact 后重试；截断工具结果；轮转 auth profile / model / thinking level；或直接失败
- **重要性：** OpenClaw 优先尝试保住当前 session，而不是轻易放弃它

---

## 11. 故障和恢复逻辑

### 故障面

| 故障面 | 检测位置 | 传播方式 | 对外翻译成什么 |
|---|---|---|---|
| 无效 CLI 或命令 override | `prepareAgentCommandExecution` | 立即抛出 | 用户可见命令错误 |
| 未知 context engine slot | `resolveContextEngine` | 抛出 | 运行启动失败 |
| CLI 子进程无输出卡死 | `executePreparedCliRun` | 包装为 `FailoverError` | timeout failover + heartbeat/system event |
| Prompt / API 超时 | `runEmbeddedAttempt`、`runEmbeddedPiAgent` | abort + retry / compaction / failover | timeout payload 或重试 |
| Context overflow | `runEmbeddedPiAgent` attempt 后 | 被观察并分类 | compaction 或 tool-result truncation |
| Auth / profile 失败 | `runEmbeddedPiAgent` 内嵌 auth controller | 标记 profile 失败并切到下一个候选 | 在另一个 auth profile 上重试 |
| Live session model switch | `runEmbeddedPiAgent` | 抛出 `LiveSessionModelSwitchError` | 在新模型选择下重启 |
| Transcript 损坏 / 孤儿状态 | `repairSessionFileIfNeeded`、history sanitation、orphan-user repair | 尽可能原地修复 | 继续运行或以退化历史继续 |

### 恢复行为

- **重试逻辑：** 主 retry loop 位于 `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`。它会在 auth rotation、过载退避、timeout-triggered compaction、overflow-triggered compaction 和 tool-result truncation 下重试。
- **Fallback 逻辑：** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal` 中的 `runWithModelFallback` 会在当前 provider / model 路径硬失败时轮转到配置中的 fallback models。
- **超时行为：** `runEmbeddedAttempt` 同时安装 overall timeout 和 idle timeout。在 compaction 期间发生的超时会触发特殊 snapshot 选择逻辑，而不会盲信部分 compact 后的 live 状态。
- **生命周期错位风险：** plugin registry 是进程全局的；session store 状态和 transcript 状态可能分叉；CLI session 复用可能过期；skill env var 若没有引用计数清理会泄露；live agent session 的一部分又属于外部 Pi 库，因此 OpenClaw 增加了多层 repair / guard 来保护这个边界。

---

## 12. 真正的核心循环

```text
一个 embedded run 的循环：
  1. 解析 workspace / plugins / auth / context engine -> `runEmbeddedPiAgent`
  2. 构建 sandbox / skills / tools / system prompt -> `runEmbeddedAttempt`
  3. 从 transcript 重建 live session -> `runEmbeddedAttempt`
  4. 清洗 history + 组装 context engine 输出 -> `runEmbeddedAttempt`
  5. prompt live agent session -> `runEmbeddedAttempt`
  6. 观察 tool / assistant / compaction 事件 -> `subscribeEmbeddedPiSession`
  7. 分类成功或失败 -> `runEmbeddedPiAgent`
  8. 如有需要则 compact / 轮转 auth / 轮转模型 / 降 thinking / 重试 -> `runEmbeddedPiAgent`
  直到成功、出现不可恢复失败、或达到重试边界
```

外层循环才是 OpenClaw 的真正核心循环。内层 tool-call loop 属于 live Pi agent session，但 OpenClaw 在它外面包裹了如此多的重试、组装和持久化控制，以至于真正的运行时控制中心其实是外层壳。

---

## 13. 架构压缩

**真正的重心：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`。如果删掉它，OpenClaw 就失去了把 session state、prompt state、tool state 和 context-engine state 汇合成一次 coherent attempt 的核心机制。

**可替换部分：**

- `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine` 背后的 `ContextEngine` 实现
- CLI 与 gateway 这类 transport ingress wrapper
- plugin-discovered 的 skill / tool / channel 广度，只要 registry contract 保持稳定
- `runCliAgent` 背后的 CLI backend provider execution

**危险接缝：**

- 围绕 `createAgentSession` 与 `SessionManager` 的 Pi 库边界
- `plugins/runtime.ts` 中进程全局 plugin registry 的兼容性
- transcript-file truth 与 session-store truth 之间的张力
- skill env 注入与 child-process isolation 之间的冲突
- layered tool policy，因为最终 tool set 是多层隐藏过滤器共同生成的

**真正有价值的抽象：**

- `ContextEngine` 值得保留，因为它确实允许 context assembly 和 compaction 在 transport 之外独立变化
- plugin slot 值得保留，因为它能为 context engine、memory provider 等模块提供互斥选择
- `runAgentAttempt` 的分裂值得保留，因为 CLI-backend 和 embedded backend 确实是两台不同的机器

**偏装饰性的抽象：**

- 默认的 `LegacyContextEngine` 更像兼容壳。它保留了 seam，但本身并没有教会你一种更强的 context architecture。

---

## 14. 设计意图推断

**第一优化目标：** 长寿命 agent session 的运维生存能力。

**证据：** transcript repair、write lock、在同一 session key 下轮转 session id、compaction retry 逻辑、tool-result truncation、auth-profile rotation、plugin-registry pinning，以及 compaction 期间 timeout 的特殊处理，都表明作者宁愿优先保住 session，也不愿优先简化代码路径。

**第二优化目标：** 通过 plugins / skills / channels 提升产品可扩展性；在 CLI、gateway、cron 之间复用统一 runtime；在边界层显式表达 trust / sandbox policy。

**它牺牲了什么：** 局部简单性、狭窄模块边界、以及首因推理的容易程度。执行路径之所以强大，是因为它吸收了大量生产故障经验；而同样的密度也让理解成本很高。

---

## 15. 安全手术图

### 更适合优先修改的部分

- `repos/openclaw/src/context-engine/types.ts` 与 `repos/openclaw/src/context-engine/registry.ts` 中非 legacy 的部分：契约与 slot 解析相对隔离，依赖负担低
- `repos/openclaw/src/agents/skills/env-overrides.ts`：一个相对自包含的安全边界
- `repos/openclaw/src/commands/agent-via-gateway.ts`：传输包装层，不拥有核心循环
- `repos/openclaw/src/agents/runtime-plugins.ts`：位于 registry resolution 之上的薄适配器

### 没有完整理解前不要碰的部分

- `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts`：transcript、prompt、tool、timeout、compaction、hook 等不变量在这里收敛
- `repos/openclaw/src/agents/pi-embedded-runner/run.ts`：retry / failover / auth / compaction 策略在这里收敛
- `repos/openclaw/src/agents/pi-tools.ts`：最终 tool surface 是多层策略共同涌现出来的
- `repos/openclaw/src/plugins/loader.ts` 与 `repos/openclaw/src/plugins/runtime.ts`：进程全局 registry 与缓存假设很容易静默失效

---

## 16. 学习价值图

### 高学习价值目标

#### Embedded turn attempt assembly

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **为什么价值高：** 它教你如何把一个 turn staging 成受控流水线，而不是一个巨大的 agent 调用
- **能学到：** 核心循环、状态模型、tool assembly、context assembly、timeout handling、transcript hygiene

#### Embedded retry / failover shell

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **为什么价值高：** 它展示了如何在 agent loop 外面包一层运维恢复逻辑，而不是把这些规则埋进 tool 或 prompt 内部
- **能学到：** runtime retry 设计、auth / model rotation、compaction recovery、outer-loop control

#### Session / model / skills 准备阶段

- **位置：** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution` 和 `agentCommandInternal`
- **为什么价值高：** 它展示了如何在任何模型调用开始前，先把用户意图转成一个 session-scoped execution bundle
- **能学到：** session truth、model override policy、skill snapshot policy、workspace ownership

#### Tool surface composition

- **位置：** `repos/openclaw/src/agents/pi-tools.ts:createOpenClawCodingTools`
- **为什么价值高：** 它提醒你“光有工具注册表还不够”；真正难的是“被策略塑形后的工具物化”
- **能学到：** policy layering、sandbox-aware tool shaping、channel/tool docking、owner gating

#### Context-engine seam

- **位置：** `repos/openclaw/src/context-engine/types.ts` 和 `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine`
- **为什么价值高：** 它提供的是一个严肃的 context lifecycle contract，而不是模糊的“memory module”
- **能学到：** contract design、slot resolution、transcript ownership boundaries

### 中等学习价值目标

#### Skill prompt 与 skill snapshot 加载

- **位置：** `repos/openclaw/src/agents/skills/workspace.ts:resolveSkillsPromptForRun`
- **为什么是中等：** 它确实教会你一个实用的文件系统技能系统，但具体扫描 / frontmatter 规则又明显带有 OpenClaw 风格
- **能学到：** skill discovery、prompt compaction、source filtering

#### Plugin registry loading

- **位置：** `repos/openclaw/src/plugins/loader.ts:resolveRuntimePluginRegistry`
- **为什么是中等：** 它代表成熟的 runtime extensibility，但产品包袱比第一代 AgentOS 需要的要重
- **能学到：** registry caching、provenance tracking、runtime activation

### 眼下学习价值较低的目标

#### Cron isolated-agent runner

- **位置：** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **为什么当前较低：** 它更多在教“如何把核心 runtime 产品化为定时任务”，而不是教你核心运行时本身
- **能学到：** cron delivery policy、scheduled job 的 session isolation

#### Channel registry

- **位置：** `repos/openclaw/src/channels/registry.ts`
- **为什么当前较低：** 它展示的是 transport discovery，而不是主 agent loop
- **能学到：** channel normalization、plugin-backed transport discovery

---

## 17. 重用复制价值图

### 优先直接复制的部分

#### Context-engine contract 与 slot resolution skeleton

- **位置：** `repos/openclaw/src/context-engine/types.ts`、`repos/openclaw/src/context-engine/init.ts`、`repos/openclaw/src/context-engine/registry.ts:registerContextEngineForOwner`、`resolveContextEngine`
- **复制价值：** 高
- **依赖负担：** 低
- **隐藏假设：** 你接受“默认情况下 transcript ownership 归 runtime，除非某个 engine 显式声明自己拥有 compaction ownership”的前提
- **为什么值得复制：** 它已经非常接近一个清晰的 `ContextEngine` 生命周期：`bootstrap`、`maintain`、`ingest`、`assemble`、`compact`

#### Plugin slot defaults

- **位置：** `repos/openclaw/src/plugins/slots.ts`
- **复制价值：** 高
- **依赖负担：** 低
- **隐藏假设：** 你也希望对 context engine、memory backend 等模块使用互斥 slot 语义
- **为什么值得复制：** 这个 slot 机制很小、很清楚，而且直接适合“一个 active context engine / 一个 active memory backend”的决策

#### Skill env override guard

- **位置：** `repos/openclaw/src/agents/skills/env-overrides.ts`
- **复制价值：** 高
- **依赖负担：** 低到中
- **隐藏假设：** 你的 skill 系统会注入 env var，而且可能 spawn child process
- **为什么值得复制：** 它已经解决了一个 AgentOS 很快就会遇到的真实安全问题

### 理解核心节奏，但不要原样复制

#### Embedded attempt pipeline

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **复制价值：** 中等，但前提是重设计
- **依赖负担：** 很高
- **隐藏假设：** Pi agent libraries、SessionManager transcript format、OpenClaw hook contract、OpenClaw tool 名称、OpenClaw sandbox contract
- **为什么不能原样复制：** 真正可迁移的是它的节奏，而不是文件本身。应该内化的顺序是：解析 sandbox -> 解析 skills / tools / system prompt -> 重建 session -> 清洗历史 -> 组装 context -> 执行 prompt -> 收尾 post-turn state。

#### Outer retry shell

- **位置：** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **复制价值：** 中等，但需要重写
- **依赖负担：** 高
- **隐藏假设：** OpenClaw 的 auth profile store、failover taxonomy、compaction helper、live session model switching
- **为什么不能原样复制：** 你想保留的是控制哲学，而不是把全部生产疤痕一次性搬过去

#### Tool policy pipeline

- **位置：** `repos/openclaw/src/agents/pi-tools.ts:createOpenClawCodingTools`
- **复制价值：** 中等
- **依赖负担：** 高
- **隐藏假设：** OpenClaw tool 名称、plugin tool metadata、channel policy、owner policy、sandbox policy、subagent policy
- **为什么不能原样复制：** 第一代 AgentOS 大概率不需要这么多策略轴线。应该复制“多层策略塑形”的思想，而不是完整矩阵。

#### Session preparation 与 skill snapshot persistence

- **位置：** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution`、`agentCommandInternal`
- **复制价值：** 中等
- **依赖负担：** 中到高
- **隐藏假设：** OpenClaw 的 session-store schema 与 model / auth override 语义
- **为什么不能原样复制：** 字段本身很产品化，但“先完成 staging，再进入运行时”的边界划分非常优秀

### 值得学习，但不必现在应用

#### Plugin loader breadth

- **位置：** `repos/openclaw/src/plugins/loader.ts:resolveRuntimePluginRegistry`
- **复制价值：** 第一阶段低
- **依赖负担：** 很高
- **隐藏假设：** 宽插件市场、进程全局 registry、来源追踪、多条 runtime surface
- **为什么建议后置：** 第一代 AgentOS 更需要窄而可检查的扩展故事

#### Cron isolated-agent execution

- **位置：** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **复制价值：** 现在低，后期中等
- **依赖负担：** 高
- **隐藏假设：** scheduled job store、delivery target、cron 特有 tool policy、session isolation semantics
- **为什么建议后置：** 当你的“数字员工”模式进入第二阶段时它会很重要，但现在会分散核心 shell 的注意力

#### Channel / session routing surfaces

- **位置：** `repos/openclaw/src/channels/registry.ts`、`repos/openclaw/src/channels/session.ts`、gateway agent delivery code
- **复制价值：** 现在低，SaaS / IM 阶段更高
- **依赖负担：** 中到高
- **隐藏假设：** 大量 channel plugin、transport-specific delivery state、last-route semantics
- **为什么建议后置：** 先学这个模式，等核心 agent shell 稳定后再实现

---

## 18. 心理模型包

- OpenClaw 的核心单位不是“一条消息”，而是“一个持久 session 中的一次 turn”。
- `agent-command.ts` 会在任何 loop 开始前，先解析 session / workspace / model / skills。
- `runEmbeddedPiAgent` 是外层运维循环，负责 queue、auth、failover、compaction 与 retry。
- `runEmbeddedAttempt` 是内层装配流水线，负责 sandbox、skills、tools、prompt、history 与 context engine。
- 真正的 live model/tool loop 在 Pi 库里，但 OpenClaw 控制着它外面几乎所有重要边界。
- 工具不是简单注册出来的，而是经过多层策略物化出来的。
- transcript truth、session-store truth 和 prompt-context truth 是三层不同真相，必须保持对齐。
- 最先复制的应是契约与 staging boundary；生产级恢复机械应当先学习，再分阶段吸收。
