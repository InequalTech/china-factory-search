# MCP client setup

One endpoint, one header — every client below is the same two facts in different config syntax:

- Endpoint (Streamable HTTP): `https://open.tianxiagongchang.com/open/mcp`
- Header: `Authorization: Bearer <API key>` — get a key at <https://www.tianxiagongchang.com/open/console>

After connecting you should see five tools: `factory_search`, `factory_detail`, `factory_contact`, `factory_deepdive`, `factory_agent_search`. Clients with WeChat Agent Pay support (WorkBuddy, QClaw) may show a sixth, `credits_topup` — see SKILL.md "In-chat top-up".

## WorkBuddy

`~/.workbuddy/mcp.json` (user-level; or `<project>/.workbuddy/mcp.json` for one project). UI path: sidebar **Plugins → MCP Servers → Configure MCP**.

```json
{
  "mcpServers": {
    "tianxiagongchang": {
      "type": "http",
      "url": "https://open.tianxiagongchang.com/open/mcp",
      "headers": {
        "Authorization": "Bearer <API key>"
      }
    }
  }
}
```

`type` must be `http` (Streamable HTTP) — there is no separate `/sse` endpoint. WorkBuddy ships the `weixinpay` plugin, so in-chat credit top-up works here when the platform has it enabled.

## Claude Code

```bash
claude mcp add --transport http tianxiagongchang \
  "https://open.tianxiagongchang.com/open/mcp" \
  --header "Authorization: Bearer <API key>"
```

Verify: `claude mcp list` → `tianxiagongchang: ... - ✓ Connected`.

## Cursor / generic `mcp.json`

`~/.cursor/mcp.json` (or project-level `.cursor/mcp.json`; same shape works for most JSON-configured clients):

```json
{
  "mcpServers": {
    "tianxiagongchang": {
      "url": "https://open.tianxiagongchang.com/open/mcp",
      "headers": {
        "Authorization": "Bearer <API key>"
      }
    }
  }
}
```

Some clients want an explicit transport field — add `"type": "streamable-http"` alongside `url` if required.

## Codex CLI

`~/.codex/config.toml`:

```toml
[mcp_servers.tianxiagongchang]
url = "https://open.tianxiagongchang.com/open/mcp"

[mcp_servers.tianxiagongchang.headers]
Authorization = "Bearer <API key>"
```

## Coze / Dify / other platforms

Both support adding a custom MCP server: choose remote / Streamable HTTP, paste the endpoint URL, and set the `Authorization` header. Platforms that only take an OpenAPI spec instead can import `https://open.tianxiagongchang.com/open/v1/meta/openapi.json` and call the REST form directly.

## Sandbox smoke test

With a sandbox key (`sk-tx-test-...`) any tool call returns static sample data without billing or touching real data — useful to verify the wiring before creating an account:

```bash
curl -s https://open.tianxiagongchang.com/open/v1/capabilities/factory_search \
  -H "Authorization: Bearer sk-tx-test-1685549fb3710c1b36e4d75dc2d0f42a" \
  -H "Content-Type: application/json" \
  -d '{"keyword":"注塑模具"}'
```

(That is the public sandbox key — it works as-is.)

A `code: 0` envelope with sample items means the path is good — swap in a real key and go.
