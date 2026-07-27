---
name: ratel-integrate
description: |
  Inspect an existing agent codebase, map each agent's tools and skills, then implement Ratel so every agent discovers and loads only its intended capabilities. Use when asked to add, wire, set up, or integrate Ratel; connect existing tools or SKILL.md playbooks to an agent through Ratel; or migrate a Vercel AI SDK, Mastra, custom TypeScript, or Python agent to Ratel. Prefer Ratel's official TypeScript adapters for AI SDK and Mastra. Edits code and dependencies, preserves capability boundaries, and verifies the result. Excludes telemetry, experiments, pilots, A/B tests, and rollout planning.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Agent
  - WebFetch
---

# /ratel-integrate — load an agent's tools and skills through Ratel

Turn an existing agent repository into a working Ratel integration. First understand the agents and the capabilities they already own; then wire those same tools and skills through Ratel with the smallest framework-native change.

The deliverable is code, dependency, and test changes in the customer's repository. Do not create a `.ratel/ratel-integrate.md` plan unless the user explicitly asks for one.

## Scope

Keep this integration deliberately narrow:

- Discover agents, tools, skills, and their current ownership boundaries.
- Register executable tools and skill bodies with Ratel.
- Give each agent Ratel's stable capability tools and, where the host adapter supports it, per-turn recall.
- Use `@ratel-ai/vercel-ai-sdk` for Vercel AI SDK agents and `@ratel-ai/mastra` for Mastra agents.
- Use the core SDK directly for other supported hosts; do not invent adapter packages.
- Preserve prompts, models, executors, schemas, middleware, memory, and request context unless a small wiring change is required.
- Do not add telemetry, experiments, feature flags, pilot logic, dashboards, A/B tests, or rollout machinery.

Observability is not a prerequisite. A small tool catalog or a skills-only agent is still valid when the user explicitly wants Ratel integrated.

## Principles

1. **Map before wiring.** Package presence is not agent topology. Find the actual agent construction, model call, tool registry, skill source, and history lifecycle.
2. **Preserve capability boundaries.** One `ratel()` instance owns one shared tool and skill catalog. Share it only among agents that intentionally have the same capabilities; otherwise create separate instances. A union catalog can become an authorization bug.
3. **Prefer the host adapter.** The AI SDK and Mastra adapters preserve native schemas, execution context, passthrough tools, and recall message shapes. Handwritten conversions recreate solved edge cases.
4. **Integrate at composition roots.** Reuse existing tool and skill definitions. Avoid rewriting individual implementations merely to satisfy Ratel.
5. **Keep model-facing tools stable.** Take `modelTools()` once per agent after passthrough tools are registered and reuse it. The live catalog can receive ordinary executable tools and skills later.

## Workflow

### Step 1 — Inspect the repository safely

Read repository instructions and check worktree state before changing anything. Detect the package manager and workspaces from lockfiles and manifests.

Use `rg` to find:

- Agent constructors, factories, supervisors, sub-agents, model calls, and tool-loop entry points.
- Vercel AI SDK imports such as `generateText`, `streamText`, and `ToolLoopAgent`.
- Mastra `Agent` construction and `createTool` definitions.
- Custom TypeScript/Python model loops and framework wrappers.
- Static tool objects, dynamic registries, tool factories, dispatchers, and MCP clients.
- Runtime skill registration plus project-scoped `SKILL.md` files under `.agents/skills`, `.claude/skills`, `.codex/skills`, or repository-specific directories.
- Existing Ratel dependencies and partial integration code.

Ignore vendored dependencies, generated output, build artifacts, and user-global skill directories unless the repository explicitly loads them.

### Step 2 — Build the per-agent capability map

For every model-facing agent, record:

| Field | What to identify |
| --- | --- |
| Agent | Stable name plus constructor and invocation files |
| Host | AI SDK, Mastra, custom TypeScript, Python, or another framework |
| Tools | Definition source, registry shape, dispatcher, and exact assigned ids |
| Skills | Definition/filesystem source, loader, and exact assigned ids |
| Scope | Shared, agent-specific, tenant-specific, or request-specific |
| History | Stateless, ephemeral tool loop, or durable multi-turn messages |
| Verification | Closest unit/integration tests and build command |

Follow the data flow, not naming conventions. A dependency on `ai` does not prove the application uses the Vercel AI SDK directly—Mastra may bring it transitively. A Markdown file is not automatically an agent skill.

If capability ownership is security-sensitive and cannot be established from code or configuration, ask one focused question before combining catalogs. Continue with all unambiguous agents while waiting.

### Step 3 — Verify the current SDK surface

Ratel moves quickly. Check documentation before editing:

1. Use Context7 when available, resolving `ratel-ai/ratel`.
2. Check the versions already pinned in the manifest and lockfile, then read their installed package READMEs when present.
3. For a new installation or APIs added on `main`, read the official SDK and adapter READMEs in `ratel-ai/ratel`.

