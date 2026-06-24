<div align="center">
  <h1>Ratel Skills</h1>
  <h4>Claude Code / Cursor / Codex skills to work on and improve AI agent codebases.</h4>

  <p>
    <a href="https://github.com/ratel-ai/ratel">Ratel</a> •
    <a href="https://skills.sh">skills.sh</a> •
    <a href="https://discord.gg/hdKpx69NR">Discord</a>
  </p>

  <p>
    <a href="./LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue" alt="license" /></a>
  </p>
</div>

> The fastest way to audit an AI agent codebase (TypeScript or Python), plan its observability, design the dashboards, integrate Ratel's context gateway, fix what the audit finds, and analyse live traces — nine skills, one engagement arc.

## Install

```bash
npx skills add ratel-ai/skills -y --all
```

That installs all nine skills globally (into `~/.claude/skills/` for Claude Code, the equivalent location for other supported agents). The `-y` flag accepts all prompts; `--all` installs all skills into all agents.

Per-skill install:

```bash
npx skills add ratel-ai/skills --skill ratel-assessment
```

Local development install (when iterating on the suite itself):

```bash
git clone https://github.com/ratel-ai/skills
npx skills add ./skills
```

The `npx skills` CLI is [Vercel Labs' open agent skills tool](https://github.com/vercel-labs/skills) — compatible with Claude Code, Cursor, Codex, OpenCode, Gemini CLI, and 40+ other coding agents.

## Quickstart — paste this into your coding agent

Want a read on your agent first? The assessment is free and static — no setup:

```text
Run npx skills add ratel-ai/skills --all and use the skills to assess the agents in this codebase and show me where Ratel would help.
```

Ready to wire Ratel in? Go straight to the integration plan:

```text
Run npx skills add ratel-ai/skills --all and use the skills to integrate Ratel in this project.
```

Either entry point pulls in the whole suite. The assessment writes a report to `.ratel/ratel-assessment-<date>.md` — plus a branded, scored HTML version (`.html`) alongside it — and ends with "Recommended next steps" — every finding routes to the right follow-up skill (observability assessment, a vendor integrate/analyze pair, the Ratel gateway integrate, or one of the two fix-skills), so you don't have to know the arc up front.

## What's inside

Nine skills. The first seven run the engagement arc, in the order a partner engagement typically uses them; the last two are the fix-skills the assessment routes to when it finds a long system prompt or weak tool/skill definitions.

| Skill | What it does | When to fire |
| --- | --- | --- |
| [`ratel-assessment`](./skills/ratel-assessment/SKILL.md) | Static read of the agent codebase (TypeScript or Python). Produces a polished assessment report at `<repo>/.ratel/ratel-assessment-<date>.{md,html}` — markdown plus a branded, scored HTML version (gauge, radar, score bars) — with a 12-dimension scorecard (0–10 per dimension + overall composite), evidence-backed findings, and conditional pointers to the right follow-up skill. | First touch. Zero partner setup. |
| [`ratel-observability-assessment`](./skills/ratel-observability-assessment/SKILL.md) | Generic, vendor-neutral: detects the observability vendor (or asks), proposes how to instrument and which dashboards to add, and routes to the matching vendor integrate skill. Writes a proposal at `<repo>/.ratel/ratel-observability-assessment.md`. | After assessment flags Observability as Weak / Missing. |
| [`ratel-langfuse-integrate`](./skills/ratel-langfuse-integrate/SKILL.md) | Langfuse-specific: concrete SDK wiring, MCP registration, naming/mapping table, file-by-file instrumentation changes, and the dashboard build-spec (Ratel-value + agent-health). | When the observability assessment detects / picks Langfuse. |
| [`ratel-langfuse-analyze`](./skills/ratel-langfuse-analyze/SKILL.md) | Reads live Langfuse traces via the Langfuse MCP server, drills into outliers, pattern-matches against a finding catalog, writes a findings report split into Ratel-flavored and stack-agnostic improvements. | After traffic accumulates under the A/B split. |
| [`ratel-langsmith-integrate`](./skills/ratel-langsmith-integrate/SKILL.md) | LangSmith-specific: concrete SDK wiring, MCP registration, run-tree naming/mapping table, file-by-file instrumentation changes, and the dashboard build-spec (Monitor + custom charts). | When the observability assessment detects / picks LangSmith. |
| [`ratel-langsmith-analyze`](./skills/ratel-langsmith-analyze/SKILL.md) | Reads live LangSmith runs via the LangSmith MCP server (SDK fallback), drills into outliers, pattern-matches against the shared finding catalog, writes a findings report split into Ratel-flavored and stack-agnostic improvements. | After traffic accumulates under the A/B split. |
| [`ratel-integrate`](./skills/ratel-integrate/SKILL.md) | Plans the Ratel **gateway** rollout: detects tool management, fetches up-to-date Ratel docs, picks integration mode (direct SDK / MCP gateway / hybrid), picks pilot scope, designs the A/B test, ties success to specific observability metrics. Covers both the TypeScript (`@ratel-ai/sdk`) and Python (`ratel-ai`) SDKs. | After observability is in. |
| [`ratel-decompose-prompt`](./skills/ratel-decompose-prompt/SKILL.md) | Breaks a long, monolithic system prompt into a lean, stable core prompt plus extractable Ratel skills (reusable playbooks) registered in a `SkillCatalog` and surfaced on demand via `search_capabilities`. Writes a plan at `<repo>/.ratel/ratel-decompose-prompt.md`. | When assessment flags Prompt decomposition (Dimension 11) as Weak / Missing. |
| [`ratel-tune-definitions`](./skills/ratel-tune-definitions/SKILL.md) | Optimizes tool and skill definitions — descriptions, names, parameter names, enums, schemas, tags — for BM25 retrievability and model usability. Writes a before/after plan at `<repo>/.ratel/ratel-tune-definitions.md`. | When assessment flags Definition quality (Dimension 12) as Weak / Missing. |

## The engagement arc

```
ratel-assessment                 → "here's what's weak; here's where Ratel fits"
        ↓ (Observability Weak/Missing)
ratel-observability-assessment   → "how to instrument + which dashboards, generically; you're on <vendor>"
        ↓ (vendor branch)
   ├─ Langfuse  → ratel-langfuse-integrate   → ratel-langfuse-analyze
   └─ LangSmith → ratel-langsmith-integrate  → ratel-langsmith-analyze
        ↓
ratel-integrate                  → "here's how to roll Ratel's gateway out + A/B it"  (needs observability in first)

ratel-assessment also branches to two fix-skills, conditional on findings:
        ├─ Prompt decomposition weak → ratel-decompose-prompt
        │                              "split the monolithic prompt into a lean core + skills"
        └─ Definition quality weak   → ratel-tune-definitions
                                       "rewrite tool/skill definitions for retrieval + selection"
```

Each skill's "Recommended next steps" section names which sibling to run next based on what it found — the arc isn't a forced sequence, it's a conditional flow.

## Shared sources of truth

The skills reference each other so the vocabulary stays consistent across an engagement. No file is duplicated. The vendor-neutral references live under `ratel-observability-assessment/references/`; each vendor's concrete renderings live under that vendor's integrate skill.

- **Semantic conventions (naming vocabulary)** — `skills/ratel-observability-assessment/references/semantic-conventions.md`. The canonical, vendor-neutral span / session / tag / attribute names. Each vendor integrate skill maps these onto its primitives.
- **Ratel feature → signal → version map** — `skills/ratel-observability-assessment/references/ratel-value-map.md`. The single, conceptual source of truth for "what Ratel ships when." Read by `ratel-assessment` and both `*-analyze` skills for their Ratel angles.
- **General agent dashboards** — `skills/ratel-observability-assessment/references/general-agent-dashboards.md`. The vendor-neutral agent-health dashboard set each vendor integrate skill renders.
- **Finding catalog** — `skills/ratel-observability-assessment/references/finding-catalog.md`. The vendor-neutral failure modes both `*-analyze` skills pattern-match against.
- **Vendor-specific mapping / hooks / widget files** — under each vendor integrate skill (e.g. `skills/ratel-langfuse-integrate/references/ratel-hooks.md`, `langfuse-value-map.md`, `widget-cheatsheet.md`). These render the conventions above onto the vendor's primitives.

When Ratel ships a new feature, update `ratel-observability-assessment/references/ratel-value-map.md` (the conceptual map) **and** the per-vendor value-map renderings (e.g. `ratel-langfuse-integrate/references/langfuse-value-map.md`). The angles in the assessment catalog and the findings in the analyze catalogs cascade from the conceptual map.

## Use them in your engagement

For Ratel-side readers — when to fire each skill:

- **First conversation** with a partner: fire `/ratel-assessment` against their repo before any sales pitch. The report is the credibility layer; "let's pilot Ratel" is the conclusion the partner reaches on their own once they've read it.
- **After they say yes** to a pilot: fire `/ratel-observability-assessment` to plan observability vendor-neutrally and route to the vendor integrate skill (`/ratel-langfuse-integrate` or `/ratel-langsmith-integrate`), which gets instrumentation and dashboards in place (a Ratel rollout you can't measure is indistinguishable from no rollout).
- **Rollout week**: fire `/ratel-integrate` for the Ratel gateway file-by-file plan + A/B design.
- **Two weeks in**: fire your vendor analyze skill (`/ratel-langfuse-analyze` or `/ratel-langsmith-analyze`) against live traces to find both Ratel-flavored deepening opportunities and stack-agnostic low-hanging fruit you can spot for them.
- **Whenever the assessment flags it**: fire `/ratel-decompose-prompt` to break a bloated system prompt into a lean core plus retrievable skills, or `/ratel-tune-definitions` to sharpen weak tool/skill definitions. Both are fix-skills — they turn an assessment finding into a concrete, implementable plan.

The reports each skill writes are markdown files written into the partner's repo under `.ratel/` — they accumulate and become the artifact of the engagement.

## Conventions

Every skill in this suite follows the same rules:

- **No emojis. Imperative voice.** The skills are for engineering audiences who tune out marketing tone.
- **Pushy multi-phrase descriptions.** Each skill's frontmatter trigger description names many phrasings so the skill fires reliably from natural language.
- **Honest skip path.** If a skill's preconditions aren't met (no agent surface, no observability, no traffic), it stops and says so. It does not fabricate a deliverable.
- **Markdown output, committed into the partner repo.** All deliverables go to `<repo>/.ratel/` so they're reviewable, diff-able, and accumulate across runs.
- **Cross-skill references, not duplication.** Shared vocabulary lives in one place; every other skill links to it.

## License

[MIT](./LICENSE). Free to use, modify, redistribute.
