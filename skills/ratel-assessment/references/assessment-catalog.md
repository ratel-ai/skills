# Assessment catalog

Ten dimensions. For each: what it covers, how to score it, the concrete findings that surface under it, and the Ratel angle where one exists.

**Ratel angles point to entries in [`../../ratel-langfuse-dashboards/references/ratel-value-map.md`](../../ratel-langfuse-dashboards/references/ratel-value-map.md)** — that file is the single source of truth for "what Ratel ships when." Do not invent angles here; if a finding feels Ratel-relevant but doesn't match a value-map entry, leave the Ratel angle off.

**Severity tags throughout**:

- **Critical** — the agent is broken or unsafe in production today.
- **Major** — measurable hit to cost / quality / reliability; will bite within weeks.
- **Minor** — real but contained; the team can defer with low cost.
- **Info** — observation worth noting; not a defect.

**Scorecard mapping per dimension**:

- **Strong** — no findings worse than Info.
- **Adequate** — at most two Minor findings; no Major or Critical.
- **Weak** — at least one Major finding (no Criticals).
- **Missing** — at least one Critical, or the dimension's surface is entirely absent where it should exist.

---

## 1. Agent topology

What it covers: how the agent is structured — single loop, supervisor + workers, graph of nodes — and whether the boundaries are explicit enough to reason about.

**Detection inputs**:

- Count of named agent functions / classes / nodes.
- Presence vs absence of named sub-agents (vs the same loop run with different prompts).
- Handoff sites: where one agent calls another.
- Frameworks that make topology explicit (LangGraph, CrewAI, Mastra agents) vs ones that don't (bare `generateText` loops).

**Common findings**:

### 1.a Flat topology where the agent actually delegates

*Detection*: the codebase has one entry point, one loop, no named sub-agents — but the system prompt mentions delegation ("first plan, then research, then write") or the conversation history shows multi-phase work.

*Severity*: Minor by default; Major if the loop is >200 lines or the prompt is >2000 tokens of role-mixing instructions.

*Recommendation*: split into explicit sub-agents (supervisor + workers). The split is mechanical once the boundaries are named.

*Ratel angle*: matches the "Multi-agent decomposition hints" entry (v0.1.10, roadmap). Ratel will propose decompositions automatically once shipped; until then, the manual split is the right move.

### 1.b Sub-agents exist but handoffs are implicit

*Detection*: agent functions call other agent functions, but no `session_id` / `agent_role` / context is passed through the call boundary.

*Severity*: Major — multi-turn analysis and observability are silently broken across the boundary.

*Recommendation*: thread `session_id`, `user_id`, and a `parent_role` through every handoff. The Langfuse propagation pattern (`propagate_attributes(...)`) handles this once observability is wired.

*Ratel angle*: none directly — fix in [`/ratel-langfuse-instrument`](../../ratel-langfuse-instrument/SKILL.md).

### 1.c Topology is too deep to debug

*Detection*: ≥4 levels of nesting, or recursive sub-agent calls without a depth bound.

*Severity*: Major — small bugs (a tool misfire) blow up into runaway sessions.

*Recommendation*: add a max-depth bound and a circuit breaker on sub-agent recursion. Flatten the topology if the depth doesn't carry information the model uses.

*Ratel angle*: none.

---

## 2. Tool surface

What it covers: tool count, naming quality, description quality, schema rigor, duplication, dead tools.

**Detection inputs**:

- Total tool count — grep for `defineTool`, `createTool`, `@tool`, `@function_tool`, `tools: {`, MCP `listTools` returns.
- Per-tool description length and quality.
- JSON schema presence and tightness.
- Duplicate / near-duplicate tools (e.g., `read_file` and `get_file_contents`).
- Tools registered but never referenced in any prompt, dispatch site, or fixture (dead).
- Tool count growth over time (git log on tool registration files).

**Common findings**:

### 2.a Tool sprawl

*Detection*: ≥20 tools exposed on a single model call (or ≥40 in the catalog total). Severity scales with size.

*Severity*: Minor at 20–30, Major at 30–60, Critical at >60. Adjust down if descriptions are exceptional; the count is a proxy for input-token pressure.

