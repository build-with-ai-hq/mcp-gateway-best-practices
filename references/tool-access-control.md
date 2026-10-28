# Tool access control, approvals and auth

Read this when scoping which agent, user, team or environment can call which MCP tool, when deciding what may auto-execute, or when setting up authentication for hosts and upstream servers.

## Model the access you need first

Write a small matrix before configuring anything:

| Identity | Servers | Tools | Auto-execute | Upstream credential |
| - | - | - | - | - |
| support-agent (prod) | crm, docs | `search_contacts`, `get_contact`, docs `*` | reads only | shared service token, read scope |
| support-lead (human, Claude Desktop) | crm | reads + `update_contact` | none (host prompts) | per-user OAuth |
| ci-review-bot | repo | `get_pull_request`, `list_files` | all (read-only) | fine-grained token, single repo |

Every row maps to one gateway credential (a virtual key), so it can be revoked without touching the others.

## Vendor-neutral rules

- Deny by default. New identities and new servers grant nothing until listed.
- Grant tools, not servers, whenever a server mixes reads and writes.
- Separate keys per environment; production keys get the smallest list.
- Destructive and side-effecting tools are never auto-executed by an unattended loop.
- Approval must show the exact tool name and arguments, and a denial must be fed back to the model ("Denied by user. Do not retry.") so it does not loop.
- The model's identity is not the user's identity. For systems with per-user permissions (email, docs, tickets), use per-user upstream credentials so the agent cannot exceed what the human can do.
- Never forward caller-supplied headers to an upstream server as if they were trusted identity.

## Bifrost: three filter levels

A tool is available only if it passes every level.

1. **Server baseline** - `tools_to_execute` on the MCP client (`mcp.client_configs[]`). `["*"]` all, `[]` none, or a list.
2. **Virtual key ceiling** - `mcp_configs` on the key. No `mcp_configs` means no MCP tools, except from servers marked `allow_by_default`. A server not listed on the key is blocked (unless `allow_by_default`). An empty tool list blocks that server for that key.
3. **Request narrowing** - headers on a request can only narrow within the key's ceiling:
   - `x-bf-mcp-include-clients: crm,docs`
   - `x-bf-mcp-include-tools: crm-search_contacts,docs-*` (format `clientName-toolName`, `clientName-*` for all tools of a server)
   - An empty header value blocks all tools for that request.

Calls to `/mcp` (`tools/list` and `tools/call`) and to `/v1/mcp/tool/execute` are checked against the same allow-list; inactive or expired keys get `403`.

### Virtual key with tool grants

API:

```bash
curl -X POST http://localhost:8080/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{
    "name": "support-agent-prod",
    "mcp_configs": [
      { "mcp_client_name": "crm",  "tools_to_execute": ["search_contacts", "get_contact"] },
      { "mcp_client_name": "docs", "tools_to_execute": ["*"] }
    ]
  }'
```

