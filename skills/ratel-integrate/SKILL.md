---
name: ratel-integrate
description: |
  Inspect a customer codebase, figure out how it manages tools and which agent framework it uses, fetch Ratel documentation, and propose a plan to integrate Ratel's tool-search gateway (BM25 retrieval + `search_tools` / `invoke_tool` gateway tools). The plan includes the integration mode (direct SDK vs MCP gateway vs hybrid), an A/B test design, and the Langfuse metrics that will prove the integration is working. Use whenever the user wants to add Ratel to a partner codebase, asks "wire up Ratel here", "how would Ratel fit", "integrate the gateway", "add tool search", "Ratel pilot for this customer", or invokes `/ratel-integrate`. Trigger on phrases like "let's add Ratel", "integrate the context gateway", "set up the Ratel SDK in this repo", "plan a Ratel rollout for them", "Ratel A/B for this agent" — even if the user doesn't say "skill" or "ratel-integrate" by name. This skill writes a markdown plan and asks clarifying questions when the codebase is ambiguous; it does not edit the agent code itself.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
  - WebFetch
---

# /ratel-integrate — plan a Ratel rollout for a customer agent

Most partner engagements eventually want Ratel itself in the picture, not just observability around it. This skill turns "let's pilot Ratel here" into a concrete week-one plan: which integration mode to use, which tools to pilot first, how to A/B test the impact, and which Langfuse metrics from the trio (`/ratel-langfuse-instrument`, `/ratel-langfuse-dashboards`, `/ratel-langfuse-analyze`) will tell you whether it worked.

The deliverable is a markdown plan the customer can implement and a clear answer to the question "how will we know if it helped." Both halves matter.

This skill complements the Langfuse trio:

- **Run after** [`/ratel-langfuse-instrument`](../ratel-langfuse-instrument/SKILL.md) so the customer's trace vocabulary (sessions, feature_flag tags, `metadata.gateway_origin`) is already in place. If it isn't, the skill will point them back there before continuing.
- **Drives** [`/ratel-langfuse-dashboards`](../ratel-langfuse-dashboards/SKILL.md) — the Ratel-value dashboards in that catalog measure the integration this skill is planning. The skill should explicitly name which dashboards the customer should build / refresh after the rollout.
- **Feeds** [`/ratel-langfuse-analyze`](../ratel-langfuse-analyze/SKILL.md) — once traffic is flowing under the A/B split, the analyze skill will surface findings from the integration.

## Philosophy

Three rules from past partner engagements that the plan should follow:

1. **Don't migrate the whole catalog in one shot.** Pick a pilot scope (one trace_name, one agent role, or a subset of tools) and prove the lift before broadening. Big-bang migrations bury the win in confounding factors.
2. **Always A/B.** A Ratel rollout without a control arm produces inconclusive numbers no matter how good the win is. The plan must include the A/B strategy, even if the strategy is "ship behind a flag at 10% and ramp."
3. **Pick the simplest integration mode that works.** Direct SDK in the agent's process beats MCP gateway for raw control; MCP gateway beats direct SDK when the customer is already speaking MCP. Don't over-architect — Ratel is a library, not a platform.

## Workflow

### Step 1 — Detect stack and tool management approach

Read manifest files and scan how the customer's agent learns about tools today. Concretely:

```bash
# Manifest
test -f package.json && jq -r '.dependencies // {}, .devDependencies // {} | keys[]' package.json | sort -u
test -f pyproject.toml && cat pyproject.toml
test -f uv.lock && head -50 uv.lock

# Tool registration sites
grep -rEn 'tools:\s*\{|tools:\s*\[|\.register\(|@tool\b|new Tool|defineTool|createTool|registerTool|McpServer\(|StdioClientTransport|listTools' \
  --include='*.ts' --include='*.tsx' --include='*.js' --include='*.py' \
  | head -100
```

Classify the tool-management approach into one of these buckets (impacts the integration mode in step 4):

| Approach in codebase | Signal | Likely integration mode |
| --- | --- | --- |
| Static tool list on every model call | `tools: { ... }` literal passed to `generateText` / `chat.completions.create` / `messages.create` | Direct SDK, replace-mode pre-filter |
| Dynamic registry + dispatcher | A central tool map + a `dispatch(toolId, args)` function | Direct SDK, replace-mode pre-filter (easiest swap) |
| MCP client consuming upstream servers | `Client` from `@modelcontextprotocol/sdk` / `mcp.client` | MCP gateway (Ratel ingests upstreams, agent talks to Ratel) |
| Mixed (some local tools + some MCP) | both signals present | Hybrid — Direct SDK for local + Ratel ingestion for MCP |
| LangGraph / CrewAI node tools | framework-managed tool surfaces | Direct SDK at the node boundary, framework-agnostic |

