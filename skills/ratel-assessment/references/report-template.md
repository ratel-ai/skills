# Report template

The canonical structure of the assessment report. The skill writes the report by walking [`assessment-catalog.md`](assessment-catalog.md), filling findings under each dimension, and then assembling the sections below in order.

The worked example at the bottom is a fictional partner — use it to calibrate tone, density, and evidence-citation style. Do not copy any of its findings into a real report unless they match real evidence in the partner's codebase.

## Structure

### Header

```markdown
# Ratel assessment — <partner / repo name>

**Date**: <YYYY-MM-DD>
**Stack**: <vercel-ai-sdk | ts-generic | python-generic | python-agentic | mixed>
**Scope**: <which entry points / agents were assessed>
**Data sources**: static code analysis<; live Langfuse sample (N traces, last 24h)>
```

### 1. Executive summary

One paragraph (4–6 sentences). What the agent is for, the headline scorecard read, the two or three findings worth raising in a partner meeting, and one sentence on what's strong.

Then the scorecard:

```markdown
| Dimension | Score |
| --- | --- |
| Agent topology | Strong / Adequate / Weak / Missing |
| Tool surface | ... |
| Context management | ... |
| Decomposition | ... |
| Model routing | ... |
| Error handling | ... |
| Observability | ... |
| Cost discipline | ... |
| Eval / quality gates | ... |
| Safety | ... |
```

### 2. Findings

Grouped by severity. Within a severity, order by dimension. Each finding follows this shape:

```markdown
### <Short title>

- **Dimension**: <dimension name>
- **Severity**: <Critical | Major | Minor | Info>
- **Evidence**: <file path with line range, count, or short quoted snippet>

<One paragraph: why it matters in this codebase.>

**Recommendation**: <One-to-two-sentence concrete action.>

**Ratel angle** (optional): <One line tied to a specific Ratel feature / version from the value map. Omit the line entirely if there is no angle.>
```

Skip an empty severity tier. If there are no Critical findings, the section starts at Major.

### 3. Where Ratel fits

A short narrative (3–8 sentences, plus a short bulleted list if helpful) consolidating the threaded Ratel-angle tags into one story. Tied to specific Ratel feature/version anchors from [`../../ratel-langfuse-dashboards/references/ratel-value-map.md`](../../ratel-langfuse-dashboards/references/ratel-value-map.md). Be honest about shipped vs roadmap.

If only one or two findings carried Ratel angles, this section is short — say so plainly ("Two of the findings above are addressable with Ratel today: ...") and stop. Do not pad.

If no findings carried Ratel angles, omit this section entirely. Do not invent a connection.

### 4. Recommended next steps

Conditional pointers per the mapping table in the main `SKILL.md`. Each pointer is one line, naming the specific finding(s) that drive it:

```markdown
- `/ratel-langfuse-instrument` — addresses *No observability wired* (Critical, Dimension 7).
- `/ratel-integrate` — addresses *Tool sprawl* (Major, Dimension 2) and *Bloated tool descriptions* (Major, Dimension 2).
```

If no findings warrant any follow-up, omit this section.

### 5. Appendix (optional)

Two things go in the appendix if relevant:

- **Inventory snapshot** — a compact list of the surfaces the assessment looked at (entry points, tool count, prompt files, observability config). Useful as a snapshot the partner can diff against next quarter.
- **Live data caveats** — if a Langfuse sample was pulled, note the window, trace count, and any caveats (sample is non-representative, only one trace_name, env filter applied). If no live data was pulled, omit.

Do not include a full per-tool dump or per-prompt dump. The catalog cites — it does not catalog.

## Formatting rules

- **Headings**: `#` for the title, `##` for sections (1–5 above), `###` for findings.
- **No emojis. No marketing adjectives.** "Critical" carries weight only because the report uses the word sparingly.
- **No links to internal Ratel issues / tickets / Slack threads.** The report is shared with the partner; keep it standalone.
- **One trailing newline** at end of file.
- **Date in the filename, not in the headings beyond the header block.** Re-reading two reports side by side is the use case — keep dates out of body content.

---

## Worked example

What follows is a fictional assessment of a fictional partner ("Northcrop AI"). It hits each section so you can calibrate density and tone. Real reports will be longer or shorter; the structure stays the same.

