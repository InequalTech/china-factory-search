---
name: china-factory-search
description: Use when the user wants to find, vet, compare, or contact manufacturers, factories, or suppliers in China — e.g. "find manufacturers in China", "source a supplier for [product]", "is this Chinese factory legit", "get contact info for a Chinese factory", "找工厂", "找供应商", "验厂", "外贸客户开发". Searches a database of 4.8M verified Chinese factories (Tianxia Gongchang open platform) via MCP tools or plain REST.
---

# China Factory Search (Tianxia Gongchang)

Search 4.8 million verified Chinese factories by product, industry, and region; fetch full company profiles and contact info; run AI deep-dives on a single factory; or hand one natural-language request to a built-in sourcing agent.

Backed by the [Tianxia Gongchang open platform](https://www.tianxiagongchang.com/open/docs) — a factory database built by aggregating, cleaning, and cross-validating Chinese manufacturer records, with factory-vs-trader identification.

## Capabilities

| Capability | What it does | Speed |
|---|---|---|
| `factory_search` | Search factories by keyword + province/city/district + industry code, paginated | fast |
| `factory_detail` | Full profile of one factory (registration, capital, scope, products, tags) | fast |
| `factory_contact` | Contact info (phones etc.) for one factory | fast |
| `factory_deepdive` | AI deep-dive: what does this factory actually make, and how good is it | 30–90 s |
| `factory_agent_search` | One natural-language request → agent plans, searches, verifies, returns a vetted factory list | 30–90 s |

## Choose your path

1. **MCP tools already configured** (tool names above visible): call them directly. Skip to "Usage rules".
2. **Agent supports MCP but not configured**: set it up once — see [references/clients.md](references/clients.md). Endpoint: `https://open.tianxiagongchang.com/open/mcp` (Streamable HTTP), header `Authorization: Bearer <API key>`.
3. **No MCP support**: use REST. Same five capabilities at `POST https://open.tianxiagongchang.com/open/v1/capabilities/<capability_name>`, same auth header, JSON body = tool arguments. Full schemas: [references/api.md](references/api.md), machine-readable spec: `GET https://open.tianxiagongchang.com/open/v1/meta/openapi.json` (public, no key).

## API key

- Read the key from the `TIANXIA_API_KEY` environment variable if set; otherwise ask the user.
- No key? Get one free at the developer console: <https://www.tianxiagongchang.com/open/console> (sign-up includes welcome credits to try real calls).
- **Sandbox / instant trial**: keys prefixed `sk-tx-test-` return static sample data — no billing, no real data. Use the public sandbox key `sk-tx-test-1685549fb3710c1b36e4d75dc2d0f42a` to verify wiring end-to-end before the user has an account. Never present sandbox output as real factory data.

## Usage rules

1. **Search in Simplified Chinese.** The underlying index is Chinese. Translate product/company keywords to Simplified Chinese before calling (`"injection molds"` → `"注塑模具"`). Region names too: `province: "浙江省"`, `city: "宁波市"`. Present results back to the user in their language.
2. **Strict input validation.** Unknown JSON fields are rejected (this catches typos — do not retry with the same body). Send only the documented fields.
3. **Check `code`, not HTTP status.** Every response is an envelope `{code, message, request_id, credits_charged, credits_balance, data}`. `code: 0` = success. MCP always returns HTTP 200; REST mirrors the business code in the HTTP status, but the envelope `code` is authoritative.
4. **Typical funnel**: `factory_search` (or `factory_agent_search` for fuzzy asks) → shortlist → `factory_detail` → `factory_contact` only for finalists (it costs the most per useful record; repeat unlocks of the same company are free for your key).
5. **Prefer `factory_agent_search` when the ask is fuzzy** ("find me mold makers around Zhejiang with export experience") — pass the user's request as `query`, in Chinese for best results. Prefer `factory_search` when you have concrete filters. Slow lane (`deepdive`, `agent_search`) is rate-limited to 6 calls/min — don't fan out in parallel.
6. **Industry codes** (`industry`, e.g. `"C35"`) come from `GET /open/v1/meta/industries`; region trees from `GET /open/v1/meta/regions`. Both free with any key. Don't guess codes — an unknown code returns `code: 40000`, it is never silently ignored.
7. **Credits are transparent.** Every envelope reports `credits_charged` and `credits_balance`. If the user asks about cost, read it from the response instead of estimating. On `40201` (insufficient credits), point the user to the console to top up.

## REST quickstart

```bash
# Search: injection mold factories in Ningbo
curl -s https://open.tianxiagongchang.com/open/v1/capabilities/factory_search \
  -H "Authorization: Bearer $TIANXIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"keyword":"注塑模具","province":"浙江省","city":"宁波市","per_page":10}'

# Then: full profile and contacts for one result (id from items[])
curl -s .../factory_detail  -H "Authorization: Bearer $TIANXIA_API_KEY" \
  -H "Content-Type: application/json" -d '{"company_id":123456}'
curl -s .../factory_contact -H "Authorization: Bearer $TIANXIA_API_KEY" \
  -H "Content-Type: application/json" -d '{"company_id":123456}'
```

`factory_search` data shape: `{total, page, per_page, items: [{id, name, region, industry, core_name, product, ...}]}`.

## Common errors

| `code` | Meaning | What to do |
|---|---|---|
| `40000` | Invalid argument (unknown field, bad industry code) | Fix the body; don't retry as-is |
| `40100` | Missing/invalid API key | Check header; get a key at the console |
| `40201` | Insufficient credits | Ask user to top up in the console |
| `40300` | Capability not enabled for this app | Enable it in the console app settings |
| `40400` | Company not found | Verify `company_id` came from a search result |
| `42900` | Rate limited (300/min general; 6/min slow lane; contact 1 QPS & 500/day) | Back off (1s/2s/4s); serialize slow-lane calls |

Full API reference, pricing, and client setup: [references/api.md](references/api.md) · [references/clients.md](references/clients.md)
