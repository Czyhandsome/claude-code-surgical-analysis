[Back to Index](../README.md) · [中文版](claude-code-agent-runtime-surgical-analysis.cn.md) | **English**

**Table of Contents:** [0](#0-scope) · [1](#1-one-sentence-machine-definition) · [2](#2-why-this-part-exists) · [3](#3-top-level-control-picture) · [4](#4-concrete-entrypoints) · [5](#5-happy-path-surgical-walkthrough) · [6](#6-dispatch-map) · [7](#7-state-model) · [8](#8-data-transformation-chain) · [9](#9-side-effect-map) · [10](#10-important-branch-points) · [11](#11-failure-and-recovery-logic) · [12](#12-the-real-core-loop) · [13](#13-architectural-compression) · [14](#14-design-intent-inference) · [15](#15-safe-surgery-map) · [16](#16-learning-value-map) · [17](#17-reuse--copy-value-map) · [18](#18-mental-model-packet)

---

# Claude Code — Code Surgical Analysis Report

**Analyzed Codebase:** `repos/claude-code`
**Analysis Perspective:** Builder of "Reel Creator" — an AI-native social media content creation Agent OS, analogous to Claude Code but for content creation instead of coding
**Report Date:** 2026-04-01
**Analysis Passes Completed:** 7

---

## 0. Scope

This report surgically dissects the Claude Code codebase (`repos/claude-code/src/`) across seven analytical passes: entrypoint tracing, happy-path walkthrough, state architecture, dispatch mechanics, side-effect mapping, failure surface cataloging, and architectural compression. Every finding is tied to a specific file and line number. The terminal deliverable (sections 16–17) answers the Reel Creator builder's primary question: for each architectural component, is it COPY-PASTE, USE-THE-ESSENCE, or JUST-FOR-LEARN in Phase 1 (deadline: April 15)?

The codebase ships approximately 150+ TypeScript source files under `src/`. Key files by line count: `src/main.tsx` (4,683 lines), `src/query.ts` (1,729 lines), `src/skills/loadSkillsDir.ts` (1,086 lines), `src/utils/forkedAgent.ts` (689 lines), `src/Tool.ts` (792 lines), `src/services/tools/toolOrchestration.ts` (188 lines), `src/state/store.ts` (34 lines), `src/query/deps.ts` (40 lines).

---

## 1. One-Sentence Machine Definition

Claude Code executes a stateful, permission-gated, tool-calling agent loop that ingests a natural-language prompt, iteratively dispatches LLM API calls and tool invocations until no tool-use blocks remain, and persists the full transcript as append-only JSONL while maintaining a multi-level context window compaction stack to keep token usage within model limits.

---

## 2. Why This Part Exists

The system exists to solve a specific reliability problem: raw LLM completions cannot safely write files, run shell commands, or perform multi-step operations on a real filesystem without (a) a deterministic dispatch layer that routes each model-requested tool call to the correct implementation, (b) a permission gate that enforces human-in-the-loop consent before destructive actions, and (c) a memory architecture that survives context window overflow across long sessions. Without all three, a pure chat interface would either silently fail on complex tasks or destructively modify the filesystem without user awareness.

Reel Creator faces the identical problem domain shifted to content: raw LLM completions cannot safely publish to TikTok/Instagram, call asset generation APIs, or manage a multi-platform publishing pipeline without the same three components. The architectural motive transfers exactly; the tools change entirely.

---

## 3. Top-Level Control Picture

```
CLI entrypoint (src/entrypoints/cli.tsx)
    │
    ▼
Commander.js flag parsing (src/main.tsx)
    │
    ├─── Interactive REPL path ──► Ink/React UI (src/replLauncher.tsx)
    │                                    │
    │                              REPL input loop
    │                                    │
    └─── Headless/print path ──► QueryEngine.submitMessage()
                                        │
                                        ▼
                               src/query.ts:queryLoop()   ◄─── while(true) iteration engine
                                        │
                          ┌─────────────┼──────────────┐
                          ▼             ▼               ▼
                   compaction    callModel()        runTools()
                   checks        (LLM API)          (tool dispatch)
                          │             │               │
                          │      streaming SSE    permission gate
                          │      tool_use blocks  → tool.call()
                          │             │               │
                          └─────────────┴───────────────┘
                                        │
                                   transcript
                                   JSONL write
                                  (~/.claude/projects/)
```

The loop terminates when the model emits a response with zero `tool_use` content blocks. State lives in three places simultaneously: the reactive `AppState` store (UI-facing), the `mutableMessages[]` array in `QueryEngine` (conversation history), and the append-only JSONL on disk (durable transcript).

---

## 4. Concrete Entrypoints

**Primary binary entrypoint:** `src/entrypoints/cli.tsx:main()` — reads `process.argv`, performs fast-path exits for `--version` and `--daemon-worker` flags, then delegates to `src/main.tsx`.

**CLI program definition:** `src/main.tsx` — constructs a Commander.js program with a `[prompt]` positional argument and flags including `--print` (`-p`), `--model`, `--resume`, `--agent`, `--permission-mode`, `--output-format`, `--max-turns`, `--allowedTools`, `--disallowedTools`, and subcommands `mcp`, `auth`, `plugin`, `marketplace`, `agents`, `open`.

**REPL slash commands:** `src/commands.ts:COMMANDS()` — returns approximately 80+ in-REPL commands registered at startup: `/compact`, `/clear`, `/help`, `/model`, `/memory`, `/config`, `/mcp`, `/skills`, `/tasks`, `/review`, `/vim`, `/permissions`, `/plan`, `/hooks`, `/agents`, `/plugin`, `/rewind`, `/cost`, `/diff`, `/files`, `/branch`, `/resume`, `/session`, `/export`, `/stats`, `/effort`, `/fast`, `/passes`. Dynamic skill and plugin commands merge via `getCommands()` at line 476.

**SDK/headless entrypoint:** `src/QueryEngine.ts:184` — `QueryEngine` class constructor initializes `mutableMessages[]`, `abortController`, and `readFileState`. `submitMessage()` at line 209 assembles the full system prompt, processes user input, and calls `query()`.

**Agent loop entrypoint:** `src/query.ts:queryLoop()` at line 241 — async generator that drives all LLM calls and tool executions. Called by `query()` at line 219 which wraps the generator.

---

## 5. Happy-Path Surgical Walkthrough

**Scenario:** User runs `claude "fix the bug in auth.ts"` from the terminal.

**Step 1 — Binary reads argv:** `src/entrypoints/cli.tsx:main()` receives `process.argv = ["node", "claude", "fix the bug in auth.ts"]`. No fast-path flags detected. Delegates to `src/main.tsx`.

**Step 2 — Commander parses flags:** `src/main.tsx` (Commander program definition, approximately line 200) parses `"fix the bug in auth.ts"` as the `[prompt]` positional argument. Because a prompt is present without an explicit `-p` flag, the code activates the headless print path. Constructs `headlessStore` via `createStore(headlessInitialState)` at `src/state/store.ts:createStore()`.

**Step 3 — QueryEngine initializes:** `src/QueryEngine.ts:200` — constructor sets `this.mutableMessages = []`, creates `AbortController`, initializes `readFileState`. No API calls yet.

**Step 4 — submitMessage() assembles context:** `src/QueryEngine.ts:209:submitMessage()` calls `fetchSystemPromptParts()` at line 292, which calls:
- `getSystemPrompt()` — returns the base system prompt string
- `getUserContext()` at `src/utils/queryContext.ts` — loads CLAUDE.md hierarchy via `src/utils/claudemd.ts` (managed → user → project → local layers, supporting `@include` directives), appends current date
- `getSystemContext()` — appends git status output

**Step 5 — User input processes:** `src/QueryEngine.ts:416:processUserInput()` at `src/utils/processUserInput/processUserInput.ts:85` receives `"fix the bug in auth.ts"`. `parseSlashCommand()` finds no leading slash. Creates a `UserMessage` with the raw text. `submitMessage()` pushes to `mutableMessages` at line 431, records transcript entry at line 451 via `src/utils/sessionStorage.ts:recordTranscript()`.

**Step 6 — query() delegates to queryLoop():** `src/query.ts:219:query()` calls `queryLoop()` at line 241, passing assembled messages, system prompt, tools list, and dependency injection object (`deps`).

**Step 7 — while(true) loop begins:** `src/query.ts:307` — loop body. Compaction checks run first (all pass on turn 1). Line 659: `for await (const message of deps.callModel({messages, system, tools, ...}))` — `deps.callModel` resolves to `queryModelWithStreaming` in `src/services/api/claude.ts:752`.

**Step 8 — LLM streams response:** `src/services/api/claude.ts:752:queryModelWithStreaming()` constructs an Anthropic API request, calls `client.beta.messages.stream()` via `@anthropic-ai/sdk`. The model responds with a `tool_use` block requesting `FileReadTool` for `auth.ts`. Streaming events yield `AssistantMessage` and `StreamEvent` objects back to the loop. Tool use blocks accumulate in `toolUseBlocks` at lines 830–834.

**Step 9 — Tools dispatch:** After streaming completes, `src/query.ts` lines 1380–1408 call `runTools(toolUseBlocks, ...)` at `src/services/tools/toolOrchestration.ts:runTools()`. `runTools()` partitions blocks: `FileReadTool` is read-only, so it lands in the concurrent batch (up to 10 concurrent). Calls `runToolUse()` for each.

**Step 10 — Permission gate and tool execution:** `src/services/tools/toolExecution.ts:337:runToolUse()` locates `FileReadTool` by name via `findToolByName()`. Validates the input `{path: "auth.ts"}` with Zod schema. Calls `checkPermissionsAndCallTool()` at line 599, which calls `canUseTool()` → `src/utils/permissions/permissions.ts:hasPermissionsToUseTool()`. Read-only file operations in default mode resolve to `allow`. Calls `tool.call(parsedInput, toolUseContext)`.

**Step 11 — Tool result feeds back:** `FileReadTool` reads `auth.ts` contents from disk. Result yields as a `UserMessage` with a `tool_result` content block. Transcript recorded. Loop iteration ends, `continue` back to step 7.

**Step 12 — Second LLM call with file contents:** Loop iteration 2 calls `deps.callModel()` with messages now including the `auth.ts` contents. Model returns `FileEditTool` tool_use block with proposed patch.

**Step 13 — Write permission check interrupts:** `checkPermissionsAndCallTool()` for `FileEditTool` — write operation in default mode. `hasPermissionsToUseTool()` resolves to `ask`. `src/hooks/useCanUseTool.tsx:93` queues a confirmation dialog. User sees Accept/Reject UI. User accepts.

**Step 14 — Edit applies and loop terminates:** `FileEditTool.call()` applies the patch to `auth.ts`. Tool result yields success message. Loop iteration 3: model returns response with zero `tool_use` blocks. `src/query.ts:1062` — stop condition met. `queryLoop()` yields final `AssistantMessage` and returns. Transcript finalized at `~/.claude/projects/<hash>/<sessionId>.jsonl`.

---

## 6. Dispatch Map

**LLM → tool dispatch:** The model emits `tool_use` content blocks with a `name` field. No agent-side routing logic selects tools. `runTools()` at `src/services/tools/toolOrchestration.ts` receives the raw blocks, partitions by read-only classification, and dispatches each via `runToolUse()` at `src/services/tools/toolExecution.ts:337`. `findToolByName()` at `src/Tool.ts` locates the correct tool object by string name match.

**Tool registry:** `src/tools.ts:getAllBaseTools()` — hardcoded list of all tool objects: `AgentTool`, `BashTool`, `FileEditTool`, `FileReadTool`, `FileWriteTool`, `GlobTool`, `GrepTool`, `NotebookEditTool`, `TodoWriteTool`, `SkillTool`, `ToolSearchTool`, and others. Feature-gated tools include conditionally. `getTools()` filters the list by `isEnabled()` returning true, by deny rules, and by tool preset configuration. MCP tools dynamically instantiate as `MCPTool` objects discovered from connected servers.

**Slash command dispatch:** `src/utils/processUserInput/processUserInput.ts:85:processUserInput()` calls `parseSlashCommand()` to detect a leading `/`, then `findCommand()` to locate the matching `Command` object from `COMMANDS()`. Commands with a local `call()` function execute immediately in-process. Commands returning a prompt string forward to the model as a user message.

**Subagent spawning:** `src/tools/AgentTool/runAgent.ts:runAgent()` creates an isolated execution context via `createSubagentContext()` from `src/utils/forkedAgent.ts`. The context clones the parent's file state cache, creates a child `AbortController`, assigns a new `agentId`, and calls `query()` recursively with a fresh empty `messages[]` array. Agent definitions load from `.claude/agents/` markdown files with YAML frontmatter.

**MCP tool loading:** `src/services/mcp/client.ts:connectToServer()` creates transport (stdio, SSE, HTTP, or WebSocket based on config), calls `client.listTools()` to discover available tools, and instantiates `MCPTool` objects for each. MCP prompts become `PromptCommand` skill objects. Resources become available via `ListMcpResourcesTool` and `ReadMcpResourceTool`.

**Skill loading and invocation:** `src/skills/loadSkillsDir.ts` scans `~/.claude/skills/`, `.claude/skills/`, and managed paths. Each `.md` file with valid YAML frontmatter (name, description, whenToUse, tools, model) becomes a `PromptCommand`. Bundled skills ship in `src/skills/bundled/`. When the model calls `SkillTool` at `src/tools/SkillTool/SkillTool.ts`, it calls `getPromptForCommand()` and runs the skill as a forked agent sub-query via `runAgent()`.

**Deferred tool discovery:** `src/tools/ToolSearchTool/ToolSearchTool.ts` — tools absent from the initial tool list become discoverable at runtime. When the model calls `ToolSearch` with a query, it returns full JSON schemas for matching tools, allowing the model to call them in subsequent turns.

---

## 7. State Model

**AppState** (`src/state/AppStateStore.ts:89`, typed as `DeepImmutable`): the primary reactive store for all UI-facing and session-level state. Fields include: `settings` (merged from all config layers), `verbose`, `mainLoopModel`, `statusLineText`, `toolPermissionContext` (permission mode + allow/deny/ask rule sets keyed by source), `mcp` (connected clients, tools, commands, resources), `plugins`, `tasks` (map of `taskId → TaskState`), `agentNameRegistry`, `foregroundedTaskId`, `speculation`, `fileHistory`, `attribution`, `denialTracking`, `fastMode`, `effortValue`, `advisorModel`, `notifications`, `elicitationQueue`, `todoLists`, `bridge`, `remote`, `theme`, and all Ink UI display fields.

**Query loop state** (`src/query.ts:204`): local mutable state inside `queryLoop()` — `messages[]`, `toolUseContext`, `autoCompactTracking`, `maxOutputTokensRecoveryCount`, `hasAttemptedReactiveCompact`, `maxOutputTokensOverride`, `pendingToolUseSummary`, `stopHookActive`, `turnCount`, `transition`. Lives only for the lifetime of one `queryLoop()` call.

**Bootstrap state** (`src/bootstrap/state.ts`): module-level global mutable variables — `originalCwd`, `sessionId`, `totalCostUSD`, `modelUsage`, `mainLoopModelOverride`, `isInteractive`, `registeredHooks`, `invokedSkills`, `agentColorMap`. The comment at line 32 reads "DO NOT ADD MORE STATE HERE," indicating acknowledged technical debt.

**Conversation history**: dual representation — in-memory as `mutableMessages: Message[]` in `QueryEngine` (line 186) and as `state.messages` inside the query loop, plus on-disk as append-only JSONL at `~/.claude/projects/<hash>/<sessionId>.jsonl` written by `src/utils/sessionStorage.ts:recordTranscript()`.

**Tool permission state** (`src/Tool.ts:123` — `ToolPermissionContext`): `mode` (one of `default` / `plan` / `bypassPermissions` / `auto`), `alwaysAllowRules`, `alwaysDenyRules`, `alwaysAskRules` each keyed by source (`localSettings`, `userSettings`, `projectSettings`, `policySettings`, `cliArg`, `command`, `session`), plus `additionalWorkingDirectories`. Stored inside `AppState.toolPermissionContext`.

**Context window assembly** (assembled each loop iteration): `getMessagesAfterCompactBoundary()` → `applyToolResultBudget()` → snip → microcompact → context collapse → autocompact → `prependUserContext(CLAUDE.md content)` → `appendSystemContext(git status)` → full system prompt concatenation.

---

## 8. Data Transformation Chain

The raw user string `"fix the bug in auth.ts"` transforms through the following pipeline:

1. **String → Commander ParsedArgs** (`src/main.tsx`): Commander.js parses argv into `{prompt: "fix the bug in auth.ts", model: undefined, print: false, ...}`.

2. **ParsedArgs → UserMessage** (`src/utils/processUserInput/processUserInput.ts:85`): `processUserInput()` wraps the string in a typed `UserMessage` object with `role: "user"` and `content: [{type: "text", text: "fix the bug in auth.ts"}]`.

3. **UserMessage → API Request** (`src/QueryEngine.ts:209`): `submitMessage()` assembles `{system: [systemPrompt, claudeMdContent, gitStatus], messages: [UserMessage], tools: [FileReadTool schema, FileEditTool schema, ...], model: "claude-opus-4-5", max_tokens: 8192, ...}`.

4. **API Request → Streaming Events** (`src/services/api/claude.ts:752`): `@anthropic-ai/sdk` streams `message_start`, `content_block_start`, `content_block_delta`, `message_delta` events. Assembled into typed `AssistantMessage`.

5. **AssistantMessage → ToolUseBlock[]** (`src/query.ts:830`): loop collects `content` blocks where `type === "tool_use"` into `toolUseBlocks[]`.

6. **ToolUseBlock → Zod-validated input** (`src/services/tools/toolExecution.ts:617`): each block's `input` field passes through the tool's Zod schema. Invalid inputs reject with `InputValidationError` before any filesystem touch.

7. **Validated input → Tool result** (`tool.call(parsedInput, ctx)`): tool executes, returns `{type: "tool_result", tool_use_id, content: string | ContentBlock[]}`.

8. **Tool result → UserMessage** (`src/query.ts`): wrapped as `UserMessage` with `role: "user"` and `content: [tool_result block]`, appended to `messages[]` for next iteration.

9. **Final AssistantMessage → JSONL** (`src/utils/sessionStorage.ts:recordTranscript()`): each message appends as a typed JSONL entry at `~/.claude/projects/<hash>/<sessionId>.jsonl`.

---

## 9. Side-Effect Map

**Filesystem writes:**
- `src/utils/sessionStorage.ts:recordTranscript()` — appends JSONL to `~/.claude/projects/<hash>/<sessionId>.jsonl`
- `FileWriteTool.call()` — creates/overwrites files in the working directory
- `FileEditTool.call()` — applies patches to existing files
- `NotebookEditTool.call()` — mutates Jupyter notebook cells
- `TodoWriteTool.call()` — writes todo state to disk
- `src/utils/config.ts:saveGlobalConfig()` — persists merged config to `~/.claude.json`
- `src/memdir/` memory writes — creates/updates `~/.claude/memory/MEMORY.md` and topic files
- `src/utils/fileHistory.ts` — writes undo snapshots for reversibility

**LLM API calls** (`src/services/api/claude.ts:752`): primary model call per loop iteration, plus additional calls for: compaction summaries (autocompact forked agent), memory extraction (`src/services/extractMemories/`), agent summaries (`src/services/AgentSummary/`), prompt suggestions (`src/services/PromptSuggestion/`), tool use summaries, side questions via `SideQuestionTool`, auto-mode permission classifier.

**Subprocess spawns:**
- `BashTool.call()` — `child_process.spawn()` with macOS sandbox-exec wrapper (`src/utils/sandbox/sandbox-adapter.ts:SandboxManager`)
- `PowerShellTool.call()` — Windows subprocess equivalent
- `LocalShellTask` — long-running background task processes
- MCP stdio servers — spawned via `StdioClientTransport` when MCP config references executable paths

**MCP network calls** (`src/services/mcp/client.ts`): `client.callTool()`, `client.listTools()`, `client.listPrompts()`, `client.listResources()`, `client.readResource()` over stdio, SSE, HTTP, or WebSocket transports.

**Persistence path inventory:**
- `~/.claude/projects/<hash>/<sessionId>.jsonl` — session transcript
- `~/.claude/projects/<hash>/<sessionId>/agents/<agentId>.jsonl` — subagent transcripts
- `~/.claude.json` — global config (model, permissions, API key)
- `.claude/settings.json` — project-level settings
- `~/.claude/settings.json` — user-level settings
- `~/.claude/memory/MEMORY.md` — persistent memory index
- `.claude/memory/` — project-scoped memory files
- `~/.claude/sessions/` — session metadata index

---

## 10. Important Branch Points

**Permission mode branch** (`src/utils/permissions/permissions.ts:hasPermissionsToUseTool()`): evaluates deny rules first across all sources (if any deny rule matches → `deny`), then allow rules, then falls through to mode-based default. Modes: `default` (dangerous ops → ask), `plan` (all writes → deny), `bypassPermissions` (everything → allow), `auto` (LLM classifier decides). Rule source priority enforced as: `policySettings > projectSettings > localSettings > userSettings > cliArg > command > session`.

**Interactive vs. headless branch** (`src/main.tsx`): presence of `--print`/`-p` flag or a positional prompt argument without a TTY routes to `QueryEngine` headless path. Interactive terminal with no initial prompt routes to `src/replLauncher.tsx` Ink/React REPL.

**Tool batching branch** (`src/services/tools/toolOrchestration.ts:runTools()`): partitions `toolUseBlocks[]` into read-only (concurrent, up to 10 simultaneous) versus non-read-only (serial, one at a time). Tool's `isReadOnly()` method determines partition.

**Context window overflow branch** (`src/query.ts:628`): before each API call, measures assembled messages token count against model limit. Triggers compaction stack in order: Snip → Microcompact → Context Collapse → Autocompact → Reactive Compact (emergency on 413 response). Each mechanism fires only if the previous was insufficient.

**Subagent vs. inline execution branch** (`src/tools/AgentTool/runAgent.ts`): `AgentTool` spawns isolated `query()` recursion with cloned context. `SkillTool` similarly forks via `runAgent()`. All other tools execute inline within the parent loop.

**Fallback model branch** (`src/query.ts:894`): `FallbackTriggeredError` switches `mainLoopModel` to the configured fallback and retries the current iteration without losing message history.

**MCP tool discovery branch** (`src/services/mcp/client.ts`): at session startup, each configured MCP server connects and discovers tools. Discovered tools merge into the active tool set for the session. Failure to connect one server does not block others.

---

## 11. Failure and Recovery Logic

**Permission denial:** `src/services/tools/toolExecution.ts:599:checkPermissionsAndCallTool()` gates every tool call. Denial yields a `tool_result` with `is_error: true` and a human-readable message explaining the permission refusal. The model receives this result and may ask the user to change permissions or suggest an alternative approach.

**Input validation failure:** `src/services/tools/toolExecution.ts:617` — Zod parse of `tool_use.input` fails → `InputValidationError` thrown → caught at line 469 → tool_result with `is_error: true` returned to model. The model re-attempts with corrected input.

**Runtime tool error:** `src/services/tools/toolExecution.ts:469` — any exception from `tool.call()` is caught, wrapped as `is_error: true` tool_result, and fed back to the model. Post-failure hooks run via `runPostToolUseFailureHooks()`. The model retries or abandons the operation.

**LLM API rate limiting:** `src/services/api/withRetry.ts` — exponential backoff with jitter on 429 responses. Configurable retry count and delay multiplier.

**Fallback model:** `src/query.ts:894–951` — `FallbackTriggeredError` caught in the loop body. Switches `mainLoopModel` to `fallbackModel` from config, reconstructs the API call, retries the current turn.

**Max output tokens recovery:** `src/query.ts:1188–1256` — on `max_tokens` stop reason, up to 3 retries by appending a continuation meta-message. Counter tracked in `maxOutputTokensRecoveryCount`.

**Context window overflow stack** (five mechanisms, each triggered at progressively higher severity):
1. **Snip** — removes oldest non-essential messages from history
2. **Microcompact** — truncates oversized individual tool results, replaces content with file references (`src/utils/sessionStorage.ts` persists replacement records for `--resume` reconstruction)
3. **Context Collapse** — collapses entire earlier turns into a summary block
4. **Autocompact** — forks a new agent that summarizes the entire conversation, replaces messages with summary, records "context collapse commit" in JSONL
5. **Reactive Compact** — emergency autocompact triggered by receiving a 413 `prompt_too_long` error from the API (`src/query.ts:1062–1183`, `hasAttemptedReactiveCompact` flag prevents infinite retry)

**Human-in-the-loop for uncertain permissions** (`src/hooks/useCanUseTool.tsx:93–167`): when permission resolves to `ask`, the coordinator routes through swarm worker → speculative classifier check → `handleInteractivePermission()`. This queues a confirmation dialog in the Ink/React UI via `setToolUseConfirmQueue`. User sees Accept / Reject / Always Allow options. Per-tool dialog components live in `src/components/permissions/`.

---

## 12. The Real Core Loop

The load-bearing core consists of exactly seven interdependent pieces. Remove any one and the system stops functioning:

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

**The seven load-bearing files:**
1. `src/query.ts:queryLoop()` — iteration engine, compaction orchestration, stop detection
2. `src/services/api/claude.ts:queryModelWithStreaming()` — LLM call, streaming assembly, provider routing (first-party / Bedrock / Vertex)
3. `src/services/tools/toolExecution.ts:checkPermissionsAndCallTool()` — tool dispatch + permission gate
4. `src/Tool.ts` — tool type system, `findToolByName()`, `ToolPermissionContext` definition
5. `src/tools.ts:getAllBaseTools()` — tool registry, `getTools()` filtering
6. `src/state/store.ts:createStore()` — reactive state container (34 lines, pure)
7. `src/utils/permissions/permissions.ts:hasPermissionsToUseTool()` — permission evaluation with source-priority ordering

Everything else — skills, MCP, subagents, memory, compaction, UI — layers on top of these seven. Reel Creator needs equivalent implementations of all seven, with different tools and a different LLM configuration.

---

## 13. Architectural Compression

**The permission model** operates in four modes (`default` / `plan` / `bypassPermissions` / `auto`) with deny rules evaluated before allow rules across seven source layers. Denial is conservative: any deny rule from any source blocks. Allow requires explicit permission from the highest-priority source. The `auto` mode adds an LLM classifier that predicts whether the current tool call matches the session's original intent, using denial tracking to build a prior.

**The memory architecture** operates at four independent time horizons:
- **CLAUDE.md** (`src/utils/claudemd.ts`): static instructions injected at session start, 4-layer hierarchy (managed → user → project → local), supports `@include` directives. Survives process restarts unchanged.
- **MEMORY.md / topic files** (`src/memdir/`): model-writable persistent memory, index file pointing to topic files. Survives sessions. Model writes via standard `FileWriteTool`.
- **Session transcript** (`src/utils/sessionStorage.ts`): append-only JSONL, resumable via `--resume`. Records tool results, file history snapshots, context collapse commits, content replacement records.
- **Invoked skills cache** (`src/bootstrap/state.ts:invokedSkills`): tracks skills called in the current session, survives compaction so the model doesn't re-invoke already-used skills.

**The skill system** uses markdown files with YAML frontmatter as the skill definition format. Discovery runs across five locations in priority order. Skills compose with subagents: `SkillTool` invokes a skill by forking a new `query()` call with the skill's prompt as system context. This means skills are first-class agents, not just prompt templates.

**The most sophisticated mechanisms** (in order of complexity):
1. **Five-level compaction stack** — Snip → Microcompact → Context Collapse → Autocompact → Reactive Compact. Each fires at a different threshold. Together they allow unbounded session length within finite context windows.
2. **Streaming tool executor** (`src/services/tools/StreamingToolExecutor.ts`) — begins executing read-only tools as tool_use blocks stream in, before the full model response completes. Reduces latency on multi-tool responses.
3. **Permission classifier** — LLM call in `auto` mode to predict approve/deny, with confidence levels and denial tracking as prior.
4. **Prompt cache sharing** (`src/utils/forkedAgent.ts:CacheSafeParams`) — subagent queries reuse the parent's API-side prompt cache by sending identical system prompt prefixes. Reduces cost for compaction summaries and memory extraction.
5. **Skill prefetch** — asynchronously discovers relevant skills while the model streams, using a signal/settle pattern to surface results before the current turn completes.
6. **Tool result budget management** — tracks per-message size, replaces oversized tool results with file references in the JSONL, persists replacement records so `--resume` can reconstruct the full content.

**Clean and well-isolated files** (safe to read and adapt independently):
- `src/state/store.ts` — 34 lines, pure reactive store with no external dependencies
- `src/services/tools/toolOrchestration.ts` — 188 lines, clear read-only/serial batching logic
- `src/query/deps.ts` — 40 lines, clean dependency injection interface for the query loop
- `src/utils/forkedAgent.ts` — well-documented cache-sharing semantics with explicit comments
- `src/skills/loadSkillsDir.ts` — clean frontmatter parsing and multi-path discovery
- `src/Task.ts` — clean type definitions with no side effects

**Tangled files** (read with caution, do not copy wholesale):
- `src/main.tsx` — 4,683 lines: CLI parsing + trust dialogs + REPL launch + headless execution + MCP init + plugin init + migrations + telemetry. Everything touches everything.
- `src/query.ts` — 1,729 lines: deeply nested recovery logic inside the `while(true)` body; compaction, fallback, streaming, and tool dispatch all interleaved.
- `src/bootstrap/state.ts` — global mutable singleton with 200+ fields; acknowledged as anti-pattern in its own comment.
- `src/Tool.ts` — 792 lines mixing tool definitions, permission context type, progress callback types, and UI interaction hooks.

---

## 14. Design Intent Inference

**The loop-as-generator pattern** (`async function* queryLoop()`) signals that the architects optimized for streaming UI updates: each `yield` pushes a message fragment to the React/Ink UI without blocking the loop. The generator pattern also makes the loop pausable — the `--resume` flag reconstructs the message array from JSONL and re-enters a new generator at the correct turn.

**The dependency injection object** (`src/query/deps.ts`) wrapping `callModel`, `runTools`, and `recordTranscript` signals that the loop was designed for testability and for swapping the LLM backend. Reel Creator should adopt the same pattern: `deps.callModel` becomes the swappable interface whether calling Claude, GPT-4o, or a local model.

**The YAML frontmatter skill format** signals that the skill system was designed for non-engineer authorship. Skills are documentation-first: the `whenToUse` field is human-readable prose that informs both the model's tool selection and the skill prefetch system. Reel Creator's workflow definitions should adopt the same pattern — markdown-first, with structured frontmatter for machine consumption.

**The four-source permission hierarchy** (`policySettings > projectSettings > localSettings > userSettings > cliArg > command > session`) signals intent to operate in enterprise environments where IT policy overrides user preferences. Reel Creator's equivalent is platform policy overriding creator preferences (e.g., a brand account cannot override the brand's content policy).

