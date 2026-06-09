# Finding catalog

The canonical catalog of patterns to look for, the heuristic that detects each one, and the recommended fix. Update this file when new patterns emerge from engagements — it's the only place these live.

Each entry has:
- **Pattern** — short name.
- **Category** — `ratel` (we fix this by integrating Ratel) or `generic` (anyone could fix it).
- **Detection** — the specific query / aggregate that triggers it.
- **Recommended action** — what to put in the finding.
- **Solved by** (Ratel patterns only) — Ratel version that ships the fix.

Don't emit a finding that isn't in the catalog without adding it here first. If a one-off finding emerges during analysis, write the template down before shipping.

---

## Data quality patterns (generic)

These break dashboards and analytics. Always fix first.

### Missing session_id

- **Detection**: any non-trivial fraction of traces (>5%) has empty `session_id`.
- **Action**: identify the agent entry point that's not setting session_id; cite the spot from the instrumentation plan. Multi-turn analysis is impossible until this is fixed.
- **Severity**: high if >25% of traces affected; medium otherwise.

### Missing user_id when product is authenticated

- **Detection**: product has logged-in users, but `user_id` is empty on >5% of traces.
- **Action**: trace upstream from the entry point to find where the user context drops. Usually a missing prop in a fan-out.
- **Severity**: medium.

### Inconsistent trace naming

