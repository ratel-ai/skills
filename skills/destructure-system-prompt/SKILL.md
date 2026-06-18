---
name: destructure-system-prompt
description: |
  Take a large, monolithic system prompt and break it into a lean core prompt plus a set of focused, load-on-demand skills. Map the prompt into cohesive blocks, decide which stay always-on (identity, universal rules, output contract) and which become skills (conditional workflows, domain procedures, reference-heavy instructions), then author each skill with a reliable trigger description and rewrite the slimmed core. Use whenever the user wants to shrink a bloated prompt, asks "split this system prompt into skills", "destructure our prompt", "this prompt is too long, modularize it", "turn these instructions into skills", "extract skills from our agent prompt", "reduce always-on context", or invokes `/destructure-system-prompt`. Trigger on phrases like "our system prompt is 8k tokens and growing", "carve this prompt into reusable pieces", "move these conditional instructions out of the base prompt", "make this prompt load on demand" — even if the user doesn't say "skill" or "destructure-system-prompt" by name. This skill writes skill files and a slimmed core prompt; it preserves behavior and reports a coverage map so nothing is silently dropped.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
---

# /destructure-system-prompt — split a monolithic prompt into a lean core + skills

A system prompt that grows by accretion ends up paying full token cost on every single turn for instructions that only matter on a fraction of them. The fix is the same shape as code refactoring: keep the always-relevant core small, and move conditional, domain-specific, or reference-heavy material into skills that load only when their trigger fires.

This skill performs that decomposition. It reads the monolithic prompt, finds the seams, decides what stays versus what becomes a skill, authors each skill with a trigger description tuned to fire reliably, and rewrites the slimmed core. The deliverable is a set of `SKILL.md` files (plus reference files where a skill is large) and a rewritten core prompt — together with a **coverage map** proving every line of the original lands somewhere.

## Philosophy

Three rules. Break any of them and the decomposition either loses behavior or fails to fire when needed.

1. **Preserve behavior — prove it with a coverage map.** Every block of the original prompt must end up either in the slimmed core or in exactly one skill. Produce a line-range → destination table. If a block has no home, it is not done. Never silently drop instructions because they seem redundant; flag them and let the user decide.
2. **The core is for what is always true.** Identity, voice, universal safety rules, the output contract, and anything referenced on nearly every turn stay in the core. Everything conditional moves out. When unsure, ask: "does this matter on a turn where the user is doing something unrelated?" If no, it is a skill.
3. **A skill is only worth extracting if its trigger is nameable.** A skill that cannot describe *when* it should load is dead weight — it either never fires or has to be invoked by hand. If you cannot write a crisp trigger condition for a block, it belongs in the core or merged into a sibling skill, not split out.

## Workflow

### Step 1 — Locate and read the prompt

Find the monolithic prompt. It may be a markdown/text file, a large string literal in code, a template, or pasted inline by the user.

```bash
# Common locations for prompt text
rg -l --glob '*.{md,txt,py,ts,tsx,js,jsx,yaml,yml,j2}' \
  -e 'system_prompt|systemPrompt|SYSTEM_PROMPT|You are |You're an? (assistant|agent)' . | head -20

# Large string literals that look like prompts (long, multi-line)
rg -n --multiline -e '"""[\s\S]{800,}"""' --glob '*.py' . | head
rg -n -e '`[\s\S]{800,}`' --glob '*.{ts,tsx,js,jsx}' . | head
```

