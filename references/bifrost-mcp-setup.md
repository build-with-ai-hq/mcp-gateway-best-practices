# Bifrost as an MCP gateway: setup

Bifrost acts as an MCP client to your servers (stdio, HTTP, SSE, or an OpenAPI document turned into tools) and as an MCP server to your hosts (Claude Code, Claude Desktop, Cursor, custom apps) at one `/mcp` endpoint. This file covers connecting servers, connecting hosts, and keeping connections healthy.

Source docs: https://docs.getbifrost.ai/mcp/overview.md , https://docs.getbifrost.ai/mcp/connecting-to-servers.md , https://docs.getbifrost.ai/mcp/connections.md

## 1. Run the gateway

```bash
npx -y @maximhq/bifrost                       # or:
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost
# UI + API on http://localhost:8080
```

Production: pin a version (`maximhq/bifrost:<version>` or `npx -y @maximhq/bifrost --transport-version <version>`), turn on dashboard auth and `enforce_auth_on_inference`, serve over TLS behind a reverse proxy, and restrict network access to `/mcp`.

## 2. Connect MCP servers

Each upstream server is an "MCP client" in Bifrost terms. Names must be unique, ASCII, with no spaces or hyphens (use `web_search`, not `web-search`).

### Connection types

| `connection_type` | Use for | Key fields |
| - | - | - |
| `stdio` | Local process servers (Node, Python, CLIs) | `stdio_config.command`, `args`, `envs` (env var names passed through) |
| `http` | Remote streamable HTTP servers | `connection_string` (URL or `env.VAR`) |
| `sse` | Servers that only speak the legacy SSE transport | `connection_string` |
| `openapi` (beta) | REST APIs with an OpenAPI/Swagger doc and no MCP server | `openapi_config.spec_file`, `base_url` |

### config.json

MCP servers live under the top-level `mcp` key in `client_configs`. Global tool-manager settings live in `mcp.tool_manager_config`.

```json
{
  "mcp": {
    "client_configs": [
      {
        "name": "filesystem",
        "connection_type": "stdio",
        "stdio_config": {
          "command": "npx",
          "args": ["-y", "@modelcontextprotocol/server-filesystem", "/srv/shared-docs"],
          "envs": ["HOME", "PATH"]
        },
        "auth_type": "none",
        "is_ping_available": true,
        "tools_to_execute": ["read_text_file", "list_directory", "search_files"]
      },
      {
        "name": "web_search",
        "connection_type": "http",
        "connection_string": "env.WEB_SEARCH_MCP_URL",
        "auth_type": "headers",
        "headers": { "Authorization": "env.WEB_SEARCH_MCP_AUTH" },
        "tools_to_execute": ["search", "fetch_url"],
        "tools_to_auto_execute": ["search"]
      },
      {
        "name": "petstore",
        "connection_type": "openapi",
        "auth_type": "none",
        "tools_to_execute": ["getInventory", "getPetById"],
        "openapi_config": { "spec_file": "specs/petstore.yaml", "base_url": "https://petstore3.swagger.io/api/v3" }
      }
    ],
    "tool_manager_config": {
      "tool_execution_timeout": "60s",
      "max_agent_depth": 10,
      "disable_auto_tool_inject": false
    }
  }
}
```

Notes:

- `env.VAR` reads an environment variable; values are redacted in the UI and API. Use it for URLs that embed keys and for every header secret.
- `tools_to_execute`: `["*"]` all, `[]` or omitted none (deny by default), or an explicit list. List tools explicitly for anything that can write.
- Check the exact tool names a server exposes (UI client sheet or `GET /api/mcp/clients`) before writing allow-lists; names differ between server versions. Pin stdio package versions in `args`.
- In Docker, stdio servers need `npx`/`python` in the image. Build a custom image or run the server separately and connect over `http`.
- Upstream auth types: `none`, `headers`, `oauth`, `per_user_oauth`, `per_user_headers`, and `token_exchange`. Clients declared in config.json with OAuth or per-user types start in `pending_verification` until an admin completes the one-time step in the UI. See https://docs.getbifrost.ai/mcp/auth/overview.md

