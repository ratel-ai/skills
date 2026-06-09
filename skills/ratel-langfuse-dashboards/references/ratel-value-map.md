# Ratel value map — feature → signal → dashboard

The canonical mapping from a Ratel capability (shipped or roadmapped) to the observable Langfuse signal it produces and the dashboard widget that visualises it. **Update this file whenever Ratel ships a new feature** — every other skill in the trio reads it.

Each row has:
- **Feature** — the Ratel capability, with the version that shipped (or will ship) it.
- **Status** — `shipped` / `rc` / `roadmap`. Dashboards for `roadmap` features go in the customer plan's "Out of scope" section, not in the active dashboard list.
- **Signal** — the Langfuse data that proves the feature is working.
- **Dashboard** — which dashboard in this catalog owns the widget that surfaces it.

## Shipped today (v0.1.5 line)

| Feature | Status | Signal | Dashboard |
| --- | --- | --- | --- |
| BM25 tool retrieval (top-K via `search_tools`) | shipped | `ratel.search_tools` observations with `top_k`, `hit_count`, `top_hit_score`, `took_ms` | Retrieval Quality |
| Replace-by-default pre-filter (top-K injected, full catalog hidden) | shipped | `metadata.replace_mode=true` on `chat-turn` traces; `input_tokens` on root generation drops by 50–85% | Token Cost & Savings |
| Gateway tools (`search_tools`, `invoke_tool`) | shipped | `metadata.gateway_origin in [direct, agent]` on every Ratel observation; count by origin | Gateway Origin Split |
| MCP server ingestion (upstream namespace prefix) | shipped | `ratel.upstream.invoke` observations with `server_name` and `tool_id` | Upstream Health |
| OAuth 2.1 / PKCE auth flows | shipped | `ratel.auth.refresh`, `ratel.auth.needs`, `ratel.auth.flow_start`, `ratel.auth.flow_end` events | Upstream Health |
| Trace stream (JSONL sink + future Langfuse sink) | shipped | every observation above exists in Langfuse | foundation for all dashboards |

## Coming soon (next minor versions)

| Feature | Status | Signal it will add | Dashboard impact |
| --- | --- | --- | --- |
| TOON encoding (v0.1.6) | rc | `metadata.encoding=toon` vs `json` on `ratel.invoke_tool`; per-call token delta | Token Cost & Savings adds a "TOON savings" widget |
| First-class skills (v0.1.7) | roadmap | `ratel.search_skills`, `ratel.invoke_skill` observations; skill ids in `metadata` | New "Skill Invocation Health" dashboard |
| LLM-driven suggestions (v0.1.9) | roadmap | `ratel.suggestion.generated` events; `score_name = suggestion_adopted` | New "Suggestion Adoption" dashboard |
| Multi-agent decomposition hints (v0.1.10) | roadmap | `ratel.decomposition.proposed` events; per-sub-agent catalog sizes | New "Decomposition Outcome" dashboard |
| Semantic search + hybrid ranking (v0.1.12–v0.1.13) | roadmap | `metadata.ranker = bm25 | semantic | hybrid`; per-ranker top-hit score | Retrieval Quality adds a "Ranker comparison" widget |
| Re-ranking (v0.1.14 LLM, v0.1.15 XGBoost) | roadmap | `ratel.rerank` observations with `before_order` / `after_order` | Retrieval Quality adds a "Re-rank lift" widget |
| Chat compaction (v0.2.x) | roadmap | `ratel.compact` observations with token-in / token-out | New "Compaction" dashboard |
| Memory orchestration (v0.3.x) | roadmap | `ratel.memory.retrieve` observations with hit count, ranking | New "Memory Recall" dashboard |

## Recommended dashboards (Ratel-value group)

### Token Cost & Savings

The headline dashboard. Shows the partner is spending less per turn.

Widgets:
1. **Daily input tokens, split by feature flag** — line, sum, `dim: day, tag.feature_flag`, filter `trace_name = chat-turn`, `tag.env = prod`. Two lines: `tool_pool=full` and `tool_pool=ratel`.
2. **Daily total cost per session** — line, avg, `dim: day, tag.feature_flag`, filter same as above.
3. **Single-stat: tokens saved this week** — single-stat, sum of difference. Computed widget; if Langfuse v4 doesn't support computed widgets natively, ship two single-stats side by side and a footnote.
4. **TOON savings** (only when v0.1.6 is in) — bar, avg, `dim: metadata.encoding`, metric `input_tokens`, filter `observation_name = ratel.invoke_tool`.

### Retrieval Quality

Shows Ratel is finding the right tools, not just any tools.

Widgets:
1. **Top-hit score distribution** — histogram, `metric: metadata.top_hit_score`, filter `observation_name = ratel.search_tools`.
2. **Recall@5 (with ground truth)** — line, avg, `dim: day, tag.feature_flag`, filter `score_name = top_k_recall_at_5`. Only shown when ground-truth labelling is in place (per `ratel-hooks.md`).
3. **Hit count over time** — line, avg of `metadata.hit_count`, `dim: day`.
4. **Ranker comparison** (v0.1.12+) — line, avg `metadata.top_hit_score`, `dim: day, metadata.ranker`.
5. **Re-rank lift** (v0.1.14+) — scatter, `metadata.before_order_top_hit` vs `metadata.after_order_top_hit`, filter `observation_name = ratel.rerank`.

### Gateway Origin Split

Shows whether the agent is using Ratel as a pre-filter or as a discovery surface.

Widgets:
1. **Daily observations by origin** — stacked-bar, count, `dim: day, metadata.gateway_origin`.
2. **Agent-origin invokes (the agent reached for `search_tools`)** — single-stat, count, filter `observation_name = ratel.search_tools`, `metadata.gateway_origin = agent`.
3. **Top tools called via gateway** — table, count, `dim: metadata.tool_id`, filter `observation_name = ratel.invoke_tool`, `metadata.gateway_origin = agent`. Top 20.

### Upstream Health

Shows MCP upstreams aren't quietly failing.

Widgets:
1. **Daily upstream invokes, by server** — stacked-bar, count, `dim: day, metadata.server_name`, filter `observation_name = ratel.upstream.invoke`.
2. **Upstream error rate, by server** — line, ratio of errors, `dim: day, metadata.server_name`.
3. **Auth events** — table, count, `dim: observation_name, metadata.upstream`, filter `observation_name starts with ratel.auth`.

### Skill Invocation Health (v0.1.7+)

Placeholder for when skills ship. Mirror structure of Gateway Origin Split, swapping `search_tools` → `search_skills`, `invoke_tool` → `invoke_skill`.

### Suggestion Adoption (v0.1.9+)

Placeholder for when LLM suggestions ship.

Widgets (when ready):
1. **Suggestions generated per week** — bar, count, `dim: week`, filter `observation_name = ratel.suggestion.generated`.
2. **Adoption rate** — line, avg of `score_value`, filter `score_name = suggestion_adopted`.
3. **Accuracy delta of adopted suggestions** — bar, `dim: metadata.suggestion_kind` (description rewrite / new skill / merge / etc.), metric `score_value` of `tool_selection_accuracy` after vs before.

### Decomposition Outcome (v0.1.10+)

Placeholder. When ready:
1. **Decomposition proposals over time** — line.
2. **Pre/post catalog sizes per sub-agent** — bar.
3. **Accuracy of decomposed agents vs monolith** — line, score split by `metadata.decomposition_arm`.
