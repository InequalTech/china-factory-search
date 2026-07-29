# China Factory Search — Agent Skill

Give your AI agent the ability to search **4.8 million verified Chinese factories** — by product, industry, and region — fetch full company profiles and contact info, and run AI-powered supplier vetting. Works with Claude Code, Cursor, Codex, and any agent that supports [Agent Skills](https://agentskills.io) or MCP.

Powered by the [Tianxia Gongchang open platform](https://www.tianxiagongchang.com/open/docs) (天下工厂开放平台).

## Install the skill

```bash
npx skills add InequalTech/china-factory-search
```

Or copy this repo into your agent's skills directory (e.g. `.claude/skills/china-factory-search/`).

## Or just connect the MCP server

```bash
claude mcp add --transport http tianxiagongchang \
  "https://open.tianxiagongchang.com/open/mcp" \
  --header "Authorization: Bearer <API key>"
```

More clients (Cursor, Codex, Coze, Dify, generic `mcp.json`): [references/clients.md](references/clients.md).

## What you get

| Tool | What it does |
|---|---|
| `factory_search` | Search factories by keyword + region + industry code |
| `factory_detail` | Full company profile |
| `factory_contact` | Contact info |
| `factory_deepdive` | AI deep-dive on one factory (30–90 s) |
| `factory_agent_search` | One natural-language request → vetted factory list (30–90 s) |

Example asks, once installed:

> "Find injection mold factories around Ningbo with export experience, shortlist 5, and get contacts for the top 2."

> "帮我在浙江找做不锈钢餐具的工厂，要有自营出口的。"

## API key

Free sign-up at the [developer console](https://www.tianxiagongchang.com/open/console) — includes welcome credits for real calls. Transparent per-call pricing; every response reports credits spent and balance ([details](references/api.md#pricing-2026-07)).

Want to try before signing up? Sandbox keys (`sk-tx-test-` prefix) return static sample data at zero cost — see [SKILL.md](SKILL.md#api-key).

## Why this database

Tianxia Gongchang aggregates, cleans, and cross-validates Chinese manufacturer records — registration data, business scope, products, web presence — and applies factory-vs-trader identification, so searches return **actual manufacturers**, not trading companies wearing factory names.

## Links

- Docs: <https://www.tianxiagongchang.com/open/docs>
- OpenAPI spec (public): `https://open.tianxiagongchang.com/open/v1/meta/openapi.json`
- MCP Registry entry: `com.tianxiagongchang/factory-search`
- Site: <https://www.tianxiagongchang.com>

## License

MIT — see [LICENSE](LICENSE). The skill and docs are open source; the backing API is a commercial service with free trial credits.
