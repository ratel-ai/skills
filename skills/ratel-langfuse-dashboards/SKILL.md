---
name: ratel-langfuse-dashboards
description: |
  Decide which Langfuse dashboards a customer should build now that their agent is instrumented, and produce a markdown spec they can follow in the Langfuse UI. Use whenever the user asks "what should we put on the Langfuse board", "design our dashboards", "what should we measure", "what would prove out Ratel's value here", or invokes `/ratel-langfuse-dashboards`. Trigger on phrases like "set up dashboards", "we need a metrics view", "what dashboards do partners want", "Ratel-value dashboards", "agent health dashboards" — even if the user doesn't say "skill" or "ratel-langfuse-dashboards" by name. Assumes instrumentation is already in place (run `/ratel-langfuse-instrument` first). Output is a markdown spec; this skill does not call the Langfuse API.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# /ratel-langfuse-dashboards — design Langfuse dashboards for an agent

Pair this skill with `/ratel-langfuse-instrument`. That skill writes the trace vocabulary; this one builds the dashboard specs that turn that vocabulary into visible value — both Ratel's value and general agent-health visibility.

The output is a markdown plan with one section per dashboard. The customer builds the dashboards by clicking through the Langfuse UI; this is intentional. Auto-creating dashboards via API requires assumptions about the Langfuse instance that we don't make in v1.

## Why two groups of dashboards

Partner startups want two different stories from the same data:

1. **"Ratel is moving the numbers"** — token spend down, retrieval quality up, fewer "tool not found" errors, lower cost per session. These dashboards justify the engagement to whoever signs the cheque. The set evolves as Ratel ships new features (today it's pre-filter + retrieval; v0.1.7 adds skills; v0.1.9 adds suggestion adoption; v0.1.12+ adds semantic + re-rank). The shipped ones go in the customer's dashboard plan; the roadmapped ones get a "we'll add this when X ships" footnote.

2. **"Our agent is healthy"** — latency percentiles, error rates per tool, abandoned-session rates, score distributions. These are useful regardless of Ratel. They build trust because they help the customer's engineers find their own bugs.

We always include both. Ratel-only dashboards feel like a sales pitch; agent-health-only dashboards feel like we forgot why we're there.

## Workflow

### Step 1 — Read prior instrumentation context

Look for `<repo>/docs/ratel-langfuse-instrument.md` (the deliverable of `/ratel-langfuse-instrument`). If present, read the vocabulary table (trace names, observation names, tags, metadata keys). Every dashboard widget references one or more of these — if a widget cites a name/tag/key that isn't in the table, either the table is incomplete or the widget needs to drop.

If the file is missing, ask the user three quick questions before continuing:

1. What's the canonical trace name for one chat turn / one job?
2. What `env`, `stack`, and `agent_version` tag values are in use?
3. Is Ratel instrumented in any form (gateway, SDK, or planned)?

Don't proceed without answers. Dashboards built on guessed vocabulary will look right but pivot wrong.

### Step 2 — Generate the two dashboard groups

Open [`references/ratel-value-map.md`](references/ratel-value-map.md) and [`references/general-agent-dashboards.md`](references/general-agent-dashboards.md). These hold the catalog of dashboards we recommend. Pick the subset that matches what's actually instrumented in this customer's setup.

Default selection:

- **Ratel-value group** (only if Ratel is in or coming): Token Cost & Savings, Retrieval Quality, Gateway Origin Split. Add roadmap-conditional ones only if the customer is on a Ratel pre-release that has the feature, or the customer has explicitly signed up to adopt it (e.g., skill invocation health for v0.1.7).
- **Agent-health group**: Latency & Cost Overview, Error Surface, Tool Usage, Session Quality, Model & Prompt Drift.

You can add more from the catalog if the customer's instrumentation supports them, or drop any that don't have backing data. Skip rather than fake.

### Step 3 — For each dashboard, write the spec

Each dashboard section in the output must have, in this order:

1. **Name** — short, action-oriented (`Token Cost & Savings`, not `Dashboard 1`).
2. **Why it matters** — one paragraph, plain English. The customer's PM should be able to read this and know whether this dashboard is for them.
3. **Required data** — the trace name(s), observation type(s), tags, and metadata keys this dashboard depends on. If anything is missing from the instrumentation, list it here as a TODO blocker.
4. **Widgets** — for each widget in the dashboard, fill in the fields from [`references/widget-cheatsheet.md`](references/widget-cheatsheet.md):
   - Title
   - Data source: traces / observations / scores
   - Metric: count / latency / cost / tokens / score / custom
   - Aggregation: sum / avg / p50 / p95 / p99 / distinct count
   - Dimension(s): time bucket, user, model, trace name, tool name, tag, metadata key
   - Filter(s): tag values, metadata constraints, type constraints
   - Visualization: time-series line / bar / stacked-bar / table / scatter / single-stat
5. **Pivots / drill-downs** — what to click on when the dashboard shows something odd, and where that drill-down lives (a saved trace filter URL the customer pastes in once the dashboard exists).
6. **Roadmap footnote** (Ratel dashboards only) — if the dashboard reaches a state where it would benefit from a Ratel feature that hasn't shipped yet, name the feature and the target version. Reuse the version map in `references/ratel-value-map.md` so this stays current.

### Step 4 — Write the plan

Output to `<repo>/docs/ratel-langfuse-dashboards.md`. Structure:

```
# Langfuse dashboards for <project>

## Summary
- N Ratel-value dashboards, M agent-health dashboards.
- Built on the instrumentation defined in docs/ratel-langfuse-instrument.md.
- Build order: <ordered list, simplest first>.

## Ratel-value dashboards
<one section per dashboard, in build order>

## Agent-health dashboards
<one section per dashboard, in build order>

## Out of scope (for now)
- Dashboards we'd add once the customer adopts <Ratel feature> at v<X.Y.Z>.
- Dashboards that need data we don't have today and haven't planned to add.
```

Print the dashboard list inline in chat (numbered, with the "why it matters" one-liner). Tell the user the file path. Do not paste full widget specs into chat — the file is the artifact.

## Honest skip path

Three skip cases:

1. **No instrumentation in place.** Tell the user to run `/ratel-langfuse-instrument` first, point them at the file path it would produce, and stop.
2. **Instrumentation present but no live data yet.** Build the plan, but mark every dashboard "blocked: no data — re-evaluate after first prod traffic". Empty dashboards demoralise teams.
3. **Customer wants only Ratel-value dashboards and has no Ratel.** Tell them the Ratel dashboards need at least the gateway path wired up; recommend a small Ratel pilot (the gateway alone is a half-day integration) before designing dashboards that depend on it. Do not stuff Ratel hooks into an agent that isn't using Ratel.

## Reference files

- [`references/ratel-value-map.md`](references/ratel-value-map.md) — Ratel feature → observable signal → recommended widget (source of truth)
- [`references/general-agent-dashboards.md`](references/general-agent-dashboards.md) — stack-agnostic dashboard catalog
- [`references/widget-cheatsheet.md`](references/widget-cheatsheet.md) — Langfuse widget vocabulary