**The `DeepImmutable<AppState>` type** on the reactive store signals that UI components were forbidden from mutating state directly. All mutations route through the store's dispatch mechanism. This prevents the class of bugs where a React component side-effects the agent loop state. Reel Creator must enforce the same boundary.

**The "DO NOT ADD MORE STATE HERE" comment** at `src/bootstrap/state.ts:32` is an architectural confession: the bootstrap singleton grew organically and became the dumping ground for cross-cutting concerns. The intent was always to keep it minimal. Reel Creator should pre-enforce this by defining the bootstrap singleton's type contract before writing any code.

**The five-level compaction stack** signals that context window management was not designed upfront — it grew incrementally as users encountered longer sessions. Each mechanism was added when the previous proved insufficient. Reel Creator should treat context management as a first-class concern from day one rather than retrofitting.

---

## 15. Safe Surgery Map

The following changes can be made to the codebase without risk of cascading failures, listed from safest to least safe:

**Safe to add** (no existing code paths break):
- New tool class implementing the `Tool` interface (`src/Tool.ts`) registered in `getAllBaseTools()` at `src/tools.ts`
- New skill `.md` file in `.claude/skills/` — picked up by `loadSkillsDir.ts` automatically
- New slash command in `COMMANDS()` at `src/commands.ts`
- New MCP server in `.claude/mcp.json` — loaded at session start via `src/services/mcp/client.ts`
- New permission rule source — extend the source priority list at `src/utils/permissions/permissions.ts`

