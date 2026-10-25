# Cutting tool-definition tokens: tool budgets and Code Mode

Read this when an agent has 3+ MCP servers, 50+ tools, or when tool definitions are a large share of input tokens.

## The problem

Classic MCP tool calling sends every available tool definition on every model turn. With 10 servers and 150 tools that can be tens of thousands of tokens per turn, repeated across every turn of an agent loop, and every intermediate tool result also flows back through the model. Costs grow with the catalog size, latency grows with it, and tool selection gets worse as similar tools compete.

## Fix in order

1. **Cut the catalog** (always do this first, no tooling needed).
   - Remove servers the agent does not use. Check logs for tools never called in 30 days.
   - Allow-list tools per agent (`tools_to_execute`, virtual key `mcp_configs`).
   - Scope per request when the task is known (`x-bf-mcp-include-clients: crm,docs`).
2. **Shrink each definition.** Short descriptions, narrow schemas, no giant enums. A 600-token description is usually 100 tokens of signal.
3. **Stop sending the catalog at all** when many servers remain: use on-demand discovery. Bifrost implements this as Code Mode.

## How Bifrost Code Mode works

For servers marked as Code Mode clients, the model no longer sees their individual tools. It sees four meta-tools:

| Meta-tool | Purpose |
| - | - |
| `listToolFiles` | List virtual `.pyi` stub files, one per server (or per tool) |
| `readToolFile` | Read compact Python signatures for a server's tools (supports line ranges) |
| `getToolDocs` | Full docs for one tool when the signature is not enough |
| `executeToolCode` | Run a Python (Starlark) script that calls the tools, e.g. `crm.search_contacts(query="acme")` |

The script runs in a sandbox inside the gateway. Intermediate tool results stay in the sandbox; the model receives only what the script prints and the `result` variable. That is where the savings come from: fewer turns and no repeated catalog.

Bifrost's published benchmark (same query set, Code Mode off vs on): 96 tools / 6 servers: -58.2% input tokens; 251 tools / 11 servers: -84.5%; 508 tools / 16 servers: -92.8%, with pass rates unchanged or better. Results depend on the workload; measure your own (see Verify below). Source: https://docs.getbifrost.ai/mcp/code-mode.md

## When to use it

Use Code Mode for:

- 3+ servers, or any "heavy" server (web search, documents, databases, large SaaS APIs).
- Multi-step workflows where tool outputs feed other tool calls (look up, filter, then act).
- Large tool results the model does not need to read in full.

Keep classic tools for:

- 1 to 2 small servers with simple, direct calls.
- Tools the model must call reliably on almost every turn (a single lookup tool).

You can mix: Code Mode for heavy servers, direct tools for a couple of small utilities.

## Enable it

Per server, in `config.json`:

```json
{
  "mcp": {
    "client_configs": [
      {
        "name": "docs",
        "connection_type": "http",
        "connection_string": "env.DOCS_MCP_URL",
        "auth_type": "none",
        "tools_to_execute": ["*"],
        "is_code_mode_client": true
      },
      {
        "name": "crm",
        "connection_type": "http",
        "connection_string": "env.CRM_MCP_URL",
        "auth_type": "headers",
        "headers": { "Authorization": "env.CRM_MCP_AUTH" },
        "tools_to_execute": ["search_contacts", "get_contact", "update_contact"],
        "tools_to_auto_execute": ["search_contacts", "get_contact"],
        "is_code_mode_client": true
      }
    ],
    "tool_manager_config": {
      "code_mode_binding_level": "server",
      "tool_execution_timeout": "30s"
    }
  }
}
```

Or at runtime: `PUT /api/mcp/client/{id}` with `"is_code_mode_client": true`, or toggle **Code Mode Server** on the client in the MCP Gateway UI.

Binding level (global):

- `"server"` (default): one stub file per server. Good for 5 to 20 tools per server.
- `"tool"`: one stub file per tool. Good for servers with 30+ tools or large schemas, so the model reads only what it needs.

## Permissions and approvals still apply

- With a virtual key, discovery (`listToolFiles`, `readToolFile`, `getToolDocs`) only shows tools that key is granted. Nested calls run with the original request's authorization and tool filters.
- In agent mode, `listToolFiles` and `readToolFile` are always auto-executable (read-only). `executeToolCode` runs automatically only if every tool call in the script is in `tools_to_auto_execute`. A pre-flight scan checks literal `server.tool(...)` calls; each actual nested call is checked again at call time, which is the authoritative check. A script that calls a non-auto tool is returned to your app for approval.
- When your app approves and executes a returned script via `/v1/mcp/tool/execute`, nested calls are still limited to `tools_to_execute`.

## Sandbox limits and behavior

- Starlark (a Python subset): no imports, classes, file I/O or network; `json` and `math` are predeclared.
- Tool calls are synchronous; use keyword arguments; index dicts with `r["key"]`.
- A failed nested call returns an error value (`hasattr(r, "error")`); the script decides whether to recover or `fail(...)`. Dropped connections and timeouts stop the script.
- Default limits per execution (configurable globally): 64 KiB source, 1,000,000 steps, 64 MiB memory budget, 64 KiB print output, 64 nested tool calls, 1 MiB per value. Split large jobs into several executions.

## Gotchas

- Stock MCP hosts connected to `/mcp` (Claude Code, Cursor) see Code Mode servers as a handful of meta-tools, not the raw tools. That is expected; it is also why "zero tools from server X" can be correct.
- Code Mode servers' `initialize` instructions are not added to every request; they appear in the stub file the model reads. Good for tokens, but verify the model actually reads stubs before calling.
- Models vary in how well they write Starlark. Run your selection evals with Code Mode on before switching production traffic.

## Verify

1. Record baseline: average input tokens per request and turns per task for 20 to 50 representative tasks (provider usage fields, or `bifrost_input_tokens_total` in Prometheus).
2. Enable Code Mode for the heavy servers only. Rerun the same tasks.
3. Compare input tokens, turns, task pass rate, and p95 latency. Keep Code Mode where pass rate holds and tokens drop; revert per server where it does not.
