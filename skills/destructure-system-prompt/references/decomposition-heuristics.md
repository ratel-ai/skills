# Decomposition heuristics

The rubric for turning one monolithic prompt into a lean core plus skills. The `SKILL.md` workflow walks this file at Step 3 (classify) and Step 4 (boundaries).

## Finding the seams

A monolithic prompt is rarely a single thought. Look for the natural boundaries:

- **Section headings** (`## Tone`, `# Tools`, `### When the user asks for X`). The author already segmented it; trust their boundaries first, then question them.
- **Conditional language** — "If the user...", "When working with...", "For requests involving...". A conditional opener is the single strongest signal of a skill: it names its own trigger.
- **Procedure blocks** — numbered or bulleted step lists. A procedure is a cohesive unit; keep it whole.
- **Voice shifts** — a jump from "you are a warm assistant" (identity) to "always return valid JSON matching this schema" (output contract) to "to query the warehouse, use the following SQL conventions" (domain procedure). Each shift is a seam.
- **Reference dumps** — long tables, enumerations, example galleries, style guides. These are the highest-value extractions: they are heavy and usually conditional.

## Keep-vs-extract rubric

Classify each block. Score it against these questions; the answers point to a disposition.

| Question | If YES → |
| --- | --- |
| Is it the agent's identity, persona, or voice? | CORE |
| Is it a universal safety, legal, or compliance rule that must hold on every turn? | CORE |
| Is it the output contract applied to essentially all responses? | CORE |
| Is it referenced or relied on across most turns regardless of task? | CORE |
| Does it open with a condition ("if/when/for requests about...")? | SKILL |
| Is it a procedure for one tool, one domain, or one task type? | SKILL |
| Is it a long reference (table, catalog, examples) used only sometimes? | SKILL (with a reference file) |
| Would a turn on an unrelated task be unaffected if this were absent? | SKILL |
| Is it duplicated elsewhere, contradicted, or apparently dead? | DROP (flag only) |

Tie-breakers:

- **CORE wins ties on safety.** If a rule is safety-relevant and you are unsure whether it is always needed, keep it in the core. A skill that fails to load must never be able to drop a guardrail.
- **SKILL wins ties on weight.** A heavy block (long tables, many examples) that is even sometimes irrelevant is worth extracting purely for the token savings on the turns it does not apply.
- **Never DROP on your own.** Redundant-looking blocks sometimes encode a deliberate emphasis. List them with the reason and let the user decide; default to keeping.

## Sizing intuition

The goal is to shrink the always-on context. A useful frame:

- **Core target**: identity + universal rules + output contract. For most agents this is well under half the original prompt; often a quarter.
- **Per-skill cost**: a skill costs ~0 tokens until its trigger fires, then costs its own size. So extracting a 600-token domain procedure that fires on 10% of turns saves ~540 tokens of average per-turn context.
- **Break-even**: a block that fires on nearly every turn saves almost nothing by being a skill, and adds trigger-matching overhead. Such blocks belong in the core.

## Boundary checks

After grouping SKILL blocks into proposed skills, run these checks:

1. **One trigger per skill.** Write the trigger condition in one sentence. If it needs "and also when...", you have two skills.
2. **Co-firing merge.** If two proposed skills would always load together, merge them — separate skills that never fire independently are pure overhead.
3. **Dependency sweep.** For every term, variable, or definition a moved block uses, grep the original for where it was defined. If the definition stays in the core, fine. If it lived in a different block, either co-locate the two or promote the definition into the core. This is the most common correctness bug in a decomposition.

   ```bash
   # After deciding a block moves, find references to terms it defines/uses
   rg -n 'TERM_OR_NAME' path/to/original-prompt
   ```

4. **Self-containment.** Read each proposed skill body as if the rest of the prompt does not exist. If it references "as described above" or assumes adjacent context, rewrite it to stand alone or pull in what it needs.

## Coverage map

The decomposition is only trustworthy if it is lossless. Maintain a table mapping every original line range to its destination:

```
| Original lines | Summary                    | Destination          |
| -------------- | -------------------------- | -------------------- |
| 1–18           | Identity + voice           | CORE                 |
| 19–34          | Universal safety rules     | CORE                 |
| 35–88          | SQL query conventions      | skill: sql-queries   |
| 89–140         | Chart/format catalog       | skill: chart-output  |
| 141–146        | Duplicate of lines 19–24   | DROP (flagged)       |
```

Every line of the original must appear in exactly one row. Gaps mean lost behavior; overlaps mean duplicated instructions. Resolve both before declaring the decomposition done.