Update with `PUT /api/governance/virtual-keys/{vk_id}` and the full `mcp_configs` list. If the same key is also used for model calls, add its `provider_configs` (see https://docs.getbifrost.ai/features/governance/virtual-keys.md).

config.json (keys live under `governance.virtual_keys`; `value` can reference an env var):

```json
{
  "governance": {
    "virtual_keys": [
      {
        "id": "vk-support-prod",
        "name": "support-agent-prod",
        "value": "env.SUPPORT_AGENT_VK",
        "is_active": true,
        "mcp_configs": [
          { "mcp_client_name": "crm",  "tools_to_execute": ["search_contacts", "get_contact"] },
          { "mcp_client_name": "docs", "tools_to_execute": ["*"] }
        ]
      },
      {
        "id": "vk-support-dev",
        "name": "support-agent-dev",
        "value": "env.SUPPORT_AGENT_DEV_VK",
        "is_active": true,
        "mcp_configs": [
          { "mcp_client_name": "crm",  "tools_to_execute": ["*"] },
          { "mcp_client_name": "docs", "tools_to_execute": ["*"] }
        ]
      }
    ]
  }
}
```

A disabled MCP server stays unavailable even if a key grants it. Keys can carry an expiry; expired keys fail closed for both inference and tool calls.

## Approval gate code (Python and TypeScript)

Python approval gate around an explicit-execution loop (works with any OpenAI-compatible endpoint that returns `tool_calls`; shown against a Bifrost gateway):

```python
import json, httpx

BASE = "http://localhost:8080"
HEADERS = {"Authorization": "Bearer sk-bf-your-virtual-key"}
AUTO_OK = {"crm_search_contacts", "crm_get_contact"}  # exact names as returned; read-only only

def run_turn(messages, model="openai/gpt-4o-mini"):  # use the model the project already uses
    r = httpx.post(f"{BASE}/v1/chat/completions", headers=HEADERS,
                   json={"model": model, "messages": messages}, timeout=120).json()
    msg = r["choices"][0]["message"]
    messages.append(msg)
    for call in msg.get("tool_calls") or []:
        name, args = call["function"]["name"], call["function"]["arguments"]
        if name not in AUTO_OK and not ask_human(f"Run {name} with {args}?"):
            messages.append({"role": "tool", "tool_call_id": call["id"],
                             "content": "Denied by user. Do not retry this call."})
            continue
        result = httpx.post(f"{BASE}/v1/mcp/tool/execute", headers=HEADERS, json=call, timeout=60).json()
        messages.append(result)  # {"role": "tool", "content": ..., "tool_call_id": ...}
    return messages
```

TypeScript equivalent:

```ts
const BASE = "http://localhost:8080";
const headers = { Authorization: "Bearer sk-bf-your-virtual-key", "Content-Type": "application/json" };
const AUTO_OK = new Set(["crm_search_contacts", "crm_get_contact"]); // exact names as returned

export async function runTurn(messages: any[], model = "openai/gpt-4o-mini") {
  const res = await fetch(`${BASE}/v1/chat/completions`, { method: "POST", headers, body: JSON.stringify({ model, messages }) });
  const msg = (await res.json()).choices[0].message;
  messages.push(msg);
  for (const call of msg.tool_calls ?? []) {
    const { name, arguments: args } = call.function;
    if (!AUTO_OK.has(name) && !(await askHuman(`Run ${name} with ${args}?`))) {
      messages.push({ role: "tool", tool_call_id: call.id, content: "Denied by user. Do not retry this call." });
      continue;
    }
    const exec = await fetch(`${BASE}/v1/mcp/tool/execute`, { method: "POST", headers, body: JSON.stringify(call) });
    messages.push(await exec.json());
  }
  return messages;
}
```

Use the exact tool names the endpoint returns (log one response first); gateways prefix tool names with the server name. Keep `AUTO_OK` in config, reviewed like code.

## Bifrost: approvals and agent mode

- **Default (explicit execution)**: on the LLM path (`/v1/chat/completions` etc.), Bifrost returns tool calls as suggestions. Your app reviews them and calls `POST /v1/mcp/tool/execute`. Put the approval gate there.
- **Agent mode**: Bifrost runs the loop itself and auto-executes only tools in `tools_to_auto_execute` (which must also be in `tools_to_execute`). Tools outside that list come back to your app in `tool_calls` for approval, with `finish_reason: "stop"`. Limits: `max_agent_depth` (default 10) and `tool_execution_timeout` (default 30s). Agent mode does not work with streaming requests.

```json
{
  "name": "filesystem",
  "connection_type": "stdio",
  "stdio_config": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/srv/data"], "envs": ["HOME", "PATH"] },
  "tools_to_execute": ["*"],
  "tools_to_auto_execute": ["read_text_file", "list_directory"]
}
```

Safe to auto-execute: reads, lists, searches, fetches of trusted sources. Require approval: writes, edits, deletes, command execution, sending messages, payments, deployments.

- **Gateway mode (`/mcp`)**: when a host such as Claude Code, Claude Desktop or Cursor runs the loop, `tools_to_auto_execute` is ignored. Configure approvals in the host (permission prompts, allow lists) and keep destructive tools off the key entirely when no human is present.

## Bifrost: authenticating hosts to `/mcp`

- Header credentials (default `mcp_server_auth_mode: "headers"`): virtual key via `Authorization: Bearer`, `x-bf-vk`, `X-Api-Key` or `x-goog-api-key`. Simple; good for services and CI.
- OAuth 2.1 (`"both"` or `"oauth"`): Bifrost acts as an authorization server with discovery, dynamic client registration and PKCE. Interactive hosts open a browser consent page and bind the grant to a virtual key instead of storing the raw key in client config. Requires `oauth2_server_config.issuer_url`.

```json
{
  "client": {
    "enforce_auth_on_inference": true,
    "mcp_server_auth_mode": "both",
    "oauth2_server_config": { "issuer_url": "https://bifrost.example.com", "access_token_ttl": 600 }
  }
}
```

A request must carry exactly one credential type. With `enforce_auth_on_inference: true`, anonymous and session-only callers are rejected. Details: https://docs.getbifrost.ai/mcp/gateway-auth.md

## Bifrost: authenticating to upstream servers

| `auth_type` | Use when |
| - | - |
| `none` | Local or public servers with no credential |
| `headers` | One shared service token (`"headers": {"Authorization": "env.X"}`) |
| `oauth` | One shared OAuth grant, refreshed by the gateway |
| `per_user_oauth` | Each user signs in to the upstream service as themselves |
| `per_user_headers` | Each user submits their own API key or tenant header (`per_user_header_keys`) |
| `token_exchange` | Exchange the caller's identity-provider token for a short-lived upstream token |

Per-user types return an `mcp_auth_required` tool result with a URL on first use; after the user completes it, Bifrost stores the credential against their identity. Details: https://docs.getbifrost.ai/mcp/auth/overview.md

`allowed_extra_headers` on a client forwards listed caller headers (tenant ID, trace ID) to that server. These values come from the caller, so the upstream server must treat them as untrusted, never as proof of identity.

## Verify access control

```bash
# Allowed key: lists only granted tools
curl -s -X POST http://localhost:8080/mcp -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer sk-bf-support-prod' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | jq '[.result.tools[].name]'

# Header cannot widen the grant: expect no update tool
curl -s -X POST http://localhost:8080/v1/chat/completions -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer sk-bf-support-prod' \
  -H 'x-bf-mcp-include-tools: crm-update_contact' \
  -d '{"model":"openai/gpt-4o-mini","messages":[{"role":"user","content":"Update contact 7 email"}]}'

# No key with enforcement on: expect 401/403
curl -s -o /dev/null -w '%{http_code}\n' -X POST http://localhost:8080/mcp \
  -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Add these as automated tests so a config change that widens access fails CI.