*Recommendation*: pre-filter the tool list per turn so the model only sees the top-K relevant tools. The full catalog stays addressable via a discovery tool.

*Ratel angle*: matches "BM25 tool retrieval" and "Replace-by-default pre-filter" (v0.1.5, shipped). This is the textbook case Ratel was built for.

### 2.b Bloated tool descriptions

*Detection*: at least one tool description >300 tokens, or median description >120 tokens. Calculate by character count if tokenization is unavailable (≈4 chars/token).

*Severity*: Minor if isolated to a few tools; Major if it's the pattern across the catalog.

*Recommendation*: trim descriptions to one short sentence ("what it does") + a one-line "when to use." Move examples and edge-case detail into a separate spec the agent doesn't see every turn.

*Ratel angle*: matches "BM25 tool retrieval" — Ratel's relevance score also surfaces which descriptions are confusing the ranker; the dashboard will show low top-hit scores for over-described tools.

### 2.c Anemic tool descriptions

*Detection*: at least one tool with description <8 tokens, or descriptions that name the tool rather than describe it ("call this tool" / "read", "write").

*Severity*: Major — the model can't select correctly without knowing what each tool does. Cascading misroutes.

*Recommendation*: rewrite descriptions to answer "what does it do" and "when do you call it." The model is the audience, not the human reader.

*Ratel angle*: matches "LLM-driven suggestions" (v0.1.9, roadmap). Ratel will propose description rewrites once shipped; in the meantime this is a hand-edit pass.

### 2.d Duplicate / near-duplicate tools

*Detection*: two or more tools with overlapping descriptions or schemas (`read_file` + `get_file_contents` + `cat_file`). Common when multiple MCP upstreams ship similar tools.

*Severity*: Major — the model picks one inconsistently and reasoning becomes unstable across turns.

*Recommendation*: consolidate or namespace explicitly. If both must exist (different upstreams), make the descriptions sharply distinguishing ("local repo only" vs "remote workspace").

*Ratel angle*: matches "MCP server ingestion (upstream namespace prefix)" (v0.1.5, shipped) — Ratel namespaces upstream tools automatically and ranks them, so the model only sees the relevant variant.

### 2.e Dead tools

*Detection*: tools registered but never referenced anywhere in prompts, fixtures, or the dispatch sites. If live data is available: tools with zero invocations over the sample window.

*Severity*: Minor.

*Recommendation*: delete or feature-flag off. Every registered tool consumes input tokens whether or not the agent ever calls it.

*Ratel angle*: none directly; fixed once Ratel pre-filtering ships (the tool is in the catalog but never makes the top-K).

### 2.f Loose / missing tool schemas

*Detection*: tools whose `inputSchema` is missing, `{}`, or has top-level `additionalProperties: true` with no constraints.

*Severity*: Major — the model invents inputs; runtime validation has to clean up.

*Recommendation*: tighten the schema. Required fields explicit, types narrow, enums where the value space is finite.

*Ratel angle*: none.

### 2.g Verbose tool outputs eating context

*Detection*: tools that return raw blobs (full file contents, raw HTTP bodies, full database rows) without summarization. Grep for `JSON.stringify(rows)` / `return await response.text()` / similar.

*Severity*: Major when ≥3 tools do this and they're called frequently; Minor otherwise.

*Recommendation*: summarize or paginate at the tool boundary. The tool's output is part of the agent's context; treat it accordingly.

*Ratel angle*: matches "TOON encoding" (v0.1.6, rc). Ratel will compress structured tool outputs once shipped; for unstructured blobs, summarization is still the customer's job.

---

## 3. Context management

What it covers: prompt size, externalization, versioning, retrieval / RAG, conversation memory, summarization, compaction.

**Detection inputs**:

- System-prompt size (in tokens / characters).
- Prompts inline vs externalized to files / a prompt-management service.
- Prompt versioning (file-history-based, hash-based, or none).
- RAG pipeline presence (vector store, retriever, embedding model).
- Memory layer (summarization, key-value cache, recall heuristics).
- Conversation-history handling — full history vs windowed vs compacted.

