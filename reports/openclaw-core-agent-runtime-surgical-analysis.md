[Back to Index](../README.md) · [中文版](openclaw-core-agent-runtime-surgical-analysis.cn.md) | **English**

**Table of Contents:** [0](#0-scope) · [1](#1-one-sentence-machine-definition) · [2](#2-why-this-part-exists) · [3](#3-top-level-control-picture) · [4](#4-concrete-entrypoints) · [5](#5-happy-path-surgical-walkthrough) · [6](#6-dispatch-map) · [7](#7-state-model) · [8](#8-data-transformation-chain) · [9](#9-side-effect-map) · [10](#10-important-branch-points) · [11](#11-failure-and-recovery-logic) · [12](#12-the-real-core-loop) · [13](#13-architectural-compression) · [14](#14-design-intent-inference) · [15](#15-safe-surgery-map) · [16](#16-learning-value-map) · [17](#17-reusecopy-value-map) · [18](#18-mental-model-packet)

---

# OpenClaw Core Agent Runtime — Code Surgical Analysis Report

**Analyzed:** `repos/openclaw` core agent runtime relevant to an AgentOS-style content-creation shell
**Date:** 2026-04-01
**Analyst question:** Which OpenClaw mechanisms should an AgentOS team copy directly, internalize deeply, or study later when building a local-first then SaaS agent shell?

---

## 0. Scope

**What is analyzed:** the OpenClaw execution spine from CLI/gateway ingress into one agent turn, centered on `src/entry.ts`, `src/cli/run-main.ts`, `src/cli/program/register.agent.ts`, `src/commands/agent-via-gateway.ts`, `src/agents/agent-command.ts`, `src/agents/command/attempt-execution.ts`, `src/agents/pi-embedded-runner/run.ts`, `src/agents/pi-embedded-runner/run/attempt.ts`, `src/context-engine/*`, `src/agents/pi-tools.ts`, `src/agents/openclaw-tools.ts`, `src/agents/skills/*`, `src/plugins/loader.ts`, and the cron isolated-agent path in `src/cron/isolated-agent/run.ts`.

**Question this report answers:** how OpenClaw actually executes one agent turn, where its real control center sits, and which pieces are high-value for a new AgentOS shell versus too product-specific to copy early.

**Intentionally excluded:** browser extension internals, macOS app UI, most media pipelines, provider-specific plugin breadth, and channel-specific implementation details beyond the dispatch surfaces they expose. Those parts enlarge product breadth, but they do not define the core agent machine your team lead described.

---

## 1. One-Sentence Machine Definition

> This subsystem is fundamentally a sessioned agent-turn machine that turns a routed prompt plus workspace/session state into a persisted, tool-augmented reply under constraints of sandbox policy, model/auth failover, plugin-loaded capability surfaces, and transcript consistency.

---

## 2. Why This Part Exists

**Problem it solves:** a modern agent shell cannot just send text to a model; it must resolve who is speaking, which agent/workspace/session owns the turn, which tools and skills are legal, how transcript state stays consistent, how retries/failover/compaction recover, and how the same core runtime survives CLI, gateway, cron, and channel delivery.

**Why the logic lives here:** OpenClaw keeps product transport thin and pushes durable execution semantics into the agent runtime. `src/entry.ts:runMainOrRootHelp`, `src/commands/agent-via-gateway.ts:agentCliCommand`, and `src/gateway/server-methods/agent.ts:dispatchAgentRunFromGateway` mostly route; `src/agents/agent-command.ts:agentCommandInternal` and `src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent` decide how the machine actually runs.

**What boundary it owns:** this subsystem owns the unit of work “one agent turn in one session.” It resolves session/workspace/model/tool/context state before the turn, drives the live model/tool loop during the turn, and persists or repairs state after the turn.

---

## 3. Top-Level Control Picture

```text
[CLI / Gateway / Cron ingress]
  -> [agent session + model resolution]
  -> [runAgentAttempt dispatch]
       -> [CLI backend runner] -----------------> [child CLI process result]
       -> [embedded runner]
            -> [plugins + skills + sandbox + tools]
            -> [SessionManager + context engine]
            -> [live agent session prompt/tool loop]
            -> [afterTurn / compaction / persistence]
  -> [payload normalization + optional delivery]
```

**Reading this diagram:** left-to-right follows one turn. Each arrow means “passes the current turn state into the next reducer/dispatcher.” The important split is not CLI versus gateway; it is CLI-backend execution versus embedded execution.

---

## 4. Concrete Entrypoints

### Packaged CLI bootstrap

- **File / function / class:** `repos/openclaw/src/entry.ts:runMainOrRootHelp`
- **Input type:** process argv plus environment
- **First meaningful downstream hop:** `repos/openclaw/src/cli/run-main.ts:runCli`
- **Sync / async:** async bootstrap

### `openclaw agent ...` command

- **File / function / class:** `repos/openclaw/src/cli/program/register.agent.ts:registerAgentCommands`
- **Input type:** Commander-parsed CLI options
- **First meaningful downstream hop:** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **Sync / async:** async request/response

### Trusted local agent run

- **File / function / class:** `repos/openclaw/src/agents/agent-command.ts:agentCommand`
- **Input type:** `AgentCommandOpts` from CLI or internal caller
- **First meaningful downstream hop:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **Sync / async:** async request/response

### Network-facing agent run

- **File / function / class:** `repos/openclaw/src/agents/agent-command.ts:agentCommandFromIngress`
- **Input type:** ingress options with explicit trust flags like `senderIsOwner` and `allowModelOverride`
- **First meaningful downstream hop:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **Sync / async:** async request/response

### Gateway RPC `agent`

- **File / function / class:** `repos/openclaw/src/gateway/server-methods/agent.ts:dispatchAgentRunFromGateway`
- **Input type:** gateway RPC request plus request-scoped client identity/capabilities
- **First meaningful downstream hop:** `repos/openclaw/src/agents/agent-command.ts:agentCommandFromIngress`
- **Sync / async:** async request/response with background completion callback

### Scheduled isolated run

- **File / function / class:** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **Input type:** cron job payload plus resolved session key
- **First meaningful downstream hop:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent` or `repos/openclaw/src/agents/cli-runner.ts:runCliAgent`
- **Sync / async:** async scheduled job

---

## 5. Happy-Path Surgical Walkthrough

**Chosen scenario:** a user runs `openclaw agent --local --agent ops --message "Summarize logs"` and OpenClaw completes one embedded turn with tools enabled.

**Why this scenario:** it exposes the real AgentOS-value path: local shell ingress, explicit session/workspace/model resolution, tool/skill/context assembly, live tool loop, and transcript persistence without the extra noise of gateway delivery plumbing.

### Step 1 — Parse the `agent` command

- **Location:** `repos/openclaw/src/cli/program/register.agent.ts:registerAgentCommands`
- **What enters:** Commander options like `message`, `agent`, `thinking`, `local`, `deliver`
- **Assumptions already true:** Node/runtime bootstrap already normalized env and argv
- **What this step does:** dispatches the CLI subcommand into `agentCliCommand`
- **What leaves:** a normalized `opts` object
- **Why this step matters:** this is where OpenClaw decides whether the same user command routes to gateway mode or embedded mode

### Step 2 — Select gateway-or-local execution

- **Location:** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **What enters:** parsed CLI opts and runtime logger
- **Assumptions already true:** command-level validation already ensured a message exists
- **What this step does:** selects embedded execution immediately when `--local` is true, otherwise tries gateway first and falls back to embedded
- **What leaves:** a call into `agentCommand`
- **Why this step matters:** OpenClaw keeps the product default transport separate from the core agent runtime; that split is useful for a future CLI-plus-SaaS AgentOS

### Step 3 — Resolve session, workspace, and policy-bearing defaults

- **Location:** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution`
- **What enters:** `AgentCommandOpts` plus runtime env
- **Assumptions already true:** this is a trusted local caller, so `senderIsOwner` is already set by `agentCommand`
- **What this step does:** validates message and overrides, loads config, resolves session identity via `resolveSession`, resolves the agent workspace and agent dir, computes timeout, and derives outbound context
- **What leaves:** a prepared execution bundle containing `sessionId`, `sessionKey`, `sessionEntry`, `workspaceDir`, `agentDir`, `timeoutMs`, and current persisted overrides
- **Why this step matters:** this is the first place OpenClaw turns a raw user request into a sessioned unit of work

### Step 4 — Snapshot skills and persist session-side run state

- **Location:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **What enters:** prepared execution bundle and current `SessionEntry`
- **Assumptions already true:** session/workspace already resolved and workspace bootstrap files already exist
- **What this step does:** builds a workspace `skillsSnapshot` when the session is new or missing one, persists thinking/verbose overrides, selects the effective model/provider, and resolves the session transcript file through `resolveSessionTranscriptFile`
- **What leaves:** a fully prepared session state plus `sessionFile`
- **Why this step matters:** OpenClaw stabilizes skill availability and transcript location before entering any model loop

### Step 5 — Dispatch the attempt to the embedded runner

- **Location:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal` via `repos/openclaw/src/agents/model-fallback.ts:runWithModelFallback` into `repos/openclaw/src/agents/command/attempt-execution.ts:runAgentAttempt`
- **What enters:** provider/model selection, session identifiers, transcript path, prompt text, skills snapshot, tool/runtime options
- **Assumptions already true:** fallback policy and auth-profile compatibility are already resolved enough to start an attempt
- **What this step does:** chooses the embedded path because the selected provider is not a CLI backend, then calls `runEmbeddedPiAgent`
- **What leaves:** a running embedded attempt
- **Why this step matters:** this is the main dispatch seam between “drive an external coding CLI” and “drive OpenClaw’s own embedded loop”

### Step 6 — Serialize the turn and load runtime plugins

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **What enters:** the full embedded run params bundle
- **Assumptions already true:** session state and transcript path already exist
- **What this step does:** enqueues the turn through a session lane and a global lane, resolves the workspace, loads runtime plugins with `ensureRuntimePluginsLoaded`, resolves auth/model state, resolves the context engine once, and enters the retry/failover loop
- **What leaves:** one concrete attempt invocation into `runEmbeddedAttempt`
- **Why this step matters:** OpenClaw treats a turn as serialized operational work, not a free-floating async function call

### Step 7 — Build sandbox, skills, tools, and the runtime system prompt

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **What enters:** current model config, workspace, session ids, skills snapshot, prompt text, and runtime context
- **Assumptions already true:** plugin registry is active and the context engine instance is already resolved
- **What this step does:** resolves sandbox context, restores skill env overrides, loads workspace/bootstrap files, builds the skills prompt, constructs tools through `createOpenClawCodingTools`, attaches MCP/LSP tools, computes runtime capabilities, and builds the final system prompt via `buildEmbeddedSystemPrompt`
- **What leaves:** `effectiveTools`, `systemPromptText`, and runtime metadata
- **Why this step matters:** this is where OpenClaw materializes “AgentOS policy” into the exact prompt/tool surface the model will see

### Step 8 — Acquire transcript ownership and reconstruct session state

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:acquireSessionWriteLock`, `SessionManager.open`, `prepareSessionManagerForRun`, and `createAgentSession`
- **What enters:** `sessionFile`, `effectiveTools`, `systemPromptText`, auth/model registries
- **Assumptions already true:** workspace and sandbox directories already exist
- **What this step does:** acquires a transcript write lock, repairs transcript damage, opens `SessionManager`, prepares settings/resource loaders, splits built-in versus custom tools, and creates the live agent session object
- **What leaves:** `activeSession`
- **Why this step matters:** the live model/tool loop does not run on raw files; it runs on a guarded in-memory session reconstructed from transcript state

### Step 9 — Sanitize history and assemble context-engine output

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:sanitizeSessionHistory` and `assembleAttemptContextEngine`
- **What enters:** `activeSession.messages`, transcript policy, token budget, prompt text
- **Assumptions already true:** the session object is live and the tool set is fixed for this attempt
- **What this step does:** strips invalid history patterns, limits turns, repairs tool-use/result pairing, then lets the context engine replace messages and prepend `systemPromptAddition`
- **What leaves:** the exact message history and system prompt that the model call will consume
- **Why this step matters:** OpenClaw explicitly separates stored transcript truth from model-context truth

### Step 10 — Prompt the live agent session and observe the tool loop

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:activeSession.prompt`
- **What enters:** effective prompt text plus optional prompt-local images
- **Assumptions already true:** session history, tool schemas, and stream wrappers are already in place
- **What this step does:** subscribes to assistant/tool events, forwards tool outputs and deltas outward, runs the model prompt, lets the underlying agent execute tool calls, and waits for compaction retries when needed
- **What leaves:** mutated session messages, assistant texts, tool metadata, usage totals, and possible prompt/tool errors
- **Why this step matters:** this is the true inner agent loop, even though OpenClaw delegates the low-level loop mechanics to `@mariozechner/pi-coding-agent`

### Step 11 — Finalize post-turn state

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:finalizeAttemptContextEngineTurn`
- **What enters:** message snapshot, prompt error state, token budget, runtime context
- **Assumptions already true:** the attempt has either completed, aborted, or timed out
- **What this step does:** lets the context engine ingest/maintain/compact after the turn, records prompt-error entries when relevant, computes the final attempt result, and tears down the live session while flushing pending tool results safely
- **What leaves:** `EmbeddedRunAttemptResult`
- **Why this step matters:** OpenClaw does not treat post-turn maintenance as optional cleanup; it feeds directly back into future turn quality

### Step 12 — Normalize payloads and return to the caller

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent` back through `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **What enters:** attempt result plus retry/failover state
- **Assumptions already true:** the session transcript and session store are already updated enough to continue future turns
- **What this step does:** retries on recoverable failures, builds final payloads and agent metadata, optionally delivers through channel transport, and returns to the original CLI call
- **What leaves:** the user-visible reply payload plus run metadata
- **Why this step matters:** this is where OpenClaw turns a noisy operational attempt into a stable turn result

---

## 6. Dispatch Map

### Gateway-vs-local command dispatch

- **What selects:** `--local` flag plus gateway failure
- **How it selects:** `agentCliCommand` branches to `agentCommand` locally or `agentViaGatewayCommand` first
- **Where the resolver lives:** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **Risk this introduces:** callers can misread gateway mode as the core runtime when it is only a transport wrapper

### Trusted-vs-ingress execution policy

- **What selects:** local call path versus network ingress path
- **How it selects:** separate exported functions inject different trust defaults
- **Where the resolver lives:** `repos/openclaw/src/agents/agent-command.ts:agentCommand` and `repos/openclaw/src/agents/agent-command.ts:agentCommandFromIngress`
- **Risk this introduces:** copying only the local path can accidentally erase the explicit trust boundary your SaaS phase needs

### CLI-backend-vs-embedded runtime

- **What selects:** whether the chosen provider is a CLI backend
- **How it selects:** `isCliProvider` inside `runAgentAttempt`
- **Where the resolver lives:** `repos/openclaw/src/agents/command/attempt-execution.ts:runAgentAttempt`
- **Risk this introduces:** session reuse semantics diverge sharply between child-process CLIs and embedded sessions

### Model/auth/fallback dispatch

- **What selects:** stored overrides, explicit overrides, auth-profile order, failover reason, configured fallbacks
- **How it selects:** `agentCommandInternal` resolves the initial model; `runWithModelFallback` and `runEmbeddedPiAgent` rotate across models/profiles/thinking levels
- **Where the resolver lives:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal` and `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **Risk this introduces:** selection state is spread across session store, config, runtime auth state, and retry loop state

### Context-engine slot dispatch

- **What selects:** `config.plugins.slots.contextEngine` or the default slot id
- **How it selects:** registry lookup by slot value
- **Where the resolver lives:** `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine`
- **Risk this introduces:** the seam is clean, but the default engine is intentionally weak; swapping engines without matching runtime expectations can silently reduce quality

### Plugin-registry dispatch

- **What selects:** plugin config, workspace path, load paths, runtime subagent mode, explicit plugin scopes
- **How it selects:** cache-keyed registry resolution in `resolveRuntimePluginRegistry`
- **Where the resolver lives:** `repos/openclaw/src/plugins/loader.ts:resolveRuntimePluginRegistry`
- **Risk this introduces:** runtime behavior depends on process-global plugin state and cache compatibility, not just local function parameters

### Tool-surface dispatch

- **What selects:** sandbox state, model/tool support, agent policy, group policy, owner status, subagent depth, message channel, plugin tool metadata
- **How it selects:** `createOpenClawCodingTools` composes raw tools, then filters them through `applyToolPolicyPipeline`
- **Where the resolver lives:** `repos/openclaw/src/agents/pi-tools.ts:createOpenClawCodingTools`
- **Risk this introduces:** the tool list is an emergent result of many policy layers; debugging a missing tool requires understanding the whole policy pipeline

### Skill-source dispatch

- **What selects:** workspace skills, bundled skills, plugin skill dirs, config allow/deny, agent skill filter, runtime eligibility
- **How it selects:** filesystem scanning plus prompt filtering
- **Where the resolver lives:** `repos/openclaw/src/agents/skills/workspace.ts:resolveSkillsPromptForRun`
- **Risk this introduces:** prompt-visible capabilities can diverge from installed capabilities if snapshot/filter logic is misunderstood

---

## 7. State Model

### `SessionEntry`

- **Type:** session-store record
- **Authoritative vs derived:** authoritative for routing, overrides, delivery context, skill snapshot, model/runtime metadata
- **Persistent vs transient:** persistent in the session store
- **Created at:** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution` via `resolveSession`
- **Mutated at:** `agentCommandInternal`, `persistSessionEntry`, cron isolated-agent paths, gateway session updates
- **Read by:** model selection, delivery logic, skill snapshot reuse, CLI session binding reuse
- **Invariants that matter:** session key remains the durable bucket; `sessionId` may rotate; overrides must match valid provider/model/auth state

### Transcript JSONL session file

- **Type:** `SessionManager`-owned transcript file
- **Authoritative vs derived:** authoritative for turn history
- **Persistent vs transient:** persistent on disk
- **Created at:** `resolveSessionTranscriptFile`
- **Mutated at:** live agent session, transcript repair, prompt-error entry persistence, compaction, ACP transcript persistence
- **Read by:** `SessionManager.open`, history sanitation, context assembly, compaction, session-history tools
- **Invariants that matter:** one writer at a time, valid session header, sane tool-use/result ordering, no orphaned trailing user turn

### `skillsSnapshot`

- **Type:** serialized skill prompt/metadata snapshot
- **Authoritative vs derived:** derived from filesystem skill state at snapshot time, but authoritative for later turns in that session
- **Persistent vs transient:** persistent in `SessionEntry`
- **Created at:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **Mutated at:** new-session or refresh points
- **Read by:** `runEmbeddedAttempt` and cron isolated-agent runs
- **Invariants that matter:** the snapshot must match the workspace/config assumptions under which the turn runs

### Live embedded attempt state

- **Type:** retry-loop-local state inside `runEmbeddedPiAgent`
- **Authoritative vs derived:** authoritative for this single run only
- **Persistent vs transient:** transient
- **Created at:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **Mutated at:** auth rotation, fallback selection, compaction counts, usage accumulation, timeout bookkeeping
- **Read by:** retry logic and final result shaping
- **Invariants that matter:** provider/model/auth/think-level transitions must remain internally coherent across retries

### Active plugin registry

- **Type:** process-global plugin registry snapshot
- **Authoritative vs derived:** authoritative for runtime-discoverable plugin tools/channels/routes
- **Persistent vs transient:** process-transient, cache-backed
- **Created at:** `repos/openclaw/src/plugins/loader.ts:loadOpenClawPlugins`
- **Mutated at:** `setActivePluginRegistry`
- **Read by:** channel registry, tool resolution, provider capability/runtime helpers
- **Invariants that matter:** the active registry must match the cache key assumptions of the current runtime

### Context engine instance

- **Type:** `ContextEngine`
- **Authoritative vs derived:** authoritative for context assembly/maintenance once selected
- **Persistent vs transient:** process-transient, resolved per run
- **Created at:** `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine`
- **Mutated at:** engine-owned methods such as `bootstrap`, `assemble`, `afterTurn`, `compact`
- **Read by:** `runEmbeddedPiAgent` and `runEmbeddedAttempt`
- **Invariants that matter:** the engine must obey the runtime contract around transcript ownership and compaction ownership

---

## 8. Data Transformation Chain

1. CLI/gateway/cron input arrives as transport-specific args or RPC payload.
2. `prepareAgentCommandExecution` normalizes that transport input into session-scoped execution input.
3. `agentCommandInternal` merges persisted session state, skills snapshot state, and model override state into a run-ready bundle.
4. `runAgentAttempt` transforms the bundle into either CLI-backend params or embedded-run params.
5. `runEmbeddedPiAgent` transforms run params into a retry-managed attempt context with resolved auth/model/context-engine state.
6. `runEmbeddedAttempt` transforms workspace/config/session state into prompt-visible artifacts: tools, skills prompt, bootstrap context, runtime system prompt, sanitized history.
7. `activeSession.prompt(...)` transforms prompt plus history plus tool schemas into assistant messages and tool-call cycles.
8. Post-turn hooks and context-engine methods transform raw messages into maintained transcript/context state.
9. `runEmbeddedPiAgent` transforms noisy attempt outcomes into stable payloads, usage meta, and recoverable failure paths.

---

## 9. Side-Effect Map

### Session store persistence

- **Where triggered:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`, `repos/openclaw/src/agents/command/attempt-execution.ts:persistSessionEntry`
- **Guards:** session key and store path must exist
- **Retryable:** conditional; merge-write retries depend on caller
- **Idempotent:** mostly yes when writing the same merged entry
- **Failure consequence:** future turns lose overrides, skills snapshot reuse, or CLI session bindings

### Transcript file mutation

- **Where triggered:** `SessionManager.open` path in `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts`, plus ACP transcript persistence in `persistAcpTurnTranscript`
- **Guards:** session lock acquired, session file repaired/prepared
- **Retryable:** conditional; safe only while transcript invariants still hold
- **Idempotent:** no
- **Failure consequence:** history corruption or duplicate/orphaned turn state

### Plugin loading

- **Where triggered:** `repos/openclaw/src/agents/runtime-plugins.ts:ensureRuntimePluginsLoaded`
- **Guards:** compatible config/workspace/cache context
- **Retryable:** yes
- **Idempotent:** effectively yes for the same cache key
- **Failure consequence:** tools, channels, providers, hooks, or routes silently disappear from the runtime surface

### Skill env injection

- **Where triggered:** `repos/openclaw/src/agents/skills/env-overrides.ts:applySkillEnvOverrides`
- **Guards:** skill config, env sanitation, allowed sensitive keys
- **Retryable:** yes
- **Idempotent:** conditionally; reference-counted per key
- **Failure consequence:** skills lose credentials or leak unsafe env vars into child processes

### Child CLI process execution

- **Where triggered:** `repos/openclaw/src/agents/cli-runner/execute.ts:executePreparedCliRun`
- **Guards:** provider must resolve to a CLI backend; process supervisor must spawn successfully
- **Retryable:** conditional; session-expired path retries without the stale CLI session id
- **Idempotent:** no
- **Failure consequence:** no turn result; possible partial external side effects from the child CLI

### Model/API streaming

- **Where triggered:** live session prompt in `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:activeSession.prompt`
- **Guards:** auth resolved, stream wrappers installed, timeout/abort signals active
- **Retryable:** yes under failover/compaction/thinking downgrade policy
- **Idempotent:** no
- **Failure consequence:** partial assistant output, prompt errors, retry/failover escalation

### Tool execution

- **Where triggered:** inside the live agent session after tool schemas are registered
- **Guards:** tool policy pipeline, sandbox policy, owner policy, provider/tool support
- **Retryable:** tool-specific
- **Idempotent:** tool-specific
- **Failure consequence:** prompt continuation failure, side-effect duplication, or tool-loop escalation

### Cron delivery

- **Where triggered:** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **Guards:** resolved delivery target and cron delivery contract
- **Retryable:** conditional
- **Idempotent:** no
- **Failure consequence:** scheduled work completes but never reaches the target channel

---

## 10. Important Branch Points

### Gateway-first versus local-first command path

- **Condition:** `opts.local === true`
- **Where checked:** `repos/openclaw/src/commands/agent-via-gateway.ts:agentCliCommand`
- **Alternate paths:** local embedded/CLI runtime immediately; gateway RPC first with embedded fallback on failure
- **Why it matters:** product topology changes, but the core turn machine does not

### ACP versus ordinary agent runtime

- **Condition:** ACP session resolution returns `ready`
- **Where checked:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **Alternate paths:** ACP control-plane turn; ordinary CLI-backend or embedded turn
- **Why it matters:** ACP bypasses a large part of the embedded runtime and uses different session semantics

### CLI backend versus embedded backend

- **Condition:** `isCliProvider(providerOverride, cfg)`
- **Where checked:** `repos/openclaw/src/agents/command/attempt-execution.ts:runAgentAttempt`
- **Alternate paths:** child-process CLI run; embedded live session run
- **Why it matters:** this is the highest-leverage runtime seam in the repo

### New session versus reused session

- **Condition:** session freshness/reset outcome and existing `skillsSnapshot`
- **Where checked:** `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal`
- **Alternate paths:** snapshot and transcript path creation; reuse existing snapshot and transcript lineage
- **Why it matters:** this decides whether the turn reuses established operating memory

### Sandbox-enabled versus host-workspace run

- **Condition:** sandbox context resolves as enabled
- **Where checked:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **Alternate paths:** use sandbox workspace/fs bridge/browser bridge; use host workspace directly
- **Why it matters:** tool set, filesystem guards, and spawned-subagent workspace inheritance all change here

### Context-engine-owned compaction versus runtime-owned compaction

- **Condition:** `contextEngine.info.ownsCompaction === true`
- **Where checked:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **Alternate paths:** engine runs compaction lifecycle itself; runtime drives legacy compaction flow
- **Why it matters:** ownership of transcript reduction moves across the plugin seam

### Compaction retry versus failover rotation

- **Condition:** timeout/overflow/tool-result-size signals inside the retry loop
- **Where checked:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **Alternate paths:** compact and retry same model; truncate tool results; rotate auth profile/model/thinking level; fail the run
- **Why it matters:** OpenClaw favors preserving the current session before abandoning it

---

## 11. Failure and Recovery Logic

### Failure Surfaces

| Surface | Where Detected | How It Propagates | Translated To |
|---------|---------------|-------------------|---------------|
| Invalid CLI or command overrides | `prepareAgentCommandExecution` | throws immediately | user-facing command error |
| Unknown context engine slot | `resolveContextEngine` | throws | run startup failure |
| CLI child no-output stall | `executePreparedCliRun` | wrapped in `FailoverError` | timeout failover plus heartbeat/system event |
| Prompt/API timeout | `runEmbeddedAttempt` and `runEmbeddedPiAgent` | abort + retry/compaction/failover | timeout payload or retry |
| Context overflow | `runEmbeddedPiAgent` after attempt | observed and classified | compaction or tool-result truncation |
| Auth/profile failure | embedded auth controller in `runEmbeddedPiAgent` | profile failure marked, candidate advances | retry on another auth profile |
| Live session model switch | `runEmbeddedPiAgent` | throws `LiveSessionModelSwitchError` | restart under the new selection |
| Transcript corruption/orphaned state | `repairSessionFileIfNeeded`, history sanitation, orphan-user repair | repaired in place where possible | continued run or degraded history |

### Recovery Behavior

- **Retry logic:** the main retry loop lives in `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`. It retries under auth rotation, overload backoff, timeout-triggered compaction, overflow-triggered compaction, and tool-result truncation.
- **Fallback logic:** `runWithModelFallback` in `repos/openclaw/src/agents/agent-command.ts:agentCommandInternal` rotates to configured fallback models when the current provider/model path fails hard enough.
- **Timeout behavior:** `runEmbeddedAttempt` installs both overall timeout and idle timeout. Timeouts that strike during compaction trigger special snapshot selection rather than trusting partially compacted live state.
- **Lifecycle mismatch risks:** plugin registry is process-global; session store state and transcript state can diverge; CLI session reuse can go stale; skill env vars can leak without reference-counted cleanup; the live agent session belongs partly to external Pi libraries, so OpenClaw carries several repair/guard layers to keep that boundary safe.

---

## 12. The Real Core Loop

```text
loop for one embedded run:
  1. resolve workspace/plugins/auth/context engine -> `runEmbeddedPiAgent`
  2. build sandbox/skills/tools/system prompt -> `runEmbeddedAttempt`
  3. reconstruct live session from transcript -> `runEmbeddedAttempt`
  4. sanitize history + assemble context engine output -> `runEmbeddedAttempt`
  5. prompt live agent session -> `runEmbeddedAttempt`
  6. observe tool/assistant/compaction events -> `subscribeEmbeddedPiSession`
  7. classify failure or success -> `runEmbeddedPiAgent`
  8. compact / rotate auth / rotate model / downgrade thinking / retry if needed -> `runEmbeddedPiAgent`
  until success, unrecoverable failure, or retry limit
```

The outer loop is OpenClaw’s real core loop. The inner tool-call loop belongs to the live Pi agent session, but OpenClaw wraps it so aggressively that the outer retry/assembly/persistence loop is the true center of runtime control.

---

## 13. Architectural Compression

**True center of gravity:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`. If you delete it, OpenClaw loses the mechanism that turns session state, prompt state, tool state, and context-engine state into one coherent attempt.

**Replaceable parts:**

- `ContextEngine` implementations behind `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine`
- transport ingress layers like CLI and gateway wrappers
- plugin-discovered skill/tool/channel breadth, assuming the registry contract remains
- CLI backend provider execution behind `runCliAgent`

**Dangerous seams:**

- the Pi library boundary around `createAgentSession` and `SessionManager`
- process-global plugin registry compatibility in `plugins/runtime.ts`
- transcript-file truth versus session-store truth
- skill env injection versus child-process isolation
- layered tool policy, because the final tool set is the product of many hidden filters

**Real abstractions:**

- `ContextEngine` earns its complexity because it genuinely lets runtime context assembly and compaction swap independently of transport
- plugin slots earn their complexity because they select exclusive implementations such as context engine or memory provider
- `runAgentAttempt` earns its split because CLI-backend and embedded backends are materially different machines

**Decorative abstractions:**

- the default `LegacyContextEngine` is mostly a compatibility shell. It preserves the seam, but it does not yet teach a strong alternative context architecture by itself.

---

## 14. Design Intent Inference

**Primary optimization:** operational survivability of long-lived agent sessions.

**Evidence for this inference:** transcript repair, write locks, session-id rotation under one session key, compaction retry logic, tool-result truncation, auth-profile rotation, plugin-registry pinning, and special handling for timeouts during compaction all favor keeping the session alive over keeping the codepath simple.

**Secondary optimizations:** product extensibility through plugins/skills/channels; unified runtime reuse across CLI, gateway, and cron; explicit trust/sandbox policy at the edge.

**What was sacrificed:** local simplicity, narrow module boundaries, and ease of reasoning. The execution path is powerful because it accreted many production failure lessons, but that same density makes first-principles understanding expensive.

---

## 15. Safe Surgery Map

### Safer to Modify First

- `repos/openclaw/src/context-engine/types.ts` and the non-legacy parts of `repos/openclaw/src/context-engine/registry.ts` — isolated contract and slot resolution with low dependency weight
- `repos/openclaw/src/agents/skills/env-overrides.ts` — self-contained security boundary for skill env injection
- `repos/openclaw/src/commands/agent-via-gateway.ts` — transport wrapper that does not own the core loop
- `repos/openclaw/src/agents/runtime-plugins.ts` — thin adapter over registry resolution

### Do Not Touch Without Full Understanding

- `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts` — transcript, prompt, tool, timeout, compaction, and hook invariants converge here
- `repos/openclaw/src/agents/pi-embedded-runner/run.ts` — retry/failover/auth/compaction policy converges here
- `repos/openclaw/src/agents/pi-tools.ts` — the final tool surface emerges from many policy layers
- `repos/openclaw/src/plugins/loader.ts` and `repos/openclaw/src/plugins/runtime.ts` — process-global registry and cache assumptions can fail silently

---

## 16. Learning-Value Map

### High Learning-Value Targets

#### Embedded turn attempt assembly

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **Why high value:** it teaches how to stage one turn as a controlled pipeline instead of a monolithic agent call
- **Teaches:** core loop, state model, tool assembly, context assembly, timeout handling, transcript hygiene

#### Embedded retry/failover shell

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **Why high value:** it teaches how to wrap an agent loop with operational recovery without burying those rules inside tools or prompts
- **Teaches:** runtime retry design, auth/model rotation, compaction recovery, outer-loop control

#### Session/model/skills preparation

- **Location:** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution` and `agentCommandInternal`
- **Why high value:** it teaches how to convert user intent into a session-scoped execution bundle before any model call starts
- **Teaches:** session truth, model override policy, skill snapshot policy, workspace ownership

#### Tool surface composition

- **Location:** `repos/openclaw/src/agents/pi-tools.ts:createOpenClawCodingTools`
- **Why high value:** it teaches that “tool registry” is not enough; the real problem is policy-shaped tool materialization
- **Teaches:** policy layering, sandbox-aware tool shaping, channel/tool docking, owner gating

#### Context-engine seam

- **Location:** `repos/openclaw/src/context-engine/types.ts` and `repos/openclaw/src/context-engine/registry.ts:resolveContextEngine`
- **Why high value:** it teaches a minimal but serious contract for context lifecycle instead of a vague “memory module”
- **Teaches:** contract design, slot resolution, transcript ownership boundaries

### Medium Learning-Value Targets

#### Skill prompt and skill snapshot loading

- **Location:** `repos/openclaw/src/agents/skills/workspace.ts:resolveSkillsPromptForRun`
- **Why medium value:** it teaches a practical filesystem skill system, but the exact scanning/frontmatter rules are OpenClaw-shaped
- **Teaches:** skill discovery, prompt compaction, source filtering

#### Plugin registry loading

- **Location:** `repos/openclaw/src/plugins/loader.ts:resolveRuntimePluginRegistry`
- **Why medium value:** it teaches mature runtime extensibility, but with more product baggage than a first AgentOS needs
- **Teaches:** registry caching, provenance tracking, runtime activation

### Low Immediate Learning-Value Targets

#### Cron isolated-agent runner

- **Location:** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **Why lower immediate value:** it teaches scheduled productization on top of the core runtime, not the core runtime itself
- **Teaches:** cron delivery policy, session isolation for scheduled jobs

#### Channel registry

- **Location:** `repos/openclaw/src/channels/registry.ts`
- **Why lower immediate value:** it teaches runtime discovery of channel plugins, but not the main agent loop
- **Teaches:** channel normalization, plugin-backed transport discovery

---

## 17. Reuse/Copy-Value Map

### Copy-Paste First

#### Context-engine contract and slot resolution skeleton

- **Location:** `repos/openclaw/src/context-engine/types.ts`, `repos/openclaw/src/context-engine/init.ts`, `repos/openclaw/src/context-engine/registry.ts:registerContextEngineForOwner`, `resolveContextEngine`
- **Copy value:** high
- **Dependency weight:** low
- **Hidden assumptions:** you adopt the same idea that transcript ownership stays in the runtime unless an engine explicitly claims compaction ownership
- **Why copy:** this is already shaped like your lead’s `ContextEngine` module and gives you a concrete lifecycle: `bootstrap`, `maintain`, `ingest`, `assemble`, `compact`

#### Plugin slot defaults

- **Location:** `repos/openclaw/src/plugins/slots.ts`
- **Copy value:** high
- **Dependency weight:** low
- **Hidden assumptions:** you want exclusive slot semantics for modules like memory/context engine
- **Why copy:** the slot mechanism is small, clear, and directly useful for “one active context engine / one active memory backend” decisions

#### Skill env override guard

- **Location:** `repos/openclaw/src/agents/skills/env-overrides.ts`
- **Copy value:** high
- **Dependency weight:** low-to-medium
- **Hidden assumptions:** your skill system injects env vars and can spawn child processes
- **Why copy:** it already solves a real security problem your AgentOS will hit as soon as user-defined skills touch external APIs

### Learn-The-Heart, Do Not Copy Raw

#### The embedded attempt pipeline

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run/attempt.ts:runEmbeddedAttempt`
- **Copy value:** medium only after heavy adaptation
- **Dependency weight:** very high
- **Hidden assumptions:** Pi agent libraries, SessionManager transcript format, OpenClaw hook contracts, OpenClaw tool names, OpenClaw sandbox contracts
- **Why not copy raw:** the value is architectural rhythm, not the literal file. You should internalize its order: resolve sandbox -> resolve skills/tools/system prompt -> reconstruct session -> sanitize history -> assemble context -> run prompt -> finalize post-turn state.

#### The outer retry shell

- **Location:** `repos/openclaw/src/agents/pi-embedded-runner/run.ts:runEmbeddedPiAgent`
- **Copy value:** medium only after redesign
- **Dependency weight:** high
- **Hidden assumptions:** OpenClaw auth profile store, failover taxonomy, compaction helpers, live session model switching
- **Why not copy raw:** you want the control philosophy, not the entire production scar tissue on day one

#### Tool policy pipeline

- **Location:** `repos/openclaw/src/agents/pi-tools.ts:createOpenClawCodingTools`
- **Copy value:** medium
- **Dependency weight:** high
- **Hidden assumptions:** OpenClaw tool names, plugin tool metadata, channel policy, owner policy, sandbox policy, subagent policy
- **Why not copy raw:** your first AgentOS should probably keep fewer policy axes. Copy the layered-policy idea, not the full matrix.

#### Session preparation and skill snapshot persistence

- **Location:** `repos/openclaw/src/agents/agent-command.ts:prepareAgentCommandExecution`, `agentCommandInternal`
- **Copy value:** medium
- **Dependency weight:** medium-to-high
- **Hidden assumptions:** OpenClaw session-store schema and model/auth override semantics
- **Why not copy raw:** the exact fields are product-specific, but the staging boundary is excellent

### Good To Learn, But Not To Apply Immediately

#### Plugin loader breadth

- **Location:** `repos/openclaw/src/plugins/loader.ts:resolveRuntimePluginRegistry`
- **Copy value:** low in phase one
- **Dependency weight:** very high
- **Hidden assumptions:** wide plugin marketplace, process-global registry, provenance tracking, multiple runtime surfaces
- **Why defer:** your first AgentOS needs a narrower, more inspectable extension story

#### Cron isolated-agent execution

- **Location:** `repos/openclaw/src/cron/isolated-agent/run.ts:runCronIsolatedAgentTurn`
- **Copy value:** low in phase one, medium later
- **Dependency weight:** high
- **Hidden assumptions:** scheduled job store, delivery targets, cron-specific tool policy, session isolation semantics
- **Why defer:** useful in phase two when your “digital employee” mode arrives, but it will distract the first local CLI shell

#### Channel/session routing surfaces

- **Location:** `repos/openclaw/src/channels/registry.ts`, `repos/openclaw/src/channels/session.ts`, gateway agent delivery code
- **Copy value:** low now, higher in SaaS/IM phase
- **Dependency weight:** medium-to-high
- **Hidden assumptions:** many channel plugins, transport-specific delivery state, last-route semantics
- **Why defer:** learn the pattern now; implement only once your core agent shell stabilizes

---

## 18. Mental Model Packet

- OpenClaw’s core unit is not “a message”; it is “a turn in a durable session.”
- `agent-command.ts` resolves session/workspace/model/skills before any loop starts.
- `runEmbeddedPiAgent` is the outer operational loop: queue, auth, failover, compaction, retry.
- `runEmbeddedAttempt` is the inner assembly pipeline: sandbox, skills, tools, prompt, history, context engine.
- The live model/tool loop sits inside Pi libraries, but OpenClaw controls everything around it.
- Tools are not merely registered; they are materialized through layered policy.
- Transcript truth, session-store truth, and prompt-context truth are separate and must stay aligned.
- Copy the contracts and staging boundaries first; study the production recovery machinery before adopting it whole.
