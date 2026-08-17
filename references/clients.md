# MCP 客户端配置

一个端点、一个请求头——下面每个客户端都是这两件事换一种配置语法而已：

- 端点（Streamable HTTP）：`https://open.tianxiagongchang.com/open/mcp`
- 请求头：`Authorization: Bearer <API 密钥>`——密钥到[开发者控制台](https://www.tianxiagongchang.com/open/console)申请

连上之后应该能看到五个工具：`factory_search`、`factory_detail`、`factory_contact`、`factory_deepdive`、`factory_agent_search`。支持微信 Agent Pay 的客户端（WorkBuddy、QClaw）可能会多出第六个 `credits_topup`，见 SKILL.md「对话内充值」。

## WorkBuddy

配置文件 `~/.workbuddy/mcp.json`（用户级；只对单个项目生效就放 `<项目目录>/.workbuddy/mcp.json`）。界面路径：侧边栏 **插件 → MCP 服务器 → 配置 MCP**。

```json
{
  "mcpServers": {
    "tianxiagongchang": {
      "type": "http",
      "url": "https://open.tianxiagongchang.com/open/mcp",
      "headers": {
        "Authorization": "Bearer <API 密钥>"
      }
    }
  }
}
```

`type` 必须是 `http`（Streamable HTTP），平台没有单独的 `/sse` 端点。WorkBuddy 自带 `weixinpay` 插件，所以平台侧开启该功能后，可以在对话内直接充值积分。

## Claude Code

```bash
claude mcp add --transport http tianxiagongchang \
  "https://open.tianxiagongchang.com/open/mcp" \
  --header "Authorization: Bearer <API 密钥>"
```

验证：`claude mcp list` → `tianxiagongchang: ... - ✓ Connected`。

## Cursor 及通用 `mcp.json`

配置文件 `~/.cursor/mcp.json`（或项目级的 `.cursor/mcp.json`；大多数用 JSON 配置的客户端都吃这个结构）：

```json
{
  "mcpServers": {
    "tianxiagongchang": {
      "url": "https://open.tianxiagongchang.com/open/mcp",
      "headers": {
        "Authorization": "Bearer <API 密钥>"
      }
    }
  }
}
```

有些客户端要求显式写传输方式，那就在 `url` 旁边补一个 `"type": "streamable-http"`。

## Codex CLI

配置文件 `~/.codex/config.toml`：

```toml
[mcp_servers.tianxiagongchang]
url = "https://open.tianxiagongchang.com/open/mcp"

[mcp_servers.tianxiagongchang.headers]
Authorization = "Bearer <API 密钥>"
```

## 扣子 Coze / Dify 等平台

两者都支持添加自定义 MCP 服务器：选远程／Streamable HTTP，粘贴端点 URL，设置 `Authorization` 请求头即可。只接受 OpenAPI 规范的平台，可以直接导入 `https://open.tianxiagongchang.com/open/v1/meta/openapi.json`，按 REST 形式调用。

## 沙箱冒烟测试

用沙箱密钥（`sk-tx-test-...`）调任何工具都会返回固定示例数据，不计费、不碰真实数据——适合在用户开户之前先把链路跑通：

```bash
curl -s https://open.tianxiagongchang.com/open/v1/capabilities/factory_search \
  -H "Authorization: Bearer sk-tx-test-1685549fb3710c1b36e4d75dc2d0f42a" \
  -H "Content-Type: application/json" \
  -d '{"keyword":"注塑模具"}'
```

（这就是公共沙箱密钥，可以直接用。）

返回 `code: 0` 并带示例条目，说明链路通了——换成正式密钥即可开跑。