**Safe to replace** (well-isolated, single caller):
- `src/state/store.ts:createStore()` — 34 lines, pure function, single clear interface
- `src/services/tools/toolOrchestration.ts:runTools()` — 188 lines, single caller in query loop, well-defined input/output types
- `src/query/deps.ts` — 40 lines, dependency injection interface; swap the `callModel` implementation to point at a different LLM provider

**Safe to extend** (additive only):
- `src/utils/claudemd.ts` — add a new hierarchy layer without breaking existing four layers
- `src/utils/sessionStorage.ts` — add new JSONL entry types (existing readers ignore unknown types)
- `src/bootstrap/state.ts` — add fields only if absolutely cross-cutting; prefer AppState instead

**Risky to touch** (high coupling):
- `src/query.ts` — modifying the `while(true)` body risks breaking one of the five compaction mechanisms, fallback model switching, or streaming coordination
- `src/main.tsx` — the god file; changes to initialization order break MCP setup, plugin loading, or trust dialog sequencing
- `src/services/api/claude.ts:queryModelWithStreaming()` — streaming assembly is tightly coupled to the SDK's event shapes; changes break the tool_use block accumulation logic
- `src/utils/permissions/permissions.ts` — permission evaluation order is security-critical; wrong priority ordering creates privilege escalation

