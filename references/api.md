# 接口参考——天下工厂开放平台

两种可互换的传输方式，能力相同，出入参完全一致：

- **MCP**（Streamable HTTP）：`https://open.tianxiagongchang.com/open/mcp`——五个工具，名字与下面的能力名一一对应。
- **REST**：`POST https://open.tianxiagongchang.com/open/v1/capabilities/<能力名>`——JSON body 就是工具入参。

两者鉴权方式相同：`Authorization: Bearer <API 密钥>`。密钥在[开发者控制台](https://www.tianxiagongchang.com/open/console)创建。

机器可读规范（公开，不需要密钥）：`GET https://open.tianxiagongchang.com/open/v1/meta/openapi.json`。
给人看的文档：[开放平台文档中心](https://www.tianxiagongchang.com/open/docs)。

## 统一响应格式

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

`code: 0` 表示成功，其余都是错误（错误码表见 SKILL.md）。**以响应体里的 `code` 为准**——MCP 响应永远是 HTTP 200，REST 会把业务码映射到 HTTP 状态码。

入参校验严格：未知字段一律拒绝，返回 `code: 40000`。

## 能力

### factory_search

检索工厂。所有筛选条件与关键词之间是 AND 关系。

入参：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `keyword` | string | 是 | 产品词或公司词，**用简体中文**召回最好 |
| `intent` | string | 否 | 检索意图提示 |
| `province` | string | 否 | 例如 `"浙江省"`（带后缀的全称） |
| `city` | string | 否 | 例如 `"宁波市"` |
| `district` | string | 否 | 例如 `"北仑区"` |
| `industry` | string | 否 | 取自 `/meta/industries` 的行业码，例如 `"C35"` 或 `"C3591"`（按前缀展开整棵子树）。未知码 → `40000` |
| `page` | int | 否 | 默认 1，**最大 100**（超过报 `40000`，绝不静默截断） |
| `per_page` | int | 否 | 默认 20，**最大 50**（超过报 `40000`，绝不静默截断） |

出参 `data`：`{total, page, per_page, items: []}`。每条 item 形如 `{id, name, region, industry, core_name, product, ...}`——其中 `id` 就是其他所有能力用的 `company_id`。

翻页上限：`per_page` 50 × `page` 100 = **单次查询最多 5000 行**。要导出更多，就按地区或行业子码切分查询条件，再按 `items[].id` 去重（见 SKILL.md「批量导出名单」）。计费按调用次数、与 `per_page` 无关，零命中的检索同样计费。

### factory_detail

入参：`{"company_id": <int>}`（必填，取自检索结果）。

出参 `data`：完整企业档案——工商注册信息、注册资本、成立日期、经营状态、经营范围、产品与服务、标签（例如已核实工厂）、地址。字段名是 snake_case。

### factory_contact

入参：`{"company_id": <int>}`（必填）。

出参 `data`：联系方式记录，每条包含号码本身（`val`）、持有人姓名、以及已知的职务。

计费说明：**同一个应用重复解锁同一家公司免费**，去重在服务端做。批量解锁是主要成本项，先收敛短名单再解锁。

### factory_deepdive

对单家工厂做 AI 深度尽调：它实际做什么、能力信号如何、并对着公开网络信息做核实。长耗时（30～90 秒）。

入参：

| 字段 | 类型 | 必填 |
|---|---|---|
| `company_id` | int | 是 |
| `company_name` | string | 是 |
| `product` | string | 否——指定某条产品线，把尽调聚焦在这条产品线上 |

出参 `data`：结构化的尽调报告。

### factory_agent_search

自然语言找厂：内置 Agent 解析意图、规划筛选条件、检索、核实候选，返回一份筛过的名单。长耗时（30～90 秒），中文提问效果最好。

入参：

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `query` | string | 是 | 例如 `"浙江一带做注塑模具、有出口经验的厂"` |
| `conversation_id` | string | 否 | 传上一次响应里的 conversation id，可以多轮迭代收敛 |

出参 `data`：筛过的工厂名单，外加下一轮要用的 `conversation_id`。走 REST 时是同步返回，没有流式。

## 免费的元数据接口（任意有效密钥，不扣积分）

| 接口 | 返回 |
|---|---|
| `GET /open/v1/meta/regions` | 省/市/区三级树（官方标准名称） |
| `GET /open/v1/meta/industries` | 四级行业码树——`industry` 入参的取值来源 |
| `GET /open/v1/meta/openapi.json` | OpenAPI 3.1 规范（公开，不需要密钥） |

## 计价（2026-07）

2000 积分 = ¥1。每个响应都会写明本次实际的 `credits_charged` 与 `credits_balance`。

| 能力 | 积分/次 | 约合人民币 |
|---|---|---|
| `factory_search` | 400 | ¥0.20 |
| `factory_detail` | 200 | ¥0.10 |
| `factory_contact` | 400（同一家公司重复解锁：0） | ¥0.20 |
| `factory_deepdive` | 500 | ¥0.25 |
| `factory_agent_search` | 200 | ¥0.10 |

注册即送体验积分，可以直接打真实调用。沙箱密钥（`sk-tx-test-` 前缀）不计费，返回固定示例数据。

## 限流与超时

- 通用：每个密钥 10 QPS、300 次/分钟，所有能力合并计算。配额按应用算，且 MCP 与 REST 共享——换传输方式不会多给配额。
- 慢通道（`factory_deepdive` + `factory_agent_search`，共用计数器）：6 次/分钟。沙箱密钥豁免。
- `factory_contact` 额外限制：1 QPS，且每个应用每天最多 500 次。
- 超限返回 `code: 42900`，不计费。响应不带 `Retry-After` 头，请用固定的指数退避（1 秒 / 2 秒 / 4 秒），慢通道调用串行化而不是并发扇出。
- 客户端超时：`factory_search` / `factory_detail` / `factory_contact` 给 30 秒足够；`factory_deepdive` 要设 **≥ 120 秒**，`factory_agent_search` 要设 **≥ 180 秒**——默认 10 秒的超时会把它们从中间掐断，看起来就像服务挂了。
