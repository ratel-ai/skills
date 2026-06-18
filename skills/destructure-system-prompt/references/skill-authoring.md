# Skill authoring

How to write each extracted skill so it fires when it should and reads cleanly when loaded. The `SKILL.md` workflow walks this file at Step 5.

## File layout

One skill is one directory:

```
<skill-name>/
  SKILL.md                 # frontmatter + body; the entry point
  references/              # optional; long tables, catalogs, examples
    <topic>.md
```

`name` in the frontmatter must equal the directory name, kebab-case. Keep `SKILL.md` scannable — if the body grows past a screen or two of dense reference material, push that material into `references/` and link it. This is progressive disclosure: the model loads `SKILL.md` on trigger, and only opens a reference file when the procedure says to.

## Frontmatter

```yaml
---
name: sql-queries
description: |
  <trigger text — see below>
allowed-tools:
  - Bash
  - Read
---
```

- **`name`** — kebab-case, matches the directory.
- **`description`** — the load-bearing field. It is the only thing the runtime sees when deciding whether to load the skill. Write it to fire reliably (next section).
- **`allowed-tools`** — include only if the skill should run with a constrained tool set. Omit to inherit the host's defaults. Match what sibling skills in the repo do.

## Writing a trigger description that fires

The description is matched against the user's actual request. It must say *what the skill does* and *when it applies*, in the words a user would use.

- **Lead with the capability**, plainly: "Generate a SQL query against the analytics warehouse following the house conventions."
- **Name many phrasings of the trigger.** Users do not speak in your section headings. List the symptoms and asks: "use whenever the user asks to pull data, write a query, 'get me the numbers for...', 'how many X last month', or mentions a table by name." Redundancy here is a feature — more surface area means more reliable firing.
- **Include the explicit invocation** if the host supports slash commands: "or invokes `/sql-queries`."
- **State the negative space** when a skill is easily confused with a sibling: "This handles read queries; for writes/migrations use `/schema-changes`."
- **Do not pad with what it does not do** beyond disambiguation. The description is for routing, not documentation.

Calibration: too narrow ("when the user types EXACTLY 'run sql'") never fires; too broad ("for any data-related question") fires on everything and reintroduces bloat. Aim for the band of real phrasings a user employs for this task and no others.

## Body

The body is the extracted instructions, rewritten to stand alone:

- **Imperative voice, second person.** "Validate the table name against the allowlist before querying" — not "the agent should validate...".
- **Self-contained.** The block used to sit next to other prompt text; that context is gone when the skill loads in isolation. Pull in any definition it relied on, or link to a reference file. Strip phrases like "as noted above."
- **Preserve behavior on the first pass.** Move the wording as-is; do not "improve" it while extracting. If the wording should change, propose that as a separate, explicit step the user approves — a refactor that silently rewrites instructions is indistinguishable from a behavior change.
- **Keep the procedure shape.** If the original was a numbered workflow, keep it numbered. Structure that survived in the monolith should survive the move.

## Naming

- Name skills after the **task or trigger**, not the implementation: `sql-queries`, not `warehouse-bigquery-helper`.
- Kebab-case, short, guessable. The name shows up in the trigger map and in any `/`-invocation.
- Avoid collisions with existing skills in the repo — grep the skills directory first.

## Consistency with the host repo

Before authoring, read one or two existing skills in the target repository and match their conventions: voice (imperative vs. descriptive), emoji policy, frontmatter fields, whether they use `references/`, and how they cross-link siblings. New skills should be indistinguishable in style from the ones already there.