---

## 16. Learning-Value Map

The following mechanisms in Claude Code carry the highest learning value for a Reel Creator builder, independent of whether the code is reusable:

**Study deeply:**

1. **The `queryLoop()` / `deps.callModel()` / `runTools()` triangle** (`src/query.ts:307`, `src/query/deps.ts`, `src/services/tools/toolOrchestration.ts`): reveals how to structure the core agentic loop so it remains testable despite being an infinite generator. The dependency injection pattern specifically teaches how to swap the LLM backend without touching loop logic. Directly applicable to Reel Creator's content generation loop.

2. **CLAUDE.md 4-layer hierarchy** (`src/utils/claudemd.ts`): the layering of managed → user → project → local with `@include` directive support teaches how to build a flexible instruction injection system that works for both individual creators and enterprise brand accounts. The "managed" layer (enterprise IT policy) maps directly to Reel Creator's "brand content policy" layer.

3. **Skill frontmatter format** (`src/skills/loadSkillsDir.ts`): the YAML frontmatter fields (`name`, `description`, `whenToUse`, `tools`, `model`) teach how to structure workflow definitions that are both human-readable and machine-consumable. Reel Creator's "content workflow" definitions should derive from this schema.

4. **Permission mode system** (`src/utils/permissions/permissions.ts:hasPermissionsToUseTool()`): the four-mode design (default / plan / bypassPermissions / auto) teaches how to expose a graduated safety model to users. The deny-before-allow evaluation order teaches a conservative default that is correct for content publishing (do not accidentally publish).

