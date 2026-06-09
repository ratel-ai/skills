---
name: ratel-assessment
description: |
  Read a partner's agent codebase end-to-end and produce a polished assessment report that names concrete weaknesses with file-level evidence, threads Ratel-relevant findings through the report, and ends with conditional pointers to the right follow-up skills. Static-only by default; if a Langfuse project is already wired and reachable via MCP, pull a small live sample to enrich findings (graceful degrade if not). Use whenever the user wants a first-touch review of a partner agent, asks "assess our agent", "review our agent", "audit this agent", "where can we improve", "what would Ratel notice in this codebase", "give us an honest read of our agent", or invokes `/ratel-assessment`. Trigger on phrases like "look at this agent and tell us what's weak", "we want a Ratel-flavored review", "first-pass audit before we engage", "spot the low-hanging fruit in this repo" — even if the user doesn't say "skill" or "ratel-assessment" by name. This skill writes a markdown report; it does not edit the agent code itself.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
---

# /ratel-assessment — front-door audit of a partner agent codebase

The first conversation with a partner startup is rarely "wire up Ratel." It is "show us you understand our agent." This skill is the front door: it reads the codebase, runs every dimension in the assessment catalog, and produces a markdown report the partner could (and would) share internally — whether or not they end up using Ratel.

The deliverable is a polished, evidence-led report at `<repo>/docs/ratel-assessment-<YYYY-MM-DD>.md`. The report carries credibility because every finding cites a file path, a line range, or a concrete count. It carries narrative because the Ratel-relevant findings cluster into a "Where Ratel fits" section that ties them to specific shipped features and roadmap versions. It carries momentum because it ends with conditional pointers to the right follow-up skill from the suite ([`/ratel-langfuse-instrument`](../ratel-langfuse-instrument/SKILL.md), [`/ratel-langfuse-dashboards`](../ratel-langfuse-dashboards/SKILL.md), [`/ratel-langfuse-analyze`](../ratel-langfuse-analyze/SKILL.md), [`/ratel-integrate`](../ratel-integrate/SKILL.md)).

This skill complements the rest of the suite:

- **Runs before** the other four. It is the only one safe to fire on a cold repo with no setup from the partner.
- **Routes into** the rest. The "Recommended next steps" section names which follow-up is appropriate given what was found.
- **Does not duplicate** the catalogs of the others. Ratel angles point to [`../ratel-langfuse-dashboards/references/ratel-value-map.md`](../ratel-langfuse-dashboards/references/ratel-value-map.md); naming references point to [`../ratel-langfuse-instrument/references/naming-conventions.md`](../ratel-langfuse-instrument/references/naming-conventions.md); the trace-event mapping comes from [`../ratel-langfuse-instrument/references/ratel-hooks.md`](../ratel-langfuse-instrument/references/ratel-hooks.md).

## Philosophy

Three rules. Break any of them and the report becomes a sales document, which the partner will see through immediately.

1. **Every finding must cite evidence.** A file path with a line range, a count, or a quote from the codebase. Vague claims ("consider improving observability") earn nothing. Specific claims ("47 tools registered across `src/agents/*.ts`, none with `description` longer than 12 words") earn the next meeting. If you can't cite, drop the finding.
2. **Honest skip beats fabrication.** If the codebase is not an agent, say so and stop — do not invent findings. If a dimension has nothing wrong with it, mark it Strong and move on; do not stretch to find a flaw. Partners notice the absence of fluff faster than they notice its presence.
3. **Ratel is a finding category, not a conclusion.** Ratel angles attach to findings that genuinely match Ratel's surface (tool sprawl, retrieval, decomposition, observability). They do not get bolted onto every finding to inflate the "Where Ratel fits" section. If only two findings legitimately map to Ratel, the section has two entries — not eight padded ones.

## Workflow

### Step 1 — Detect the agent codebase

Read manifests and grep for agent surfaces. The customer's stack drives which patterns the rest of the workflow looks for.

```bash
# Manifests
test -f package.json && jq -r '.dependencies // {}, .devDependencies // {} | keys[]' package.json | sort -u
test -f pyproject.toml && head -200 pyproject.toml
test -f requirements.txt && cat requirements.txt
test -f uv.lock && head -50 uv.lock

# Agent surfaces — tool definitions, LLM clients, agent frameworks
grep -rEn 'tools:\s*\{|tools:\s*\[|defineTool|createTool|@tool\b|@function_tool\b|McpServer\(|StdioClientTransport|listTools|streamText|generateText|chat\.completions|messages\.create|invoke_model|graph\.add_node|Agent\(|Crew\(|@agent\b' \
  --include='*.ts' --include='*.tsx' --include='*.js' --include='*.py' \
  | head -100
```

