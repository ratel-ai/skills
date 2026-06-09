# Ratel integration patterns

The per-mode and per-framework code shapes for wiring Ratel into an agent. Read this **after** Step 3 of the skill — i.e., after you have the up-to-date docs in hand. If anything here disagrees with the latest docs, trust the docs and flag this file for an update.

Public Ratel surface (as of v0.1.5):

- TS SDK: `@ratel-ai/sdk` — `ToolCatalog`, `searchToolsTool`, `invokeToolTool`, `registerMcpServer`
- CLI: `@ratel-ai/cli` — `ratel serve`, `ratel mcp add | list | edit`, `ratel inspect`
- MCP server: `@ratel-ai/mcp-server` (also published as `ratel-mcp`) — exposes `search_tools` and `invoke_tool` to any MCP client

A Python SDK is on the v0.5.x roadmap; before it ships, Python customers integrate by running `ratel serve` as an MCP gateway and pointing their MCP-aware agent at it.

## Mode 1 — Direct SDK (TypeScript)

Best when:
- The agent process is Node/TS.
- There's a single dispatcher or a static `tools:` parameter.
- You want full control over when retrieval runs vs when the gateway tools are exposed.

Shape:

```ts
import { ToolCatalog, searchToolsTool, invokeToolTool } from "@ratel-ai/sdk";

// 1. Build the catalog once at process start.
const catalog = new ToolCatalog({
  trace: { kind: "jsonl", sessionId: process.env.SESSION_ID ?? "boot", path: "..." },
  // future: { kind: "langfuse", ... } once the native sink ships
});

// 2. Register every tool the agent should know about.
for (const tool of allTools) {
  catalog.register({
    id: tool.id,
    name: tool.name,
    description: tool.description,
    inputSchema: tool.inputSchema,
    outputSchema: tool.outputSchema,
    execute: tool.run,
  });
}

// 3a. Replace-mode (pre-filter): swap the model's tool list for top-K hits.
const hits = catalog.search(currentUserMessage, /* topK = */ 8, "direct");
const filteredTools = hits.flatMap(({ toolId }) => catalogTools.filter(t => t.id === toolId));
const result = await generateText({ model, tools: filteredTools, /* ... */ });

// 3b. Gateway-mode: expose `search_tools` and `invoke_tool` so the agent can reach more on demand.
const result = await generateText({
  model,
  tools: { search_tools: searchToolsTool(catalog), invoke_tool: invokeToolTool(catalog) },
});
```

For most pilots, use **replace-mode with topK=8** as the default. Gateway mode is more powerful but adds an extra model turn per discovery.

### Vercel AI SDK specifics

The two patterns above drop into `generateText` / `streamText` directly. Wrap the call in the customer's existing observability span (per [`ratel-langfuse-instrument/references/stack-vercel-ai-sdk.md`](../../ratel-langfuse-instrument/references/stack-vercel-ai-sdk.md)) so the Ratel-emitted trace events land under the same trace.

### Mastra / generic TS specifics

Same shape — register tools into the catalog instead of (or in addition to) Mastra's tool registry. For the pilot, keep the customer's existing registry and tee tools into the Ratel catalog; the agent's tool surface is whatever you pass to the model call.

## Mode 2 — MCP gateway

Best when:
- The customer's agent already speaks MCP (Claude Desktop, Cursor, Goose, custom MCP client).
- Tools come from one or more MCP upstream servers and the customer wants Ratel in front of them.
- The agent process is Python and the Python SDK hasn't shipped yet.

Setup steps for the plan:

1. **Install** `@ratel-ai/mcp-server` (npm) or `ratel-mcp` (whichever the customer prefers).
2. **Configure upstreams**: `ratel mcp add <name> --transport stdio --command "<cmd>"` or `--transport sse --url ...`. Each upstream's tools are ingested into the catalog with namespace prefix `upstream__toolName`.
3. **Run** `ratel serve` as the MCP server the agent connects to. Point the agent's MCP config at this process.
4. **Auth (if needed)**: for SSE/HTTP upstreams that use OAuth, run `ratel mcp auth <name>` once per upstream — Ratel handles refresh and re-auth after that.
5. **Trace stream**: by default lands in `~/.ratel/telemetry/<project-slug>/<session-id>.jsonl`. Wire the forwarder from [`ratel-langfuse-instrument/references/ratel-hooks.md`](../../ratel-langfuse-instrument/references/ratel-hooks.md) to push to Langfuse, or wait for the native Langfuse sink.

