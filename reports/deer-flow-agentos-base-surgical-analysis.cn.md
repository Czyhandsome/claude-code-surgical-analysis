[返回目录](../README.cn.md) · **中文版** | [English](deer-flow-agentos-base-surgical-analysis.md)

**目录：** [0](#0-范围) · [1](#1-一句话机器定义) · [2](#2-为什么存在这部分) · [3](#3-顶层控制图) · [4](#4-具体入口点) · [5](#5-正常路径外科演练) · [6](#6-调度图) · [7](#7-状态模型) · [8](#8-数据转换链) · [9](#9-副作用图) · [10](#10-重要分支点) · [11](#11-故障和恢复逻辑) · [12](#12-真正的核心循环) · [13](#13-架构压缩) · [14](#14-设计意图推断) · [15](#15-安全手术图) · [16](#16-学习价值图) · [17](#17-重用复制价值图) · [18](#18-心理模型包)

---

# DeerFlow 作为 AgentOS 基座的适配性 — 代码外科分析报告

**分析对象：** `repos/deer-flow`，重点覆盖 `backend/packages/harness/deerflow`、`backend/app/gateway`、`backend/app/channels`、`backend/langgraph.json`，以及 `backend/packages/harness/deerflow/client.py` 中的嵌入式客户端
**日期：** 2026-04-02
**分析问题：** DeerFlow 能否作为一个先 local-first、再走向 SaaS 的 AgentOS shell 的第一代基础依赖、基础 fork，或代码供体？继承边界应该停在哪里？

---

## 0. 范围

**本报告分析什么：** DeerFlow 从 run 入口进入单个 agent turn 的执行脊柱，重点围绕 `repos/deer-flow/backend/langgraph.json`、`repos/deer-flow/backend/app/gateway/routers/thread_runs.py:stream_run`、`repos/deer-flow/backend/app/gateway/services.py:start_run`、`repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent`、`repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`、`repos/deer-flow/backend/packages/harness/deerflow/agents/factory.py:create_deerflow_agent`、`repos/deer-flow/backend/packages/harness/deerflow/client.py:DeerFlowClient`、`repos/deer-flow/backend/packages/harness/deerflow/tools/tools.py:get_available_tools`、`repos/deer-flow/backend/packages/harness/deerflow/skills/loader.py:load_skills`、`repos/deer-flow/backend/packages/harness/deerflow/sandbox/*`，以及 `repos/deer-flow/backend/app/channels/manager.py:ChannelManager` 中的 channel 派发路径。

**本报告回答什么：** DeerFlow 是否适合作为你的内容创作 shell 的第一代 AgentOS 基座，哪些子系统属于可复用基础设施，哪些属于产品意见，以及最小安全采纳路径是什么。

**有意不分析什么：** 前端样式、`frontend/public/demo` 下的示例线程数据、大部分公共技能内容内部实现，以及媒体生成 prompt craft。这些内容会扩大 DeerFlow 的表面积，但不会决定它是否是一个安全的 AgentOS 基座。

**方法：** 对 gateway、harness、client、sandbox、skill、memory 和 channel 路径做静态追踪，并辅以三次轻量验证运行：`uv run pytest tests/test_harness_boundary.py -q`、`uv run pytest tests/test_gateway_services.py -q`、`uv run pytest tests/test_client.py -q -k GatewayConformance`。

---

## 1. 一句话机器定义

> DeerFlow 本质上是一个运行在 LangGraph 之上、以文件系统为底座的 agent 产品壳：它把 threaded chat message 加上上传文件，转化为受配置文件、中间件和 sandbox 约束的流式回复与输出工件；它真正的重心不在自有运行时内核，而在围绕借来的 LangChain/LangGraph loop 所做的 prompt / tool / middleware 组装。

---

## 2. 为什么存在这部分

**它解决的问题：** DeerFlow 想把一个通用的 LLM tool-calling loop 变成可用的终端用户产品：上传文件落在每线程工作区里，sandbox 把这些文件暴露给工具，模型以 prompt 形式看到 skills 和 tools，结果通过 SSE 流出，web 或 IM 客户端都能挂到同一个线程生命周期上。

**为什么逻辑在这里：** `repos/deer-flow/backend/packages/harness/deerflow` 定义模型真正能看到的运行时表面，而 `repos/deer-flow/backend/app/gateway` 与 `repos/deer-flow/backend/app/channels` 则把产品流量路由到这个表面。harness 决定 model、prompt、tools、middleware、sandbox、memory 和 persistence；app 层主要负责序列化、流式输出，并把 harness 适配给 web 和 IM 传输。

**它拥有的边界：** DeerFlow 拥有“一个产品化的 agent turn”：归一化输入、重建线程上下文、组装 prompt 和 tools、调用 LangGraph agent、序列化流式输出、持久化 checkpoint 状态，并暴露 artifact。它不拥有一个全新的 agent kernel。它拥有的是套在 kernel 外面的包装层。

---

## 3. 顶层控制图

```text
[Frontend / IM channel / Python client]
  -> [Gateway route or embedded client]
  -> [start_run / stream / chat]
  -> [RunManager + Worker + StreamBridge]
  -> [make_lead_agent]
       -> [model factory]
       -> [tool loader]
       -> [skill + memory + soul prompt assembly]
       -> [middleware chain]
       -> [LangChain create_agent / LangGraph loop]
            -> [sandbox tools / MCP / subagents]
            -> [checkpoint + store]
  -> [SSE / client events / IM reply]
```

**如何阅读这张图：** 从左到右表示一个 agent turn 的流动。LangChain loop 之前和之后，基本都是 DeerFlow 自己的代码在主导。核心事实是：内部决策 loop 并不在 DeerFlow 代码里。DeerFlow 做的是把 prompt、tools、middleware、state 和 persistence 注入一个框架拥有的 loop。

---

## 4. 具体入口点

### LangGraph 图入口

- **文件 / 函数 / 类：** `repos/deer-flow/backend/langgraph.json` -> `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`
- **输入类型：** 来自 LangGraph Server 的 `RunnableConfig`
- **第一个有意义的下游跳转：** `make_lead_agent` 内部的 `langchain.agents.create_agent(...)`
- **同步 / 异步：** graph factory

### Gateway 流式运行

- **文件 / 函数 / 类：** `repos/deer-flow/backend/app/gateway/routers/thread_runs.py:stream_run`
- **输入类型：** 通过 HTTP 进入的 `RunCreateRequest`
- **第一个有意义的下游跳转：** `repos/deer-flow/backend/app/gateway/services.py:start_run`
- **同步 / 异步：** 异步请求 + SSE 流

### 本地嵌入 API

- **文件 / 函数 / 类：** `repos/deer-flow/backend/packages/harness/deerflow/client.py:DeerFlowClient.stream`
- **输入类型：** Python 字符串消息，可选 `thread_id`
- **第一个有意义的下游跳转：** `repos/deer-flow/backend/packages/harness/deerflow/client.py:_ensure_agent`
- **同步 / 异步：** 包装框架流式接口的同步 generator

### IM channel 派发

- **文件 / 函数 / 类：** `repos/deer-flow/backend/app/channels/manager.py:ChannelManager._dispatch_loop`
- **输入类型：** 入队的 `InboundMessage`
- **第一个有意义的下游跳转：** `repos/deer-flow/backend/app/channels/manager.py:_resolve_run_params`
- **同步 / 异步：** 异步消息消费者

### 线程状态恢复 / 人工修补

- **文件 / 函数 / 类：** `repos/deer-flow/backend/app/gateway/routers/threads.py:update_thread_state`
- **输入类型：** `ThreadStateUpdateRequest`
- **第一个有意义的下游跳转：** `checkpointer.aput(...)`
- **同步 / 异步：** 异步请求-响应

### 缺失的入口

- **文件 / 函数 / 类：** harness 包里没有一等的终端用户 CLI 入口
- **输入类型：** 无
- **第一个有意义的下游跳转：** 无
- **同步 / 异步：** 缺失

DeerFlow 有一些工具脚本和一个 Python 嵌入客户端，但它并没有交付一个像 Claude Code 或你描述的 local-first AgentOS shell 那样的 shell-first 产品运行时。

---

## 5. 正常路径外科演练

**选定场景：** 一个 web 客户端向 `POST /api/threads/{thread_id}/runs/stream` 发送一条用户消息，DeerFlow 执行默认的 lead agent，并向调用方返回流式事件以及持久化后的线程状态。

**为什么选这个场景：** 这是判断 AgentOS 基座是否可用时最有信息量的路径，因为它会跨过你真正会继承的所有接缝：run 生命周期、线程状态、prompt 组装、tool 加载、sandbox 注入、框架 loop 与 streaming。

### Step 1 — 接收 run 请求

- **位置：** `repos/deer-flow/backend/app/gateway/routers/thread_runs.py:stream_run`
- **输入：** `thread_id`、`RunCreateRequest` 和 FastAPI `Request`
- **前置假设：** gateway app 已经在 `repos/deer-flow/backend/app/gateway/deps.py:langgraph_runtime` 中初始化了 `stream_bridge`、`checkpointer`、`store` 和 `run_manager`
- **本步骤做什么：** 把请求派发到 `start_run`，并准备一个 SSE `StreamingResponse`
- **输出：** 一个 `RunRecord` 加上一个存活中的 SSE 响应 generator
- **重要性：** 这是 DeerFlow 面向产品的入口。之后所有语义都继承自这里定义的 run 语义。

### Step 2 — 创建瞬时 run 状态

- **位置：** `repos/deer-flow/backend/app/gateway/services.py:start_run`
- **输入：** 校验后的请求体、`thread_id` 和请求作用域内的单例访问器
- **前置假设：** 线程可能已经存在于 store 或 checkpointer 中，也可能不存在
- **本步骤做什么：** 选择断连行为，调用 `RunManager.create_or_reject`，向 store upsert 一条轻量线程记录，解析 agent factory，归一化输入，并构建 `RunnableConfig`
- **输出：** 一个 pending `RunRecord`、归一化后的 `graph_input` 和 DeerFlow 风格的 `config` dict
- **重要性：** DeerFlow 在这里已经拆分了 run 真相：`RunRecord` 在内存里，线程元数据在 store 里，agent 状态则会放进 checkpointer。

### Step 3 — 调度后台执行

- **位置：** `repos/deer-flow/backend/app/gateway/services.py:start_run`
- **输入：** `RunRecord`、`graph_input`、`config`、`checkpointer`、`store` 和 `make_lead_agent`
- **前置假设：** 请求尚未流出任何 agent 输出
- **本步骤做什么：** 把 `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent` 作为 `asyncio.Task` 调度出去，然后在完成后安排一次尽力而为的标题同步
- **输出：** 一个 `RunRecord`，其 `task` 字段指向 worker task
- **重要性：** gateway 不会内联执行 agent。它把每个 turn 都变成一个拥有独立内存生命周期的后台作业。

### Step 4 — 注入 DeerFlow 运行时上下文

- **位置：** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent`
- **输入：** bridge、run manager、run record、checkpointer、store、agent factory、归一化输入和 config
- **前置假设：** run 仍处于 pending，且还没有 stream item 被发布
- **本步骤做什么：** 将 run 标记为 running，记录运行前 checkpoint id，向 stream bridge 发布 run 元数据，创建带 `thread_id` 的 LangGraph `Runtime`，把它注入 `config["configurable"]["__pregel_runtime"]`，再构建 `RunnableConfig`
- **输出：** 一个与框架兼容的 runtime config，以及第一个 SSE 可见的元数据事件
- **重要性：** 这里是 DeerFlow 手工补齐框架预期的地方，也是单进程假设开始出现的地方。

### Step 5 — 组装 lead agent

- **位置：** `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`
- **输入：** `RunnableConfig`
- **前置假设：** `thread_id` 已经存在于 runtime context 中，且 config metadata 仍可变
- **本步骤做什么：** 解析模型名，若存在 `agent_name` 则解析自定义 agent 配置，计算运行时元数据，调用 `create_chat_model`、`get_available_tools`，通过 `_build_middlewares` 构建中间件，利用 `apply_prompt_template` 渲染巨大的 system prompt，最后调用 `langchain.agents.create_agent`
- **输出：** 一个带有 DeerFlow 状态模型与中间件的 LangChain/LangGraph agent graph
- **重要性：** 这是 DeerFlow 真正的重心。如果你继承 DeerFlow，你继承的就是这种组装哲学。

### Step 6 — 物化线程文件系统状态

- **位置：** `repos/deer-flow/backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py:ThreadDataMiddleware.before_agent`
- **输入：** 当前 `ThreadState` 与 runtime context
- **前置假设：** `thread_id` 已经存在于 runtime context 或 configurable metadata 中
- **本步骤做什么：** 解析 `.deer-flow/threads/{thread_id}/user-data/...` 下的 workspace、uploads 和 outputs 路径，并把它们注入到 `state["thread_data"]`
- **输出：** 写入框架 state 的 `thread_data`
- **重要性：** DeerFlow 把每个对话线程都变成了一个文件系统命名空间。这是少数能直接映射到你 AgentOS 方向上的设计之一。

### Step 7 — 用上传文件记账信息重写最后一条用户消息

- **位置：** `repos/deer-flow/backend/packages/harness/deerflow/agents/middlewares/uploads_middleware.py:UploadsMiddleware.before_agent`
- **输入：** 最后一条 `HumanMessage` 和解析出的 uploads 目录
- **前置假设：** `thread_data` 已经存在
- **本步骤做什么：** 扫描当前消息附带的文件、扫描磁盘上的历史上传文件，把它们序列化成一个 `<uploaded_files>` 块，并把该块前置到 human message 内容前面
- **输出：** 被改写的 human message 内容，以及 `uploaded_files` 状态
- **重要性：** DeerFlow 通过 prompt 注入而不是 typed context channel 来解决文件感知问题。这很快，但也很脆弱。

### Step 8 — 绑定 tools 与 prompt

- **位置：** `repos/deer-flow/backend/packages/harness/deerflow/tools/tools.py:get_available_tools` 与 `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/prompt.py:apply_prompt_template`
- **输入：** 当前 app config、model 名称、feature flags、启用中的 skills、MCP cache、ACP config、memory 文件和可选 agent soul
- **前置假设：** harness 可以从磁盘读取 config 文件、extensions config 和 skills 目录
- **本步骤做什么：** 通过反射解析配置化工具，追加 `present_file`、`ask_clarification` 等内建工具，并视情况追加 `task`、`view_image`、MCP tools、ACP tools、deferred tool search、skill 列表、memory 注入和 subagent prompt 段落
- **输出：** 一组 tool 列表和一个巨大的 system prompt 字符串
- **重要性：** 这已经不只是组合了，而是 DeerFlow 把自己的世界观编码进 prompt 与 registry 形状里。

### Step 9 — 把内部 loop 交给框架

- **位置：** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent` -> `agent.astream(...)`
- **输入：** `graph_input`、runnable config、stream modes、checkpointer 和 store
- **前置假设：** DeerFlow 特有的 prompt、state schema 与 middleware 都已经绑定完毕
- **本步骤做什么：** 把控制权交给框架拥有的 LangChain/LangGraph loop，由它去 prompt 模型、选择 tool call、通过 middleware wrapper 执行工具、持久化 checkpoint，并在最终回答或 interrupt 时停下
- **输出：** 以 LangGraph `values`、`messages`、`updates` 或其他 stream mode 形式流出的 chunk
- **重要性：** DeerFlow 不拥有关键算法 loop。它是在装饰它。

### Step 10 — 序列化并入队流式事件

- **位置：** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent` 与 `repos/deer-flow/backend/packages/harness/deerflow/runtime/serialization.py:serialize`
- **输入：** LangGraph chunks
- **前置假设：** DeerFlow 请求的 stream modes 是 `agent.astream` 能够产出的
- **本步骤做什么：** 将 stream mode 映射为 SSE 事件名，剥离 `__interrupt__` 等内部 LangGraph key，把 messages 和 state 序列化成 JSON-safe 结构，再发布到 `MemoryStreamBridge`
- **输出：** 入队的 `StreamEvent` 对象
- **重要性：** 这是让 DeerFlow 看起来像产品而不是裸 graph 库的关键接缝。

### Step 11 — 交付 SSE 并持久化 run 后元数据

- **位置：** `repos/deer-flow/backend/app/gateway/services.py:sse_consumer` 与 `repos/deer-flow/backend/app/gateway/services.py:_sync_thread_title_after_run`
- **输入：** bridge 条目、请求断连状态、已完成的后台 task
- **前置假设：** worker 已经发布了元数据、stream events 和结束哨兵
- **本步骤做什么：** 格式化 SSE frame；如果客户端以 `on_disconnect=cancel` 方式断开，则取消该 run；然后读取最终 checkpoint，把 `title` 回写到 store
- **输出：** HTTP 流输出以及更新后的 store 元数据
- **重要性：** DeerFlow 用三种不同的持久化视图结束一次 turn：checkpoint state、store metadata，以及已经发送出去的 SSE output。

---

## 6. 调度图

### Assistant 路由

- **选择对象：** `assistant_id`
- **如何选择：** `repos/deer-flow/backend/app/gateway/services.py:resolve_agent_factory` 永远返回 `make_lead_agent`；自定义 assistant 只是注入 `configurable["agent_name"]`
- **解析器位置：** `repos/deer-flow/backend/app/gateway/services.py:resolve_agent_factory` 和 `repos/deer-flow/backend/app/gateway/services.py:build_run_config`
- **引入的风险：** DeerFlow 宣传 assistant，但所有 assistant 实际都走同一个 factory。变化的是 prompt/config，而不是 runtime。

### 模型 provider 调度

- **选择对象：** 来自 config 或请求上下文的模型名
- **如何选择：** `AppConfig.get_model_config(name)`，再加上 `resolve_class(model_config.use, BaseChatModel)` 的反射类加载
- **解析器位置：** `repos/deer-flow/backend/packages/harness/deerflow/models/factory.py:create_chat_model`
- **引入的风险：** 每个模型 provider 细节都会透过 config 与 reflection 渗出。这样虽保留灵活性，但把正确性绑定到了 YAML 上。

### Tool surface 调度

- **选择对象：** tool group、模型能力、MCP 启用状态、ACP config 和 `subagent_enabled`
- **如何选择：** 基于 config 的过滤，再通过 `resolve_variable(tool.use, BaseTool)` 反射解析，并按条件追加内建工具
- **解析器位置：** `repos/deer-flow/backend/packages/harness/deerflow/tools/tools.py:get_available_tools`
- **引入的风险：** tool surface 高度动态，而且部分真实来源藏在 config 文件和 extension state 里。排查“为什么模型会看到这个工具”会很贵。

### Skill 调度

- **选择对象：** 磁盘上启用的 skills，以及模型自己判断要读哪个 skill
- **如何选择：** `load_skills(enabled_only=True)` 把一组 SKILL.md 路径注入 system prompt；LLM 再通过 `read_file` 自行选择
- **解析器位置：** `repos/deer-flow/backend/packages/harness/deerflow/skills/loader.py:load_skills` 与 `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/prompt.py:get_skills_prompt_section`
- **引入的风险：** 这不是运行时 registry，而是 prompt 驱动的自我调度。你会继承 recall error、prompt 膨胀和非确定性 skill 选择。

### Middleware 顺序调度

- **选择对象：** feature flag、显式 middleware 实例，以及可选的 `@Next` / `@Prev` anchor
- **如何选择：** 在 `_build_middlewares` 或 `_assemble_from_features` 中顺序组装，再做 anchor 插入
- **解析器位置：** `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:_build_middlewares` 与 `repos/deer-flow/backend/packages/harness/deerflow/agents/factory.py:_insert_extra`
- **引入的风险：** 顺序是承重结构。轻微重排都会静默改变运行时行为，因为 thread path、uploads、sandbox、tool patch、memory 和 clarification 都依赖相对位置。

### Sandbox backend 调度

- **选择对象：** `config.sandbox.use`
- **如何选择：** 通过反射构造 provider
- **解析器位置：** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/sandbox_provider.py:get_sandbox_provider`
- **引入的风险：** 接口看起来稳定，但 `LocalSandboxProvider` 与 `AioSandboxProvider` 的语义差异非常大。

### 持久化 backend 调度

- **选择对象：** `checkpointer.type`
- **如何选择：** 面向 memory、sqlite 或 postgres 的分支 factory
- **解析器位置：** `repos/deer-flow/backend/packages/harness/deerflow/agents/checkpointer/provider.py:get_checkpointer` 与 `repos/deer-flow/backend/packages/harness/deerflow/runtime/store/provider.py:get_store`
- **引入的风险：** DeerFlow 虽然镜像了 store 与 checkpointer 的 backend 选择，但逻辑上的状态分裂并没有消失。换存储并不能消除这种分裂。

### Channel session 调度

- **选择对象：** 用户级、channel 级和默认 session override
- **如何选择：** 分层 dict merge 到 `run_config` 与 `run_context`
- **解析器位置：** `repos/deer-flow/backend/app/channels/manager.py:_resolve_run_params`
- **引入的风险：** IM 投递又多加了一个配置平面，可以独立于 gateway 调用方去覆盖模型与 agent 行为。

---

## 7. 状态模型

### `ThreadState`

- **类型：** `repos/deer-flow/backend/packages/harness/deerflow/agents/thread_state.py:ThreadState`
- **权威还是派生：** 对 DeerFlow 而言，它是框架可见的每线程 agent state 的权威表示
- **持久还是瞬时：** 如果存在 checkpointer，则通过 LangGraph checkpoint 持久化；否则是瞬时状态
- **创建位置：** `make_lead_agent(... state_schema=ThreadState)` 与 `DeerFlowClient._ensure_agent(...)`
- **修改位置：** `ThreadDataMiddleware.before_agent`、`UploadsMiddleware.before_agent`、`TitleMiddleware`、`MemoryMiddleware`、`ViewImageMiddleware` 等中间件，以及 `present_file_tool` 之类的工具
- **读取位置：** LangGraph agent 执行、gateway thread state endpoint、序列化辅助逻辑
- **关键不变量：** `messages` 仍是标准会话列表；在文件系统感知工具运行前 `thread_data` 必须存在；`artifacts` 是 append-merge 状态，不是不可变真相

### `RunRecord`

- **类型：** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/manager.py:RunRecord`
- **权威还是派生：** 只对实时 HTTP/API run 生命周期而言是权威状态
- **持久还是瞬时：** 瞬时、进程本地、仅内存
- **创建位置：** `RunManager.create` 或 `RunManager.create_or_reject`
- **修改位置：** `RunManager.set_status`、`RunManager.cancel`，以及 `start_run` 附加 `task` 时
- **读取位置：** gateway run endpoint 与 `sse_consumer`
- **关键不变量：** 一个 run id 对应一个 thread id；状态迁移在同一个 asyncio lock 下发生；清理后状态即消失

### 线程 store 记录

- **类型：** 命名空间 `("threads",)` 下的宽松 dict
- **权威还是派生：** 是线程元数据与 title 的派生索引，不是完整 agent 真相
- **持久还是瞬时：** 当 store backend 为 sqlite/postgres 时可持久；内存后端时则为瞬时
- **创建位置：** `threads.create_thread` 或 `services._upsert_thread_in_store`
- **修改位置：** `_store_upsert`、`_sync_thread_title_after_run`、`threads.patch_thread`
- **读取位置：** `threads.search_threads`、`threads.get_thread`
- **关键不变量：** 它可能落后于 checkpointer；它是搜索索引，而不是完整状态机

### `AppConfig` 与 extension config 单例

- **类型：** `AppConfig`、`ExtensionsConfig`，以及若干模块级 config cache
- **权威还是派生：** 权威运行时配置
- **持久还是瞬时：** 磁盘持久，进程内缓存
- **创建位置：** `get_app_config()`、`ExtensionsConfig.from_file()` 及相关单例 getter
- **修改位置：** config reload、gateway skill/MCP update endpoint、文件写入
- **读取位置：** model factory、tool loader、sandbox provider、prompt builder、memory config、channel service
- **关键不变量：** harness 假设文件存在且保持一致；很多运行时决策会反复从磁盘读取 config，以跨越进程边界

### Memory 数据与更新队列

- **类型：** JSON memory payload 加 `ConversationContext` 队列条目
- **权威还是派生：** 派生出来的长期摘要状态，不是原始会话真相
- **持久还是瞬时：** memory payload 持久化到 JSON；队列停留在进程内
- **创建位置：** `get_memory_storage().load(...)` 与 `MemoryUpdateQueue.add`
- **修改位置：** `MemoryMiddleware.after_agent`、`MemoryUpdater.update_memory`、手工 memory CRUD helper
- **读取位置：** `_get_memory_context` 中的 prompt 注入
- **关键不变量：** memory 可能滞后于真实会话 turn；队列去抖逻辑会按设计丢弃同一线程的中间上下文

### Sandbox provider cache

- **类型：** provider 单例以及 `_thread_sandboxes`、`_warm_pool`、`_singleton` 等内部 map
- **权威还是派生：** 对单进程内的 sandbox 复用来说是权威
- **持久还是瞬时：** 瞬时进程内存
- **创建位置：** `get_sandbox_provider`
- **修改位置：** `acquire`、`release`、空闲清理、shutdown
- **读取位置：** sandbox middleware 与 sandbox tools
- **关键不变量：** `LocalSandboxProvider` 本质上是所有人共享的一层 host shell；`AioSandboxProvider` 则假设了线程级挂载与容器生命周期所有权

---

## 8. 数据转换链

**起始形态：** HTTP 请求体或 client 调用，例如 `{"input": {"messages": [{"role": "user", "content": "..."}]}}`

| 步骤 | 转换前形态 | 转换后形态 | 增加 / 丢失 / 压缩了什么意义 |
|------|-------------|-------------|-----------------------------------|
| `normalize_input` | LangGraph 风格的 role/content dict | 带 `HumanMessage` 对象的 LangChain state dict | DeerFlow 会把非 human role 折叠成 `HumanMessage` 占位；消息类型精度在早期就丢失了 |
| `build_run_config` | request config + context + metadata | 与 `RunnableConfig` 兼容的 dict | thread id、assistant 路由和 DeerFlow context flag 被合并进框架 config |
| `_build_middlewares` | config flag | 有序 middleware 列表 | 运行时策略变成了位置行为 |
| `get_available_tools` | config 中的 tool 规格 + runtime flag | 已绑定的 `BaseTool` 列表 | 反射式 config 被转成可执行 tool surface |
| `apply_prompt_template` | memory、skills、soul、feature flag | 一个巨大的 system prompt 字符串 | 结构化运行时概念被压平成 prompt 文本 |
| `UploadsMiddleware.before_agent` | `HumanMessage("user text")` | `HumanMessage("<uploaded_files>...user text")` | 文件元数据被压进 prompt 文本，而不是 typed context |
| `agent.astream` | framework state + config | LangGraph stream chunks | DeerFlow 在此把控制权交给框架 loop |
| `serialize(..., mode="values")` | 带内部 key 的原始 state dict | JSON-safe state dict | 内部 LangGraph 控制 key 被剥离 |
| `format_sse` | event name + payload | SSE wire frame | DeerFlow 把框架输出重新打包成产品级流式格式 |

**最终形态：** SSE 事件流，加上被 checkpoint 的 channel value，以及一个携带 `title` 等搜索元数据的 store 记录。

**关键有损转换：** `normalize_input` 会降低消息 role 精度；prompt 组装会把 memory、skills 和文件列表压成字符串；序列化会剥离内部 LangGraph 控制 key；store 同步则把丰富状态压缩成 metadata 和 title。

---

## 9. 副作用图

### 模型 API 调用

- **触发位置：** `repos/deer-flow/backend/packages/harness/deerflow/models/factory.py:create_chat_model`，经由 `run_agent` 启动的框架 loop
- **守卫条件：** 模型名必须能在 config 中解析；provider credential 必须存在
- **可重试性：** 条件性可重试，而且主要委托给 provider/client 行为，而不是 DeerFlow 自己的运行时策略
- **是否幂等：** 否
- **失败后果：** run 切到 `error`；worker 发出 SSE `"error"` 事件

### Checkpoint 持久化

- **触发位置：** LangGraph loop 内部；另外也会在 `threads.create_thread` 和 `threads.update_thread_state` 中显式写入
- **守卫条件：** 存在 checkpointer backend，且它能序列化当前 state
- **可重试性：** 条件性可重试
- **是否幂等：** 不严格幂等；每次更新都会写一个新的 checkpoint id
- **失败后果：** thread history、多 turn state 与 resume 语义都会损坏

### Thread store 写入

- **触发位置：** `services._upsert_thread_in_store`、`threads._store_put`、`_sync_thread_title_after_run`
- **守卫条件：** store backend 存在
- **可重试性：** 是
- **是否幂等：** 对 upsert 来说基本是
- **失败后果：** search/list/title 视图会与 checkpoint 真相漂移，但 run 仍然能完成

### Thread 文件系统写入

- **触发位置：** `ThreadDataMiddleware`、`uploads/manager.py`、sandbox tools、`present_file`
- **守卫条件：** 有效 `thread_id`、允许的路径、sandbox 或宿主文件系统可用
- **可重试性：** 条件性可重试
- **是否幂等：** 任意写入不是幂等；目录创建基本是
- **失败后果：** uploads、artifacts 和基于 workspace 的 tool 执行会失败

### Sandbox 命令执行

- **触发位置：** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/local/local_sandbox.py:execute_command` 或 AIO sandbox backend
- **守卫条件：** sandbox provider 能解析；host bash 可能被禁用；路径校验通过
- **可重试性：** 条件性可重试
- **是否幂等：** 否
- **失败后果：** tool call 返回错误，或副作用部分落到 host/container 状态中

### Memory LLM 更新

- **触发位置：** `repos/deer-flow/backend/packages/harness/deerflow/agents/memory/updater.py:MemoryUpdater.update_memory`
- **守卫条件：** memory config 已启用，且过滤后的会话至少包含一条 human 消息和一条 assistant 消息
- **可重试性：** 理论上可以，但 DeerFlow 当前实现是记录日志后跳过
- **是否幂等：** 否，因为重复摘要可能产出不同的 memory JSON
- **失败后果：** 个性化会静默退化；核心 turn 执行仍然成功

### IM 出站投递

- **触发位置：** `repos/deer-flow/backend/app/channels/*.py` 下的 channel 类
- **守卫条件：** channel credential、文件大小限制、消息格式
- **可重试性：** 条件性可重试
- **是否幂等：** 否
- **失败后果：** 即便 run 本身成功，Feishu/Slack/Telegram 上的用户仍可能收不到响应

---

## 10. 重要分支点

### 默认 agent 与 bootstrap agent

- **条件：** `configurable["is_bootstrap"]`
- **检查位置：** `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`
- **可选路径：** bootstrap 路径只追加 `setup_agent`；默认路径则解析配置中的 tool group 和 agent soul
- **重要性：** DeerFlow 已经把产品初始化流程也塞进同一个 agent factory，这扩大了核心表面积。

### 自定义 assistant 名称与默认 lead agent

- **条件：** `assistant_id != "lead_agent"`
- **检查位置：** `repos/deer-flow/backend/app/gateway/services.py:build_run_config` 与 `repos/deer-flow/backend/app/channels/manager.py:_resolve_run_params`
- **可选路径：** 默认 assistant 直接路由；自定义 assistant 会被规整成 `agent_name`，但仍然使用 lead-agent factory
- **重要性：** DeerFlow 的 custom agent 本质上是 prompt/config overlay，而不是独立运行时。

### Local sandbox 与 AIO sandbox

- **条件：** `config.sandbox.use`
- **检查位置：** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/sandbox_provider.py:get_sandbox_provider`
- **可选路径：** `LocalSandboxProvider` 把虚拟路径映射到 host shell；`AioSandboxProvider` 则去申请容器
- **重要性：** 同一个 “sandbox” 抽象，藏住了极其不同的信任边界。

### 是否启用 host bash

- **条件：** `sandbox.allow_host_bash`
- **检查位置：** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/security.py:is_host_bash_allowed`
- **可选路径：** bash tool 要么暴露，要么被过滤掉
- **重要性：** DeerFlow 的本地体验会因为一个 config bit，从“没有 shell”静默切换成“直接 host shell”。

### 是否启用 subagent 模式

- **条件：** `configurable["subagent_enabled"]`
- **检查位置：** `make_lead_agent`、`_build_middlewares` 和 `get_available_tools`
- **可选路径：** `task` tool 和 `SubagentLimitMiddleware` 要么出现，要么完全不存在
- **重要性：** subagent 不是始终存在的运行时原语，而是可选的 prompt / tool 膨胀项。

### 持久化后端还是内存后备

- **条件：** config 是否声明 sqlite/postgres checkpointer/store
- **检查位置：** `get_checkpointer`、`get_store` 及其异步版本
- **可选路径：** 持久化后端可跨重启保留状态；内存 fallback 重启即丢
- **重要性：** DeerFlow 很容易看起来像“可上生产”，但实际上可能默默运行在易失状态上。

### 多任务策略

- **条件：** `multitask_strategy`
- **检查位置：** `RunManager.create_or_reject`
- **可选路径：** 拒绝、打断或回滚正在运行的 run
- **重要性：** DeerFlow 虽然暴露了并发策略，但 `run_agent` 里的 rollback 仍只是一个 stub。

---

## 11. 故障和恢复逻辑

### 故障表面

| 故障面 | 检测位置 | 传播方式 | 转译结果 |
|---------|---------------|-------------------|---------------|
| 未知或无效模型 | `make_lead_agent` | 异常冒泡进 `run_agent` | run 状态变 `error` + SSE `"error"` |
| Tool 异常 | `ToolErrorHandlingMiddleware.wrap_tool_call` | 被包装 | 转成 `ToolMessage(status="error")`，让模型继续 |
| Guardrail 拒绝 | `GuardrailMiddleware` | 被包装 | `ToolMessage(status="error")` |
| 缺失 config 文件或 YAML 无效 | `get_app_config` / `load_agent_config` / `ExtensionsConfig.from_file` | 依调用点而定，可能冒泡也可能只记日志 | 启动失败、run 失败，或功能静默缺失 |
| Store 写入 / 标题同步失败 | gateway helper | 记录日志并吞掉 | 搜索元数据陈旧 |
| Stream bridge 队列溢出 | `MemoryStreamBridge.publish` | 记录日志并丢弃 | 缺失部分 stream event |
| Memory 更新的 JSON 解析或 LLM 失败 | `MemoryUpdater.update_memory` | 记录日志并吞掉 | memory 陈旧 |
| 客户端断连 | `sse_consumer` | 显式 cancel 路径 | run 被中断 |
| Rollback 请求 | `run_agent` | 只有部分 stub | 状态会变，但 checkpoint rollback 实际不会发生 |

### 恢复行为

- **重试逻辑：** DeerFlow 在运行时层面几乎不做重试。它最主要的恢复手段不是 retry，而是 degrade。Tool error 会变成 `ToolMessage`，store 失败会记录日志后继续，memory 失败也只记日志继续，`run_agent` 里不支持的 stream mode `"events"` 会被跳过。
- **回退逻辑：** 当 config 缺失时，`get_checkpointer` 与 `get_store` 会退回内存实现；`create_chat_model` 可以退回默认模型名；`build_run_config` 为了兼容较新的 LangGraph 行为，会优先用 `context` 而不是 `configurable`。
- **超时行为：** `LocalSandbox.execute_command` 会把子进程执行硬性限制在 600 秒内。Subagent 轮询在 `task_tool` 中会限制为执行超时加缓冲。Stream bridge 默认每 15 秒发一次 heartbeat。
- **生命周期错配风险：** 最大的问题是真相被拆开了。`RunRecord`、checkpoint state 和 store metadata 可能互相不一致。另一个错配点是进程局部性：`RunManager` 与 `MemoryStreamBridge` 都在进程内，而 channels 与其他客户端却可能假设这是一个更耐久的 server。

---

## 12. 真正的核心循环

DeerFlow 并没有实现一个自研的一阶原理 agent loop。它真正的工作单元，是一个被 DeerFlow wrapper 塑形过的框架调用。

```text
request lifecycle:
  1. normalize run request
     -> `repos/deer-flow/backend/app/gateway/services.py:start_run`
  2. create transient run state and schedule worker
     -> `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent`
  3. assemble agent from config, prompt, tools, and middleware
     -> `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`
  4. annotate state with thread paths, uploads, sandbox, title/memory hooks
     -> middleware chain under `repos/deer-flow/backend/packages/harness/deerflow/agents/middlewares/*`
  5. prompt framework-owned model/tool loop
     -> `langchain.agents.create_agent(...).astream(...)`
  6. serialize framework output and enqueue stream events
     -> `repos/deer-flow/backend/packages/harness/deerflow/runtime/serialization.py:serialize`
     -> `repos/deer-flow/backend/packages/harness/deerflow/runtime/stream_bridge/memory.py:publish`
  7. stream to client and sync title/store metadata
     -> `repos/deer-flow/backend/app/gateway/services.py:sse_consumer`
until final answer, interrupt, or error
```

最直白的事实是：DeerFlow 拥有 loop 外面的 harness；LangChain/LangGraph 拥有 loop 本身。

---

## 13. 架构压缩

**真正的重心：** DeerFlow 自有的重力中心在 `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`，因为所有主要路径最终都会汇入这里，而且它决定模型、tool surface、middleware 顺序、state schema 和 system prompt。更深层的运行时重心仍在 `make_lead_agent` 调用的外部 `langchain.agents.create_agent` loop 里。

**可替换部件：** frontend、IM channels、gateway routers、model provider class、checkpointer backend、store backend 和 sandbox provider 都可以被替换，而无需重写整个仓库。`repos/deer-flow/backend/tests/test_harness_boundary.py:test_harness_does_not_import_app` 这个 harness 边界测试，证明团队正在主动把 harness 做成可替换件。

**危险接缝：** `build_run_config` 和 `_resolve_run_params` 通过松散类型的 dict 合并运行时真相；skill 选择依赖 prompt 自我派发；`ThreadState`、`RunRecord` 和 store metadata 各自持有不同片段的真相；`LocalSandboxProvider` 一边宣称自己是 sandbox，一边实际在 host 上执行；`MemoryStreamBridge` 和 `RunManager` 则假设只有一个进程。

**真实抽象：** sandbox provider 接口、checkpointer/store backend factory、stream bridge 抽象，以及半成品的 `create_deerflow_agent` factory 都是真抽象。它们确实支持真实的运行时变体。

**装饰性抽象：** assistant 模型大多是装饰性的，因为所有 assistant 都路由到 `make_lead_agent`；skill registry 也大多是装饰性的，因为最终还是 LLM 自己决定读哪个 markdown；`create_deerflow_agent` 的纯 SDK 叙事尚未完成，因为关键运行时部件仍然会读全局 config 或文件系统。

---

## DeerFlow 作为 AgentOS 基座：可行性、采纳策略与继承边界

**是否可行：** 只适合作为选择性代码供体和临时参考 harness。它不适合作为安全的直接依赖，也不适合作为你 AgentOS 的整仓基础。

**为什么结论不能更乐观：** DeerFlow 在一些外壳问题上确实做得足够好，值得学习：每线程文件系统布局、流式 run 生命周期、部分 harness/app 切分、嵌入客户端形状，以及更接近真实隔离的容器 sandbox 路径。但你真正需要的核心 agent machine，仍然过度受框架支配、过度依赖配置文件，而且带着明显的产品意见。

**最直白的模式判断：**

- **直接依赖：** 不行。DeerFlow 仍把太多运行时真相藏在 config 单例、LangGraph state 和 prompt 组装里。你会把时间花在修补继承行为，而不是构建自己的 OS contract。
- **作为基础 fork：** 只适合一次性 spike。如果你把 DeerFlow fork 成主线基座，你会立即继承 LangChain/LangGraph 税、状态双脑、伪本地 sandbox 语义、prompt 驱动 skill，以及 gateway/channel 的产品假设。
- **选择性代码供体 / 仅作参考：** 可以。这才是正确姿势。

**应该继承什么：**

- `repos/deer-flow/backend/packages/harness/deerflow/config/paths.py`
- `repos/deer-flow/backend/packages/harness/deerflow/uploads/manager.py`
- `repos/deer-flow/backend/packages/harness/deerflow/runtime/stream_bridge/base.py` 中的接口思路
- `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/manager.py` 中轻量 run 生命周期的形状
- `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py` 的部分选定片段
- `repos/deer-flow/backend/packages/harness/deerflow/client.py` 的嵌入客户端外形，但不继承其下方当前的 agent factory

**必须尽早重写什么：**

- 主 agent loop contract
- 权威的 turn/session state model
- 权限与 human approval model
- tool 与 skill registry contract
- “completed / blocked / failed” turn 的结果 schema
- local-first CLI shell

**应该延后什么：**

- `app/gateway/routers/*` 下的 LangGraph 兼容 endpoint
- `app/channels/*` 下的 IM channels
- title generation、follow-up suggestion 和当前 memory summarization
- 除非 tool 数量真的爆炸，否则 deferred tool search 也可以后置

**永远不要继承什么：**

- 当前 `lead_agent` prompt 作为你的 system prompt 基座
- prompt 驱动的 skill 选择作为你的 skills 架构
- `LocalSandboxProvider` 作为安全故事
- “assistant 只是一个图上的 `agent_name` overlay” 这种假设
- “LangGraph checkpoint state 应该就是你的 AgentOS 领域状态” 这种假设

### 主要子系统决策矩阵

| 子系统 | DeerFlow 当前设计 | 为什么存在 | 优点 | 缺点 / 风险 | 与我们的 AgentOS 适配度 | 建议：KEEP / FORK-AND-MODIFY / REWRITE / IGNORE | phase-1 用法 | 长期用法 |
|---|---|---|---|---|---|---|---|---|
| Lead agent 组装 | `make_lead_agent` 围绕 `create_agent` 组装 model、prompt、tools 和 middleware | 给所有表面提供一个默认 agent factory | 组合快；入口明确 | 巨型隐式 I/O factory；借来的 inner loop；prompt 税重 | 低 | REWRITE | 只做学习对象 | 无 |
| 偏“纯”的 agent factory | `create_deerflow_agent` 加 `RuntimeFeatures` | 暴露一个更像 SDK 的 harness 路径 | 边界比 `make_lead_agent` 更好；插入模型可测试 | 仍非 config-free；文档也承认还有隐式 I/O | 中 | FORK-AND-MODIFY | 复制 API 外形 | 只保留改写后的版本 |
| Run 生命周期 | `RunManager` + `run_agent` + `StreamBridge` | 把 graph 执行变成产品 run | 薄而好懂；对 streaming 友好 | 单进程；rollback 只是 stub；run 真相与 checkpoint 真相分裂 | 中 | FORK-AND-MODIFY | 本地原型可接受 | 替换底层持久化与调度器 |
| Checkpointer + store backend factory | 配置驱动的 memory/sqlite/postgres 选择 | 持久化 thread state 与搜索元数据 | 很实用的 backend 切换接缝 | 绑死在 LangGraph state model 上 | 中低 | FORK-AND-MODIFY | 只有在继续使用 LangGraph 时才保留 | 若 AgentOS 自有状态引擎则重写 |
| Thread 文件系统 + uploads + outputs | `.deer-flow/threads/{thread_id}/user-data/...` 加上传工具 | 给每个线程一个具体工作区 | 直接可用；契合产出 artifact 的 shell | 不同模块对 thread id 规则不完全一致；仍偏文件中心 | 高 | KEEP | 几乎可直接复制 | 保留并做少量清理 |
| Sandbox 抽象 | 基于本地或容器 sandbox 的 provider 接口 | 让工具执行脱离 Python 进程内存 | 是真抽象；后端可替换 | 不同 provider 的语义差异过大 | 中 | FORK-AND-MODIFY | 只保留接口 | 保留改造后的版本 |
| Local sandbox | 带路径映射的单例 host shell | 零摩擦本地开发 | 易运行；易调试 | 不是真 sandbox；共享 host shell；权限叙事不成立 | 很低 | REWRITE | 不要把它当安全边界继承 | 永不 |
| AIO sandbox | 带线程挂载与 warm pool 的容器/K8s provider | 从 host shell 走向隔离执行 | 这里最接近真实运行时资产 | Docker/K8s 成本重；挂载假设强；运维负担大 | 中 | FORK-AND-MODIFY | 除非 sandbox 很急，否则先后置 | 也许可有选择保留 |
| Tool loader + deferred tool search | 反射式 config tools 加 MCP 延迟公开 | 保持 tool surface 灵活并减小上下文 | 灵活；deferred schema 的点子很聪明 | config 驱动不透明；MCP 很重；排障成本高 | 中 | FORK-AND-MODIFY | 只复制 deferred schema 思路 | 只有 tool catalog 很大时再保留 |
| Skill 系统 | 扫描 SKILL.md，让模型通过 prompt 去读 skill | 在不写 codegen 的情况下提供廉价可扩展性 | 编写门槛低 | 不是实际 registry；匹配不可靠；prompt 膨胀 | 低 | REWRITE | 最多把它当人工 playbook | 用显式 skill contract 替换 |
| Memory 系统 | 把会话摘要成 JSON 再注入 prompt | 提供轻量个性化 | 简单；不阻塞核心 run path | 不是 ContextEngine；可信度低；异步滞后；文件绑定 | 低 | REWRITE | phase 1 直接忽略 | 自建 context engine |
| Subagent 系统 | `task` tool + 后台线程池 + 轮询 | 并行化较大工作 | 至少证明了一种可行 UX 形状 | 调度器粗糙；共享假设强；prompt/tool 耦合高 | 中低 | FORK-AND-MODIFY | 以后也许可复制事件 UX | 若 subagent 重要则重写 |
| DeerFlowClient | 绕过 gateway/langgraph server 的嵌入式 Python client | 让本地嵌入成为可能 | 对 local mode 很有价值；有 gateway-conformance 测试 | 是 Python SDK，不是 CLI 产品；仍受 config 绑定 | 中高 | FORK-AND-MODIFY | 很适合作为内部 SDK facade 供体 | 只保留重写后的 facade |
| Gateway LangGraph 兼容层 | 模仿 LangGraph Platform 的 FastAPI route | 让 frontend 和 SDK 说同一协议 | 很实用的产品适配层 | 带来 split-brain 架构和更多状态重复 | 低 | IGNORE | 无 | 只有在 SDK 兼容性重要时再考虑 |
| IM channels | 经 `langgraph_sdk` 接到 Feishu/Slack/Telegram | 把同一个 agent 推进聊天应用 | 是有价值的产品参考 | 传输特化逻辑和 session override 会污染 runtime 视角 | 低 | IGNORE | 无 | 核心运行时稳定后再看 |
| Frontend 与 demo artifact | 网关之上的 Next.js UI 加 demo threads | 展示完整产品 | 演示价值好 | 对 runtime-base 选择几乎无关 | 很低 | IGNORE | 无 | 无 |

### 这个仓库里最大的架构谎言 / 幻觉

- **“DeerFlow 有自己的 agent runtime。”** 不。DeerFlow 有的是套在 LangChain/LangGraph runtime 外面的 harness。这个区别很重要，因为你一旦和核心行为打架，最后打的还是框架。
- **“`LocalSandboxProvider` 是 sandbox。”** 不是。它只是一个带路径改写的 host-shell adapter。
- **“Skills 是 registry。”** 也不是。它们只是模型可能会去读、也可能不会去读的 markdown prompt。
- **“Assistants 是一等运行时变体。”** 也不是。大多数情况下，它们只是 `lead_agent` 加一个 `agent_name`。
- **“Runs、threads 和 state 有一个统一真相源。”** 没有。Run status、checkpoint state 与 store metadata 是按设计分裂的。

### 这个仓库让哪些事看起来比真实更容易

- 让 LangGraph 看起来像一个产品 API
- 让 local mode 与 server mode 共享同一份运行时真相
- 让 skill 扩展性看起来像确定性机制
- 让 sandbox 安全看起来只是换一个 provider
- 让 long-term memory 看起来像已解决子系统，而不是一个有损摘要任务

### 在真实产品压力下最可能先坏掉的地方

- 使用 `MemoryStreamBridge` 的多进程或多 pod streaming
- 如果有人真的信任 `LocalSandboxProvider`，那么权限与安全预期会先崩
- 当更多 skills、tools 和内容创作指令堆进来后，prompt 质量会先崩
- 当 run 跨 transport 被打断、回滚或重连时，状态一致性会先出问题
- 任何试图让当前 memory 系统承载严肃个性化负荷的尝试

---

## 14. 设计意图推断

**主要优化目标：** 先在 LangGraph 之上尽快交付一个广而可演示的通用 agent 产品，再在之后逐步雕出可复用 harness。

**推断依据：** 仓库同时交付了 frontend、gateway、channels、公共 skills、重 prompt 的行为塑形、LangGraph 兼容 route 和嵌入客户端。`repos/deer-flow/backend/docs/rfc-create-deerflow-agent.md` 里的 RFC 还明确承认当前 harness 里存在“`6 处隐式 I/O`”，并把真正 config-free 的 runtime 视为后续目标。

**次级优化目标：** 功能广度、传输覆盖面和用户可见精致度。DeerFlow 在构建自有 runtime contract 之前，已经先投入了 title generation、suggestion、skills、channel adapter 和兼容性 API。

**被牺牲的东西：** 运行时显式性、local-shell-first 易用性、确定性的权限模型、清晰的状态所有权，以及未来摆脱 LangChain/LangGraph 假设的自由度。

---

## 15. 安全手术图

### 更适合先改的地方

- `repos/deer-flow/backend/packages/harness/deerflow/config/paths.py`，因为它已经表达了一个干净的文件系统 contract，且与框架耦合有限
- `repos/deer-flow/backend/packages/harness/deerflow/uploads/manager.py`，因为它是纯业务逻辑，不依赖 FastAPI 或 LangGraph
- `repos/deer-flow/backend/packages/harness/deerflow/runtime/stream_bridge/base.py`，因为抽象很小，而且痛点非常清楚
- `repos/deer-flow/backend/packages/harness/deerflow/client.py`，因为它是通往本地嵌入式叙事最干净的桥，尽管其内部仍需更换

### 高风险、不要过早改的地方

- `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/prompt.py`，因为这个 prompt 已经背负了太多策略
- `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py`，因为 middleware 顺序和 tool surface 都是承重结构
- 如果你还依赖 LangGraph 兼容性，那么 `repos/deer-flow/backend/app/gateway/services.py` 加 `runtime/runs/worker.py` 都属于高风险改动点
- `repos/deer-flow/backend/app/channels/*`，因为这些路径会立刻再引入一层 state/config plane

### 最小可行采纳路径

1. 复制 `config/paths.py` 和 `uploads/manager.py` 中的 thread 文件系统 contract。
2. 复制 `RunManager`、`StreamBridge` 和 `DeerFlowClient` 的形状，而不是复制它们的实现。
3. 先写出你自己显式的 `TurnState`、`RunResult` 和权限 contract。
4. 如果你很快就需要容器 sandbox，只 fork `AioSandboxProvider` 的概念，不要 fork 本地 provider。
5. 如果你要继续用 LangGraph，也必须把它藏在你自己的 runtime facade 后面。
6. 在 phase 1 里，把 DeerFlow skills 只当 prompt playbook，不要把它们当长期 skill registry。

### 开放问题

- 当你加入显式 turn-result contract、human approval 和更强的 ContextEngine 之后，你还需要 LangGraph 吗？
- 你希望 generation gateway 在 agent runtime 之外，还是让它也变成同一 contract 下的一个 tool provider？
- 你希望 local CLI 与 cloud SaaS 共享的是同一个 state model，还是只共享 tool/skill contract？

### 下一步推荐调查

- 把 DeerFlow 的容器 sandbox 路径拿去和 OpenClaw 的 permission / session model 对比，而不是只跟 DeerFlow 的 local mode 对比
- 在复制任何 LangGraph 形状的 state 之前，先决定你的 AgentOS 到底需要 framework-owned loop，还是 self-owned loop
- 在采用任何现有持久化层之前，先原型化你自己的显式 `completed / blocked / failed` run result schema

---

## 16. 学习价值图

### 最值得学习的部分

1. `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent`
   它能教你的：如何在不 fork 框架 server 的前提下，把一个 framework graph 包成产品级 run lifecycle、stream serialization 和 abort 语义。

2. `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py:AioSandboxProvider`
   它能教你的：如何把线程级文件系统挂载、warm-pool 复用和容器生命周期接进 agent runtime。

3. `repos/deer-flow/backend/packages/harness/deerflow/agents/factory.py:create_deerflow_agent`
   它能教你的：一个团队在先交付 config-heavy 产品运行时之后，如何试图把系统重新拉回可复用 SDK 层。

4. `repos/deer-flow/backend/packages/harness/deerflow/tools/builtins/tool_search.py:DeferredToolRegistry`
   它能教你的：一种很干净的办法，能够在不把工具彻底从平台移除的情况下，降低 tool-schema 上下文压力。

5. `repos/deer-flow/backend/tests/test_harness_boundary.py:test_harness_does_not_import_app`
   它能教你的：当系统处在切分过程中时，边界测试往往比 README 文案更能说明真实架构意图。

**尽管表面复杂，但学习价值不高的部分：** 当前的 lead-agent prompt、公共 skills catalog，以及大多数 gateway router。它们教的是产品包装，而不是内核设计。

---

## 17. 重用/复制价值图

### 最值得复制的部分

| 候选项 | 依赖重量 | 隐藏假设 | 复制价值 |
|---|---|---|---|
| `repos/deer-flow/backend/packages/harness/deerflow/config/paths.py` | 低 | 假设是 thread-oriented 文件系统布局，且 home 在 `.deer-flow` | 高；接近可直接复制 |
| `repos/deer-flow/backend/packages/harness/deerflow/uploads/manager.py` | 低 | 假设 uploads 生活在 thread 文件系统 contract 下 | 高；接近可直接复制 |
| `repos/deer-flow/backend/packages/harness/deerflow/runtime/stream_bridge/base.py` | 低 | 除 event-stream 语义外几乎无 | 高；复制接口即可 |
| `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/manager.py` | 中低 | 假设是进程内调度器且只在一个 host 上运行 | 中；适合改造成 durable run |
| `repos/deer-flow/backend/packages/harness/deerflow/client.py` | 中 | 假设 LangGraph 形状的 agent state 和 DeerFlow config 文件存在 | 中；复制 facade pattern，不要复制 guts |
| `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py` | 高 | 假设 Docker/K8s、挂载所有权和 thread-id 文件系统 contract | 中；只适合作为选择性供体 |
| `repos/deer-flow/backend/packages/harness/deerflow/models/factory.py` 及 provider patch | 中 | 假设配置驱动的反射式 provider 加载 | 中；如果你继续用 Python 且保持类似模型广度，会有用 |
| `repos/deer-flow/backend/packages/harness/deerflow/tools/builtins/tool_search.py` | 中 | 假设 deferred tool 在 prompt 侧揭示 schema 后仍可调用 | 中；是不错的可选供体 |

### 更适合晚点复制，或根本不要复制的部分

- **现在不要复制：** `lead_agent` prompt、skill loading 与 skill prompt injection、memory updater、channel stack、gateway LangGraph 兼容层。
- **原因：** 这些部分在 DeerFlow 内部变化很快，因为它们编码的是产品行为，而不是可复用的 runtime 法则。

---

## 18. 心理模型包

- DeerFlow 是一个围绕借来的 LangGraph loop 搭起来的产品壳。
- `make_lead_agent` 是 DeerFlow 自有的重力中心。
- 最值得复用的资产是文件系统布局、uploads、stream/run wrapper，以及部分 sandbox 思路。
- 最大的继承风险是 prompt 驱动 skills、伪本地 sandbox 和分裂的状态真相。
- Custom assistant 大多数时候只是同一个 factory 上的不同 prompt/config overlay。
- harness/app 切分体现了真实意图，但 harness 还没有真正做到 config-free。
- DeerFlow 更适合做代码供体，而不是依赖项或整仓基础 fork。
- 先建立你自己的 state、permission 和 skill contract，再去吸收 DeerFlow 的想法。