**Common findings**:

### 3.a Monolithic system prompt

*Detection*: a single system prompt >3000 tokens that mixes role definition, tool docs, output formatting, examples, safety rules, and edge-case handling.

*Severity*: Major.

*Recommendation*: split by responsibility — role prompt is short and stable; tool docs live in the tool descriptions; output format goes in a contract section; examples go in a few-shot store the agent retrieves from on demand.

*Ratel angle*: none directly; this is a hand-edit pass.

### 3.b Inline-string prompts with no versioning

*Detection*: prompts as inline template strings in agent code, no commit hash captured, no `prompt_version` field anywhere.

*Severity*: Minor on its own; Major if there is no observability either (you can't regression-detect on a prompt you can't pin).

*Recommendation*: externalize prompts to files (or to Langfuse Prompt Management). Attach `prompt_version` as observation metadata.

*Ratel angle*: none; this is a [`/ratel-langfuse-instrument`](../../ratel-langfuse-instrument/SKILL.md) follow-up.

### 3.c No retrieval where retrieval is obvious

*Detection*: the agent has a body of static knowledge (docs, FAQs, policies) that is dumped into the system prompt rather than retrieved.

*Severity*: Major — input tokens scale with the size of the knowledge dump.

*Recommendation*: move static knowledge into a vector / BM25 store; retrieve top-K per turn. The model only sees what's relevant.

*Ratel angle*: indirect — Ratel itself is BM25-over-tools, not BM25-over-documents, but the customer's win pattern is the same (filter the context surface). Worth noting; do not over-claim.

### 3.d Full-history replay every turn

*Detection*: every model call replays the entire conversation history with no summarization or windowing.

*Severity*: Minor at short sessions; Major if sessions can grow >50 turns.

*Recommendation*: summarize older turns into a compact memory; replay only the recent N turns verbatim.

*Ratel angle*: matches "Chat compaction" (v0.2.x, roadmap) and "Memory orchestration" (v0.3.x, roadmap). Both are roadmap-only — be honest about the timeline.

### 3.e Skill-shaped subroutines inlined as prompt blocks

*Detection*: recurring instruction blocks across prompts — "draft an email," "summarise and extract," "fetch then format" — written inline as system-prompt instructions and copy-pasted across multiple agents or call sites. Grep for repeated multi-line instruction blocks; check whether the same multi-step pattern appears as inline prose in three or more locations.

*Severity*: Minor — the agent works; the prompts duplicate.

*Recommendation*: extract the recurring blocks into named, retrievable units. In the meantime, deduplicate via shared prompt fragments.

*Ratel angle*: matches "First-class skills" (v0.1.7, roadmap). Ratel will treat skills as retrievable first-class units alongside tools; until then, this is a hand-extraction pass.

---

## 4. Decomposition

What it covers: whether the agent decomposes complex tasks into steps or tries to one-shot them; whether sub-tasks fan out cleanly.

**Detection inputs**:

- System prompt: does it instruct the model to plan? Are the steps named?
- Code: does any logic plan a sequence of sub-calls before invoking tools?
- For frameworks like LangGraph: is the graph structured around decomposition?

**Common findings**:

### 4.a Monolithic single-turn solution attempts

*Detection*: the agent's main loop is "tool-call → answer," with no planning phase, no fan-out, and no critic / verifier step.

*Severity*: Minor for simple agents; Major if the agent handles multi-step tasks (research, writing, code generation, data extraction across sources).

*Recommendation*: introduce an explicit planning step — even a single "plan first, then act" prompt round-trip improves outcomes measurably.

*Ratel angle*: matches "Multi-agent decomposition hints" (v0.1.10, roadmap). State the version honestly.

### 4.b Fan-out without fan-in

*Detection*: the agent calls multiple sub-agents in parallel (or sequentially) but has no aggregation / critic step to reconcile their outputs.

*Severity*: Major — sub-agent disagreement compounds silently.

*Recommendation*: add a verifier / aggregator agent that takes the fan-out outputs and produces a single reconciled result.

*Ratel angle*: none directly.

---

## 5. Model routing