Classify into a stack using [`references/stack-detection.md`](references/stack-detection.md). The stack drives where to look for tools, prompts, sub-agents, and observability config. If no agent surface is detected at all, use the [honest skip path](#honest-skip-path).

### Step 2 — Inventory the agent surface

Launch one **Explore** agent (or do it directly for small repos) with a single, scoped prompt. The agent's job is to fill an inventory the assessment catalog will then evaluate. Ask it to map:

1. **Agent entry points** — HTTP handlers, queue consumers, CLI verbs, chat-platform webhooks. One bullet per entry point with file path.
2. **Tool definitions** — every place tools are declared (registries, decorators, MCP client setup). Total count. For each tool: id, name, description length, schema presence, inline vs externalized.
3. **Prompt templates** — every system / user / tool prompt. Where they live (inline string, separate file, prompt-management service). Approximate token weight if the file makes that easy to estimate.
4. **Sub-agent boundaries** — supervisor / worker / role-specialized agents; handoff sites; whether sub-agents are explicit (named) or implicit (the same loop with different prompts).
5. **Session/context handling** — where the conversation thread / job correlation id originates; whether anything propagates it through sub-agents.
6. **Model routing** — which models get called for which tasks; whether routing is task-aware or uniform.
7. **Error handling** — try/except shape around tool calls; retry presence; backoff; dead-letter.
8. **Memory / retrieval** — vector store, RAG pipeline, conversation memory, summarization.
9. **Observability config** — Langfuse / Langsmith / OTel / OpenInference / OpenLLMetry / Helicone / homegrown logging. Env vars, init sites.
10. **Eval surface** — eval directory, fixtures, ground-truth dataset, CI invocations.

Keep the inventory inside the analysis context — it is not a deliverable. The report cites it; it does not include it.

### Step 3 — Detect observability and probe Langfuse if reachable

Two checks:

1. **Static**: are any of the following present? Langfuse SDK in manifest; Langfuse env vars in `.env*` / sample envs / docker-compose; Langsmith SDK; OTel exporter pointing somewhere agent-shaped; OpenInference / OpenLLMetry instrumentation.
2. **Live (best effort)**: if `LANGFUSE_PUBLIC_KEY` and `LANGFUSE_SECRET_KEY` are reachable and the Langfuse MCP server is configured for this session, attempt one cheap aggregate query (e.g., trace count in the last 24h on the most recent trace_name). If it returns data, fetch a small sample (≤50 traces) and use it to enrich the relevant findings (tool surface count, error rate, latency). If the MCP call fails for any reason — auth, network, no project, no traces — **fall back silently** to static-only. Do not surface the failure as a finding; it is not the partner's bug.

The catalog distinguishes between findings that need live data ("Untyped tool observations" — observable only with traces) and findings that don't ("Tool sprawl by count" — observable statically). Static-only runs simply skip the live-data findings; they don't degrade the rest.

### Step 4 — Run the assessment

Walk every dimension in [`references/assessment-catalog.md`](references/assessment-catalog.md). For each:

- Apply the detection heuristic against the inventory from Step 2 (and the live sample from Step 3 if available).
- Decide severity per the rubric: Critical / Major / Minor / Info.
- Attach evidence: file path + line range, count, or quoted snippet.
- If the catalog entry has a Ratel angle, attach it as a one-line tag pointing to the relevant Ratel feature/version from [`../ratel-langfuse-dashboards/references/ratel-value-map.md`](../ratel-langfuse-dashboards/references/ratel-value-map.md).

Do not invent dimensions not in the catalog. If you spot something genuinely novel, note it inline but mark it `dimension: ad-hoc` in the report so the catalog can absorb it later.

### Step 5 — Score the scorecard

Each of the ten dimensions gets exactly one of:

- **Strong** — well-handled; no findings worse than Info.
- **Adequate** — works; one or two Minor findings; no Major or worse.
- **Weak** — material gaps; at least one Major finding.
- **Missing** — dimension is absent or broken; at least one Critical finding.

The scorecard goes at the top of the report. It is the first thing the partner reads. Make it accurate; do not soften.

### Step 6 — Generate the report

Output to `<repo>/docs/ratel-assessment-<YYYY-MM-DD>.md` using the structure in [`references/report-template.md`](references/report-template.md). Date-stamped so multiple runs accumulate without overwriting.

Sections, in order:

1. **Executive summary** — one paragraph, then the scorecard table.
2. **Findings**, grouped by severity (Critical → Major → Minor → Info). Each finding: title, dimension, severity, evidence, why it matters, recommendation, optional one-line Ratel angle.
3. **Where Ratel fits** — consolidates the threaded Ratel angles into one narrative. Tied to specific Ratel features / roadmap versions from the value map. If only one or two findings carry Ratel angles, this section is short — that's fine.
4. **Recommended next steps** — conditional skill pointers based on what was found. Bullet list, each pointer one line. See [Conditional pointers](#conditional-pointers) below.

### Step 7 — Inline summary

Print to chat, in this order:

1. The scorecard table (or a compact one-line version: "Topology Adequate · Tools Weak · Context Strong · ...").
2. The top 3 findings with one-line summaries each.
3. The file path of the full report.
4. The recommended next-step skill(s).

Do not paste the full report body into chat. The file is the artifact.

## Conditional pointers

The "Recommended next steps" section is conditional on findings. Never list a skill that does not apply. Map findings to pointers as follows:

| If the assessment found... | Point to |
| --- | --- |
| Any Observability finding (`Weak` or `Missing` dimension) | `/ratel-langfuse-instrument` |
| Observability is `Strong` *or* `Adequate` but no dashboards mentioned | `/ratel-langfuse-dashboards` |
| Observability is `Strong` *and* live data was reachable | `/ratel-langfuse-analyze` |
| Any Tool surface finding tagged with a Ratel angle | `/ratel-integrate` |
| Decomposition / topology Ratel-angle finding present | `/ratel-integrate` (with a note that decomposition support is roadmap v0.1.10) |
| None of the above | omit the section entirely |

Each pointer is one line: skill name + the *specific* finding from this report it follows up on. Do not paraphrase the skill descriptions; the partner can read them.

## Tone and evidence

- **Imperative voice, second person.** "Wrap tool calls as `type: tool` observations" — not "It might be a good idea to..."
- **No emojis. No marketing adjectives.** "Critical" carries weight only because the report uses the word sparingly.
- **No hedging on severity.** If it's a Critical, write Critical. If it's Info, do not inflate it.
- **Evidence-first finding bodies.** First paragraph cites evidence; second explains why it matters; third gives the recommendation. The Ratel angle is one line at the end, not woven through.
- **Time-bind nothing in the report.** Avoid "in the last sprint" / "since this morning." The partner re-reads this in three months.

## Honest skip path

If after Step 1 you cannot find any LLM client import, tool definition, agent framework, or model call in the codebase, stop. Do not write a report. Tell the user:

> No agent surface detected — checked `<files looked at>`. If the agent lives somewhere unusual, point me at it and I'll re-run. Otherwise this codebase isn't a fit for `/ratel-assessment`.

If the codebase is an agent but the catalog finds *nothing worse than Info* across all ten dimensions, write the report anyway — the scorecard will read all-Strong or near-all-Strong, the findings section will be small, and the conclusion will be "this agent is in good shape; the Ratel-relevant angle is `<one-or-zero items>`." That is itself a credibility-building report; do not pad it.

## Reference files

- [`references/assessment-catalog.md`](references/assessment-catalog.md) — the ten dimensions, detection heuristics, severity rubrics, recommendations, and Ratel angles. The load-bearing file.
- [`references/stack-detection.md`](references/stack-detection.md) — quick playbook for classifying the agent stack; cross-links to `/ratel-langfuse-instrument`'s per-stack files for deeper detail.
- [`references/report-template.md`](references/report-template.md) — canonical report structure and a worked example.

Reads from (does not duplicate):

- [`../ratel-langfuse-instrument/references/naming-conventions.md`](../ratel-langfuse-instrument/references/naming-conventions.md) — shared trace/observation/tag/metadata vocabulary
- [`../ratel-langfuse-instrument/references/ratel-hooks.md`](../ratel-langfuse-instrument/references/ratel-hooks.md) — Ratel trace events → Langfuse observation mapping
- [`../ratel-langfuse-dashboards/references/ratel-value-map.md`](../ratel-langfuse-dashboards/references/ratel-value-map.md) — the single source of truth for Ratel feature → observable signal → version