```markdown
# Ratel assessment — Northcrop AI / research-agent

**Date**: 2026-06-09
**Stack**: vercel-ai-sdk
**Scope**: `src/agents/research-agent/*` and the `chat-turn` entry point at `src/api/chat/route.ts`. Out of scope: the offline summarization job at `scripts/digest.ts`.
**Data sources**: static code analysis. No live Langfuse data — project has no observability wired yet.

## Executive summary

Northcrop's research agent is a Vercel AI SDK loop that exposes 41 tools to the model on every turn through `src/agents/research-agent/tools.ts:18`. The top-level shape is sound — a single named agent, clean entry point, no recursion — but the tool surface is the dominant cost and quality risk in the codebase today: descriptions are long, several pairs are near-duplicates, and there is no pre-filtering. There is no observability wired anywhere, which means none of the cost or quality claims in this report are independently verifiable today; closing that loop is the highest-leverage next step. The eval suite under `evals/` is real and runs on PRs — strongest dimension in the assessment.

| Dimension | Score |
| --- | --- |
| Agent topology | Strong |
| Tool surface | Weak |
| Context management | Adequate |
| Decomposition | Adequate |
| Model routing | Adequate |
| Error handling | Adequate |
| Observability | Missing |
| Cost discipline | Weak |
| Eval / quality gates | Strong |
| Safety | Adequate |

## Findings

### No observability wired

- **Dimension**: Observability
- **Severity**: Critical
- **Evidence**: no `langfuse`, `langsmith`, `@opentelemetry/*`, `openinference`, `openllmetry`, or `helicone` packages in `package.json`. No telemetry init in `src/api/chat/route.ts`. No tracing env vars in `.env.example`.

The agent runs in production (`README.md` describes a live customer surface) but there is no way for anyone outside the running process to observe what it does. Regressions across deploys are invisible; cost spikes are invisible; per-tool failure modes are invisible. None of the other findings in this report are independently verifiable without this loop closed.

**Recommendation**: wire Langfuse via the patterns in `/ratel-langfuse-instrument` before any other change in this report. Two-day landing window is typical.

**Ratel angle**: routes to `/ratel-langfuse-instrument` — and downstream, once data is flowing, `/ratel-langfuse-dashboards` to build the cost and retrieval-quality dashboards that will measure the rest of the report's recommendations.

### Tool sprawl on the chat turn

- **Dimension**: Tool surface
- **Severity**: Major
- **Evidence**: `src/agents/research-agent/tools.ts:18-487` registers 41 tools. All are passed in `tools:` on `generateText` at `src/api/chat/route.ts:42`. Average description length 187 tokens (rough estimate from character count); longest is `web_research_brief` at 612 tokens.

Every chat turn sends all 41 tool descriptions to the model. At ~140 tokens average × 41 tools, the tool catalog alone is ≈5,700 input tokens before any user content or conversation history. This is most of the input cost on a typical turn and a significant fraction of context budget on small models.

**Recommendation**: pre-filter the tool list per turn so the model only sees the top-K (typically 8) most-relevant tools for the user's message. The full catalog remains addressable via a discovery surface.

**Ratel angle**: matches Ratel's BM25 tool retrieval + replace-mode pre-filter (v0.1.5, shipped). Textbook fit for what Ratel was built for; expected input-token reduction is 50–85% on the catalog portion of the prompt.

### Bloated tool descriptions

- **Dimension**: Tool surface
- **Severity**: Major
- **Evidence**: `src/agents/research-agent/tools.ts` median description 156 tokens. Five tools (`web_research_brief`, `crawl_pages`, `summarize_findings`, `cite_sources`, `synthesize_report`) have descriptions over 350 tokens each, all with multi-paragraph examples inline.

Long descriptions inflate the catalog cost (compounding finding 2.a) and also confuse the model's tool selection — the inline examples often mention other tools, which the model treats as routing hints.

**Recommendation**: trim descriptions to one short "what it does" sentence + one "when to use" line. Move examples and edge-case detail to a separate spec document the agent does not see every turn.

**Ratel angle**: matches Ratel's BM25 retrieval (v0.1.5, shipped) — the dashboard surfaces low top-hit scores for confusing descriptions, giving a direct measurement of which ones to rewrite. The roadmap entry for LLM-driven suggestions (v0.1.9) will eventually propose the rewrites automatically.

### Near-duplicate tools

- **Dimension**: Tool surface
- **Severity**: Major
- **Evidence**: three pairs flagged — `web_research_brief` vs `crawl_pages`; `summarize_findings` vs `synthesize_report`; `cite_sources` vs `format_citations`. Descriptions overlap >40% by token bag.

The model picks among near-duplicates inconsistently, which makes the agent's behavior across turns less predictable than it should be. Several open issues in `evals/results/` mention "wrong tool selected" without naming the disambiguation.

**Recommendation**: consolidate each pair, or rewrite descriptions to be sharply distinguishing ("brief: fast, free-text overview" vs "crawl: deep, structured fetch with page bodies").

**Ratel angle**: matches Ratel's BM25 retrieval (v0.1.5, shipped) — top-hit-score distribution will surface the duplication empirically.

### No max_tokens cap on chat-turn generations

- **Dimension**: Cost discipline
- **Severity**: Major
- **Evidence**: `src/api/chat/route.ts:42-58` calls `generateText` without `maxTokens`. Same pattern in `src/agents/research-agent/summarizer.ts:31`.

A worst-case generation can fill the model's context budget. On a Sonnet-class model that is a 16K+ token output for a turn that frequently only needs 500 tokens.

**Recommendation**: cap per call. Sensible defaults: 4096 for chat turns, 1024 for the summarizer step.

### Anemic prompt versioning

- **Dimension**: Context management
- **Severity**: Minor
- **Evidence**: system prompt is an inline template string in `src/agents/research-agent/prompt.ts:5`. No version constant, no hash, no git-tracked prompt id.

Prompt regressions across deploys will be undetectable once observability is wired.

**Recommendation**: extract the prompt to `prompts/research-agent.v1.md` and attach a `prompt_version` constant. Roll the version on every change.

### Eval suite covers tasks but not tool selection

- **Dimension**: Eval / quality gates
- **Severity**: Minor
- **Evidence**: `evals/research-agent.jsonl` has 84 labeled tasks with expected outputs but no `expected_tool_id` field. CI gate in `.github/workflows/evals.yml` runs on every PR.

The eval suite catches output regressions but cannot detect tool-selection regressions (e.g., the agent producing a correct-looking answer using the wrong tool). Given the tool sprawl finding above, this is worth closing.

**Recommendation**: label `ground_truth_tool_id` (or the equivalent list of acceptable tool ids) on each fixture. The catalog metadata vocabulary defines the key.

**Ratel angle**: matches the Retrieval Quality dashboard's `recall@5` widget (v0.1.5, shipped) — once ground truth exists, retrieval quality becomes measurable in dashboards and CI.

## Where Ratel fits

Three findings in this report (tool sprawl, bloated descriptions, near-duplicates) are the textbook fit for Ratel's shipped v0.1.5 surface: BM25 tool retrieval with replace-mode pre-filter, paired with the Retrieval Quality dashboard. Once observability lands, the dashboard will show the input-token reduction directly and surface low top-hit scores for the descriptions that need rewriting.

The ground-truth labeling finding (Eval / quality gates, Minor) unlocks the Retrieval Quality dashboard's `recall@5` widget, which closes the measurement loop on the integration: not just "we sent fewer tokens" but "we sent the *right* tools." This is the partner-facing proof.

The roadmap entry for LLM-driven suggestions (v0.1.9) will eventually propose the description rewrites this report calls out manually. Honest timing: that's roadmap, not shipped. Mention only if asked.

## Recommended next steps

- `/ratel-langfuse-instrument` — addresses *No observability wired* (Critical, Dimension 7). Required before any of the below can be measured.
- `/ratel-integrate` — addresses *Tool sprawl on the chat turn*, *Bloated tool descriptions*, and *Near-duplicate tools* (all Major, Dimension 2). Pilot scope should be the `chat-turn` trace, A/B split via a feature flag.
- `/ratel-langfuse-dashboards` — after the above land, build the Token Cost & Savings and Retrieval Quality dashboards to measure the integration's impact.

## Appendix — inventory snapshot

- Entry points: `src/api/chat/route.ts:42` (single).
- Sub-agents: `src/agents/research-agent/summarizer.ts:14` (called from the main loop; not separately tool-equipped).
- Tools: 41 registered in `src/agents/research-agent/tools.ts`. All used at one call site.
- Prompts: 1 inline template (`src/agents/research-agent/prompt.ts:5`). No versioning.
- Observability: none.
- Evals: 84 fixtures in `evals/research-agent.jsonl`; CI gate in `.github/workflows/evals.yml`.
```

That is the worked example. The structure is the contract; the content varies per repo.