5. **Reactive store pattern** (`src/state/store.ts`): 34 lines that show a minimal, framework-agnostic reactive container. Study to understand the `DeepImmutable` contract enforcement pattern.

6. **Subagent context isolation** (`src/utils/forkedAgent.ts`): teaches how to run parallel agent workstreams (e.g., simultaneous caption generation + thumbnail creation) without cross-contaminating message histories or file state caches.

7. **Tool result budget + content replacement records** (`src/utils/sessionStorage.ts`): teaches how to handle the case where a generated asset (image, video) is too large to keep inline in the message history. The pattern of replacing content with file references and persisting replacement records for reconstruction is directly applicable to Reel Creator handling generated media.

8. **Streaming tool executor** (`src/services/tools/StreamingToolExecutor.ts`): teaches how to begin executing tools as they stream in, reducing latency on parallel tool calls. Applicable to Reel Creator's parallel asset generation pipelines.

**Study for future phases:**

9. **Five-level compaction stack** (`src/query.ts`): premature for Phase 1 but teaches what failure modes emerge at scale. Study the order and triggering conditions to inform Reel Creator's token budget design from the start.

10. **Prompt cache sharing via `CacheSafeParams`** (`src/utils/forkedAgent.ts`): cost optimization pattern for subagent queries. Study after Reel Creator reaches production traffic.