What it covers: whether the agent picks the right model for the right sub-task (cheap model for classification, expensive model for reasoning) or runs everything through one expensive model.

**Detection inputs**:

- Number of distinct model ids called in the codebase.
- Whether sub-agents call different models or the same one.
- Whether classification / routing / formatting tasks share the same model as deep reasoning.

**Common findings**:

### 5.a Single-model agent doing everything

*Detection*: every model call uses the same provider model, including small classification / formatting steps that don't need it.

*Severity*: Minor unless cost is a stated pain point; then Major.

*Recommendation*: route by task — small / fast models for classification and routing, larger models for reasoning. The router can be deterministic (per-tool or per-role) before going LLM-as-router.

*Ratel angle*: none.

### 5.b Latest-and-greatest used as default for everything

*Detection*: every model call goes through the current frontier model even where a smaller / older model would suffice.

*Severity*: Minor — this is mostly a cost smell, not a quality smell. Flag and move on.

*Recommendation*: pick model per task. Use evals (Dimension 9) to confirm the cheaper model maintains quality on the relevant tasks before routing them away from the frontier model.

*Ratel angle*: none.

---

## 6. Error handling

What it covers: retry behavior, backoff, dead-letter, user-facing failure UX, partial-result handling.

**Detection inputs**:

- `try` / `except` / `catch` shape around tool calls and model calls.
- Retry loops — bounded vs unbounded, backoff type, jitter.
- Dead-letter / failure-state persistence.
- User-facing error messages — exposed raw or wrapped.

**Common findings**:

### 6.a Unbounded retry loops

*Detection*: `while True: try: ... except: continue` around tool / model calls, or retry without a max attempt count.

*Severity*: Critical — one transient upstream failure burns through tokens and quota.

*Recommendation*: bound retries (3 attempts typical), exponential backoff with jitter, circuit-breaker on repeated failure.

*Ratel angle*: none.

### 6.b Bare except / swallow-all error handling

*Detection*: `except: pass` or `catch (e) {}` around tool execution. Errors disappear silently.

*Severity*: Major — silent failures look like bad model output on the user side.

*Recommendation*: catch specific exceptions, log with structured context, surface a partial-result indicator to the model so it can recover.

*Ratel angle*: none.

### 6.c Raw error strings leak to user

*Detection*: model is told "here is the error: <stack trace>" or the user-facing channel receives raw exception messages.

*Severity*: Major — security (information disclosure) and UX both.

*Recommendation*: classify errors (transient / permanent / configuration) and produce user-appropriate messages. The model can still see internal detail; users should not.

*Ratel angle*: none.

---

## 7. Observability

What it covers: presence and consistency of tracing, naming, scoring, session boundaries. The most important dimension for partner engagements because everything else hinges on the ability to measure.

**Detection inputs**:

- SDK presence — Langfuse, Langsmith, OTel, OpenInference, OpenLLMetry, Helicone.
- Env vars and init sites.
- Naming consistency vs the vocabulary in [`../../ratel-langfuse-instrument/references/naming-conventions.md`](../../ratel-langfuse-instrument/references/naming-conventions.md).
- Session-id sourcing and propagation.
- Tool-call observation typing (`type: tool` vs untyped `event`).
- Score wiring (any `score()` calls or ingestion of eval-driven scores).
- (Live, if reachable) trace count, naming uniformity, tag coverage.

**Common findings**:

### 7.a No observability at all

*Detection*: no observability SDK in the manifest, no init site, no tracing env vars.

*Severity*: Critical for any agent in production; Major otherwise.

*Recommendation*: wire Langfuse via [`/ratel-langfuse-instrument`](../../ratel-langfuse-instrument/SKILL.md).

*Ratel angle*: routes to `/ratel-langfuse-instrument`.

### 7.b Observability wired but no session_id

*Detection*: Langfuse / Langsmith init exists, but no `session_id` is set on traces (grep for `session_id`, `setSessionId`, `propagate_attributes(session_id`).

*Severity*: Major — multi-turn analysis is impossible without sessions.

*Recommendation*: set `session_id` at the entry point and propagate through sub-agents.

