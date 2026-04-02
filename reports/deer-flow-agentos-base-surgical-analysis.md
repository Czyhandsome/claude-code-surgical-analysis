[Back to Index](../README.md) · [中文版](deer-flow-agentos-base-surgical-analysis.cn.md) | **English**

**Table of Contents:** [0](#0-scope) · [1](#1-one-sentence-machine-definition) · [2](#2-why-this-part-exists) · [3](#3-top-level-control-picture) · [4](#4-concrete-entrypoints) · [5](#5-happy-path-surgical-walkthrough) · [6](#6-dispatch-map) · [7](#7-state-model) · [8](#8-data-transformation-chain) · [9](#9-side-effect-map) · [10](#10-important-branch-points) · [11](#11-failure-and-recovery-logic) · [12](#12-the-real-core-loop) · [13](#13-architectural-compression) · [14](#14-design-intent-inference) · [15](#15-safe-surgery-map) · [16](#16-learning-value-map) · [17](#17-reusecopy-value-map) · [18](#18-mental-model-packet)

---

# DeerFlow as an AgentOS Base — Code Surgical Analysis Report

**Analyzed:** `repos/deer-flow` with emphasis on `backend/packages/harness/deerflow`, `backend/app/gateway`, `backend/app/channels`, `backend/langgraph.json`, and the embedded client in `backend/packages/harness/deerflow/client.py`
**Date:** 2026-04-02
**Analyst question:** Can DeerFlow serve as the first base dependency, base fork, or code donor for a local-first then SaaS AgentOS shell, and where must inheritance stop?

---

## 0. Scope

**What is analyzed:** the DeerFlow execution spine from run ingress into one agent turn, centered on `repos/deer-flow/backend/langgraph.json`, `repos/deer-flow/backend/app/gateway/routers/thread_runs.py:stream_run`, `repos/deer-flow/backend/app/gateway/services.py:start_run`, `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent`, `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`, `repos/deer-flow/backend/packages/harness/deerflow/agents/factory.py:create_deerflow_agent`, `repos/deer-flow/backend/packages/harness/deerflow/client.py:DeerFlowClient`, `repos/deer-flow/backend/packages/harness/deerflow/tools/tools.py:get_available_tools`, `repos/deer-flow/backend/packages/harness/deerflow/skills/loader.py:load_skills`, `repos/deer-flow/backend/packages/harness/deerflow/sandbox/*`, and the channel dispatch path in `repos/deer-flow/backend/app/channels/manager.py:ChannelManager`.

**Question this report answers:** whether DeerFlow is a sound first AgentOS base for your content-creation shell, which subsystems are reusable infrastructure versus product opinion, and what the smallest safe adoption path is.

**Intentionally excluded:** frontend styling, demo thread data under `frontend/public/demo`, most public skill content internals, and media-generation prompt craft. Those parts enlarge DeerFlow's surface area, but they do not decide whether DeerFlow is a safe AgentOS base.

**Method:** static trace across gateway, harness, client, sandbox, skill, memory, and channel paths, plus three lightweight verification runs: `uv run pytest tests/test_harness_boundary.py -q`, `uv run pytest tests/test_gateway_services.py -q`, and `uv run pytest tests/test_client.py -q -k GatewayConformance`.

---

## 1. One-Sentence Machine Definition

> DeerFlow is fundamentally a LangGraph-hosted, filesystem-backed agent product shell that turns threaded chat messages plus uploaded files into streamed replies and output artifacts under config-file, middleware, and sandbox constraints, and its true center of gravity lies in prompt/tool/middleware assembly around a borrowed LangChain/LangGraph loop rather than in a self-owned runtime kernel.

---

## 2. Why This Part Exists

**Problem it solves:** DeerFlow tries to turn a generic LLM tool-calling loop into a usable end-user product: uploaded files land in a per-thread workspace, a sandbox exposes those files to tools, the model sees skills and tools in prompt form, results stream over SSE, and web or IM clients can attach to the same thread lifecycle.

**Why the logic lives here:** `repos/deer-flow/backend/packages/harness/deerflow` shapes the runtime surface the model sees, while `repos/deer-flow/backend/app/gateway` and `repos/deer-flow/backend/app/channels` route product traffic into that surface. The harness decides model, prompt, tools, middleware, sandbox, memory, and persistence. The app layer mostly serializes, streams, and adapts that harness to web and IM transports.

**What boundary it owns:** DeerFlow owns one productized agent turn: normalize input, reconstruct thread context, assemble prompt and tools, invoke a LangGraph agent, serialize streamed outputs, persist checkpoint state, and expose artifacts. It does not own a novel agent kernel. It owns the wrapper around one.

---

## 3. Top-Level Control Picture

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

**Reading this diagram:** left-to-right follows one agent turn. DeerFlow-owned code dominates everything before and after the LangChain loop. The central fact is that the inner decision loop does not live in DeerFlow code. DeerFlow injects prompt, tools, middleware, state, and persistence into a framework-owned loop.

---

## 4. Concrete Entrypoints

### LangGraph graph entry

- **File / function / class:** `repos/deer-flow/backend/langgraph.json` -> `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`
- **Input type:** `RunnableConfig` from LangGraph Server
- **First meaningful downstream hop:** `langchain.agents.create_agent(...)` inside `make_lead_agent`
- **Sync / async:** graph factory

### Gateway run streaming

- **File / function / class:** `repos/deer-flow/backend/app/gateway/routers/thread_runs.py:stream_run`
- **Input type:** `RunCreateRequest` over HTTP
- **First meaningful downstream hop:** `repos/deer-flow/backend/app/gateway/services.py:start_run`
- **Sync / async:** async request + SSE stream

### Embedded local API

- **File / function / class:** `repos/deer-flow/backend/packages/harness/deerflow/client.py:DeerFlowClient.stream`
- **Input type:** Python string message plus optional `thread_id`
- **First meaningful downstream hop:** `repos/deer-flow/backend/packages/harness/deerflow/client.py:_ensure_agent`
- **Sync / async:** sync generator wrapping framework streaming

### IM channel dispatch

- **File / function / class:** `repos/deer-flow/backend/app/channels/manager.py:ChannelManager._dispatch_loop`
- **Input type:** queued `InboundMessage`
- **First meaningful downstream hop:** `repos/deer-flow/backend/app/channels/manager.py:_resolve_run_params`
- **Sync / async:** async message consumer

### Thread state resume / human patch

- **File / function / class:** `repos/deer-flow/backend/app/gateway/routers/threads.py:update_thread_state`
- **Input type:** `ThreadStateUpdateRequest`
- **First meaningful downstream hop:** `checkpointer.aput(...)`
- **Sync / async:** async request/response

### What is missing

- **File / function / class:** no first-class end-user CLI entrypoint in the harness package
- **Input type:** none
- **First meaningful downstream hop:** none
- **Sync / async:** absent

DeerFlow has utility scripts and a Python embedded client. It does not ship a shell-first product runtime comparable to Claude Code or the local-first AgentOS shell you described.

---

## 5. Happy-Path Surgical Walkthrough

**Chosen scenario:** a web client posts one user message to `POST /api/threads/{thread_id}/runs/stream`, DeerFlow executes the default lead agent, and the caller receives streamed events plus persisted thread state.

**Why this scenario:** it is the most revealing path for an AgentOS base decision because it crosses the exact seams you would inherit: run lifecycle, thread state, prompt assembly, tool loading, sandbox injection, framework loop, and streaming.

### Step 1 — Accept the run request

- **Location:** `repos/deer-flow/backend/app/gateway/routers/thread_runs.py:stream_run`
- **What enters:** `thread_id`, `RunCreateRequest`, and a FastAPI `Request`
- **Assumptions already true:** the gateway app has already initialized `stream_bridge`, `checkpointer`, `store`, and `run_manager` in `repos/deer-flow/backend/app/gateway/deps.py:langgraph_runtime`
- **What this step does:** dispatches the request into `start_run` and prepares an SSE `StreamingResponse`
- **What leaves:** a `RunRecord` plus a live SSE response generator
- **Why this step matters:** this is DeerFlow's product-facing ingress. Everything downstream inherits its run semantics.

### Step 2 — Create transient run state

- **Location:** `repos/deer-flow/backend/app/gateway/services.py:start_run`
- **What enters:** validated request body, `thread_id`, and request-scoped singleton accessors
- **Assumptions already true:** the thread may or may not already exist in the store or checkpointer
- **What this step does:** selects disconnect behavior, calls `RunManager.create_or_reject`, upserts a lightweight thread record into the store, resolves the agent factory, normalizes input, and builds a `RunnableConfig`
- **What leaves:** a pending `RunRecord`, normalized `graph_input`, and a DeerFlow-configured `config` dict
- **Why this step matters:** DeerFlow already splits run truth here: `RunRecord` lives in memory, thread metadata lives in store, and agent state will live in the checkpointer.

### Step 3 — Schedule background execution

- **Location:** `repos/deer-flow/backend/app/gateway/services.py:start_run`
- **What enters:** `RunRecord`, `graph_input`, `config`, `checkpointer`, `store`, and `make_lead_agent`
- **Assumptions already true:** the request has not yet streamed any agent output
- **What this step does:** schedules `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent` as an `asyncio.Task`, then schedules best-effort title sync after completion
- **What leaves:** a `RunRecord` whose `task` field points at the worker task
- **Why this step matters:** the gateway does not execute the agent inline. It turns every turn into a background job with its own in-memory lifecycle.

### Step 4 — Inject DeerFlow runtime context

- **Location:** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent`
- **What enters:** bridge, run manager, run record, checkpointer, store, agent factory, normalized input, and config
- **Assumptions already true:** the run is still pending and no stream item has been published yet
- **What this step does:** marks the run as running, snapshots the pre-run checkpoint id, publishes run metadata to the stream bridge, creates a LangGraph `Runtime` with `thread_id`, injects it into `config["configurable"]["__pregel_runtime"]`, and builds a `RunnableConfig`
- **What leaves:** a framework-compatible runtime config and the first SSE-visible metadata event
- **Why this step matters:** this is where DeerFlow patches framework expectations manually. It is also where single-process assumptions begin.

### Step 5 — Assemble the lead agent

- **Location:** `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`
- **What enters:** `RunnableConfig`
- **Assumptions already true:** `thread_id` already sits in runtime context and config metadata is mutable
- **What this step does:** resolves the model name, resolves custom agent config if `agent_name` exists, computes runtime metadata, calls `create_chat_model`, calls `get_available_tools`, builds middleware through `_build_middlewares`, renders the giant system prompt through `apply_prompt_template`, and finally calls `langchain.agents.create_agent`
- **What leaves:** a compiled LangChain/LangGraph agent graph with DeerFlow state schema and middleware
- **Why this step matters:** this is DeerFlow's real center of gravity. If you inherit DeerFlow, you inherit this assembly philosophy.

### Step 6 — Materialize thread filesystem state

- **Location:** `repos/deer-flow/backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py:ThreadDataMiddleware.before_agent`
- **What enters:** current `ThreadState` and runtime context
- **Assumptions already true:** `thread_id` exists in runtime context or configurable metadata
- **What this step does:** resolves workspace, uploads, and outputs paths under `.deer-flow/threads/{thread_id}/user-data/...` and injects them into `state["thread_data"]`
- **What leaves:** `thread_data` inside the framework state
- **Why this step matters:** DeerFlow turns every conversation thread into a filesystem namespace. That is one of the few pieces that maps cleanly to your AgentOS direction.

### Step 7 — Rewrite the last user message with upload bookkeeping

- **Location:** `repos/deer-flow/backend/packages/harness/deerflow/agents/middlewares/uploads_middleware.py:UploadsMiddleware.before_agent`
- **What enters:** the last `HumanMessage` and resolved uploads directory
- **Assumptions already true:** `thread_data` already exists
- **What this step does:** scans the current message's attached files, scans historical uploads on disk, serializes them into an `<uploaded_files>` block, and prepends that block to the human message content
- **What leaves:** mutated human message content plus `uploaded_files` state
- **Why this step matters:** DeerFlow solves file awareness by prompt injection, not by a typed context channel. That is fast, but it is also brittle.

### Step 8 — Bind tools and prompt

- **Location:** `repos/deer-flow/backend/packages/harness/deerflow/tools/tools.py:get_available_tools` and `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/prompt.py:apply_prompt_template`
- **What enters:** current app config, model name, feature flags, enabled skills, MCP cache, ACP config, memory files, and optional agent soul
- **Assumptions already true:** the harness can read config files, extensions config, and the skills directory from disk
- **What this step does:** resolves configured tools by reflection, appends built-ins like `present_file` and `ask_clarification`, optionally appends `task`, `view_image`, MCP tools, ACP tools, deferred tool search, skill listings, memory injection, and subagent prompt sections
- **What leaves:** a tool list and a giant system prompt string
- **Why this step matters:** this is not just composition. This is DeerFlow's worldview encoded as prompt and registry shape.

### Step 9 — Let the framework own the inner loop

- **Location:** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent` -> `agent.astream(...)`
- **What enters:** `graph_input`, runnable config, stream modes, checkpointer, and store
- **Assumptions already true:** DeerFlow-specific prompt, state schema, and middleware have already been bound
- **What this step does:** yields control to the framework-owned LangChain/LangGraph loop, which prompts the model, selects tool calls, routes tool execution through middleware wrappers, persists checkpoints, and stops on final answer or interrupt
- **What leaves:** streamed chunks in LangGraph `values`, `messages`, `updates`, or other stream modes
- **Why this step matters:** DeerFlow does not own the critical algorithmic loop. It decorates it.

### Step 10 — Serialize and enqueue stream events

- **Location:** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent` and `repos/deer-flow/backend/packages/harness/deerflow/runtime/serialization.py:serialize`
- **What enters:** LangGraph chunks
- **Assumptions already true:** DeerFlow requested stream modes that `agent.astream` can yield
- **What this step does:** maps stream modes to SSE event names, strips internal LangGraph keys like `__interrupt__`, serializes messages and state into JSON-safe shapes, and publishes them into `MemoryStreamBridge`
- **What leaves:** queued `StreamEvent` objects
- **Why this step matters:** this is the seam that makes DeerFlow feel like a product rather than a raw graph library.

### Step 11 — Deliver SSE and persist post-run metadata

- **Location:** `repos/deer-flow/backend/app/gateway/services.py:sse_consumer` and `repos/deer-flow/backend/app/gateway/services.py:_sync_thread_title_after_run`
- **What enters:** bridge entries, request disconnect state, finished background task
- **Assumptions already true:** the worker has published metadata, stream events, and an end sentinel
- **What this step does:** formats SSE frames, cancels the run if the client disconnects with `on_disconnect=cancel`, then reads the final checkpoint to sync `title` back into the store
- **What leaves:** HTTP stream output plus updated store metadata
- **Why this step matters:** DeerFlow ends the turn with three different persisted views of reality: checkpoint state, store metadata, and already-sent SSE output.

---

## 6. Dispatch Map

### Assistant routing

- **What selects:** `assistant_id`
- **How it selects:** `repos/deer-flow/backend/app/gateway/services.py:resolve_agent_factory` always returns `make_lead_agent`; custom assistants only inject `configurable["agent_name"]`
- **Where the resolver lives:** `repos/deer-flow/backend/app/gateway/services.py:resolve_agent_factory` and `repos/deer-flow/backend/app/gateway/services.py:build_run_config`
- **Risk this introduces:** DeerFlow advertises assistants, but all assistants route through one factory. Variation is prompt/config variation, not runtime variation.

### Model provider dispatch

- **What selects:** model name from config or request context
- **How it selects:** `AppConfig.get_model_config(name)` plus reflective class loading in `resolve_class(model_config.use, BaseChatModel)`
- **Where the resolver lives:** `repos/deer-flow/backend/packages/harness/deerflow/models/factory.py:create_chat_model`
- **Risk this introduces:** every model provider detail bleeds through config and reflection. That keeps the surface flexible, but it couples correctness to YAML.

### Tool surface dispatch

- **What selects:** tool groups, model capability, MCP enablement, ACP config, and `subagent_enabled`
- **How it selects:** config filtering plus reflection in `resolve_variable(tool.use, BaseTool)` plus conditional built-in append
- **Where the resolver lives:** `repos/deer-flow/backend/packages/harness/deerflow/tools/tools.py:get_available_tools`
- **Risk this introduces:** the tool surface is highly dynamic and partly hidden in config files and extension state. Debugging "why the model saw this tool" gets expensive fast.

### Skill dispatch

- **What selects:** enabled skills on disk and the model's own judgment about which skill to read
- **How it selects:** `load_skills(enabled_only=True)` injects a list of SKILL.md paths into the system prompt; the LLM self-selects by calling `read_file`
- **Where the resolver lives:** `repos/deer-flow/backend/packages/harness/deerflow/skills/loader.py:load_skills` and `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/prompt.py:get_skills_prompt_section`
- **Risk this introduces:** this is not a runtime registry. It is prompt-mediated self-dispatch. You inherit recall errors, prompt bloat, and non-deterministic skill selection.

### Middleware ordering dispatch

- **What selects:** feature flags or explicit middleware instances plus optional `@Next` / `@Prev` anchors
- **How it selects:** sequential assembly in `_build_middlewares` or `_assemble_from_features`, then anchored insertion
- **Where the resolver lives:** `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:_build_middlewares` and `repos/deer-flow/backend/packages/harness/deerflow/agents/factory.py:_insert_extra`
- **Risk this introduces:** order is load-bearing. A small reorder silently changes runtime behavior because thread paths, uploads, sandbox, tool patching, memory, and clarification all depend on relative position.

### Sandbox backend dispatch

- **What selects:** `config.sandbox.use`
- **How it selects:** reflective provider construction
- **Where the resolver lives:** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/sandbox_provider.py:get_sandbox_provider`
- **Risk this introduces:** the interface looks stable, but the semantics differ radically between `LocalSandboxProvider` and `AioSandboxProvider`.

### Persistence backend dispatch

- **What selects:** `checkpointer.type`
- **How it selects:** branching factory for memory, sqlite, or postgres
- **Where the resolver lives:** `repos/deer-flow/backend/packages/harness/deerflow/agents/checkpointer/provider.py:get_checkpointer` and `repos/deer-flow/backend/packages/harness/deerflow/runtime/store/provider.py:get_store`
- **Risk this introduces:** DeerFlow mirrors store and checkpointer backends, but the logical state split stays the same. Switching storage does not remove the split.

### Channel session dispatch

- **What selects:** user-level, channel-level, and default session overrides
- **How it selects:** layered dict merge into `run_config` and `run_context`
- **Where the resolver lives:** `repos/deer-flow/backend/app/channels/manager.py:_resolve_run_params`
- **Risk this introduces:** IM delivery adds another config plane that can override model and agent behavior independently of gateway callers.

---

## 7. State Model

### `ThreadState`

- **Type:** `repos/deer-flow/backend/packages/harness/deerflow/agents/thread_state.py:ThreadState`
- **Authoritative vs derived:** authoritative for DeerFlow's framework-visible per-thread agent state
- **Persistent vs transient:** persistent through LangGraph checkpoints when a checkpointer exists; otherwise transient
- **Created at:** `make_lead_agent(... state_schema=ThreadState)` and `DeerFlowClient._ensure_agent(...)`
- **Mutated at:** middlewares like `ThreadDataMiddleware.before_agent`, `UploadsMiddleware.before_agent`, `TitleMiddleware`, `MemoryMiddleware`, `ViewImageMiddleware`, tools like `present_file_tool`
- **Read by:** LangGraph agent execution, gateway thread state endpoints, serialization helpers
- **Invariants that matter:** `messages` remains the canonical conversation list; `thread_data` must exist before filesystem-aware tools run; `artifacts` is append-merge state, not immutable truth

### `RunRecord`

- **Type:** `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/manager.py:RunRecord`
- **Authoritative vs derived:** authoritative only for the live HTTP/API run lifecycle
- **Persistent vs transient:** transient, process-local, in-memory only
- **Created at:** `RunManager.create` or `RunManager.create_or_reject`
- **Mutated at:** `RunManager.set_status`, `RunManager.cancel`, `start_run` when attaching `task`
- **Read by:** gateway run endpoints and `sse_consumer`
- **Invariants that matter:** a run id maps to one thread id; status transitions happen under one asyncio lock; state vanishes after cleanup

### Thread store record

- **Type:** loose dict under namespace `("threads",)`
- **Authoritative vs derived:** derived index of thread metadata and title, not full agent truth
- **Persistent vs transient:** persistent when store backend is sqlite or postgres; transient when in-memory
- **Created at:** `threads.create_thread` or `services._upsert_thread_in_store`
- **Mutated at:** `_store_upsert`, `_sync_thread_title_after_run`, `threads.patch_thread`
- **Read by:** `threads.search_threads`, `threads.get_thread`
- **Invariants that matter:** it can lag the checkpointer; it is a search index, not the full state machine

### `AppConfig` and extension config singletons

- **Type:** `AppConfig`, `ExtensionsConfig`, plus several module-level config caches
- **Authoritative vs derived:** authoritative runtime configuration
- **Persistent vs transient:** persistent on disk, cached in-process
- **Created at:** `get_app_config()`, `ExtensionsConfig.from_file()`, and related singleton getters
- **Mutated at:** config reloads, gateway skill/MCP update endpoints, file writes
- **Read by:** model factory, tool loader, sandbox provider, prompt builder, memory config, channel service
- **Invariants that matter:** the harness assumes files exist and stay coherent; many runtime decisions re-read config from disk to bridge process boundaries

### Memory data and update queue

- **Type:** JSON memory payload plus `ConversationContext` queue entries
- **Authoritative vs derived:** derived long-term summary state, not raw conversation truth
- **Persistent vs transient:** memory payload persists to JSON; queue stays in-process
- **Created at:** `get_memory_storage().load(...)` and `MemoryUpdateQueue.add`
- **Mutated at:** `MemoryMiddleware.after_agent`, `MemoryUpdater.update_memory`, manual memory CRUD helpers
- **Read by:** prompt injection in `_get_memory_context`
- **Invariants that matter:** memory can lag conversation turns; queue debouncing can drop intermediate contexts per thread by design

### Sandbox provider caches

- **Type:** provider singletons and internal maps like `_thread_sandboxes`, `_warm_pool`, or `_singleton`
- **Authoritative vs derived:** authoritative for sandbox reuse inside one process
- **Persistent vs transient:** transient process memory
- **Created at:** `get_sandbox_provider`
- **Mutated at:** `acquire`, `release`, idle cleanup, shutdown
- **Read by:** sandbox middleware and sandbox tools
- **Invariants that matter:** `LocalSandboxProvider` is effectively one host shell for everyone; `AioSandboxProvider` assumes thread-specific mounts and container lifecycle ownership

---

## 8. Data Transformation Chain

**Starting shape:** HTTP request body or client call such as `{"input": {"messages": [{"role": "user", "content": "..."}]}}`

| Step | Before Shape | After Shape | Meaning Added / Lost / Compressed |
|------|-------------|-------------|-----------------------------------|
| `normalize_input` | LangGraph-style dict with role/content dicts | LangChain state dict with `HumanMessage` objects | DeerFlow collapses non-human roles into `HumanMessage` placeholders; message-type precision is lost early |
| `build_run_config` | request config + context + metadata | `RunnableConfig`-compatible dict | thread id, assistant routing, and DeerFlow context flags get merged into framework config |
| `_build_middlewares` | config flags | ordered middleware list | runtime policy becomes positional behavior |
| `get_available_tools` | config tool specs + runtime flags | bound `BaseTool` list | reflective config turns into executable tool surface |
| `apply_prompt_template` | memory, skills, soul, feature flags | one giant system prompt string | structured runtime concepts get flattened into prompt text |
| `UploadsMiddleware.before_agent` | `HumanMessage("user text")` | `HumanMessage("<uploaded_files>...user text")` | file metadata gets flattened into prompt text instead of typed context |
| `agent.astream` | framework state + config | LangGraph stream chunks | DeerFlow hands control to the framework loop |
| `serialize(..., mode="values")` | raw state dict with internal keys | JSON-safe state dict | internal LangGraph control keys get stripped |
| `format_sse` | event name + payload | SSE wire frame | DeerFlow repackages framework output into product streaming format |

**Final shape:** SSE event stream, plus checkpointed channel values, plus a store record carrying search metadata like `title`.

**Key lossy transformations:** `normalize_input` downgrades message roles; prompt assembly flattens memory, skills, and file listings into strings; serialization strips internal LangGraph control keys; store sync reduces rich state down to metadata and title.

---

## 9. Side-Effect Map

### Model API invocation

- **Where triggered:** `repos/deer-flow/backend/packages/harness/deerflow/models/factory.py:create_chat_model` via the framework loop started in `run_agent`
- **Guards:** model name must resolve in config; provider credentials must exist
- **Retryable:** conditional, mostly delegated to provider/client behavior rather than DeerFlow runtime policy
- **Idempotent:** no
- **Failure consequence:** the run flips to `error`; the worker emits an SSE `"error"` event

### Checkpoint persistence

- **Where triggered:** inside the LangGraph loop; explicitly written in `threads.create_thread` and `threads.update_thread_state`
- **Guards:** a checkpointer backend exists and can serialize the current state
- **Retryable:** conditional
- **Idempotent:** not strictly; each update writes a fresh checkpoint id
- **Failure consequence:** thread history, multi-turn state, and resume semantics break

### Thread store writes

- **Where triggered:** `services._upsert_thread_in_store`, `threads._store_put`, `_sync_thread_title_after_run`
- **Guards:** store backend exists
- **Retryable:** yes
- **Idempotent:** mostly yes for upserts
- **Failure consequence:** search/list/title views drift from checkpoint truth, but runs can still finish

### Thread filesystem writes

- **Where triggered:** `ThreadDataMiddleware`, `uploads/manager.py`, sandbox tools, `present_file`
- **Guards:** valid `thread_id`, permitted paths, sandbox or host filesystem availability
- **Retryable:** conditional
- **Idempotent:** no for arbitrary writes; yes for directory creation
- **Failure consequence:** uploads, artifacts, and workspace-based tool execution fail

### Sandbox command execution

- **Where triggered:** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/local/local_sandbox.py:execute_command` or AIO sandbox backends
- **Guards:** sandbox provider resolves; host bash may be disabled; path validation passes
- **Retryable:** conditional
- **Idempotent:** no
- **Failure consequence:** tool calls return errors or side effects partially land on host/container state

### Memory LLM update

- **Where triggered:** `repos/deer-flow/backend/packages/harness/deerflow/agents/memory/updater.py:MemoryUpdater.update_memory`
- **Guards:** memory config enabled, filtered conversation includes at least one human and one assistant message
- **Retryable:** yes in theory, but DeerFlow currently logs and skips on failure
- **Idempotent:** no, because repeated summarization can yield different memory JSON
- **Failure consequence:** personalization degrades quietly; core turn execution still succeeds

### IM outbound delivery

- **Where triggered:** channel classes under `repos/deer-flow/backend/app/channels/*.py`
- **Guards:** channel credentials, file size limits, message formatting
- **Retryable:** conditional
- **Idempotent:** no
- **Failure consequence:** users on Feishu/Slack/Telegram miss responses even if the run itself succeeded

---

## 10. Important Branch Points

### Default agent vs bootstrap agent

- **Condition:** `configurable["is_bootstrap"]`
- **Where checked:** `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`
- **Alternate paths:** bootstrap path appends only `setup_agent`; default path resolves configured tool groups and agent soul
- **Why it matters:** DeerFlow already carries product bootstrapping inside the same agent factory, which broadens the core surface area.

### Custom assistant name vs default lead agent

- **Condition:** `assistant_id != "lead_agent"`
- **Where checked:** `repos/deer-flow/backend/app/gateway/services.py:build_run_config` and `repos/deer-flow/backend/app/channels/manager.py:_resolve_run_params`
- **Alternate paths:** default assistant routes directly; custom assistant gets normalized into `agent_name` but still uses the lead-agent factory
- **Why it matters:** DeerFlow custom agents are prompt/config overlays, not separate runtimes.

### Local sandbox vs AIO sandbox

- **Condition:** `config.sandbox.use`
- **Where checked:** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/sandbox_provider.py:get_sandbox_provider`
- **Alternate paths:** `LocalSandboxProvider` maps virtual paths onto the host shell; `AioSandboxProvider` provisions containers
- **Why it matters:** the same "sandbox" abstraction hides radically different trust boundaries.

### Host bash enabled vs disabled

- **Condition:** `sandbox.allow_host_bash`
- **Where checked:** `repos/deer-flow/backend/packages/harness/deerflow/sandbox/security.py:is_host_bash_allowed`
- **Alternate paths:** bash tool is either exposed or filtered out
- **Why it matters:** DeerFlow's local experience can silently shift from "no shell" to "host shell" based on one config bit.

### Subagent mode enabled vs disabled

- **Condition:** `configurable["subagent_enabled"]`
- **Where checked:** `make_lead_agent`, `_build_middlewares`, and `get_available_tools`
- **Alternate paths:** `task` tool and `SubagentLimitMiddleware` either appear or stay absent
- **Why it matters:** subagents are not always-on runtime primitives. They are optional prompt and tool inflation.

### Durable persistence vs in-memory fallback

- **Condition:** whether config declares sqlite/postgres checkpointer/store
- **Where checked:** `get_checkpointer`, `get_store`, and async equivalents
- **Alternate paths:** durable backends persist across restarts; in-memory fallback loses state on restart
- **Why it matters:** DeerFlow can look production-ready while silently running on ephemeral state.

### Multitask strategy

- **Condition:** `multitask_strategy`
- **Where checked:** `RunManager.create_or_reject`
- **Alternate paths:** reject, interrupt, or rollback inflight runs
- **Why it matters:** DeerFlow exposes concurrency policy, but rollback is still a stub in `run_agent`.

---

## 11. Failure and Recovery Logic

### Failure Surfaces

| Surface | Where Detected | How It Propagates | Translated To |
|---------|---------------|-------------------|---------------|
| Unknown or invalid model | `make_lead_agent` | exception bubbles into `run_agent` | run status `error` + SSE `"error"` |
| Tool exception | `ToolErrorHandlingMiddleware.wrap_tool_call` | wrapped | `ToolMessage(status="error")` so the model can continue |
| Guardrail denial | `GuardrailMiddleware` | wrapped | `ToolMessage(status="error")` |
| Missing config files or invalid YAML | `get_app_config` / `load_agent_config` / `ExtensionsConfig.from_file` | bubbles or logs depending on call site | startup failure, run failure, or feature silently absent |
| Store write/title sync failure | gateway helpers | logged and swallowed | stale search metadata |
| Stream bridge queue overflow | `MemoryStreamBridge.publish` | logged and dropped | missing stream events |
| Memory update JSON parse or LLM failure | `MemoryUpdater.update_memory` | logged and swallowed | stale memory |
| Client disconnect | `sse_consumer` | explicit cancel path | interrupted run |
| Rollback request | `run_agent` | partial stub | status changes, but checkpoint rollback does not actually happen |

### Recovery Behavior

- **Retry logic:** DeerFlow retries very little at the runtime level. The biggest recovery behavior is not retry; it is degradation. Tool errors turn into `ToolMessage` objects, store failures log and continue, memory failures log and continue, and unsupported stream mode `"events"` gets skipped in `run_agent`.
- **Fallback logic:** `get_checkpointer` and `get_store` fall back to in-memory implementations when config is absent. `create_chat_model` can fall back to the default model name. `build_run_config` prefers `context` over `configurable` for newer LangGraph behavior.
- **Timeout behavior:** `LocalSandbox.execute_command` hard-caps subprocess execution at 600 seconds. Subagent polling caps at execution timeout plus a buffer in `task_tool`. Stream bridge heartbeats fire every 15 seconds by default.
- **Lifecycle mismatch risks:** the biggest mismatch is split truth. `RunRecord`, checkpoint state, and store metadata can disagree. Another mismatch is process locality: `RunManager` and `MemoryStreamBridge` are in-process, while channels and other clients may assume a more durable server.

---

## 12. The Real Core Loop

DeerFlow does not implement its own first-principles agent loop. The effective unit of work is a framework invocation shaped by DeerFlow wrappers.

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

The blunt truth: DeerFlow owns the harness around the loop. LangChain/LangGraph owns the loop.

---

## 13. Architectural Compression

**True center of gravity:** DeerFlow-owned gravity sits in `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:make_lead_agent`, because every major path eventually routes through it, and because it selects the model, tool surface, middleware order, state schema, and system prompt. The deeper runtime gravity still sits in the external `langchain.agents.create_agent` loop that `make_lead_agent` calls.

**Replaceable parts:** the frontend, IM channels, gateway routers, model provider classes, checkpointer backend, store backend, and sandbox provider are all replaceable without rewriting the whole repo. The harness boundary test in `repos/deer-flow/backend/tests/test_harness_boundary.py:test_harness_does_not_import_app` proves the team is actively trying to make the harness swappable.

**Dangerous seams:** `build_run_config` and `_resolve_run_params` merge runtime truth through loosely-typed dicts; skill selection depends on prompt self-dispatch; `ThreadState`, `RunRecord`, and store metadata each hold different slices of truth; `LocalSandboxProvider` markets itself as a sandbox while executing on the host; `MemoryStreamBridge` and `RunManager` assume one process.

**Real abstractions:** the sandbox provider interface, checkpointer/store backend factories, stream bridge abstraction, and partial `create_deerflow_agent` factory are real. They enable actual runtime variation.

**Decorative abstractions:** the assistant model is mostly decorative because all assistants route through `make_lead_agent`; the skill registry is mostly decorative because the LLM still decides which markdown file to read; the pure-SDK story in `create_deerflow_agent` is not finished because key runtime pieces still read global config or the filesystem.

---

## DeerFlow as AgentOS Base: Feasibility, Adoption Strategy, and Boundary of Inheritance

**Feasible or not:** feasible only as a selective code donor and temporary reference harness. It is not a safe direct dependency, and it is not a sound whole-repo base for your AgentOS.

**Why the answer is not stronger:** DeerFlow gets several important outer-shell problems right enough to learn from: per-thread filesystem layout, streaming run lifecycle, partial harness/app split, embedded client shape, and a more real container sandbox path. But the core agent machine you need is still too framework-owned, too config-file-driven, and too product-opinionated.

**Blunt mode-by-mode judgment:**

- **Direct dependency:** no. DeerFlow still hides too much runtime truth in config singletons, LangGraph state, and prompt assembly. You would spend your time patching inherited behavior instead of building your own OS contract.
- **Forked base:** only for a disposable spike. If you fork DeerFlow as the mainline base, you inherit LangChain/LangGraph tax, split-brain state, fake-local-sandbox semantics, prompt-driven skills, and gateway/channel product assumptions immediately.
- **Selective code donor / reference only:** yes. This is the right move.

**What should be inherited:**

- `repos/deer-flow/backend/packages/harness/deerflow/config/paths.py`
- `repos/deer-flow/backend/packages/harness/deerflow/uploads/manager.py`
- the interface ideas in `repos/deer-flow/backend/packages/harness/deerflow/runtime/stream_bridge/base.py`
- the lightweight run lifecycle shape in `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/manager.py`
- selected pieces of `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py`
- the embedded-client shape in `repos/deer-flow/backend/packages/harness/deerflow/client.py`, but not the current agent factory under it

**What must be rewritten early:**

- the main agent loop contract
- the authoritative turn/session state model
- the permission and human-approval model
- the tool and skill registry contract
- the result schema for "completed / blocked / failed" turns
- the local-first CLI shell

**What should be postponed:**

- LangGraph compatibility endpoints under `app/gateway/routers/*`
- IM channels under `app/channels/*`
- title generation, follow-up suggestions, and current memory summarization
- deferred tool search unless tool count actually explodes

**What should never be inherited:**

- the current `lead_agent` prompt as your system prompt base
- prompt-driven skill selection as your skills architecture
- `LocalSandboxProvider` as a security story
- the assumption that assistants are just `agent_name` overlays on one graph
- the assumption that LangGraph checkpoint state should be your AgentOS domain state

### Major Subsystem Decision Matrix

| subsystem | DeerFlow current design | why it exists | strengths | weaknesses / risk | fit for our AgentOS | recommendation: KEEP / FORK-AND-MODIFY / REWRITE / IGNORE | phase-1 usage | long-term usage |
|---|---|---|---|---|---|---|---|---|
| Lead agent assembly | `make_lead_agent` assembles model, prompt, tools, and middleware around `create_agent` | give every surface one default agent factory | fast composition; one obvious entrypoint | giant implicit-I/O factory; borrowed inner loop; prompt tax | low | REWRITE | study only | none |
| Pure-ish agent factory | `create_deerflow_agent` plus `RuntimeFeatures` | expose a more SDK-like harness path | better boundary than `make_lead_agent`; testable insertion model | still not config-free; docs admit hidden I/O remains | medium | FORK-AND-MODIFY | copy the API shape | keep only an adapted version |
| Run lifecycle | `RunManager` + `run_agent` + `StreamBridge` | turn graph execution into product runs | thin and understandable; streaming-friendly | single-process; rollback stub; run truth split from checkpoint truth | medium | FORK-AND-MODIFY | acceptable for local prototype | replace backing persistence and scheduler |
| Checkpointer + store backend factories | config-driven memory/sqlite/postgres selection | persist thread state and search metadata | practical backend swap seam | tied to LangGraph state model | low-medium | FORK-AND-MODIFY | keep only if LangGraph stays | rewrite if AgentOS owns its own state engine |
| Thread filesystem + uploads + outputs | `.deer-flow/threads/{thread_id}/user-data/...` plus upload utilities | give each thread a concrete workspace | directly useful; aligns with artifact-producing shell | thread id rules differ between modules; still file-centric | high | KEEP | copy almost directly | keep with minor cleanup |
| Sandbox abstraction | provider interface over local or container sandboxes | let tools execute outside Python memory | real abstraction; swappable backends | semantics vary too much between providers | medium | FORK-AND-MODIFY | keep interface only | keep adapted |
| Local sandbox | singleton host shell with path mapping | zero-friction local dev | easy to run; easy to debug | not a real sandbox; shared host shell; permission lie | very low | REWRITE | do not inherit as security boundary | never |
| AIO sandbox | container/K8s provisioner with thread mounts and warm pool | move from host shell to isolated execution | closest thing to a real runtime asset here | heavy Docker/K8s tax; mount assumptions; operational weight | medium | FORK-AND-MODIFY | postpone unless sandbox is urgent | maybe keep selectively |
| Tool loader + deferred tool search | reflective config tools plus MCP deferral | keep tool surface flexible and context smaller | flexible; deferred schema idea is smart | config-driven opacity; MCP-heavy; debugging cost | medium | FORK-AND-MODIFY | copy deferred-schema idea only | keep only if tool catalog gets large |
| Skill system | scan SKILL.md and let the model read skills by prompt | cheap extensibility without codegen | low authoring friction | not a real registry; no reliable matching; prompt bloat | low | REWRITE | at most reuse as human playbooks | replace with explicit skill contracts |
| Memory system | summarize conversation into JSON and inject into prompt | give lightweight personalization | simple and independent of core run path | not a ContextEngine; low trust; async lag; file-bound | low | REWRITE | ignore for phase 1 | build your own context engine |
| Subagent system | `task` tool + background threadpool + polling | parallelize larger work | proves one workable UX pattern | crude scheduler; shared assumptions; high prompt/tool coupling | low-medium | FORK-AND-MODIFY | maybe copy event UX later | rewrite if subagents matter |
| DeerFlowClient | embedded Python client that skips gateway/langgraph server | make local embedding possible | valuable shape for local mode; gateway-conformance tests exist | Python SDK, not CLI product; still config-bound | medium-high | FORK-AND-MODIFY | strong donor for internal SDK facade | keep only a rewritten facade |
| Gateway LangGraph compat layer | FastAPI routes mirroring LangGraph Platform | let frontend and SDK speak one protocol | practical product adapter | adds split-brain architecture and more state duplication | low | IGNORE | none | maybe add later only if SDK compatibility matters |
| IM channels | Feishu/Slack/Telegram bridge via `langgraph_sdk` | push the same agent into chat apps | useful product reference | transport-specific logic and session overrides pollute the runtime picture | low | IGNORE | none | revisit after core runtime stabilizes |
| Frontend and demo artifacts | Next.js UI over gateway plus demo threads | show a complete product | good demo value | irrelevant to runtime-base choice | very low | IGNORE | none | none |

### Biggest architectural lies / illusions in this repo

- **"DeerFlow has its own agent runtime."** No. DeerFlow has a harness around a LangChain/LangGraph runtime. That matters because every core-behavior fight eventually becomes a framework fight.
- **"`LocalSandboxProvider` is a sandbox."** No. It is a host-shell adapter with path rewriting.
- **"Skills are a registry."** Not really. They are markdown prompts that the model may or may not choose to read.
- **"Assistants are first-class runtime variants."** Not really. They are mostly `lead_agent` plus `agent_name`.
- **"Runs, threads, and state have one source of truth."** No. Run status, checkpoint state, and store metadata diverge by design.

### What this repo makes look easier than it really is

- making LangGraph feel like a product API
- making local mode and server mode share the same runtime truth
- making skill extensibility look deterministic
- making sandbox security look like a provider swap
- making long-term memory look like a solved subsystem rather than a lossy summary job

### What will likely break first under real product pressure

- multi-process or multi-pod streaming with `MemoryStreamBridge`
- permission and safety expectations if anyone trusts `LocalSandboxProvider`
- prompt quality once more skills, tools, and content-creation instructions pile up
- state consistency when runs interrupt, roll back, or reconnect across transports
- any attempt to make the current memory system carry serious personalization load

---

## 14. Design Intent Inference

**Primary optimization:** ship a broad, demoable, general-purpose agent product quickly on top of LangGraph, then carve out a reusable harness afterward.

**Evidence for this inference:** the repo ships frontend, gateway, channels, public skills, prompt-heavy behavior shaping, LangGraph compatibility routes, and an embedded client. The RFC at `repos/deer-flow/backend/docs/rfc-create-deerflow-agent.md` explicitly admits "`6 处隐式 I/O`" in the current harness and frames full config-free runtime as a later goal.

**Secondary optimizations:** feature breadth, transport reach, and user-visible polish. DeerFlow invests in title generation, suggestions, skills, channel adapters, and compatibility APIs before it invests in a self-owned runtime contract.

**What was sacrificed:** runtime explicitness, local-shell-first ergonomics, deterministic permissions, crisp state ownership, and future freedom from LangChain/LangGraph assumptions.

---

## 15. Safe Surgery Map

### Safer to Modify First

- `repos/deer-flow/backend/packages/harness/deerflow/config/paths.py` because it already expresses a clean filesystem contract with limited framework coupling
- `repos/deer-flow/backend/packages/harness/deerflow/uploads/manager.py` because it is pure business logic with no FastAPI or LangGraph dependency
- `repos/deer-flow/backend/packages/harness/deerflow/runtime/stream_bridge/base.py` because the abstraction is small and the pain points are obvious
- `repos/deer-flow/backend/packages/harness/deerflow/client.py` because it is the cleanest bridge to a local embedded story, even though the internals still need replacement

### High-Risk to Modify Early

- `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/prompt.py` because the prompt is already carrying too much policy
- `repos/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py` because middleware order and tool surface are load-bearing
- `repos/deer-flow/backend/app/gateway/services.py` plus `runtime/runs/worker.py` if you still rely on LangGraph compatibility
- `repos/deer-flow/backend/app/channels/*` because those paths add another state/config plane immediately

### Smallest Viable Adoption Path

1. Copy the thread filesystem contract from `config/paths.py` and `uploads/manager.py`.
2. Copy the shape, not the implementation, of `RunManager`, `StreamBridge`, and `DeerFlowClient`.
3. Write your own explicit `TurnState`, `RunResult`, and permission contract first.
4. If you need container sandboxing soon, fork only `AioSandboxProvider` concepts, not the local provider.
5. Keep LangGraph behind your own runtime facade if you use it at all.
6. Treat DeerFlow skills as prompt playbooks during phase 1, not as your long-term skill registry.

### Open Questions

- Do you want LangGraph at all once you add explicit turn-result contracts, human approvals, and a richer ContextEngine?
- Do you want your generation gateway to sit outside the agent runtime, or do you want it to become one more tool provider under the same contract?
- Do you want local CLI and cloud SaaS to share one state model, or only one tool/skill contract?

### Next Recommended Investigations

- compare DeerFlow's container sandbox path against OpenClaw's permission and session model rather than against DeerFlow's local mode
- decide whether your AgentOS needs a framework-owned loop or a self-owned loop before you copy any LangGraph-shaped state
- prototype your explicit `completed / blocked / failed` run result schema before adopting any existing persistence layer

---

## 16. Learning-Value Map

### Most worth learning

1. `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/worker.py:run_agent`
   What it teaches: how to wrap a framework graph in product run lifecycle, stream serialization, and abort semantics without forking the framework server.

2. `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py:AioSandboxProvider`
   What it teaches: how to tie thread-specific filesystem mounts, warm-pool reuse, and container lifecycle into an agent runtime.

3. `repos/deer-flow/backend/packages/harness/deerflow/agents/factory.py:create_deerflow_agent`
   What it teaches: how a team tries to claw back a reusable SDK layer after shipping a config-heavy product runtime first.

4. `repos/deer-flow/backend/packages/harness/deerflow/tools/builtins/tool_search.py:DeferredToolRegistry`
   What it teaches: one clean way to reduce tool-schema context pressure without deleting tools from the platform entirely.

5. `repos/deer-flow/backend/tests/test_harness_boundary.py:test_harness_does_not_import_app`
   What it teaches: when a system is mid-split, boundary tests often tell you more about real architecture intent than README prose does.

**Low learning value despite surface complexity:** the current lead-agent prompt, the public skills catalog, and most gateway routers. They teach product packaging, not kernel design.

---

## 17. Reuse/Copy-Value Map

### Most worth copying

| candidate | dependency weight | hidden assumptions | copy value |
|---|---|---|---|
| `repos/deer-flow/backend/packages/harness/deerflow/config/paths.py` | low | assumes thread-oriented filesystem layout and `.deer-flow` home | high; near-direct copy |
| `repos/deer-flow/backend/packages/harness/deerflow/uploads/manager.py` | low | assumes uploads live under the thread filesystem contract | high; near-direct copy |
| `repos/deer-flow/backend/packages/harness/deerflow/runtime/stream_bridge/base.py` | low | none beyond event-stream semantics | high; copy the interface |
| `repos/deer-flow/backend/packages/harness/deerflow/runtime/runs/manager.py` | low-medium | assumes in-process scheduler and one host | medium; adapt for durable runs |
| `repos/deer-flow/backend/packages/harness/deerflow/client.py` | medium | assumes LangGraph-shaped agent state and DeerFlow config files | medium; copy the facade pattern, not the guts |
| `repos/deer-flow/backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py` | high | assumes Docker/K8s, mount ownership, and thread-id filesystem contract | medium; selective donor only |
| `repos/deer-flow/backend/packages/harness/deerflow/models/factory.py` and provider patches | medium | assumes config-driven reflective provider loading | medium; useful if you stay in Python and keep similar model breadth |
| `repos/deer-flow/backend/packages/harness/deerflow/tools/builtins/tool_search.py` | medium | assumes deferred tools stay callable after prompt-side schema reveal | medium; good optional donor |

### Most worth copying late or not at all

- **Do not copy now:** `lead_agent` prompt, skill loading and skill prompt injection, memory updater, channel stack, gateway LangGraph compatibility layer.
- **Why not:** these parts move fast inside DeerFlow because they encode product behavior, not reusable runtime law.

---

## 18. Mental Model Packet

- DeerFlow is a product shell around a borrowed LangGraph loop.
- `make_lead_agent` is the DeerFlow-owned center of gravity.
- The best reusable assets are filesystem layout, uploads, stream/run wrappers, and some sandbox ideas.
- The worst inheritance risks are prompt-driven skills, fake local sandboxing, and split state truth.
- Custom assistants are mostly one factory with different prompt/config overlays.
- The harness/app split is real intent, but the harness is still not truly config-free.
- DeerFlow is better as a code donor than as a dependency or whole-base fork.
- Build your own state, permission, and skill contracts first; import DeerFlow ideas only after that boundary exists.