---

## 17. Reuse / Copy-Value Map

### A. COPY-PASTE (use directly, license assumed OK)

These components are small, isolated, correct, and applicable to Reel Creator with minimal or zero adaptation:

**1. Reactive store — `src/state/store.ts:createStore()` (34 lines)**
A pure function that wraps a value in a reactive container with `getState()` and `setState()`. Zero dependencies on Claude Code specifics. Reel Creator needs exactly this pattern to separate agent loop state from UI state. Copy the 34-line implementation, rename types, done.

**2. Tool orchestration batch logic — `src/services/tools/toolOrchestration.ts:runTools()` (188 lines)**
Partitions tool calls into read-only concurrent (up to N) and non-read-only serial batches. The partitioning logic is tool-agnostic — it calls `tool.isReadOnly()` and dispatches via a passed-in `runToolUse` function. Reel Creator needs identical logic: asset read tools (fetch image, read caption draft) run concurrently; publish tools (post to TikTok, write to database) run serially. Copy the batching logic, swap the tool implementations.

**3. Dependency injection interface — `src/query/deps.ts` (40 lines)**
Defines the `QueryDeps` interface that decouples the agent loop from its external dependencies (`callModel`, `runTools`, `recordTranscript`). Copy this interface pattern. In Reel Creator, `callModel` becomes the swappable LLM interface, `runTools` becomes the content action dispatcher, and `recordTranscript` becomes the production session recorder.

