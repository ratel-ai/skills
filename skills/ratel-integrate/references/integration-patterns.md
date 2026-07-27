# Ratel agent integration patterns

Use these shapes after mapping each agent's capability boundary and checking current official docs. They were verified against Ratel `main` at `d36e93c` and these stable releases on 2026-07-27:

- `@ratel-ai/sdk@0.5.3`
- `@ratel-ai/vercel-ai-sdk@0.2.0`
- `@ratel-ai/mastra@0.1.0`

Do not copy the versions blindly. Respect an existing compatible pin or resolve the current stable release. Avoid prereleases unless the user explicitly asks for experimental behavior.

## Capability boundary

One `ratel()` call owns one tool catalog, one skill catalog, and one recall-id sequence. Its adapted views share that state.

- Use one core for agents that intentionally share the same tools and skills.
- Use separate cores for agents with different permissions, tenants, or assigned capability sets.
- Register shared definitions into each authorized core if necessary. Do not centralize merely to deduplicate bootstrap code.
- Treat a core's capability set as immutable after it enters a shared cache. Never incrementally register tenant- or request-specific capabilities into a process-global core; create a per-request core or cache by the exact authorization set.

```ts
const supportRatel = ratel({ method: "bm25", recallTopK: 5 }).adaptTo(aiSdk());
const adminRatel = ratel({ method: "bm25", recallTopK: 5 }).adaptTo(aiSdk());

await supportRatel.tools.register(supportTools);
await adminRatel.tools.register(adminTools);
```

## Vercel AI SDK

Use when application code directly uses `ai` v5, v6, or v7 APIs on Node.js 20.6 or newer. Verify the current peer range before editing; do not choose it merely because `ai` is present transitively.

```ts
import { aiSdk } from "@ratel-ai/vercel-ai-sdk";
import { ratel } from "@ratel-ai/sdk";
import { streamText } from "ai";

const r = ratel({ method: "bm25", recallTopK: 5 }).adaptTo(aiSdk());

await r.tools.register(agentTools);
await r.skills.register(agentSkills);

// Take once after registering passthrough tools; reuse across turns.
const modelTools = r.modelTools();

const result = streamText({
  model,
  tools: modelTools,
  messages,
  prepareStep: r.prepareStep,
});
```

`agentTools` is the existing `Record<string, Tool>`. The adapter catalogs ordinary executable tools and preserves provider-defined, dynamic, client-executed, and AI-SDK-specific lifecycle tools as eager passthroughs. Register passthroughs before taking the `modelTools()` snapshot.

Choose one recall idiom:

- `prepareStep: r.prepareStep` is the smallest change for stateless calls or when recall pairs should not enter durable history.
- `messages: await r.appendRecall(messages)` appends recall to persistent multi-turn history. Continue persisting the model's response messages: `await result.responseMessages` on AI SDK 7 or `(await result.response).messages` on AI SDK 5/6.

Do not use both on the same call.

## Mastra

Use for agents constructed from `@mastra/core >=1.11 <2` on Node.js 22.13 or newer. Verify the current peer range before editing.

```ts
import { Agent } from "@mastra/core/agent";
import { mastra } from "@ratel-ai/mastra";
import { ratel } from "@ratel-ai/sdk";

const r = ratel({ method: "bm25", recallTopK: 5 }).adaptTo(mastra());

await r.tools.register(agentTools);
await r.skills.register(agentSkills);

const agent = new Agent({
  id: "assistant",
  name: "assistant",
  instructions,
  model,
  tools: r.modelTools(),
  inputProcessors: [...existingInputProcessors, r.recallProcessor()],
});
```

`agentTools` is the existing record of Mastra `createTool()` values. The adapter preserves normalized schemas and forwards live `ToolExecutionContext`. `recallProcessor()` injects once per generation; call it separately for each Mastra agent.

## Skills

Skills are framework-neutral and register on the same adapted view:

```ts
await r.skills.register([
  {
    id: "incident-response",
    name: "Incident response",
    description: "Triage and mitigate a production incident.",
    tags: ["incident", "on-call"],
    tools: ["query_logs", "rollback_deploy"],
    metadata: { owner: ["platform"] },
    body: incidentResponseMarkdown,
  },
]);
```

Ratel does not scan `SKILL.md` directories. Reuse the repository's loader or add a small bootstrap loader that:

1. Reads only approved project-scoped skill files.
2. Parses frontmatter into `id`, `name`, `description`, optional `tags`, `tools`, and `metadata`.
3. Keeps the Markdown body intact.
4. Validates that declared tool ids exist in the same authorized catalog.
5. Registers the batch with `await r.skills.register(skills)`.

Do not load every Markdown file or user-global skill directory.

## Framework-neutral TypeScript

For a custom TypeScript loop, use the standalone core and convert only its three model-facing capability tools at the existing host boundary:

```ts
import { ratel } from "@ratel-ai/sdk";

const r = ratel({ method: "bm25", recallTopK: 5 });
await r.tools.register(...nativeExecutableTools);
await r.skills.register(agentSkills);

const capabilityTools = r.modelTools();
const recall = await r.recall(currentUserText);
```

`capabilityTools` contains `search_capabilities`, `invoke_tool`, and `get_skill_content`. Wrap those native `ExecutableTool` objects into the host's documented tool shape. Format `recall` into the host's message shape at the same boundary. Do not invent `@ratel-ai/<framework>` imports.

If the agent already consumes an upstream MCP server in process, ingest it into this same catalog rather than creating a separate gateway:

```ts
import { registerMcpServer } from "@ratel-ai/sdk";

await registerMcpServer(r.tools.catalog, { name: "existing-upstream", transport });
```

Keep the existing agent-level authorization decision around that connection.

## Python

Python has framework-neutral primitives and no framework adapter package:

```python
from ratel_ai import (
    Skill,
    SkillCatalog,
    ToolCatalog,
    get_skill_content_tool,
    invoke_tool_tool,
    search_capabilities_tool,
)

tools = ToolCatalog()
skills = SkillCatalog()

await tools.register(executable_tools)
await skills.register([
    Skill(
        id="incident-response",
        name="Incident response",
        description="Triage and mitigate a production incident.",
        body=incident_response_markdown,
    )
])

capability_tools = [
    search_capabilities_tool(tools, skills),
    invoke_tool_tool(tools),
    get_skill_content_tool(skills),
]
```

Convert only those capability tools into the framework's native tool type. Python has no generic adapter recall helper; use a current official host example for any top-K filtering or message injection. Do not claim a LangChain, LangGraph, CrewAI, Pydantic AI, or OpenAI Agents adapter exists.

## Verification targets

At minimum, exercise:

1. `modelTools()` exposes the three capability ids and only expected passthroughs.
2. Tool search and invocation preserve the original executor behavior.
3. Skill search and `get_skill_content` return the assigned body.
4. A skill's declared tool dependencies resolve.
5. Separate agent cores cannot retrieve each other's capabilities.
6. Each selected TypeScript adapter's recall hook injects once per user turn.

Run the integration twice during development; the second pass should not add duplicate registration or another Ratel instance.
