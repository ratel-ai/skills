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

> The fastest way to audit an AI agent codebase (TypeScript or Python), plan its observability, design the dashboards, integrate Ratel's context gateway, fix what the audit finds, and analyse live traces — seven skills, one engagement arc.

## Install

```bash
npx skills add ratel-ai/skills -y --all
```

That installs all seven skills globally (into `~/.claude/skills/` for Claude Code, the equivalent location for other supported agents). The `-y` flag accepts all prompts; `--all` installs all skills into all agents.

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

## Quickstart — copy-paste this into your coding agent

```text
I want you to assess my agent codebase and produce a Ratel-flavored
report so we can see what's weak and where Ratel would help.

Step 1 — install the Ratel skills suite:

  npx skills add ratel-ai/skills -y --all

Step 2 — run the `ratel-assessment` skill on this repository. It
will produce a markdown report at `.ratel/ratel-assessment-<date>.md`
with a 12-dimension scorecard, evidence-backed findings, and a
"Where Ratel fits" section.

Once we've reviewed the report together, run the `ratel-integrate`
skill to produce a concrete rollout plan (integration mode, pilot
scope, A/B test design, Langfuse metrics) at
`.ratel/ratel-integrate.md`.

Show me the scorecard inline and link to the report file.
```

Two skills are surfaced explicitly here. The other five are reachable via the assessment report's "Recommended next steps" section — every finding routes to the right follow-up skill, whether that's observability work or a concrete fix.

## What's inside

Seven skills. The first five run the engagement arc, in the order a partner engagement typically uses them; the last two are the fix-skills the assessment routes to when it finds a long system prompt or weak tool/skill definitions.

| Skill | What it does | When to fire |
| --- | --- | --- |
| [`ratel-assessment`](./skills/ratel-assessment/SKILL.md) | Static read of the agent codebase (TypeScript or Python). Produces a polished assessment report at `<repo>/.ratel/ratel-assessment-<date>.md` with a 12-dimension scorecard, evidence-backed findings, and conditional pointers to the right follow-up skill. | First touch. Zero partner setup. |
| [`ratel-langfuse-instrument`](./skills/ratel-langfuse-instrument/SKILL.md) | Detects the agent stack, maps topology, decides naming + tagging vocabulary, writes a file-by-file Langfuse instrumentation plan. | After assessment flags Observability as Weak / Missing. |
| [`ratel-langfuse-dashboards`](./skills/ratel-langfuse-dashboards/SKILL.md) | Designs two dashboard groups — Ratel-value (Token Cost & Savings, Retrieval Quality, Gateway Origin Split, Upstream Health, Skill Retrieval Health) and general agent health — as a markdown spec the partner builds by hand in the Langfuse UI. | After instrumentation lands. |
| [`ratel-integrate`](./skills/ratel-integrate/SKILL.md) | Plans the Ratel rollout: detects tool management, fetches up-to-date Ratel docs, picks integration mode (direct SDK / MCP gateway / hybrid), picks pilot scope, designs the A/B test, ties success to specific Langfuse metrics. Covers both the TypeScript (`@ratel-ai/sdk`) and Python (`ratel-ai`) SDKs. | After observability is in. |
| [`ratel-langfuse-analyze`](./skills/ratel-langfuse-analyze/SKILL.md) | Reads live Langfuse traces via the Langfuse MCP server, drills into outliers, pattern-matches against a finding catalog, writes a findings report split into Ratel-flavored and stack-agnostic improvements. | After traffic accumulates under the A/B split. |
| [`ratel-decompose-prompt`](./skills/ratel-decompose-prompt/SKILL.md) | Breaks a long, monolithic system prompt into a lean, stable core prompt plus extractable Ratel skills (reusable playbooks) registered in a `SkillCatalog` and surfaced on demand via `search_capabilities`. Writes a plan at `<repo>/.ratel/ratel-decompose-prompt.md`. | When assessment flags Prompt decomposition (Dimension 11) as Weak / Missing. |
| [`ratel-tune-definitions`](./skills/ratel-tune-definitions/SKILL.md) | Optimizes tool and skill definitions — descriptions, names, parameter names, enums, schemas, tags — for BM25 retrievability and model usability. Writes a before/after plan at `<repo>/.ratel/ratel-tune-definitions.md`. | When assessment flags Definition quality (Dimension 12) as Weak / Missing. |

## The engagement arc

```
ratel-assessment          → "here's what's weak; here's where Ratel fits"
        ↓
ratel-langfuse-instrument → "here's how to see what's happening"
        ↓
ratel-langfuse-dashboards → "here's what to put on the screens"
        ↓
ratel-integrate           → "here's how to roll Ratel out + A/B it"
        ↓
ratel-langfuse-analyze    → "here's what the data says after the rollout"

ratel-assessment also branches to two fix-skills, conditional on findings:
        ├─ Prompt decomposition weak → ratel-decompose-prompt
        │                              "split the monolithic prompt into a lean core + skills"
        └─ Definition quality weak   → ratel-tune-definitions
                                       "rewrite tool/skill definitions for retrieval + selection"
```

Each skill's "Recommended next steps" section names which sibling to run next based on what it found — the arc isn't a forced sequence, it's a conditional flow.

## Shared sources of truth

The skills reference each other so the vocabulary stays consistent across an engagement. No file is duplicated:

- **Naming vocabulary** — `skills/ratel-langfuse-instrument/references/naming-conventions.md`. The canonical trace / observation / tag / metadata names. Read by every other skill.
- **Ratel feature → signal → version map** — `skills/ratel-langfuse-dashboards/references/ratel-value-map.md`. The single source of truth for "what Ratel ships when." Read by `ratel-assessment` and `ratel-langfuse-analyze` for their Ratel angles.
- **Trace event → Langfuse observation mapping** — `skills/ratel-langfuse-instrument/references/ratel-hooks.md`. Read by `ratel-integrate`.

When Ratel ships a new feature, the value-map is the one file to update — the angles in the assessment catalog and the findings in the analyze catalog cascade from it.

## Use them in your engagement

For Ratel-side readers — when to fire each skill:

- **First conversation** with a partner: fire `/ratel-assessment` against their repo before any sales pitch. The report is the credibility layer; "let's pilot Ratel" is the conclusion the partner reaches on their own once they've read it.
- **After they say yes** to a pilot: fire `/ratel-langfuse-instrument` to get observability in place (a Ratel rollout you can't measure is indistinguishable from no rollout).
- **Before the rollout**: fire `/ratel-langfuse-dashboards` to set up what you'll measure.
- **Rollout week**: fire `/ratel-integrate` for the file-by-file plan + A/B design.
- **Two weeks in**: fire `/ratel-langfuse-analyze` against live traces to find both Ratel-flavored deepening opportunities and stack-agnostic low-hanging fruit you can spot for them.
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