**4. JSONL transcript schema and `recordTranscript()` — `src/utils/sessionStorage.ts` (key typed entry definitions)**
The append-only JSONL with typed entries (`transcript`, `fileHistorySnapshot`, `attribution`, `contentReplacement`, `contextCollapseCommit`) solves the exact problem Reel Creator faces: persisting long content creation sessions with large generated assets. Copy the JSONL pattern and the typed entry discriminated union. Replace file-centric entries with media-centric equivalents (`generatedAsset`, `publishRecord`, `revisionHistory`).

**5. Frontmatter parsing for skill definitions — `src/skills/loadSkillsDir.ts` (frontmatter parse + PromptCommand assembly)**
The markdown + YAML frontmatter skill discovery pattern (scan directory → parse frontmatter → assemble PromptCommand) is directly applicable to Reel Creator's "content workflow" definitions. A "TikTok viral short" workflow is structurally identical to a Claude Code skill. Copy the discovery and parsing logic; replace the skill execution backend with Reel Creator's content execution engine.

**6. Zod validation at tool input boundary — `src/services/tools/toolExecution.ts:617` (pattern)**
The 3-line pattern of `tool.inputSchema.parse(rawInput)` before any tool execution, with `InputValidationError` on failure, prevents malformed LLM outputs from reaching side-effectful code. This pattern copies identically into Reel Creator's tool dispatch layer. Each content action tool validates its input before touching any external API.

**7. Task type definitions — `src/Task.ts`**
Clean TypeScript type definitions for task state, progress, and status. Zero Claude Code specifics. Copy as the type foundation for Reel Creator's content task system. Rename `Task` to `ContentTask`, adjust status enum values to match content creation states (`drafting`, `generating`, `reviewing`, `publishing`, `published`).

---

### B. USE-THE-ESSENCE (adapt the pattern, not the code)

These designs are architecturally applicable but require rebuilding for Reel Creator's domain:

**1. The agent loop — `src/query.ts:queryLoop()` → Reel Creator: `contentLoop()`**
*Pattern to extract:* `while(true)` async generator that calls LLM, collects action blocks, dispatches actions, feeds results back, stops when no actions remain.
*Why not portable:* The 1,729-line implementation is deeply entangled with Claude Code's compaction stack, file tool semantics, code-specific context assembly (git status, CLAUDE.md), and code-reviewer UI components. The core loop logic is correct; the surrounding machinery is wrong for content creation.
*What to rebuild:* A `contentLoop()` async generator with the same while/call/dispatch/result/stop structure, but with content-context assembly (platform specs, brand guidelines, trending audio), content-specific compaction (summarize generation history rather than code diffs), and content-specific stop conditions (asset approved, post scheduled).

