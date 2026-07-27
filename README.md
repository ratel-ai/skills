<div align="center">
  <h1>Ratel Skills</h1>
  <h4>Claude Code / Cursor / Codex skills to work on and improve AI agent codebases.</h4>

  <p>
    <a href="https://docs.ratel.sh">Docs</a> •
    <a href="https://github.com/ratel-ai/ratel">Ratel</a> •
    <a href="https://skills.sh">skills.sh</a> •
    <a href="https://discord.gg/75vAPdjYqT">Discord</a>
  </p>

  <p>
    <a href="https://discord.gg/75vAPdjYqT"><img src="https://img.shields.io/discord/1478702964003705015?logo=discord&logoColor=white&label=Discord&color=5865F2" alt="Discord" /></a>
    <a href="./LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue" alt="license" /></a>
  </p>
</div>

> The fastest way to audit an AI agent codebase (TypeScript or Python), plan its observability, integrate Ratel, and fix what the audit finds — five skills, one engagement arc.

## Quickstart — paste this into your coding agent

Want a read on your agent first? The assessment is free and static — no setup:

```text
Run npx skills add ratel-ai/skills --all and use the skills to assess the agents in this codebase
```

Ready to wire Ratel in? Go straight to the code integration:

```text
Run npx skills add ratel-ai/skills --all and use the skills to integrate Ratel in this project.
```

Either entry point pulls in the whole suite. The assessment writes a report to `.ratel/ratel-assessment-<date>.md` — plus a branded, scored HTML version (`.html`) alongside it — and ends with "Recommended next steps": every finding routes to the right follow-up skill (the observability assessment, the Ratel integration, or one of the two fix-skills), so you don't have to know the arc up front.

## Install

```bash
npx skills add ratel-ai/skills -y --all
```

That installs all five skills into the current project (`./.claude/skills/` for Claude Code, the equivalent location for other supported agents). The `-y` flag accepts all prompts; `--all` installs all skills into all agents; add `-g` to install globally (`~/.claude/skills/`).

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

## What's inside

Five skills. The assessment routes to an optional observability proposal, a direct code integration, or one of two focused fix-skills.