### API (runtime changes)

```bash
curl -X POST http://localhost:8080/api/mcp/client \
  -H "Content-Type: application/json" \
  -d '{
    "name": "web_search",
    "connection_type": "http",
    "connection_string": "https://mcp.example.com/mcp",
    "auth_type": "none",
    "tools_to_execute": ["search", "fetch_url"]
  }'

curl http://localhost:8080/api/mcp/clients                          # list with state and tools
curl -X PUT http://localhost:8080/api/mcp/client/{id} -H "Content-Type: application/json" -d '{...}'
curl -X POST http://localhost:8080/api/mcp/client/{id}/reconnect     # recycle a sticky connection
curl -X POST http://localhost:8080/api/mcp/client/{id}/refresh-tools # re-discover tools now
curl -X DELETE http://localhost:8080/api/mcp/client/{id}
```

Safety rules enforced by Bifrost when dashboard auth is off: registering a `stdio` client via the API is refused, and HTTP/SSE clients pointing at loopback or private addresses are refused. Enable dashboard auth or declare those clients in `config.json`.

A client deleted in the UI but still present in `config.json` comes back on restart. Remove it from the file to delete it permanently.

## 3. Connect hosts to `/mcp`

`POST /mcp` handles JSON-RPC (`initialize`, `tools/list`, `tools/call`); `GET /mcp` is the SSE stream. Each caller authenticates with a virtual key via `Authorization: Bearer <vk>`, `x-bf-vk`, `X-Api-Key`, or `x-goog-api-key`, and sees only the tools that key allows.

**Claude Code**

```bash
claude mcp add --transport http bifrost http://localhost:8080/mcp \
  --header "Authorization: Bearer sk-bf-your-virtual-key" --scope user
# or
claude mcp add-json bifrost '{"type":"http","url":"http://localhost:8080/mcp","headers":{"Authorization":"Bearer sk-bf-your-virtual-key"}}'
```

Run `/mcp` inside Claude Code to confirm `bifrost` is connected with a tool count. `--scope project` writes a checked-in `.mcp.json`; never put a real key there. Use a per-developer key via `--scope user` or local scope instead.

In the default `headers` auth mode, Claude Code may log `SDK auth failed` because it probes OAuth registration endpoints that are not served in that mode; the connection still works. Switching `mcp_server_auth_mode` to `both` or `oauth` lets hosts connect through a browser OAuth flow instead of pasting keys (see [tool-access-control.md](tool-access-control.md)). After changing auth settings, remove and re-add the server in Claude Code; it caches auth per server.

**Claude Desktop / Cursor / other hosts with HTTP + headers support**

```json
{
  "mcpServers": {
    "bifrost": {
      "url": "http://localhost:8080/mcp",
      "headers": { "Authorization": "Bearer sk-bf-your-virtual-key" }
    }
  }
}
```

Exact config keys vary by host and version; check the host's MCP docs. Approval and auto-run settings for tool calls are configured in the host, not in Bifrost, when the host runs the agent loop.

**App code via the LLM gateway (no MCP client in your app)**

When your app sends chat requests through Bifrost (`/v1/chat/completions`, `/openai`, `/anthropic`), Bifrost injects allowed MCP tools into the request, returns tool calls, and your app executes approved ones with `POST /v1/mcp/tool/execute` (body = the tool call object). See the approval-gate examples in the main SKILL.md.

**Avoid double injection.** If the same host both routes model traffic through Bifrost and connects to `/mcp`, the model can see each tool twice. Set `mcp.tool_manager_config.disable_auto_tool_inject: true`; callers on the LLM path then opt in per request with `x-bf-mcp-include-tools`.

