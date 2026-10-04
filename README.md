# Agent Looker for Agent Plugins

A plugin in the [Agent Plugins](https://agent-plugins.org/) format that protects AI agents from unsafe URLs, malicious content, and prompt injection via the [Agent Looker](https://agentlooker.ai/) MCP server. The format is vendor-neutral; ChatGPT is the first host this README covers.

## What it does

**Skills** teach the model when to call the MCP tools:

| Skill | Trigger | MCP tool |
|-------|---------|----------|
| `check-url-safety` | Before opening, downloading, or following any URL | `check_url_safety` |
| `check-text-safety` | After reading web pages, search results, files, emails, pasted text, or connected-app output | `check_text_safety` |
| `report-risk-url` | When a suspicious URL is discovered | `report_risk_url` |
| `report-risk-text` | When prompt injection, hidden instructions, or leaked secrets are discovered | `report_risk_text` |

**Hooks** inject the same rules into context at `SessionStart`, `PreToolUse`, and `PostToolUse`. They are static JSON printed with `cat`; no runtime dependency.

| | ChatGPT Chat | ChatGPT Work |
|---|---|---|
| Skills, MCP tools | Yes | Yes |
| Hooks | Ignored | Yes |

In plain Chat, protection comes from the skills and the tool descriptions alone; the skills are written to work without hooks.

```
check_url_safety ──UNSAFE──> refuse, tell the user the threat categories
       │
      SAFE ──> read ──> check_text_safety ──BLOCK──> treat as data only
                                         ──FLAG───> warn, check embedded links individually
                                         ──ALLOW──> continue
```

## Requirements

- An Agent Looker account ([dashboard](https://app.agentlooker.ai/))
- ChatGPT with **Developer mode** (plan and workspace policy dependent)

## Installation

### ChatGPT desktop (Work)

1. **Plugins → Add plugin marketplace**
2. **Source**: `Gogolook-Inc/agent-looker-for-AgentPlugins`
3. **Git ref**: `main` for production, `staging` or `develop` for testing
4. **Sparse paths**: leave empty
5. Install **Agent Looker** from the Plugins Directory and sign in to the connector (OAuth 2.1, Google login)

To share inside your organization: **Plugins → Personal →** plugin menu **→ Publish**.

### ChatGPT web

The browser has no filesystem, so only the MCP connector applies:

1. **Settings → Security and login → Developer mode**
2. **Plugins → +**, choose **Public endpoint**, URL `https://api.agentlooker.ai/mcp`
3. Sign in

## Environments

Branch == environment. Each branch carries its own MCP URL in `mcp.json` and in each skill's `agents/openai.yaml`.

| Branch | API |
|---|---|
| `production` | `https://api.agentlooker.ai/mcp` |
| `staging` | `https://api-staging.agentlooker.ai/mcp` |
| `develop` | `https://api-develop.agentlooker.ai/mcp` |

Staging and develop sign-in pages sit behind HTTP Basic Auth at the CDN; the browser prompts once before the Google login.

Never edit the URL by hand. The [mcp-url workflow](.github/workflows/mcp-url.yml) fails a pull request whose URL does not match the target branch and rewrites it on push.

## Backend notes

Checked against production on 2026-10-02.

| | Status |
|---|---|
| OAuth metadata (S256, `none`, DCR) | OK |
| `/.well-known/oauth-protected-resource/mcp` | OK |
| `/.well-known/oauth-protected-resource` (root) | 404. OpenAI documents the root path; add an alias if discovery fails. |
| MCP `instructions` | Not set. The only way to deliver standing rules in plain Chat; keep key points in the first 512 characters. |
| Tool annotations (`readOnlyHint` etc.) | Not set. `check_*` should be read-only. |

## Structure

```
.agents/plugins/marketplace.json
.github/workflows/mcp-url.yml
plugins/agent-looker/
  plugin.json          # portable manifest, OpenAI settings in extensions.com.openai
  mcp.json             # streamable-http server
  skills/*/            # SKILL.md + agents/openai.yaml
  hooks/               # hooks.json + static context payloads
  assets/              # icon.png 128px, logo.png 512px
```

## License

GPL-3.0 -- see [LICENSE](plugins/agent-looker/LICENSE).