| Skill | What it does | When to fire |
| --- | --- | --- |
| [`ratel-assessment`](./skills/ratel-assessment/SKILL.md) | Static read of the agent codebase (TypeScript or Python). Produces a polished assessment report at `<repo>/.ratel/ratel-assessment-<date>.{md,html}` — markdown plus a branded, scored HTML version (gauge, radar, score bars) — with a 12-dimension scorecard (0–10 per dimension + overall composite), evidence-backed findings, and conditional pointers to the right follow-up skill. | First touch. Zero partner setup. |
| [`ratel-observability-assessment`](./skills/ratel-observability-assessment/SKILL.md) | Detects the agent surface and the OpenTelemetry backend the team runs, plans how to turn on Ratel's native OTLP telemetry, and proposes the dashboards that prove Ratel's value. Writes a proposal at `<repo>/.ratel/ratel-observability-assessment.md`. | After assessment flags Observability as Weak / Missing. |
| [`ratel-integrate`](./skills/ratel-integrate/SKILL.md) | Maps every agent's tool and skill ownership, then edits the codebase to load those capabilities through Ratel. Uses the official Vercel AI SDK or Mastra adapter when applicable and the core SDK elsewhere; preserves agent boundaries and verifies discovery, invocation, and skill loading. | When ready to integrate Ratel. No observability prerequisite. |
| [`ratel-decompose-prompt`](./skills/ratel-decompose-prompt/SKILL.md) | Breaks a long, monolithic system prompt into a lean, stable core prompt plus extractable Ratel skills (reusable playbooks) registered in a `SkillCatalog` and surfaced on demand via `search_capabilities`. Writes a plan at `<repo>/.ratel/ratel-decompose-prompt.md`. | When assessment flags Prompt decomposition (Dimension 11) as Weak / Missing. |
| [`ratel-tune-definitions`](./skills/ratel-tune-definitions/SKILL.md) | Optimizes tool and skill definitions — descriptions, names, parameter names, enums, schemas, tags — for retrievability (Ratel's hybrid lexical + semantic ranking) and model usability. Writes a before/after plan at `<repo>/.ratel/ratel-tune-definitions.md`. | When assessment flags Definition quality (Dimension 12) as Weak / Missing. |

## The engagement arc

```text
ratel-assessment                 → "here's what's weak; here's where Ratel fits"
        ├─ Ready to wire Ratel       → ratel-integrate
        │                              "load each agent's tools + skills through the right adapter"
        ├─ Observability weak        → ratel-observability-assessment
        │                              "plan native OTLP export + dashboards"
        ├─ Prompt decomposition weak → ratel-decompose-prompt
        │                              "split the monolithic prompt into a lean core + skills"
        └─ Definition quality weak   → ratel-tune-definitions
                                       "rewrite tool/skill definitions for retrieval + selection"
```

`ratel-integrate` can also run directly when the user already wants the code change. Observability is an independent branch, not an integration gate.

## How Ratel observability works

Ratel telemetry **is** OpenTelemetry. The SDKs emit the retrieval-and-tool funnel natively as `gen_ai.*` spans plus a `ratel.*` overlay (`ratel.search`, `execute_tool`, `ratel.skill.load`, …), exported as stock OTLP — so you observe a Ratel rollout by pointing those spans at the OTel backend you already run (Langfuse, LangSmith, your own collector, or [Ratel Cloud](https://docs.ratel.sh/docs/cloud), coming soon). Ratel does not emit LLM-call spans; its funnel joins the same trace next to your own model instrumentation. `ratel-observability-assessment` plans this setup — greenfield or dual-exported into an existing provider — and the dashboards that prove value.

## Shared sources of truth

The skills reference each other so the vocabulary stays consistent across an engagement. No file is duplicated. The vendor-neutral references live under `ratel-observability-assessment/references/`; the other skills link to them.

- **Semantic conventions (naming vocabulary)** — `skills/ratel-observability-assessment/references/semantic-conventions.md`. Ratel's native OpenTelemetry conventions (`gen_ai.*` + the `ratel.*` overlay): span names, session sourcing, tags, and attributes. Read by `ratel-assessment` and `ratel-observability-assessment`.
- **Native telemetry setup** — `skills/ratel-observability-assessment/references/native-telemetry-setup.md`. How to turn Ratel's OTLP telemetry on (greenfield `configureTelemetry()` / `configure_telemetry()`, or dual-export via `ratelSpanProcessor()` / `ratel_span_processor()`), in TypeScript and Python.
- **Ratel feature → signal → version map** — `skills/ratel-observability-assessment/references/ratel-value-map.md`. The single source of truth for "what Ratel ships when" and the observable signal each capability produces. Read by `ratel-assessment` and `ratel-observability-assessment`.
- **General agent dashboards** — `skills/ratel-observability-assessment/references/general-agent-dashboards.md`. The vendor-neutral agent-health dashboard set to render in whichever OTel backend you run.
- **Finding catalog** — `skills/ratel-observability-assessment/references/finding-catalog.md`. The failure modes to pattern-match against when reviewing live traces.

When Ratel ships a new feature, update `ratel-observability-assessment/references/ratel-value-map.md` — the angles in the assessment catalog and the dashboards in the observability proposal cascade from it.

## Use them in your engagement

For Ratel-side readers — when to fire each skill:

- **First conversation** with a partner: fire `/ratel-assessment` against their repo before any sales pitch. The report is the credibility layer; "let's pilot Ratel" is the conclusion the partner reaches on their own once they've read it.
- **When observability is the problem**: fire `/ratel-observability-assessment` to plan native OTLP export and value dashboards in the backend the team already runs.
- **When ready to change code**: fire `/ratel-integrate` to inventory each agent's tools and skills, wire the supported SDK or framework adapter, and verify the resulting capability boundaries.
- **Whenever the assessment flags it**: fire `/ratel-decompose-prompt` to break a bloated system prompt into a lean core plus retrievable skills, or `/ratel-tune-definitions` to sharpen weak tool/skill definitions. Both are fix-skills — they turn an assessment finding into a concrete, implementable plan.

The audit and planning skills write reviewable Markdown under `.ratel/`. `ratel-integrate` changes the agent code and dependencies directly.

## Conventions

Every skill in this suite follows the same rules:

- **No emojis. Imperative voice.** The skills are for engineering audiences who tune out marketing tone. (One deliberate exception: `ratel-assessment` uses a couple of emojis in its final chat output when offering to open the HTML report, so the reader can't miss it.)
- **Concise, trigger-rich descriptions.** Each skill's frontmatter description states what it does, when to use it, and its output boundary in a few lines — enough distinct trigger vocabulary to fire reliably from natural language, no redundant paraphrase chains. We practice the context discipline we sell.
- **Honest skip path.** If a skill's required surface is absent, it stops and says so. It does not fabricate a deliverable.
- **Reviewable output.** Audits and plans go to `<repo>/.ratel/`; implementation work stays in the source, dependency, and test files it changes.
- **Cross-skill references, not duplication.** Shared vocabulary lives in one place; every other skill links to it.

## License

[MIT](./LICENSE). Free to use, modify, redistribute.
