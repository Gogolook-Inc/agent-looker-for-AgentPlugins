# Agent Looker for Agent Plugins

採用 [Agent Plugins](https://agent-plugins.org/) 格式的 plugin，透過 [Agent Looker](https://agentlooker.ai/) MCP server 防護 AI agent 不受不安全的 URL、惡意內容與 prompt injection 影響。格式是 vendor-neutral 的，本文先以 ChatGPT 為例。

## 功能

**Skills** 告訴模型何時呼叫 MCP tools：

| Skill | 時機 | MCP tool |
|-------|------|----------|
| `check-url-safety` | 開啟、下載、跟隨任何 URL 之前 | `check_url_safety` |
| `check-text-safety` | 讀完網頁、搜尋結果、檔案、email、貼上的文字、connector 回傳內容之後 | `check_text_safety` |
| `report-risk-url` | 發現可疑 URL 時 | `report_risk_url` |
| `report-risk-text` | 發現 prompt injection、隱藏指令、外洩機密時 | `report_risk_text` |

**Hooks** 在 `SessionStart`、`PreToolUse`、`PostToolUse` 注入同一套規則，靜態 JSON 用 `cat` 輸出，沒有 runtime 依賴。

| | ChatGPT Chat | ChatGPT Work |
|---|---|---|
| Skills、MCP tools | ✅ | ✅ |
| Hooks | ❌ 忽略 | ✅ |

一般 Chat 只靠 skills 與 tool description，skills 的寫法假設 hooks 不會觸發。

## 需求

- Agent Looker 帳號（[dashboard](https://app.agentlooker.ai/)）
- ChatGPT 的 **Developer mode**（視方案與 workspace 政策）

## 安裝

### ChatGPT desktop（Work）

1. **Plugins → Add plugin marketplace**
2. **Source**：`Gogolook-Inc/agent-looker-for-AgentPlugins`
3. **Git ref**：`main` 是 production，測試填 `staging` 或 `develop`
4. **Sparse paths**：留空
5. 在 Plugins Directory 安裝 **Agent Looker**，登入 connector（OAuth 2.1，Google 登入）

要分享給組織內其他人：**Plugins → Personal →** plugin 選單 **→ Publish**。

### ChatGPT web

瀏覽器沒有檔案系統，只能接 MCP connector：

1. **Settings → Security and login → Developer mode**
2. **Plugins → +**，選 **Public endpoint**，URL 填 `https://api.agentlooker.ai/mcp`
3. 登入

## 環境

分支 = 環境。每個分支的 `mcp.json` 與各 skill 的 `agents/openai.yaml` 帶自己的 MCP URL。

| 分支 | API |
|---|---|
| `production` | `https://api.agentlooker.ai/mcp` |
| `staging` | `https://api-staging.agentlooker.ai/mcp` |
| `develop` | `https://api-develop.agentlooker.ai/mcp` |

staging / develop 的登入頁在 CDN 有 Basic Auth，瀏覽器會先問一次帳密。

URL 不要手改。[mcp-url workflow](.github/workflows/mcp-url.yml) 會讓 URL 與目標分支不符的 PR 失敗，push 時自動改寫。

## 後端待辦

2026-10-02 對 production 實測。

| | 狀態 |
|---|---|
| OAuth metadata（S256、`none`、DCR） | OK |
| `/.well-known/oauth-protected-resource/mcp` | OK |
| 根路徑 `/.well-known/oauth-protected-resource` | 404。OpenAI 文件寫根路徑，找不到就加 alias |
| MCP `instructions` | 未設定。一般 Chat 唯一能傳常駐規則的管道，重點放前 512 字 |
| Tool annotations（`readOnlyHint` 等） | 未設定。`check_*` 應標 read-only |

## 結構

```
.agents/plugins/marketplace.json
.github/workflows/mcp-url.yml
plugins/agent-looker/
  plugin.json          # portable manifest，OpenAI 設定在 extensions.com.openai
  mcp.json             # streamable-http server
  skills/*/            # SKILL.md + agents/openai.yaml
  hooks/               # hooks.json + 靜態 context payload
  assets/              # icon.png 128px、logo.png 512px
```

## 授權

GPL-3.0，見 [LICENSE](plugins/agent-looker/LICENSE)。