- **Detection**: more than ~5 distinct `trace_name` values in the same `env`, and at least one looks like a function name (`handleChat`, `POST /api/chat`) or a UUID.
- **Action**: enforce one trace name per externally meaningful unit per `[ratel-langfuse-instrument naming-conventions.md](../../ratel-langfuse-instrument/references/naming-conventions.md)`.
- **Severity**: medium (annoys dashboards but doesn't break them).

### Tool calls landing as untyped events

- **Detection**: top tools in the catalog do not appear when filtering observations by `type = tool`.
- **Action**: wrap the tool-call site to emit observations of `type: tool` with `name = tool.<tool-id>`. Cite the stack reference.
- **Severity**: high (every Ratel-value dashboard + every tool-health dashboard depends on this).

### Truncated or missing input/output

- **Detection**: >20% of observations have empty input OR empty output.
- **Action**: instrumentation is collecting structure but not content. Find the recording call and pass the input/output explicitly; don't rely on auto-capture for hand-rolled spans.
- **Severity**: medium.

---

## Cost & token patterns

### Token-heavy turns with huge tool catalogs (RATEL)

- **Category**: ratel
- **Detection**: `chat-turn` traces where input_tokens > some threshold (5000 is a useful default) AND the tool list in the system prompt or first user message is large (visible in the input field or estimable from observation count of tool-definition load events).
- **Action**: introduce Ratel as a pre-filter (`replace_mode = true`). Recommend a pilot on the top trace_name only first. Cite expected savings from the benchmark in the Ratel README (~50–85% input token reduction at pool ≥ 180 tools).
- **Solved by**: shipped, v0.1.5 line.
- **Severity**: high if it affects >25% of cost; medium otherwise.

### Tool-payload bloat (RATEL, coming)

- **Category**: ratel (roadmap)
- **Detection**: tool observations where output > 10kb AND the tool is called many times per session.
- **Action**: today, prune output before recording. Future: Ratel's TOON encoding (v0.1.6) and smart pruning (v0.2.x) handle this systematically. Mention both: an immediate workaround and the eventual fix.
- **Solved by**: v0.1.6 (TOON), v0.2.x (smart pruning).
- **Severity**: medium.

### System-prompt / context bloat (generic)

- **Category**: generic
- **Detection**: `chat-turn` generations where input_tokens is high (>5000) BUT the tool catalog is small (≤ ~30 tools) or tool schemas aren't even in the recorded input — i.e. the bloat is a large fixed system prompt and/or accumulating conversation history re-sent every turn, NOT a large tool list. Tell-tale: input is large and roughly constant across turns even when the user message is tiny; output is small (mostly tool-call decisions). This is the honest alternative to the RATEL huge-catalog pattern — do not dress it up as a tool-search opportunity when the catalog is small.
- **Action**: (1) Enable provider prompt caching on the static system-prompt prefix — for a ~10–22k-token input that is ~90% identical across turns, this is the single biggest cost lever and needs no app rewrite. (2) Trim the system prompt: move long playbooks/prose into tools or retrieved docs the agent pulls on demand. (3) Summarise or window conversation history instead of replaying it verbatim. Cite the per-turn input-token p50/p95 and the input:output ratio.
- **Severity**: high if it drives >25% of cost; medium otherwise.

### Generic cost spike

- **Category**: generic
- **Detection**: daily total cost has a >50% jump in the last 24h vs the previous 7-day avg.
- **Action**: split cost by model and by trace_name to localise. Cite the trace ids of the top 5 spend contributors.
- **Severity**: high.

### Wrong-tier model for a task

- **Category**: generic
- **Detection**: top-cost trace_names use a frontier model (Opus / GPT-5 / etc.) where a smaller model would do (heuristic: input length is short, output length is short, and the task per the prompt is structurally simple).
- **Action**: A/B the same trace on a smaller model. Suggest specific candidate (Haiku / mini variant).
- **Severity**: medium.

---

## Retrieval & tool-selection patterns

### Low recall@K (RATEL)

- **Category**: ratel
- **Detection**: `score_value` average for `score_name = top_k_recall_at_5` is below 0.7 over the window.
- **Action**: review tool descriptions for the tools that aren't being recalled. Ratel's suggestion engine (v0.1.9) will automate this; today, rewrite descriptions manually.
- **Solved by**: shipped (today) with manual fix; v0.1.9 automates it.
- **Severity**: medium (high if recall is below 0.5).

### Misrouted tool calls (RATEL)

- **Category**: ratel
- **Detection**: agent calls one tool, immediately followed by an error or a different tool call on the same input — pattern of "wrong tool first try" in the trace bodies.
- **Action**: surface the misrouting examples. Recommend ground-truth labelling + Ratel's `tool_selection_accuracy` score, then a Ratel pre-filter pilot.
- **Solved by**: shipped (today), reinforced by v0.1.9 suggestions.
- **Severity**: medium.

### Tools called once or never

- **Category**: generic
- **Detection**: 30%+ of registered tools have zero invocations over the window.
- **Action**: tools either need better descriptions (most likely) or they should be removed. List the dead tools.
- **Severity**: low (high if the dead tools constitute most of the catalog).

### Retry storms

- **Category**: generic
- **Detection**: traces with >5 invocations of the same `tool_id` within 10 seconds.
- **Action**: cite trace ids. Usually a missing retry budget or a tool returning a recoverable error that the agent doesn't handle. Recommend a tool-level retry budget.
- **Severity**: high if >5% of traces affected.

---

## Multi-agent / handoff patterns

### Flat sub-agent structure (generic)

- **Category**: generic
- **Detection**: traces with multiple `agent_role` metadata values but no parent-child nesting (siblings only).
- **Action**: parent context isn't carried across the handoff. Cite the stack reference for `propagate_attributes` / equivalent.
- **Severity**: medium.

### Runaway sub-agent loop

- **Category**: generic
- **Detection**: same sub-agent role appears >10 times in a single trace.
- **Action**: probable infinite-loop / max-step misconfiguration. Cite trace ids.
- **Severity**: high.

### Decomposition opportunity (RATEL, roadmap)

- **Category**: ratel (roadmap)
- **Detection**: single-agent traces with very large tool catalogs (>50 tools) AND clear clustering of tool calls into groups across sessions.
- **Action**: roadmap pointer to multi-agent decomposition hints (v0.1.10). Today: manually identify the cluster and propose a supervisor / sub-agent split.
- **Solved by**: v0.1.10.
- **Severity**: medium.

---

## Session & quality patterns

### Abandoned-session spike

- **Category**: generic
- **Detection**: % of single-turn sessions with no follow-up trended up 25%+ over the window.
- **Action**: read the first/last messages of a sample. Usually a regression in the first response.
- **Severity**: high.

### Score regression by agent_version

- **Category**: generic
- **Detection**: avg score for `tag.agent_version = <latest>` is materially below the previous version's avg.
- **Action**: cite both versions and the score name. Recommend a rollback or hotfix.
- **Severity**: high.

### No evaluation scores emitted (generic)

- **Detection**: `listScores` returns zero rows over the window despite traffic existing. No online or annotation scores wired.
- **Action**: wire at least one cheap online score (e.g. an LLM-judge on task completion, or a deterministic check) so quality regressions are visible. Without it, the recall/accuracy/score-regression patterns in this catalog cannot be evaluated at all — every quality finding is blind. Point back to `/ratel-langfuse-dashboards` and `/ratel-langfuse-instrument` for score setup.
- **Severity**: medium (gates all quality analysis, but not load-bearing for cost/latency dashboards).

### Prompt drift

- **Category**: generic
- **Detection**: multiple `metadata.prompt_version` values active in the same `agent_version`.
- **Action**: prompt versioning isn't enforced. Recommend pinning per-deploy.
- **Severity**: low.

---

## Model & infra patterns

### Latency outlier from a single model

- **Category**: generic
- **Detection**: one `model_id` has p95 latency >3x the median across models.
- **Action**: investigate provider-side; recommend a fallback / circuit breaker. Cite trace ids.
- **Severity**: medium.

### Generation usage missing

- **Category**: generic
- **Detection**: >10% of generations have empty token usage.
- **Action**: provider integration isn't capturing usage. Most commonly: wrong import (`from openai import OpenAI` instead of `from langfuse.openai import openai`).
- **Severity**: medium (blocks cost dashboards).

### Model swap mid-trace

- **Category**: generic
- **Detection**: a single trace contains generations with multiple distinct `model_id` values.
- **Action**: usually a fallback / cascade firing. Worth surfacing — high model swap rates indicate frontier-model rate limiting or a bug.
- **Severity**: low.

---

## Suggestion / adoption patterns (RATEL, roadmap)

### Low suggestion adoption (RATEL)

- **Category**: ratel (roadmap)
- **Detection**: when v0.1.9 ships and suggestions are being generated, `score_name = suggestion_adopted` avg is below 0.3.
- **Action**: investigate why suggestions are being declined — surface a sample, recommend tuning the suggestion model or the prompt.
- **Solved by**: v0.1.9+ tuning loop.
- **Severity**: low.

### Accuracy delta after suggestion (RATEL)

- **Category**: ratel (roadmap)
- **Detection**: after suggestions adopted, `tool_selection_accuracy` should rise. If flat or down, the suggestion engine is misfiring.
- **Action**: roll back the suggested catalog, file a Ratel issue with the trace ids.
- **Solved by**: v0.1.9 with eval coverage (v0.1.11).
- **Severity**: medium.