## 4. Health, states and recovery

- Health check: a `tools/list` over the client's connection, preceded by a `ping` only when `is_ping_available` is true. Set `is_ping_available: false` for servers on the 2026-07-28 MCP revision, which removed `ping`. Once healthy, checks run on the tool sync interval (default 10 minutes, `tool_sync_interval` per client or `mcp.tool_sync_interval` globally, Go duration strings like `"10m"`); while `unstable` they run every 10 seconds.
- States: `healthy`, `unstable` (last check failed; tool calls still attempted), `needs_reauth` (credential dead, admin must reauthorize), `pending_verification`, `disabled`, `error`, `degraded` (cluster disagreement).
- `GET /api/mcp/clients` includes `last_failure` (`stage`, `message`, `at`, `since`) for unhealthy servers. Start debugging there.
- HTTP and SSE reconnect make-before-break; stdio reconnects close-first (a brief gap) to avoid two subprocesses fighting over locks or ports.
- `needs_session_stickiness` (HTTP with server-level auth only): `true` keeps one persistent connection (lower latency); default per-call dials fresh for each tool call (more robust to dead credentials).
- `tool_execution_timeout` (default 30s) bounds each tool call; raise it per client for slow tools rather than globally.

## 5. Observability

- Built-in logs: MCP tool calls are logged alongside LLM requests (dashboard Logs). Configure captured request headers as log metadata to tag entries with tenant or trace IDs. Use `x-bf-disable-content-logging` on requests whose payloads must not be stored.
- Prometheus (`/metrics`): `bifrost_mcp_client_operation_duration_seconds` histogram with labels `mcp_client`, `mcp_tool_name`, `mcp_method`, `error_type`, and virtual key / team / customer labels. `_count` is call volume.

```promql
sum by (mcp_client, mcp_tool_name) (rate(bifrost_mcp_client_operation_duration_seconds_count{error_type!=""}[5m]))
histogram_quantile(0.95, sum by (le, mcp_tool_name) (rate(bifrost_mcp_client_operation_duration_seconds_bucket[5m])))
```

- OpenTelemetry plugin: MCP client spans with `mcp.method.name`, `gen_ai.tool.name`, `error.type` and governance identity. Docs: https://docs.getbifrost.ai/features/observability/otel.md

## 6. Troubleshooting

| Symptom | Check |
| - | - |
| Host connects but shows zero tools | Server state in `GET /api/mcp/clients`; `tools_to_execute` not empty; the virtual key has an `mcp_configs` entry for that server; server not in Code Mode (Code Mode servers appear as a few meta-tools) |
| `401`/`403` on `/mcp` | Key exists, is active and not expired; `Bearer ` prefix present; `enforce_auth_on_inference` matches your intent |
| stdio server never becomes healthy | Command exists in the runtime (container image); `envs` passes `PATH`; run the same command by hand |
| Tools changed upstream but not in Bifrost | `POST /api/mcp/client/{id}/refresh-tools` |
| Tool calls time out | Raise `tool_execution_timeout` on that client; make the tool paginate |
| Model sees each tool twice | Turn on `disable_auto_tool_inject` |

## 7. Verify the setup

```bash
# 1. Servers are connected and healthy
curl -s http://localhost:8080/api/mcp/clients | jq '.[] | {name: .config.name, state, tools: (.tools|length)}'

# 2. A scoped key sees only its granted tools
curl -s -X POST http://localhost:8080/mcp -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer sk-bf-support-key' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | jq '[.result.tools[].name]'

# 3. A tool outside the grant is refused (expect an error, not a result)
curl -s -X POST http://localhost:8080/mcp -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer sk-bf-support-key' \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"<a tool not granted>","arguments":{}}}'

# 4. Tool-call metrics are flowing
curl -s http://localhost:8080/metrics | grep bifrost_mcp_client_operation_duration_seconds_count
```
