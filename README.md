[中文版](README.cn.md) | **English**

# Code Surgical Analysis Catalog

This repository publishes bilingual surgical analyses of agent systems and adjacent runtimes. Each report is a curated publication artifact focused on concrete execution paths rather than high-level impressions.

## What This Repo Contains

- A stable landing-page index in English and Chinese.
- Published report pairs under `reports/`, with one English file and one Chinese file per analysis.
- Reusable analysis structure centered on scope, happy-path runtime tracing, state/dispatch mapping, side effects, failure recovery, and reuse guidance.

## Methodology

Each report is expected to answer the same core questions:

- What exact subsystem is being analyzed, and what is intentionally excluded?
- How does one real turn or task flow through the system on the happy path?
- Where are the actual control points for dispatch, state, tools, context, and persistence?
- Which mechanisms are worth copying directly, internalizing deeply, or only studying?

## Publication Contract

- Every published analysis ships as an EN+CN pair.
- The root README pair remains the repo homepage.
- New analyses are added by creating a new slug pair under `reports/` and a new row in the catalog below.
- English and Chinese are peer publication artifacts, not optional follow-ups.

## Report Catalog

| System | Focus | Date | English | 中文 |
|---|---|---|---|---|
| Claude Code | Agent runtime and tool-loop architecture | 2026-04-01 | [EN](reports/claude-code-agent-runtime-surgical-analysis.md) | [中文](reports/claude-code-agent-runtime-surgical-analysis.cn.md) |
| DeerFlow | AgentOS-base suitability, adoption boundary, and reusable subsystems | 2026-04-02 | [EN](reports/deer-flow-agentos-base-surgical-analysis.md) | [中文](reports/deer-flow-agentos-base-surgical-analysis.cn.md) |
| OpenClaw | Core agent runtime for a local-first then SaaS shell | 2026-04-01 | [EN](reports/openclaw-core-agent-runtime-surgical-analysis.md) | [中文](reports/openclaw-core-agent-runtime-surgical-analysis.cn.md) |

## Notes

- The Claude Code report is preserved from the repo’s original single-report README publication and moved into `reports/`.
- The DeerFlow report is curated from the comparison workspace research output and republished here under a stable slug, with a matching Chinese peer file.
- The OpenClaw report is curated from the comparison workspace research output and republished here under a stable slug.
- This repository is documentation-only; it does not define a build, test, or code generation workflow.