The agent only sees Ratel's two gateway tools (`search_tools`, `invoke_tool`). To use a tool, it calls `search_tools` first and then `invoke_tool` with the returned id. This is the most token-efficient mode at very large catalogs but requires the agent to handle the discovery step.

### Python specifics

Until the Python SDK ships, Python integrations are MCP-gateway mode by definition. For LangChain / LlamaIndex / direct-SDK agents, install an MCP client (e.g., `mcp` from PyPI) and configure it to talk to `ratel serve`. For LangGraph / CrewAI agents, the same MCP client wraps the agent's tool node.

## Mode 3 — Hybrid

Best when:
- The agent has a mix of local tools (defined in the customer's codebase) and upstream MCP tools.
- The customer wants Ratel ranking across both.

Shape:

1. Register the local tools into a `ToolCatalog` via the direct SDK (Mode 1).
2. Use `registerMcpServer(catalog, { name, transport })` to ingest each MCP upstream into the **same** catalog. Tools land with the `upstream__` prefix.
3. Expose `searchToolsTool(catalog)` + `invokeToolTool(catalog)` to the agent. Search ranks across local and upstream uniformly; invocation routes to the right executor automatically.

This mode is exactly Mode 1 plus `registerMcpServer` calls. Don't dual-instantiate.

## Per-framework callouts

### Vercel AI SDK

- Pre-filter at the call site (where `tools:` is passed). Don't try to wrap the SDK.
- If the agent uses `maxSteps > 1` for multi-turn tool loops, pre-filter once at the start of the loop — the SDK reuses the same tool list across steps. Re-running search every step is wasted work and breaks the simple "this turn used this top-K" trace pattern.

### LangChain (Python)

- Pre-filter at the agent constructor (`AgentExecutor(tools=...)`). Until the Python SDK ships, use Mode 2 (MCP gateway) and have LangChain talk MCP.
- If using Mode 2, the agent gets `search_tools` and `invoke_tool` as plain tools; document this in the plan since LangChain users don't expect two-step tool calling.

### LangGraph

- The tool node is where the tools list lives. Mode 2 wraps the tool node with an MCP client; Mode 1 (when Python SDK ships) replaces the node's tools with the catalog's search results.
- Multi-agent graphs: the catalog should be **shared** across nodes (same in-process instance for Mode 1; same `ratel serve` for Mode 2). Per-node catalogs defeat the point.

### CrewAI

- Per-agent tool lists in CrewAI map to per-agent catalogs in Mode 1 (when available). For now, Mode 2 with `ratel serve` works — each agent runs its own MCP client against the same gateway.

### Custom agent loops (no framework)

- These are the easiest case. Pre-filter wherever the tools list is materialised for the model. The dispatcher swap is a 10-line change.

## What the plan must specify

For each agent surface the plan touches, the per-file changes must answer:

1. **Mode**: direct SDK / MCP gateway / hybrid.
2. **Init site**: the file + line where the `ToolCatalog` is constructed (Mode 1) or where `ratel serve` is launched (Mode 2).
3. **Registration site**: where every tool the agent uses is registered.
4. **Swap site**: where the agent's `tools:` parameter is replaced with the top-K or with the gateway tools.
5. **Metadata wiring**: where `gateway_origin`, `top_k`, `hit_count`, `top_hit_score`, `replace_mode` get attached to the Langfuse observation. See [`ratel-langfuse-instrument/references/ratel-hooks.md`](../../ratel-langfuse-instrument/references/ratel-hooks.md).
6. **Flag check**: where the A/B feature flag is read to decide which arm of the split the request belongs to (see [`ab-test-patterns.md`](ab-test-patterns.md)).
