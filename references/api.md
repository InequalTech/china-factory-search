# API reference — Tianxia Gongchang open platform

Two interchangeable transports, same capabilities, same input/output:

- **MCP** (Streamable HTTP): `https://open.tianxiagongchang.com/open/mcp` — five tools named exactly like the capabilities below.
- **REST**: `POST https://open.tianxiagongchang.com/open/v1/capabilities/<name>` — JSON body = tool arguments.

Auth for both: `Authorization: Bearer <API key>`. Keys are created in the [developer console](https://www.tianxiagongchang.com/open/console).

Machine-readable spec (public, no key): `GET https://open.tianxiagongchang.com/open/v1/meta/openapi.json`.
Human docs: <https://www.tianxiagongchang.com/open/docs>.

## Response envelope

```json
{
  "code": 0,
  "message": "ok",
  "request_id": "req_...",
  "credits_charged": 400,
  "credits_balance": 123600,
  "data": { }
}
```

`code: 0` means success; anything else is an error (see table in SKILL.md). The envelope `code` is authoritative — MCP responses are always HTTP 200, REST mirrors the business code in the HTTP status.

Input validation is strict: unknown fields are rejected with `code: 40000`.

## Capabilities

### factory_search

Search factories. All filters are AND-combined with the keyword.

Input:

| Field | Type | Required | Notes |
|---|---|---|---|
| `keyword` | string | yes | Product/company keyword, **Simplified Chinese** for best recall |
| `intent` | string | no | Search intent hint |
| `province` | string | no | e.g. `"浙江省"` (full name with suffix) |
| `city` | string | no | e.g. `"宁波市"` |
| `district` | string | no | e.g. `"北仑区"` |
| `industry` | string | no | Industry code from `/meta/industries`, e.g. `"C35"` or `"C3591"` (prefix expands to the whole subtree). Unknown code → `40000` |
| `page` | int | no | Default 1 |
| `per_page` | int | no | Default 20 |

Output `data`: `{total, page, per_page, items: []}`. Each item: `{id, name, region, industry, core_name, product, ...}` — `id` is the `company_id` used by every other capability.

### factory_detail

Input: `{"company_id": <int>}` (required, from a search result).

Output `data`: full profile — registration info, capital, establishment date, business status, business scope, products/services, tags (e.g. verified-factory), address. Field names are snake_case.

### factory_contact

Input: `{"company_id": <int>}` (required).

Output `data`: contact records, each with the value (`val`), holder name, and position where known.

Billing note: unlocking the **same company again with the same app is free** — dedup is server-side. Batch-unlocking many companies is the main cost driver; shortlist first.

### factory_deepdive

AI deep-dive on one factory: what it actually makes, capability signals, verification against web sources. Long-running (30–90 s).

Input:

| Field | Type | Required |
|---|---|---|
| `company_id` | int | yes |
| `company_name` | string | yes |
| `product` | string | no — focus the dive on a product line |

Output `data`: structured deep-dive report.

### factory_agent_search

Natural-language sourcing: a built-in agent parses intent, plans filters, searches, verifies candidates, and returns a vetted list. Long-running (30–90 s). Best with Chinese queries.

Input:

| Field | Type | Required | Notes |
|---|---|---|---|
| `query` | string | yes | e.g. `"浙江一带做注塑模具、有出口经验的厂"` |
| `conversation_id` | string | no | Pass the previous response's conversation id to refine iteratively |

Output `data`: the vetted factory list plus the `conversation_id` for follow-up turns. Over REST this returns synchronously (no streaming).

## Free metadata endpoints (any valid key, no credits charged)

| Endpoint | Returns |
|---|---|
| `GET /open/v1/meta/regions` | Province / city / district tree (official names) |
| `GET /open/v1/meta/industries` | 4-level industry code tree — source of `industry` values |
| `GET /open/v1/meta/openapi.json` | OpenAPI 3.1 spec (public, no key needed) |

## Pricing (2026-07)

2,000 credits = ¥1. Every response reports exact `credits_charged` / `credits_balance`.

| Capability | Credits / call | ≈ CNY |
|---|---|---|
| `factory_search` | 400 | ¥0.20 |
| `factory_detail` | 200 | ¥0.10 |
| `factory_contact` | 400 (repeat same company: 0) | ¥0.20 |
| `factory_deepdive` | 500 | ¥0.25 |
| `factory_agent_search` | 200 | ¥0.10 |

Sign-up grants welcome credits for real trial calls. Sandbox keys (`sk-tx-test-`) charge nothing and return static samples.

## Rate limits & timeouts

- General: 10 QPS and 300 requests/min per key, all capabilities combined. Quota is per app and shared across MCP and REST — switching transport does not grant extra quota.
- Slow lane (`factory_deepdive` + `factory_agent_search`, shared counter): 6/min. Sandbox keys are exempt.
- `factory_contact` additionally: 1 QPS, and max 500 calls per app per day.
- Over limit → `code: 42900`, nothing is charged. No `Retry-After` header — use fixed exponential backoff (1s / 2s / 4s) and serialize slow-lane calls instead of fanning out.
- Client timeouts: 30 s is enough for `factory_search` / `factory_detail` / `factory_contact`; set **≥ 120 s** for `factory_deepdive` and **≥ 180 s** for `factory_agent_search` — default 10 s timeouts will cut them off mid-run and look like an outage.