**2. CLAUDE.md 4-layer context injection — `src/utils/claudemd.ts` → Reel Creator: Brand Context System**
*Pattern to extract:* Layered instruction injection at session start — managed (platform) > user (creator preferences) > project (campaign) > local (session overrides). Supports `@include` to compose instruction fragments.
*Why not portable:* CLAUDE.md is literally named for Claude and structured around coding workflows (project architecture, coding style). The `@include` syntax and layer discovery paths are hardcoded to `.claude/` directories.
*What to rebuild:* A `BrandContext` loader with layers: platform-policy (Reel Creator's TOS-enforced limits) > brand-guide (from brand's `.reel/brand.md`) > campaign-brief (from `.reel/campaign.md`) > session-override (creator's current session notes). Same `@include` directive syntax, different discovery paths.

**3. Permission gate system — `src/utils/permissions/permissions.ts` → Reel Creator: Publishing Gate**
*Pattern to extract:* Deny-before-allow rule evaluation with source-priority ordering. Four modes graduated from conservative to permissive. Per-action allow/deny/ask rules stored by source.
*Why not portable:* Rules are keyed to filesystem paths and bash command patterns. The Zod schemas and rule matching logic are specific to file operations and shell commands.
*What to rebuild:* A `PublishingGate` with the same four-mode structure (`draft` / `review` / `scheduled` / `live`) and the same deny-before-allow evaluation. Rules keyed to platform actions (`post_to_tiktok`, `publish_instagram_reel`, `send_to_brand_review`) rather than file paths. The `policySettings > brandSettings > creatorSettings > sessionOverride` priority order mirrors Claude Code's source hierarchy exactly.

**4. Subagent spawning with isolated context — `src/utils/forkedAgent.ts` → Reel Creator: Parallel Asset Agent**
*Pattern to extract:* Fork a new agent with cloned immutable state, independent abort controller, separate message history, and the ability to report results back to the parent.
*Why not portable:* `createSubagentContext()` clones a `fileStateCache` (read-cache of local files) and inherits a code-centric working directory model. Reel Creator's "asset state" is remote (S3/CDN URLs, platform draft IDs) not local file paths.
*What to rebuild:* A `forkContentAgent()` function that clones the current campaign context (brand guidelines, platform specs, current draft URLs), creates an isolated `AbortController`, and runs a sub-`contentLoop()`. Used for parallelizing: caption generation while thumbnail is being created, A/B variant generation, compliance review while main content generates.

**5. Memory architecture — `src/memdir/` + `src/utils/claudemd.ts` → Reel Creator: Creator Memory**
*Pattern to extract:* Two-tier memory — static instructions (CLAUDE.md equivalent) injected at session start, plus dynamic model-writable memory (MEMORY.md equivalent) that persists across sessions. Topic-file index pattern for organized persistent memory.
*Why not portable:* Memory files live in `.claude/` directories with code-project semantics. Memory extraction prompts are tuned for extracting coding preferences and project knowledge.
*What to rebuild:* A `CreatorMemory` system with `.reel/brand-memory.md` (brand-persistent facts: tone of voice, recurring hashtags, platform performance patterns) and `.reel/session-memory.md` (session-persistent context: current campaign, audience notes). Memory extraction prompts tuned for content creation preferences.

**6. Hook system — `src/hooks/` (pre/post tool execution hooks) → Reel Creator: Content Event Hooks**
*Pattern to extract:* Register functions that fire before/after specific tool calls. Pre-hooks can abort the action. Post-hooks receive the result for side effects (logging, notification, analytics).
*Why not portable:* Hooks are registered against filesystem and shell tool names (`FileWriteTool`, `BashTool`). React/Ink hook architecture is specific to the terminal UI.
*What to rebuild:* A `ContentHookRegistry` that fires before/after content actions (`BeforePublish`, `AfterGenerate`, `OnBrandViolation`). Pre-publish hook checks compliance. Post-generate hook triggers thumbnail generation. OnBrandViolation hook queues human review.

---

### C. JUST-FOR-LEARN (study, don't apply in Phase 1)

These mechanisms are educationally valuable but premature for Reel Creator's Phase 1 deadline of April 15:

**1. Five-level compaction stack (`src/query.ts` — Snip → Microcompact → Context Collapse → Autocompact → Reactive Compact)**
*What to learn:* Context windows overflow at scale. The five mechanisms represent five independently discovered failure modes, each requiring a different mitigation. The Autocompact mechanism (fork an agent to summarize the full conversation, replace messages with the summary) is the most transferable insight: treat context summarization as a first-class agentic task, not a string truncation.
*Why not Phase 1:* Reel Creator's sessions will be short (one campaign, one post) in Phase 1. Context overflow will not manifest until Phase 2+ when multi-session campaigns and cross-platform workflows are introduced. Implementing all five mechanisms before observing the failure would be premature optimization.

**2. Streaming tool executor — `src/services/tools/StreamingToolExecutor.ts`**
*What to learn:* Begin executing tools as they stream from the model's response, not after the full response completes. The signal/settle pattern for coordinating partially-received tool_use blocks with execution pipelines.
*Why not Phase 1:* Phase 1 Reel Creator content actions (generate caption, create thumbnail, schedule post) are latency-tolerant and sequential. The complexity of streaming execution is not justified until parallel high-throughput asset pipelines are needed in Phase 2.

**3. Prompt cache sharing via CacheSafeParams — `src/utils/forkedAgent.ts`**
*What to learn:* Subagent queries that share identical system prompt prefixes with the parent reuse the API provider's cached KV-store, reducing both latency and token cost for compaction summaries, memory extraction, and agent summaries.
*Why not Phase 1:* Reel Creator's Phase 1 will not have enough traffic to make prompt cache optimization a meaningful concern. Implement after traffic data reveals the cost profile of subagent calls.

**4. Permission classifier (LLM-based auto-mode) — `src/services/api/claude.ts` auto-mode calls**
*What to learn:* The classifier calls a fast model with the current tool call and the session's original intent, asking: "does this action match what the user started this session to do?" Denial tracking builds a prior that makes the classifier more accurate over time within a session.
*Why not Phase 1:* Phase 1 Reel Creator will operate in explicit `review` mode (all publish actions require confirmation). The auto-classifier only becomes valuable in Phase 3 when Reel Creator runs scheduled autonomous posting campaigns without a human online.

**5. ToolSearchTool deferred tool loading — `src/tools/ToolSearchTool/ToolSearchTool.ts`**
*What to learn:* Not all tools need to be in the initial context window. A search tool that returns full JSON schemas allows the model to discover and use tools not in its initial tool set. This keeps the initial prompt shorter for common tasks.
*Why not Phase 1:* Phase 1 Reel Creator will have fewer than 20 tools. Deferred loading only pays off when the tool set exceeds the context budget — likely Phase 3+ when Reel Creator supports 50+ platform integrations.

**6. Skill prefetch with signal/settle — `src/skills/loadSkillsDir.ts` + async discovery**
*What to learn:* While the model streams its response, asynchronously discover which skills are relevant to the current turn using keyword matching on the partial response text. Surface results before the model finishes, so relevant skills are available immediately.
*Why not Phase 1:* Phase 1 Reel Creator will have a small, known workflow library. The complexity of async prefetch is not justified until the workflow library grows to 30+ workflows that require intelligent discovery.

**7. Multi-provider API routing — `src/services/api/claude.ts` (Bedrock / Vertex / first-party branch logic)**
*What to learn:* The model call layer abstracts over three API surfaces (Anthropic first-party, AWS Bedrock, Google Vertex) behind a single `queryModelWithStreaming()` interface. Authentication, endpoint URLs, and request shapes differ per provider.
*Why not Phase 1:* Reel Creator Phase 1 targets a single LLM provider. Enterprise multi-cloud routing becomes relevant when serving brand accounts that require data residency in specific cloud regions.

---

## 18. Mental Model Packet

- `queryLoop()` iterates: call LLM, collect tool_use blocks, execute tools, feed results back, stop when zero blocks remain.
- Every tool call passes through `hasPermissionsToUseTool()` before executing; deny rules evaluate before allow rules across seven source layers.
- The reactive store (`createStore()`) enforces a hard boundary: agent loop mutates state, UI only reads it.
- CLAUDE.md injects four-layer static context at session start; MEMORY.md provides model-writable persistent cross-session memory.
- Skills are markdown files with YAML frontmatter that the model invokes like tools; each skill spawns an isolated forked `query()` call.
- Session state lives in three simultaneous places: in-memory `mutableMessages[]`, reactive `AppState`, and append-only JSONL on disk.
- The five-level compaction stack exists because context overflow manifests gradually; each level fires at a different threshold to avoid unnecessary summarization.
- `src/main.tsx` is a 4,683-line god file; everything else is relatively clean and surgically replaceable.