If after this step you cannot find any tools at all, use the [honest skip path](#honest-skip-path).

### Step 2 — Map the agent topology relevant to Ratel

Run an Explore agent (or do it directly for small repos) to answer:

1. **Where is the LLM call that takes a `tools:` parameter?** — that's the integration site for pre-filtering.
2. **What's the catalog size today and what's expected at steady state?** — Ratel's lift grows with catalog size; under ~15 tools the win is too small to justify the integration, and the plan should say so.
3. **Is there a single dispatcher** (good — drop-in replace) **or are tools dispatched inline** (need a small refactor)?
4. **What's the user-facing latency budget?** — Ratel's retrieval adds <1ms, but the customer should know.

Capture this in 4-6 bullets in the plan.

### Step 3 — Fetch up-to-date Ratel documentation

Ratel ships fast (every minor version every few weeks on the v0.1.x line). Don't recite from memory. Pull the current state at runtime.

Tier 1: try [Context7](https://github.com/upstash/context7) via the available MCP tools. Resolve the library id for `ratel-ai/ratel` and fetch the README + SDK README + CLI README. This gives you whatever version's current.

Tier 2 (fallback): WebFetch raw GitHub READMEs. Canonical paths:

```
https://raw.githubusercontent.com/ratel-ai/ratel/main/README.md
https://raw.githubusercontent.com/ratel-ai/ratel/main/src/sdk/ts/README.md
https://raw.githubusercontent.com/ratel-ai/ratel/main/src/integrations/cli/README.md
https://raw.githubusercontent.com/ratel-ai/ratel/main/src/integrations/mcp/README.md
```

Tier 3 (only if the customer already has Ratel installed): read the package's own README from `node_modules/@ratel-ai/sdk/README.md` or the Python site-packages equivalent. This is the most accurate for *their* pinned version.

Capture three things from whatever docs you read: the **current shipped version**, the **public API for tool registration / search / invoke**, and the **MCP gateway tool names**. If the public API has changed since the patterns in [`references/integration-patterns.md`](references/integration-patterns.md), trust the fetched docs and call out the discrepancy in the plan so the integration-patterns file gets updated next.

### Step 4 — Decide the integration mode

Based on Step 1's classification and Step 3's docs, pick one (and only one) primary integration mode:

- **Direct SDK** — import the Ratel SDK in the agent process, register tools into a `ToolCatalog`, swap the agent's tool list for `catalog.search(query, topK).map(asToolDef)` (replace mode) or expose the two gateway tools.
- **MCP gateway** — run `ratel serve` (or `@ratel-ai/mcp-server`) as a process; configure the customer's agent to talk to it via MCP. Their existing tool sources get ingested into the gateway as upstreams.
- **Hybrid** — Direct SDK for the agent's local tools; MCP gateway as one of the agent's MCP clients for upstream-provided tools. Only recommend this when both kinds of tool surfaces exist.

The plan should state the choice and the reason in one sentence ("Direct SDK because there's a single dispatcher in `src/agent/dispatch.ts:42` and no MCP upstreams").

Read [`references/integration-patterns.md`](references/integration-patterns.md) for the per-mode setup and the per-framework code shape.

### Step 5 — Pick the pilot scope

Don't migrate everything. Recommend a pilot scope:

- **By trace_name**: pilot on the single trace_name with the highest token spend per turn (the customer can confirm from `/ratel-langfuse-analyze` aggregates).
- **By agent role**: pilot on one sub-agent (e.g., `research-agent`) and leave the supervisor alone.
- **By tool subset**: pilot with just the top-50 most-called tools registered, leaving the long tail out for v1.
- **By traffic**: ship behind a flag at 10% and ramp on green metrics.

Pick one or two of these and justify. State explicitly what is **out of pilot scope** so the customer doesn't accidentally widen.

### Step 6 — Design the A/B test

Read [`references/ab-test-patterns.md`](references/ab-test-patterns.md). Pick a strategy and customise to the codebase:

- **Live feature flag** (preferred when traffic is healthy): tag the trace `feature_flag=tool_pool=ratel` vs `tool_pool=full`. Both arms run on real traffic.
- **Shadow mode** (when production risk is high): production keeps the original path; the Ratel path runs in parallel, logs to Langfuse, but its output isn't returned to the user.
- **Replay** (when traffic is too thin for a live split): collect inputs from the original path into a Langfuse dataset; replay through Ratel afterwards.

For each: state the trace tags / metadata the customer must emit so the dashboards from `/ratel-langfuse-dashboards` light up correctly.

If the codebase doesn't have an existing flagging pattern, **ask the user** before recommending one of your own. Common patterns to ask about: feature flag SaaS (LaunchDarkly, Statsig, GrowthBook), env-var split, percent-of-user hashing, internal experimentation framework.

Sample prompt to the user when in doubt:

> The codebase doesn't have an obvious feature-flag layer for this A/B. Do you have an internal pattern for traffic splits — e.g., a LaunchDarkly client, env-based toggles, or a percentage rollout helper — or should I propose a minimal one inline in the plan?

### Step 7 — Tie to Langfuse metrics

The integration is worthless if no one can prove it worked. Name the **exact Langfuse dashboards and scores** that measure this rollout, sourced from [`ratel-langfuse-dashboards/references/ratel-value-map.md`](../ratel-langfuse-dashboards/references/ratel-value-map.md):

- **Token Cost & Savings** dashboard — the headline. Split by `feature_flag` tag. The plan must guarantee `input_tokens` per `chat-turn` trace will land in the arm tag correctly.
- **Retrieval Quality** dashboard — needs `ratel.search_tools` observations with `top_hit_score`, `hit_count`, `top_k`. The plan must specify these get emitted (see Langfuse instrumentation's [`ratel-hooks.md`](../ratel-langfuse-instrument/references/ratel-hooks.md)).
- **Gateway Origin Split** dashboard — needs `metadata.gateway_origin = direct | agent` on every Ratel observation.
- **Scores** — recommend wiring `tool_selection_accuracy` and `top_k_recall_at_5` if any form of ground truth (gold-labelled tool ids per task, eval dataset) exists.

If the customer has not yet run `/ratel-langfuse-instrument`, **do not proceed** to Step 8. Route them back. Building a Ratel plan that nobody can measure produces an unverifiable engagement.

### Step 8 — Ask for any missing information

Before writing the plan, check what you don't know and ask. The skill must surface its assumptions, not bury them. Common questions:

- Is there a preferred Ratel version to pin to? (default: whatever's `latest` per Step 3)
- Which Ratel feature(s) does the partner most want to validate first — tool retrieval (shipped), gateway origin pattern (shipped), or a roadmap one (v0.1.7 skills, v0.1.9 suggestions, etc.)?
- Is there ground truth labelling for any task, even for a subset? (drives the score-wiring decision)
- Are there cost/latency budgets the integration must not bust?
- Is the agent in production, internal preview, or pre-launch? (changes risk tolerance for A/B)

Group these into one batched question for the user (use `AskUserQuestion` if available, or list them in chat). Don't proceed with the plan until you have the answers.

### Step 9 — Write the plan

Output to `<repo>/docs/ratel-integrate.md`. Sections, in order:

1. **Summary** — stack, tool management approach, integration mode picked, pilot scope, A/B strategy, target Ratel version. Six bullets max.
2. **Up-to-date docs reference** — note the Ratel version the plan was written against and the docs source (Context7 / GitHub raw / installed package).
3. **Topology + tool-management map** — from Steps 1-2.
4. **Integration plan** — file-by-file diff intent: where to register tools, where to swap the tool list / wire the dispatcher / connect the MCP gateway, where to set `metadata.gateway_origin`. Cite [`integration-patterns.md`](references/integration-patterns.md) rather than re-deriving.
5. **A/B test plan** — strategy from Step 6, including the exact trace tag values and the feature-flag wiring choice (deferring to the user's pattern if they provided one).
6. **Metrics & dashboards** — table mapping the Ratel-value dashboards from `/ratel-langfuse-dashboards` to "now / after rollout / after pilot expansion."
7. **Roadmap pointers** — only what's directly relevant to this customer (e.g., if they care about skills, mention v0.1.7; if they care about decomposition, mention v0.1.10). Don't list the whole roadmap.
8. **Open questions** — anything still ambiguous from Step 8.
9. **Verification checklist** — five items the customer can tick after the integration lands: pilot trace_name uses Ratel, `feature_flag` tag is split correctly, `ratel.search_tools` observations appear, Token Cost & Savings dashboard shows separation between arms, Retrieval Quality dashboard has data.

Print the table of contents inline in chat (six bullets max) and tell the user the file path. Do not paste the full plan body into the chat.

## Honest skip path

Three skip cases:

1. **No LLM tool surface in the codebase.** No `tools: { ... }` parameter, no `@tool` decorators, no MCP client. Tell the user there's nothing for Ratel to pre-filter and stop. Don't fabricate a "potential future fit."
2. **Catalog too small (<15 tools).** Ratel's benefit grows with catalog size; under ~15 well-described tools, the integration overhead exceeds the win. Tell the user this and suggest revisiting when the catalog grows.
3. **Langfuse not yet instrumented.** Route to `/ratel-langfuse-instrument` first. A Ratel rollout without observability is not measurable, and an unmeasurable rollout is indistinguishable from no rollout.

## Reference files

- [`references/integration-patterns.md`](references/integration-patterns.md) — per-mode and per-framework integration shapes
- [`references/ab-test-patterns.md`](references/ab-test-patterns.md) — feature-flag, shadow, and replay A/B strategies + how to tag traces so the dashboards split correctly

Reads from (don't duplicate):

- [`../ratel-langfuse-instrument/references/ratel-hooks.md`](../ratel-langfuse-instrument/references/ratel-hooks.md) — Ratel trace event → Langfuse observation mapping
- [`../ratel-langfuse-dashboards/references/ratel-value-map.md`](../ratel-langfuse-dashboards/references/ratel-value-map.md) — Ratel feature → Langfuse signal → dashboard widget
