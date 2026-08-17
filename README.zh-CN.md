# 中国工厂检索 — Agent Skill

[English](README.md) · **简体中文**

给你的 AI Agent 装上「在中国找工厂」的能力：按产品、行业、地区检索 **480 万家已核实的中国工厂**，拉取完整企业档案与联系方式，并做 AI 供应商尽调。适用于 WorkBuddy、Claude Code、Cursor、Codex，以及任何支持 [Agent Skills](https://agentskills.io) 或 MCP 的 Agent。

能力由[天下工厂开放平台](https://www.tianxiagongchang.com/open/docs)提供。

## 安装 skill

```bash
npx skills add InequalTech/china-factory-search
```

或者把这个仓库直接复制进 Agent 的 skills 目录：

- WorkBuddy：`~/.workbuddy/skills/china-factory-search/`（也可以把 `SKILL.md` 直接拖进聊天框导入）
- Claude Code：`~/.claude/skills/china-factory-search/` 或项目内 `.claude/skills/china-factory-search/`

## 或者只接 MCP 服务器

WorkBuddy，写进 `~/.workbuddy/mcp.json`：

```json
{
  "mcpServers": {
    "tianxiagongchang": {
      "type": "http",
      "url": "https://open.tianxiagongchang.com/open/mcp",
      "headers": { "Authorization": "Bearer <API 密钥>" }
    }
  }
}
```

Claude Code：

```bash
claude mcp add --transport http tianxiagongchang \
  "https://open.tianxiagongchang.com/open/mcp" \
  --header "Authorization: Bearer <API 密钥>"
```

更多客户端（Cursor、Codex、扣子 Coze、Dify、通用 `mcp.json`）见 [references/clients.md](references/clients.md)。

## 你会得到什么

| 工具 | 做什么 |
|---|---|
| `factory_search` | 按关键词 + 地区 + 行业码检索工厂 |
| `factory_detail` | 单家工厂的完整企业档案 |
| `factory_contact` | 单家工厂的联系方式 |
| `factory_deepdive` | 对一家工厂做 AI 深度尽调（30～90 秒） |
| `factory_agent_search` | 一句自然语言 → 一份筛过的工厂名单（30～90 秒） |

装好之后可以这样问：

> 「帮我在浙江找做不锈钢餐具的工厂，要有自营出口的，挑 5 家，把最好的 2 家联系方式拉出来。」

> 「宁波做注塑模具的厂有哪些？先给我一批名单。」

> 「这家公司是真工厂还是贸易商？」

## API 密钥

到[开发者控制台](https://www.tianxiagongchang.com/open/console)免费注册即可获取，注册就送体验积分、能打真实调用。按次透明计价，每个响应都会写明本次消耗的积分和余额（[计价明细](references/api.md)）。

想先试再注册？沙箱密钥（`sk-tx-test-` 前缀）零成本返回固定示例数据，见 [SKILL.md](SKILL.md#api-密钥)。

在 WorkBuddy 这类带微信 Agent Pay 的客户端里，积分不足时可以直接在对话内拉起微信支付充值，不用离开聊天窗口。

## 为什么用这个库

天下工厂对中国制造企业的记录做聚合、清洗与交叉校验——工商注册数据、经营范围、产品、网络痕迹——并做「工厂 vs 贸易商」识别，所以检索返回的是**真正的制造企业**，而不是挂着工厂名字的贸易公司。

## 相关链接

- 文档中心：<https://www.tianxiagongchang.com/open/docs>
- OpenAPI 规范（公开）：`https://open.tianxiagongchang.com/open/v1/meta/openapi.json`
- MCP Registry 条目：`com.tianxiagongchang/factory-search`
- 官网：<https://www.tianxiagongchang.com>

## 许可

MIT，见 [LICENSE](LICENSE)。skill 与文档是开源的；背后的 API 是商业服务，注册赠送体验积分。
