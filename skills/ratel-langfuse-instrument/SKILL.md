---
name: ratel-langfuse-instrument
description: |
  Inspect an agent codebase, decide where Langfuse tracing belongs, and produce a concrete instrumentation plan covering SDK setup, session boundaries, sub-agent handoffs, tool wrapping, and the naming/tagging vocabulary that downstream Langfuse work depends on. Use whenever the user wants to mount observability for the first time, asks "where would Langfuse go in this codebase", says "instrument the agent", "wire up tracing", "set up Langfuse here", or invokes `/ratel-langfuse-instrument`. Trigger on phrases like "add observability", "we just signed a partner, let's get Langfuse in", "instrument this for us", "trace coverage for this agent" — even if the user doesn't say "skill" or "ratel-langfuse-instrument" by name. This skill writes a markdown plan; it does not edit the agent code itself.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
---

# /ratel-langfuse-instrument — plan Langfuse coverage for an agent codebase

Mount Langfuse observability on a customer's codebase the way the Ratel team would: detect the stack, map the agent's mental model, decide one consistent naming/tagging vocabulary, and write a plan the customer can implement file by file. The plan is the deliverable. Do not edit the agent code.

This skill exists because every partner-startup onboarding starts the same way — figure out what they're running, decide what to trace, agree a vocabulary — and we want that conversation to be repeatable instead of ad-hoc. The vocabulary it lands on also becomes the contract for [`/ratel-langfuse-dashboards`](../ratel-langfuse-dashboards/SKILL.md) and [`/ratel-langfuse-analyze`](../ratel-langfuse-analyze/SKILL.md), which both expect the names/tags/metadata defined here to actually show up in traces.

## Philosophy: trace the mental model, not the call graph

A common failure mode is "wrap every function in a span." That produces traces that match the code's call graph but tell you nothing about what the agent was *trying to do*. Langfuse traces are most useful when their structure matches the conceptual structure of a turn:

- **Trace** = one externally meaningful unit of work (one chat turn, one job, one webhook). Not "one HTTP request" if a request contains multiple agent turns; not "one model call" if a turn contains many.
- **Observation** = one step the agent took inside that unit. Sub-agent invocations, tool calls, model calls, retrieval steps. Nest them to reflect delegation, not source-file layout.
- **Session** = a thread of related traces sharing a `session_id`. Usually a user conversation, an agent run-id, or a job correlation id.

Two anti-patterns to call out in the plan when you see them in the code:

1. **No session boundary at all** — every turn is a fresh trace with no `session_id`. Multi-turn analysis becomes impossible. The fix is almost always a single line at the agent entry point.
2. **Tool calls captured as untyped events** — every tool call lands as a generic `event` with the tool name in `metadata` rather than as an `observation` of type `tool` with the tool name in `name`. This blocks the entire native tool-call dashboard surface.

Refer to [`references/naming-conventions.md`](references/naming-conventions.md) for the full vocabulary the plan should adopt.

## Workflow

### Step 1 — Detect the stack

Read manifest files to identify language and framework. Branch into the matching reference file for stack-specific patterns.

```bash
# TypeScript / Node detection
test -f package.json && cat package.json | jq -r '.dependencies // {}, .devDependencies // {} | keys[]' | sort -u

# Python detection
test -f pyproject.toml && cat pyproject.toml | grep -A 200 '^\[' || true
test -f requirements.txt && cat requirements.txt
test -f uv.lock && head -50 uv.lock
```

Map dependencies to one of these stack profiles:

| Signal in manifest | Stack | Reference |
| --- | --- | --- |
| `ai`, `@ai-sdk/*` | Vercel AI SDK | [`references/stack-vercel-ai-sdk.md`](references/stack-vercel-ai-sdk.md) |
| `@mastra/core`, hand-rolled loops calling `openai` / `@anthropic-ai/sdk` directly | TypeScript generic | [`references/stack-typescript-generic.md`](references/stack-typescript-generic.md) |
| `langfuse` + `openai` / `anthropic` / `langchain` / `llama_index` | Python generic | [`references/stack-python-generic.md`](references/stack-python-generic.md) |
| `langgraph`, `crewai`, `agno`, `autogen` | Python agentic | [`references/stack-python-agentic.md`](references/stack-python-agentic.md) |

If signals overlap (e.g., both a LangGraph supervisor and raw OpenAI calls inside), pick the agentic reference as primary and note the mixed-stack callout in the plan.

