---
name: china-factory-search
description: 用于在中国找工厂、找供应商、比选工厂、验厂核实、拿工厂联系方式的场景——例如「帮我找注塑模具工厂」「浙江有哪些做不锈钢餐具的厂」「这家中国公司是真工厂还是贸易商」「外贸客户开发」「给我一批做汽车零部件的厂商名单」。基于天下工厂开放平台，检索 480 万家已核实的中国工厂，可走 MCP 工具，也可以走纯 REST。（English keywords — find manufacturers in China, source a supplier in China, verify a Chinese factory, get factory contact info, China supplier sourcing）
---

# 中国工厂检索（天下工厂开放平台）

按产品、行业、地区检索 480 万家已核实的中国工厂；拉取单家工厂的完整企业档案与联系方式；对一家工厂做 AI 深度尽调；也可以把一句自然语言诉求直接交给内置的找厂 Agent。

数据来自[天下工厂开放平台](https://www.tianxiagongchang.com/open/docs)——聚合、清洗、交叉校验中国制造企业记录建成的工厂库，带「工厂 vs 贸易商」识别能力。

## 能力一览

| 能力 | 做什么 | 耗时 |
|---|---|---|
| `factory_search` | 按关键词 + 省/市/区 + 行业码检索工厂，支持翻页 | 快 |
| `factory_detail` | 单家工厂的完整档案（工商注册、资本、经营范围、产品、标签） | 快 |
| `factory_contact` | 单家工厂的联系方式（电话等） | 快 |
| `factory_deepdive` | AI 深度尽调：这家厂到底做什么、做得怎么样 | 30～90 秒 |
| `factory_agent_search` | 一句自然语言诉求 → Agent 自主规划、检索、核实，返回一份筛过的工厂名单 | 30～90 秒 |

## 三条接入路径

1. **已配好 MCP**（能看到上面那几个工具名）：直接调用，跳到「调用规矩」。
2. **客户端支持 MCP 但还没配**：配置一次即可，见 [references/clients.md](references/clients.md)。端点 `https://open.tianxiagongchang.com/open/mcp`（Streamable HTTP），请求头 `Authorization: Bearer <API 密钥>`。
3. **客户端不支持 MCP**：走 REST。同样这五个能力，`POST https://open.tianxiagongchang.com/open/v1/capabilities/<能力名>`，鉴权头一样，JSON body 就是工具入参。完整字段表见 [references/api.md](references/api.md)，机器可读规范 `GET https://open.tianxiagongchang.com/open/v1/meta/openapi.json`（公开，不需要密钥）。

## API 密钥

- 优先从环境变量 `TIANXIA_API_KEY` 读取；没有就问用户要。
- 用户还没有密钥？到开发者控制台免费注册领取，见[开发者控制台](https://www.tianxiagongchang.com/open/console)（注册即送体验积分，可以打真实调用）。
- **沙箱密钥／即时试跑**：以 `sk-tx-test-` 开头的密钥返回固定示例数据，不计费、不碰真实数据。用户还没开户时，可用公共沙箱密钥 `sk-tx-test-1685549fb3710c1b36e4d75dc2d0f42a` 把链路先跑通。**绝不能把沙箱返回的示例数据当成真实工厂数据呈现给用户。**

## 调用规矩

1. **关键词用简体中文。** 底层索引是中文的。用户给的是英文产品词就先译成简体中文再调（`"injection molds"` → `"注塑模具"`），地区同理，`province: "浙江省"`、`city: "宁波市"`。结果回给用户时再切回用户的语言。
2. **入参严格校验。** 未知字段会被直接拒绝（这条正好帮你抓拼写错误，不要用同样的 body 重试）。只传文档里写明的字段。
3. **看 `code`，不看 HTTP 状态码。** 每个响应都是统一响应格式 `{code, message, request_id, credits_charged, credits_balance, data}`，`code: 0` 才是成功。MCP 一律返回 HTTP 200，REST 会把业务码映射到 HTTP 状态码，但以响应体里的 `code` 为准。
4. **典型漏斗**：`factory_search`（诉求模糊就用 `factory_agent_search`）→ 收敛出短名单 → `factory_detail` → 只对最终候选调 `factory_contact`（它单条有效记录的成本最高；同一家公司重复解锁对同一个密钥免费）。
5. **诉求模糊时优先 `factory_agent_search`**（例如「帮我找浙江一带做注塑模具、有出口经验的厂」）——把用户的原话作为 `query` 传进去，中文效果最好。已经有明确筛选条件时用 `factory_search`。慢通道（`factory_deepdive`、`factory_agent_search`）限流 6 次/分钟，不要并发扇出。
6. **行业码**（`industry`，例如 `"C35"`）来自 `GET /open/v1/meta/industries`，地区树来自 `GET /open/v1/meta/regions`，两个接口任意密钥免费调用。不要凭印象猜行业码——未知码直接返回 `code: 40000`，绝不会被静默忽略。
7. **费用是透明的。** 每个响应都带 `credits_charged` 和 `credits_balance`。用户问花了多少，从响应里读，不要估算。遇到 `40201`（积分不足）：如果工具列表里有 `credits_topup`，可以在对话内引导充值（见下节）；没有就引导用户去控制台充值。

## 对话内充值（微信 Agent Pay）

在带 `weixinpay` 插件的客户端里（WorkBuddy、QClaw），服务端可能额外暴露一个 `credits_topup` 工具。当调用返回 `40201`、或用户主动说要充值时：

1. 让用户自己选套餐——`p10`（¥10）、`p50`（¥50）、`p200`（¥200）。把选项列出来，不要默认推最大的那档。响应里会写明这一档实际到账多少积分。
2. 调用 `credits_topup {"pack": "p10"}`，返回结果里带 `WeixinPay-Required`。
3. 把这个值作为 `paymentCode` 交给 `weixinpay_pay` 工具，由它拉起微信支付授权。支付码约 14 分钟过期；过期未付就重新调一次 `credits_topup`，未支付的订单不产生任何费用。
4. 用户付完，积分会自动到账（通常几秒内）。重试原来那次调用；如果还是 `40201`，等一分钟再重试一次。

注意：只在余额确实不足、或用户主动提出时才引导充值，**任务进行中不要主动推销**。工具列表里没有 `credits_topup`（其他客户端，或该功能未开启）就退回控制台充值，见[开发者控制台](https://www.tianxiagongchang.com/open/console)。沙箱密钥返回的是不可支付的示例值（`SANDBOX-NOT-PAYABLE`），绝不要把它丢给 `weixinpay_pay`。

## 批量导出名单

用户要批量导出一批工厂（CSV、表格、导进 CRM）时，按下面这套流程走，不要自己临时发明翻页策略：

1. **只翻 `factory_search` 的页。** `factory_agent_search` 没有翻页而且慢、贵；`factory_detail` / `factory_contact` 是名单定下来之后的逐家补全，不是拉名单的手段。
2. **先探 `total`**：用 `per_page: 50` 调第 1 页，读出 `data.total`，据此估算工作量和费用并**先告诉用户**再继续。计费按调用次数、不按行数——所以固定用 `per_page: 50`（用 20 的话同样的数据要多付 2.5 倍）。成本 = ceil(N / 50) 次调用 × 400 积分。零命中的检索同样计费，批量翻页前先用一次调用验证查询条件。
3. **串行翻页**，从第 1 页到 ceil(total/50)；遇到 `42900` 按 1 秒/2 秒/4 秒退避后重试当前页。不要并发——配额是按应用算的，并发只会更快撞限流。
4. **硬顶 5000 行/次查询**（`page` ≤ 100）。`total` 超过这个数就切分查询条件——先按 `city` 切，再按 `district`，或者按行业子码切（两棵树都来自免费的 meta 接口）——合并后按 `items[].id` 去重。
5. **按需补全**：检索结果本身已经带了工商摘要和 `phones_count`。`factory_detail` 重复调用会重复计费（结果自己缓存）；`factory_contact` 只对最终候选调（1 QPS、每应用每天 500 次、必须串行；同一家公司重复解锁免费）。
6. **持久化进度**（已完成的页码、已见过的 id、已解锁的 id），中断后能续跑，不会重复付费。

可直接运行的参考实现（纯标准库 Python）和给人看的说明，见[列表导出最佳实践](https://www.tianxiagongchang.com/open/docs/export)。

## REST 快速上手

```bash
# 检索：宁波的注塑模具工厂
curl -s https://open.tianxiagongchang.com/open/v1/capabilities/factory_search \
  -H "Authorization: Bearer $TIANXIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"keyword":"注塑模具","province":"浙江省","city":"宁波市","per_page":10}'

# 然后：拉某一条结果的完整档案和联系方式（company_id 取自 items[].id）
curl -s .../factory_detail  -H "Authorization: Bearer $TIANXIA_API_KEY" \
  -H "Content-Type: application/json" -d '{"company_id":123456}'
curl -s .../factory_contact -H "Authorization: Bearer $TIANXIA_API_KEY" \
  -H "Content-Type: application/json" -d '{"company_id":123456}'
```

`factory_search` 的数据结构：`{total, page, per_page, items: [{id, name, region, industry, core_name, product, ...}]}`。

## 常见错误码

| `code` | 含义 | 怎么办 |
|---|---|---|
| `40000` | 入参非法（未知字段、行业码不存在） | 修 body，不要原样重试 |
| `40100` | 密钥缺失或无效 | 检查请求头；到控制台申请密钥 |
| `40201` | 积分不足 | 有 `credits_topup` 工具就走对话内充值（WorkBuddy／QClaw），否则引导去控制台充值 |
| `40300` | 该应用未开通此能力 | 到控制台的应用设置里开通 |
| `40400` | 公司不存在 | 确认 `company_id` 来自检索结果 |
| `42900` | 触发限流（通用 300 次/分钟；慢通道 6 次/分钟；contact 1 QPS 且每天 500 次） | 退避重试（1 秒/2 秒/4 秒）；慢通道调用串行化 |

完整接口字段、计价、客户端配置：[references/api.md](references/api.md) · [references/clients.md](references/clients.md)