*Ratel angle*: matches the "Sessions" section of [`../../ratel-langfuse-instrument/references/naming-conventions.md`](../../ratel-langfuse-instrument/references/naming-conventions.md). Route to `/ratel-langfuse-instrument`.

### 7.c Tool calls captured as untyped events

*Detection*: tool calls land as generic `event` observations (or as `span`) with the tool name in metadata instead of as `type: tool` observations with the tool name in `name`.

*Severity*: Major — blocks the native tool-call dashboards (Gateway Origin Split, Upstream Health, tool-level cost views).

*Recommendation*: re-wrap tool calls as `type: tool` observations. One-line fix per call site once Langfuse v4 patterns are in place.

*Ratel angle*: same Langfuse hygiene pattern; route to `/ratel-langfuse-instrument`.

### 7.d Inconsistent observation naming

*Detection*: tool observations named after the function (`handle_search`, `read_file_impl`) rather than after the tool id (`tool.search`, `tool.read_file`).

*Severity*: Minor on its own; Major if it cascades into dashboards being unreadable.

*Recommendation*: adopt the vocabulary in [`../../ratel-langfuse-instrument/references/naming-conventions.md`](../../ratel-langfuse-instrument/references/naming-conventions.md). One-pass rename.

*Ratel angle*: routes to `/ratel-langfuse-instrument`.

### 7.e Observability wired but never analyzed

*Detection*: Langfuse is wired and reachable (live check succeeds) and there is meaningful data, but no dashboards exist, no scores are ingested, and there is no recurring review cadence anywhere in the README / runbooks.

*Severity*: Minor — the data is there, the loop just isn't closed.

*Recommendation*: build the dashboards via `/ratel-langfuse-dashboards`; review live data via `/ratel-langfuse-analyze`.

*Ratel angle*: routes to `/ratel-langfuse-dashboards` and `/ratel-langfuse-analyze`.

---

## 8. Cost discipline

What it covers: per-call cost awareness in the code — model choice per task, caching, payload size, output-token caps.

**Detection inputs**:

- `max_tokens` / `maxTokens` settings on model calls — present? sensible?
- Prompt caching (Anthropic) / context caching (OpenAI) usage.
- Streaming vs blocking — streaming-only patterns where blocking would be cheaper / vice versa.
- Output size discipline — instructions to be concise, schemas that constrain output shape.

**Common findings**:

### 8.a No max_tokens cap

*Detection*: `generateText` / `chat.completions.create` calls without `maxTokens` / `max_tokens`.

*Severity*: Minor for assistants; Major for tools embedded in higher-throughput surfaces (a 16K-token answer to a yes/no question is a real cost smell).

*Recommendation*: cap per call. Different caps per role.

*Ratel angle*: none.

### 8.b No prompt caching

*Detection*: prompts have a large stable prefix (multi-thousand-token system prompt or tool catalog) but no caching is configured.

*Severity*: Major for high-frequency agents.

*Recommendation*: enable provider-side caching on the stable prefix. The win is proportional to traffic.

*Ratel angle*: indirect — Ratel pre-filter shrinks the catalog the prompt prefix would otherwise carry; combine both for compounded savings.

### 8.c Verbose model output by instruction

*Detection*: system prompt instructs verbose output ("explain your reasoning in detail") for tasks that don't need it (a classification call, a routing call).

*Severity*: Minor.

*Recommendation*: terse outputs for non-reasoning roles. Keep verbose chain-of-thought to the roles that benefit.

*Ratel angle*: none.

---

## 9. Eval / quality gates

What it covers: presence of an eval suite, ground truth, CI gates, regression detection.

**Detection inputs**:

- `evals/`, `tests/eval/`, `tests/agents/`, `prompts/test/` directories.
- Eval framework imports — `promptfoo`, `langfuse.score`, `langsmith.evaluate`, `inspect_ai`, internal eval modules.
- CI workflow files invoking evals on PRs.
- Ground-truth artifacts — labeled fixtures, gold sets.

**Common findings**:

### 9.a No eval suite

*Detection*: zero `evals/` directory, zero eval framework imports, zero CI invocations of anything eval-shaped.