If you cannot identify any agent surface at all (no LLM client imports, no agent framework, no model calls), use the [honest skip path](#honest-skip-path).

### Step 2 — Map the agent's topology

Launch one **Explore** agent (or do it directly for very small repos) to answer four questions, citing file paths:

1. **Where does a turn begin?** — entry points: an HTTP handler, a CLI verb, a queue consumer, a chat-platform webhook. This is where `session_id` lives.
2. **What are the agent units?** — supervisor function, sub-agent factories, role-specialised loops. Anything that takes a user message and returns a response. These become trace boundaries.
3. **Where are tools defined and called?** — tool registries (`tools: [...]`), `@tool` decorators, MCP server wiring. Each tool needs to surface as an observation of type `tool` (Langfuse v4).
4. **Where do sub-agents hand off to other sub-agents?** — supervisor → worker, parallel fan-out, graph node transitions. These are the spots that need `propagate_attributes()` (or the framework equivalent) so session/user/tag context survives the boundary.

Capture this as a small topology diagram in the plan (ASCII or Mermaid). It does not need to be exhaustive — it needs to give the customer a single picture they can point at while implementing.

### Step 3 — Decide naming and tagging

Read [`references/naming-conventions.md`](references/naming-conventions.md) and apply it to the topology from Step 2:

- One trace name per externally meaningful unit (`chat-turn`, `summarize-thread`, `nightly-research-job`).
- One observation name per role (`supervisor`, `research-agent`, `writer-agent`) and per tool (`tool.<tool-id>`).
- Tags: stack identifier, environment (`dev` / `staging` / `prod`), agent version, feature flag arm if relevant.
- Metadata keys: a small consistent set so dashboards can pivot — `agent_role`, `tool_id`, `model`, `prompt_version`, `user_tier`, `gateway_origin` (when Ratel is present).

The plan should list every name/tag/metadata key it introduces in one table the customer can paste into a shared doc. Skills #2 and #3 read this table; if it's missing they can't function.

### Step 4 — Decide Ratel-aware hooks (only if Ratel is or will be present)

Check whether `@ratel-ai/sdk`, `ratel-ai-core`, or the `ratel-mcp` / `@ratel-ai/mcp-server` package appears anywhere in the manifest. If yes — or if the customer is signing up to add Ratel as part of this engagement — read [`references/ratel-hooks.md`](references/ratel-hooks.md) and add a section to the plan covering:

- Mapping each Ratel `TraceEvent` variant onto a Langfuse observation (`Search` → observation type `tool` named `ratel.search_tools`, `InvokeStart`/`InvokeEnd` → observation type `tool` named `ratel.invoke_tool`, etc.).
- Required metadata on each observation: `gateway_origin` (`direct` vs `agent`), `top_k`, `hit_count`, `replace_mode`, score of top hit, latency.
- A "before / after" annotation strategy so the customer can run an A/B comparison once Ratel is wired in.

If Ratel is not present and there is no plan to introduce it, **skip this section entirely**. Do not pre-bake a Ratel sales pitch into a customer-owned doc — keep the plan honest.

### Step 5 — Write the plan

Write the plan to `<repo>/docs/ratel-langfuse-instrument.md` (create the `docs/` directory if it doesn't exist; ask the user to confirm the path if the repo already uses a different docs convention).

The plan must contain, in this order:

1. **Summary** — one paragraph: stack detected, agent topology, what's already instrumented (if anything), what this plan adds.
2. **Setup** — SDK install commands, env vars, Langfuse MCP server registration steps (for the customer's Claude Code / Cursor), and a working "hello trace" snippet they can paste and verify.
3. **Topology** — the diagram from Step 2.
4. **Naming, tagging, metadata vocabulary** — the table from Step 3.
5. **Per-file changes** — file path, what to wrap, which observation type to use, what name/tags/metadata to attach. Cite the matching pattern from the stack reference file rather than re-deriving it.
6. **Ratel hooks** (conditional, per Step 4).
7. **Verification checklist** — copy from [`references/verification-checklist.md`](references/verification-checklist.md): six items the customer can tick once instrumentation lands.

Print the table of contents inline in the chat — six bullets max — and tell the user the file path. Do not paste the full plan body into the chat; the file is the artifact.

## Honest skip path

If after Step 1 you cannot find a single LLM client import, agent loop, or model call in the codebase, stop. Do not write a plan. Tell the user:

> No agent surface detected — only checked `<files looked at>`. If this codebase has agent code in a non-standard location, point me at it and I'll re-run.

Forced instrumentation plans on a non-agent codebase produce dead documents and waste partner trust. Better to skip and ask.

If the stack is one we don't yet have a reference file for (e.g., Ruby, Go, or a niche framework), still produce a plan — but mark the stack-specific sections "by analogy with the Python generic reference" and ask the user whether to spawn a follow-up to author a new reference. Don't fake confidence.

## Reference files

- [`references/stack-vercel-ai-sdk.md`](references/stack-vercel-ai-sdk.md)
- [`references/stack-typescript-generic.md`](references/stack-typescript-generic.md)
- [`references/stack-python-generic.md`](references/stack-python-generic.md)
- [`references/stack-python-agentic.md`](references/stack-python-agentic.md)
- [`references/naming-conventions.md`](references/naming-conventions.md) — shared with `/ratel-langfuse-dashboards` and `/ratel-langfuse-analyze`
- [`references/ratel-hooks.md`](references/ratel-hooks.md)
- [`references/verification-checklist.md`](references/verification-checklist.md)