If the user pasted the prompt inline, use that. If you cannot find a prompt and none was provided, use the [honest skip path](#honest-skip-path).

Read the whole prompt before splitting anything. Note its total size (lines / approx tokens) — that is the number the decomposition is trying to shrink.

### Step 2 — Map the prompt into blocks

Walk the prompt top to bottom and segment it into cohesive blocks. A block is a run of instruction that shares one purpose — a section heading, a numbered procedure, a rules list, an output-format spec, a persona paragraph. Record each block's line range and a one-line summary.

For large prompts (more than ~400 lines), launch one **Explore** agent with a scoped prompt to produce the block map, then review it. For smaller prompts, do it directly. Do not split blocks across the seam of a single coherent procedure — keep a workflow together.

### Step 3 — Classify each block: keep vs extract

Apply the rubric in [`references/decomposition-heuristics.md`](references/decomposition-heuristics.md) to every block. Each block gets exactly one disposition:

- **CORE** — always-on. Identity, voice, universal safety/compliance rules, the output contract, anything referenced on nearly every turn.
- **SKILL** — conditional. Domain workflows, tool-specific procedures, format-specific generators, reference-heavy instructions that only apply to a subset of turns.
- **DROP (flag only)** — appears redundant, contradictory, or dead. Never drop silently: list it for the user with the reason. Default to KEEP if uncertain.

### Step 4 — Define skill boundaries

Group the SKILL blocks into coherent skills. The unit is **one trigger condition, one skill**. Two blocks that always fire together belong in one skill; a block that fires under a distinct condition is its own skill. Watch for:

- **Hidden dependencies** — a block that references a definition living in another block. Either keep them together or promote the shared definition into the core (or a reference file the skill reads).
- **Over-splitting** — five micro-skills that always co-fire add overhead without benefit. Merge them.
- **Under-splitting** — one skill bundling three unrelated triggers fires too often and loads irrelevant context. Split it.

For each proposed skill, write its trigger condition in one sentence before authoring. If you cannot, it is not a skill (rule 3).

### Step 5 — Author each skill

For each skill, write `<skill-name>/SKILL.md` following [`references/skill-authoring.md`](references/skill-authoring.md). Each skill needs:

- **Frontmatter**: `name` (kebab-case, matches directory), `description` (the load-bearing trigger text — many phrasings, written so it fires reliably from natural language), and `allowed-tools` if the skill needs a constrained tool set.
- **Body**: the extracted instructions, rewritten in imperative voice as a self-contained procedure. The skill must not assume the reader still has the surrounding prompt context that used to sit next to it.
- **Reference files**: if a skill carries long tables, catalogs, or examples, move them to `references/` and link them, so the `SKILL.md` itself stays scannable (progressive disclosure).

Match the conventions of any existing skills in the target repository — frontmatter shape, voice, file layout — so the new skills are consistent with their neighbors.

### Step 6 — Rewrite the lean core

Produce the slimmed core prompt: the CORE blocks, reassembled in a sensible order, plus — only if the host runtime does not surface skills automatically — a short pointer telling the model that skills exist and load on demand. Do not inline skill bodies back into the core; that defeats the purpose. The core should be materially shorter than the original; report the before/after size.

### Step 7 — Verify and report the coverage map

Print, in this order:

1. **Coverage map** — a table: original line range → destination (CORE / `<skill-name>` / DROP-flagged). Every line of the original appears exactly once. This is the proof that behavior was preserved.
2. **Size delta** — original core size → new core size (lines / approx tokens), and the count + names of skills extracted.
3. **Flagged drops**, if any — each with the reason, for the user to confirm.
4. **File paths** of every file written (skills + core).

If any original block is unaccounted for, stop and resolve it before declaring done.

## Anti-patterns

- **Splitting by topic instead of by trigger.** A skill exists to load conditionally. "All the formatting rules" is a topic; it is only a skill if formatting is conditional. If it is always-on, it stays in the core.
- **Triggers that never fire.** A description full of jargon the user would never type is a skill that sits idle. Write triggers in the user's words — the symptoms and phrasings they actually use.
- **Triggers that always fire.** A description so broad it matches every turn reintroduces the bloat you were removing. If a skill loads on nearly every turn, it belonged in the core.
- **Orphaned cross-references.** Moving a block that defined a term used elsewhere, without relocating the definition. The coverage map plus a grep for moved terms catches this.
- **Lossy compression disguised as refactoring.** "Tightening" instructions while moving them changes behavior. Move first, preserve wording; propose rewrites separately and let the user approve.

## Honest skip path

If you cannot locate a system prompt and the user did not provide one, stop. Do not invent a prompt to split. Tell the user:

> No system prompt found — checked `<files looked at>`. Point me at the prompt file (or paste the prompt) and I'll re-run.

If the prompt is already small and cohesive (roughly under 150 lines with no clearly conditional sections), say so and recommend against splitting rather than manufacturing skills that will never earn their overhead.

## Reference files

- [`references/decomposition-heuristics.md`](references/decomposition-heuristics.md) — the keep-vs-extract rubric, how to find seams, dependency and over/under-splitting checks. The load-bearing file.
- [`references/skill-authoring.md`](references/skill-authoring.md) — frontmatter spec, how to write a trigger description that fires reliably, progressive disclosure, and naming.