*Severity*: Critical for any agent shipping to production; Major otherwise.

*Recommendation*: stand up a minimal eval set — even 20 hand-labeled examples per critical task is enough to catch regressions. Hook to CI; gate merges.

*Ratel angle*: none — but having ground truth unlocks Ratel's retrieval-quality scoring (`top_k_recall_at_5`, `tool_selection_accuracy`) which is otherwise unobservable.

### 9.b Eval set exists but no CI gate

*Detection*: `evals/` directory has fixtures and a runner, but no CI workflow invokes it.

*Severity*: Major.

*Recommendation*: wire to CI. Block merge on regression beyond a tolerance.

*Ratel angle*: none.

### 9.c No ground truth for tool selection

*Detection*: eval fixtures exist but none label the *correct tool id* for each task.

*Severity*: Minor on its own; Major in combination with tool sprawl.

*Recommendation*: label `ground_truth_tool_id` per fixture per the metadata vocabulary in [`../../ratel-langfuse-instrument/references/naming-conventions.md`](../../ratel-langfuse-instrument/references/naming-conventions.md). Enables retrieval-quality scoring.

*Ratel angle*: matches the Retrieval Quality dashboard's `recall@5` widget — unlocks once ground truth exists.

---

## 10. Safety

What it covers: prompt-injection guards, tool input validation, secret handling, output sanitization.

**Detection inputs**:

- Tool inputs that originate from user content without validation.
- Tools that read filesystems, shell out, write to databases, send messages, call external APIs — and what guards exist around them.
- Secret handling — env vars, secret managers, leak surfaces.
- Output sanitization where the agent's output flows to a downstream sink (a webhook, a database, an LLM-as-judge).

**Common findings**:

### 10.a Tools that can shell out / write to disk without validation

*Detection*: tools whose execution path includes `exec`, `subprocess`, `child_process`, `fs.writeFile`, `os.system` — and whose inputs are not validated against an allow-list.

*Severity*: Critical when reachable from untrusted input.

*Recommendation*: allow-list inputs, sandbox execution (e.g., Vercel Sandbox or equivalent), forbid the tool from running on untrusted-input paths.

*Ratel angle*: none.

### 10.b Prompt injection: untrusted content concatenated into prompts

*Detection*: user input (or content fetched from URLs / RSS / email / scraped pages) flows into system or user prompts without delimiter discipline.

*Severity*: Major.

*Recommendation*: delimit untrusted content (XML tags, structured fields), instruct the model to treat the content as data not instructions, and run an output classifier before acting on it.

*Ratel angle*: none.

### 10.c Secrets in env files committed to the repo

*Detection*: `.env` (not `.env.example`) committed, or secrets visible in the repo's git history.

*Severity*: Critical.

*Recommendation*: rotate the leaked secrets, add `.env` to `.gitignore`, move to a secret manager. This finding is non-negotiable; promote to the top of the report regardless of other findings.

*Ratel angle*: none.

---

## How to combine findings into a dimension score

Score each dimension once, using the **worst severity** in its findings:

- Any Critical finding → **Missing**.
- Worst is Major → **Weak**.
- Worst is Minor → **Adequate** (or **Strong** if no findings beyond Info).
- No findings → **Strong**.

If the dimension has *no signal at all in the codebase* (e.g., no observability surface anywhere, no eval directory at all), score it **Missing** and write a single Critical-or-Major finding explaining the absence. Don't score **Strong** by accident on a dimension you couldn't even evaluate.

## Catalog maintenance

When Ratel ships a new feature, the corresponding row goes in [`../../ratel-langfuse-dashboards/references/ratel-value-map.md`](../../ratel-langfuse-dashboards/references/ratel-value-map.md) first. Only then does this catalog get an updated Ratel-angle line. Two reasons: (1) the value map is the source of truth, (2) a Ratel angle in the assessment that doesn't yet have a dashboard widget produces an unfinishable engagement.

If you spot a new assessment dimension worth adding (something genuinely orthogonal to the existing ten), open a PR — but bias toward adding a new finding under an existing dimension first.