Confirm the stable package versions, async registration API, skill data model, adapter compatibility, and recall idiom. Prefer stable releases; do not select a prerelease or experimental package unless the user asks.

Context7 can lag recent adapter work. When it conflicts with the released package README or current official source, trust the version the repository will actually install. Read [`references/integration-patterns.md`](references/integration-patterns.md) for the adapter selection and code shapes, then reconcile them with the live docs.

### Step 4 — Choose the supported path per agent

| Detected host | Integration |
| --- | --- |
| TypeScript importing compatible Vercel AI SDK APIs directly | `@ratel-ai/sdk` + `@ratel-ai/vercel-ai-sdk`; `ratel(...).adaptTo(aiSdk())` |
| TypeScript using a compatible Mastra release | `@ratel-ai/sdk` + `@ratel-ai/mastra`; `ratel(...).adaptTo(mastra())` |
| Custom or other TypeScript host | `@ratel-ai/sdk`; adapt its native capability tools only at the existing model boundary |
| Python host | `ratel-ai`; wrap the core capability tools into the framework's documented tool shape |
| Unsupported language/framework | Use a documented thin wrapper if the current SDK shapes fit; otherwise report the precise blocker |

Check the current peer and Node.js ranges before selecting an adapter. If it would require a framework or runtime upgrade, do not expand the task silently: use a documented core path only when it preserves semantics, otherwise report the blocker. Do not recommend an adapter that does not exist or replace an in-process integration with Ratel Local or an MCP gateway.

### Step 5 — Implement the capability loading

Use the repository's package manager and install dependencies in the package that owns the agent.

For each capability boundary:

1. Construct or cache one Ratel core per immutable authorization set at the agent's composition root. For tenant- or request-scoped capabilities, use a per-request core or an exact-scope cache; never register request-specific tools into a process-global core. Default to BM25 and a small recall budget unless the repository already has an explicit retrieval configuration.
2. Register the agent's existing framework-native tools through the adapted view. Preserve ids, descriptions, schemas, executors, approval behavior, and live execution context.
3. Ingest existing upstream MCP tools into the same catalog only when that agent already consumes the upstream. Do not broaden its MCP access.
4. Register assigned skills on `r.skills` as one batch. Ratel does not scan directories: reuse an existing loader or add the smallest loader that converts approved project skills to `{ id, name, description, tags?, tools?, metadata?, body? }`.
5. Keep each skill's `tools` field aligned with exact registered tool ids. Do not register arbitrary repository Markdown or user-global skills.
6. Replace the agent's model-facing full executable catalog with `r.modelTools()`, preserving adapter passthroughs.
7. Add the host's supported recall/discovery path:
   - AI SDK: use `prepareStep` for stateless/drop-in calls; use `appendRecall` when the application persists multi-turn response messages.
   - Mastra: append `r.recallProcessor()` to existing input processors.
   - Custom TypeScript: use the core's `r.recall(query)` only when the host has a clear message-injection boundary.
   - Python and other hosts: expose the capability tools in the framework's native shape. Add top-K filtering or message injection only when a current official example documents that path.

If the repository already has Ratel, extend or migrate the existing integration rather than creating a second catalog. A second run of this skill should produce no further changes.

### Step 6 — Verify behavior

For new loader or capability-mapping logic, write the focused failing test first. Then verify:

- Each agent exposes `search_capabilities`, `invoke_tool`, and `get_skill_content`, plus only legitimate passthrough tools.
- A representative query discovers and invokes an assigned tool through Ratel.
- A representative query discovers an assigned skill and loads its full body.
- Skill-declared tool ids resolve in the same catalog.
- Agents with different capability scopes cannot discover each other's tools or skills.
- Adapter recall is injected once per user turn and existing conversation persistence still works, when the selected adapter provides recall.

Run the narrow tests first, then the affected package's typecheck, test suite, lint, and build as available. Do not fix unrelated pre-existing failures; report them separately.

### Step 7 — Report the completed integration

Summarize:

1. Agents found and the Ratel path used for each.
2. Tool and skill sources wired to each agent.
3. Dependencies and files changed.
4. Verification commands and results.
5. Any unsupported or intentionally untouched surface.

Do not include rollout metrics or a future telemetry plan.

## Honest stop paths

- **No agent surface:** if no model-facing agent or tool/skill loading path exists, do not install Ratel. Report where you looked and ask for the non-standard location.
- **No capabilities:** if an agent has neither tools nor skills, leave it unchanged and say why.
- **Unsafe ownership ambiguity:** do not merge catalogs when doing so may widen an agent's access. Ask for the missing boundary decision.
- **Unsupported adapter:** never fabricate a package or silently weaken framework semantics. Use the current documented wrapper path or explain what prevents a safe integration.

## Reference

- [`references/integration-patterns.md`](references/integration-patterns.md) — current stable TypeScript adapter shapes, framework-neutral fallbacks, skill registration, and multi-agent boundaries
