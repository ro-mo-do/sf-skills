# sf-hosted-mcp

End-to-end setup skill for connecting any AI client (Claude Code, Cursor, ChatGPT, Postman) to **Salesforce Hosted MCP Servers** via OAuth 2.0 + PKCE.

## What This Skill Covers

| Area | Description |
|------|-------------|
| **External Client App (ECA)** | Creating and configuring the OAuth identity for MCP clients |
| **MCP Server activation** | Enabling built-in Salesforce MCP servers in Setup |
| **Claude Code CLI** | Configuring `.mcp.json` with `mcp-remote` |
| **Cursor** | Configuring `~/.cursor/mcp.json` with the `auth` block |
| **Postman** | OAuth 2.0 + PKCE setup for testing MCP connections |
| **Troubleshooting** | JWT errors, PKCE verifier failures, Cursor race conditions |

## What This Skill Does NOT Cover

| Area | Use Instead |
|------|-------------|
| Non-MCP OAuth / Connected Apps | [sf-connected-apps](../sf-connected-apps/) |
| Deploying metadata to orgs | [sf-deploy](../sf-deploy/) |
| Custom Apex `@InvocableMethod` MCP tools | [sf-apex](../sf-apex/) |
| DX MCP Server (`sfdx_dev`) | separate guide |
| Custom MCP server development | separate guide |
| MuleSoft / Heroku MCP | separate guide |

## Architecture

```
AI Client (Claude / Cursor / ChatGPT)
    │  OAuth 2.0 + PKCE  (per-user, named user tokens)
    ▼
External Client App  (consumer key + secret)
    │
    ▼
Salesforce Hosted MCP Server
  https://api.salesforce.com/platform/mcp/v1/<SERVER-NAME>           (production)
  https://api.salesforce.com/platform/mcp/v1/sandbox/<SERVER-NAME>   (sandbox/scratch)
    │
    ▼
Salesforce Platform  (FLS · sharing rules · object permissions enforced per-user)
```

## Available MCP Servers

| Server name | What it exposes | Risk |
|---|---|---|
| `platform/sobject-reads` | Read + query only | Low |
| `platform/sobject-mutations` | Create + update, no delete | Medium |
| `platform/sobject-deletes` | Delete operations only | High |
| `platform/sobject-all` | Full CRUD across all SObjects | High |
| `platform/api-catalog` | REST API endpoints as tools | Medium |
| `platform/flows` | Lightning Flows as callable tools | Medium |
| `platform/invocable-actions` | Apex `@InvocableMethod` classes | Medium |
| `platform/prompt-builder` | Prompt Builder templates | Low |
| `platform/data-360` | Unified customer data SQL queries | Low |
| `platform/tableau-next` | Tableau analytics / semantic models | Low |

**Recommended starting point:** `platform/sobject-reads` — read-only and safe for initial testing.

## Requirements

| Requirement | Value |
|---|---|
| Salesforce org type | Production, Developer, Sandbox, or Scratch (via DevHub) |
| sf CLI | v2.x |
| Node.js | Required for `mcp-remote` (Claude Code and other stdio-based clients) |
| ECA activation delay | Up to 30 minutes after creation |

## Quick Start

**Claude Code:** Install the skill and invoke it:
```
/sf-hosted-mcp
```

**Other AI assistants:** Paste the contents of `SKILL.md` as context, then ask:
```
Use these instructions to connect me to Salesforce via Hosted MCP.
```

## Key Gotchas

- **Connected Apps cannot be used for MCP** — only External Client Apps work
- **Cursor `auth` key must be `CLIENT_ID` (all caps)** — `clientId` camelCase is silently ignored
- **Claude Code native SSE does not work** — use `mcp-remote` via `npx` instead
- **Scratch orgs**: ECA cannot be created via Setup UI — use DevHub + package + install
- **Multiple Cursor servers sharing one `CLIENT_ID`**: authenticate one at a time; do not click "Connect" while the auto-triggered browser is open

## Documentation

| Document | Description |
|---|---|
| [SKILL.md](SKILL.md) | Full skill reference — steps 1 through 5 |
| [assets/](assets/) | ECA metadata templates for deployment |

## License

MIT — authored by Robert Doan
