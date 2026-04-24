---
name: sf-hosted-mcp
description: >
  Salesforce Hosted MCP Server end-to-end setup. TRIGGER when: user wants to
  connect Claude Code, Cursor, Postman, or another MCP client to a Salesforce org via the
  Hosted MCP Server feature; user needs to create an External Client App for MCP;
  user wants to activate built-in MCP servers; user asks about Salesforce MCP
  server URLs or OAuth setup for MCP. DO NOT TRIGGER for: DX MCP Server (that is
  the sfdx_dev guide), custom MCP server development, or MuleSoft/Heroku MCP.
license: MIT
allowed-tools: Bash Read Write Edit Glob Grep WebFetch AskUserQuestion TodoWrite
metadata:
  version: "1.2.0"
  author: "Robert Doan"
  docs: "https://developer.salesforce.com/docs/platform/hosted-mcp-servers/guide"
---

# sf-hosted-mcp: Salesforce Hosted MCP Server Setup

Use this skill to configure Salesforce Hosted MCP Servers end-to-end — from org login through External Client App creation, MCP server selection, and wiring up Claude Code, Cursor and/or Postman.

## Official Documentation

- Guide root: https://developer.salesforce.com/docs/platform/hosted-mcp-servers/guide
- Create ECA: https://developer.salesforce.com/docs/platform/hosted-mcp-servers/guide/create-external-client-app.html
- Connect Claude: https://developer.salesforce.com/docs/platform/hosted-mcp-servers/guide/claude.html
- Connect Cursor: https://developer.salesforce.com/docs/platform/hosted-mcp-servers/guide/cursor.html
- Server reference: https://developer.salesforce.com/docs/platform/hosted-mcp-servers/references/reference/servers-reference.html

---

## When This Skill Owns the Task

Use `sf-hosted-mcp` when the work involves:
- Connecting Claude Code, Cursor, ChatGPT, or Postman to Salesforce via MCP
- Creating an External Client App for MCP authentication
- Choosing and activating built-in Salesforce MCP servers
- Configuring `mcp.json` or Claude's connector settings for Salesforce

Delegate to skills from https://github.com/Jaganpro/sf-skills.git when the user is:
- Creating ECAs for non-MCP OAuth use cases → [sf-connected-apps](../sf-connected-apps/SKILL.md)
- Deploying metadata to orgs → [sf-deploy](../sf-deploy/SKILL.md)
- Building custom MCP server tools in Apex → [sf-apex](../sf-apex/SKILL.md)

---

## Architecture Overview

```
AI Client (Claude / Cursor / ChatGPT)
    │  OAuth 2.0 + PKCE  (per-user, named user tokens)
    ▼
External Client App  (consumer key + secret)
    │
    ▼
Salesforce Hosted MCP Server
  https://api.salesforce.com/platform/mcp/v1/<SERVER-NAME>          (production)
  https://api.salesforce.com/platform/mcp/v1/sandbox/<SERVER-NAME>  (sandbox/scratch)
    │
    ▼
Salesforce Platform  (FLS · sharing rules · object permissions enforced per-user)
```

Key constraints:
- **Connected Apps cannot be used for MCP** — only External Client Apps work
- External Client Apps cannot be created via Setup UI in scratch orgs; use a DevHub org + package + install
- Allow up to 30 minutes after ECA creation for global activation
- JWT-based access tokens are required; classic session tokens will not work

---

## Step 1 — Verify Org Login

Run this first to check for an active default org:

```bash
sf org display --json
```

If no default org is set or the command errors, log in:

```bash
# Interactive browser login (production / developer org)
sf org login web --alias my-org --set-default

# Sandbox
sf org login web --alias my-sandbox --instance-url https://test.salesforce.com --set-default

# Scratch org (requires DevHub)
sf org create scratch --definition-file config/project-scratch-def.json --alias my-scratch --set-default --duration-days 7
```

After login, confirm with `sf org display --json` and note the `instanceUrl` — you will need it for the MCP server URL.

---

## Step 2 — Create the External Client App

The ECA is the OAuth identity for your MCP client. You need one per MCP client type (one for Claude, one for Cursor, etc.) because each client has a different OAuth callback URL.

### Option A: Metadata deployment (recommended, works in all org types)

Use the templates in `assets/` to build the metadata files, then deploy.

**Minimum required files:**

| File | Metadata type |
|---|---|
| `externalClientApps/<AppName>.eca-meta.xml` | `ExternalClientApplication` |
| `extlClntAppGlobalOauthSets/<AppName>.ecaGlblOauth-meta.xml` | `ExtlClntAppGlobalOauthSettings` |
| `extlClntAppOauthSettings/<AppName>.ecaOauth-meta.xml` | `ExtlClntAppOauthSettings` |

**Step-by-step:**

1. Copy `assets/eca-mcp-template.eca-meta.xml` → `force-app/main/default/externalClientApps/<AppName>.eca-meta.xml`
2. Copy `assets/eca-mcp-global-oauth.ecaGlblOauth-meta.xml` → `force-app/main/default/extlClntAppGlobalOauthSets/<AppName>.ecaGlblOauth-meta.xml`
3. Copy `assets/eca-mcp-oauth-settings.ecaOauth-meta.xml` → `force-app/main/default/extlClntAppOauthSettings/<AppName>.ecaOauth-meta.xml`
4. Replace all `{{PLACEHOLDERS}}` (see table below)
5. Deploy: `sf project deploy start --source-dir force-app/main/default --target-org <alias>`
6. Retrieve the consumer key.

The deployment response includes the ECA record ID in 18-character format (e.g. `"id": "0xIHp0000004CcEMAU"`). Salesforce Setup URLs require the **15-character ID** — trim the last 3 characters:

```
18-char (from deploy response): 0xIHp0000004CcEMAU
15-char (for Setup URL):        0xIHp0000004CcE
```

Use the 15-char ID to build the direct Setup link:

```
https://<instance>.my.salesforce-setup.com/lightning/setup/ManageExternalClientApplication/<15-CHAR-ECA-ID>/detail
```

If that link fails, use the list page instead:

```
https://<instance>.my.salesforce-setup.com/lightning/setup/ManageExternalClientApplication/home
```

**Retrieve the Consumer Key (user action required):**
1. Open the ECA detail page using the link above
2. Navigate to **Settings → OAuth Settings**
3. Copy the **Consumer Key** — you will paste it into the MCP client config in Step 4

**Placeholder reference:**

| Placeholder | Value |
|---|---|
| `{{APP_NAME}}` | API name, no spaces (e.g., `Claude_MCP`) |
| `{{LABEL}}` | Display label (e.g., `Claude MCP Client`) |
| `{{CONTACT_EMAIL}}` | Admin email address |
| `{{DESCRIPTION}}` | Short description |
| `{{CALLBACK_URL}}` | See callback URL table below |
| `{{DISTRIBUTION_STATE}}` | `Local` (single org) or `Packageable` |

**Callback URL by client:**

| Client | Callback URL |
|---|---|
| Claude (claude.ai web) | `https://claude.ai/api/mcp/auth_callback` |
| Claude Code CLI (native SSE) | `http://localhost:8080/callback` |
| Claude Code CLI (via mcp-remote) | `http://localhost:8080/oauth/callback` |
| Cursor (native `auth` block) | Managed internally by Cursor — no entry needed in ECA |
| Cursor (via mcp-remote) | `http://localhost:8081/oauth/callback` |
| Postman (HTTP) | `https://oauth.pstmn.io/v1/callback` |
| Postman (Web) | `https://oauth.pstmn.io/v1/browser-callback` |
| ChatGPT | Retrieve from ChatGPT Advanced settings |

### Option B: Setup UI (production / developer orgs only, not scratch orgs)

1. Setup → Quick Find → **External Client App Manager** → **New External Client App**
2. Fill in Basic Information
3. Check **Enable OAuth**
4. Enter the callback URL for your client (see table above)
5. Add OAuth scopes:
   - **Access MCP servers** (`MCP`)
   - **Perform requests at any time** (`RefreshToken`)
6. Under Security, select:
   - ✅ Issue JSON Web Token (JWT)-based access tokens for named users
   - ✅ Require Proof Key for Code Exchange (PKCE) extension
   - ☐ Deselect all other security options
7. Click **Create**
8. Go to Settings → OAuth Settings → copy **Consumer Key**

---

## Step 3 — Activate MCP Servers in Setup

Each MCP server must be activated in Salesforce Setup before it can accept connections. This is a manual UI step — it cannot be automated via metadata deployment.

**How to activate:**

1. In Setup, go to **Integrations → API Catalog → MCP Servers**
2. Find the server you want (e.g., `Salesforce SObject Reads`)
3. Click **Activate**

The direct URL for each server follows this pattern (replace `<instance>` and the server ID):

```
https://<instance>.my.salesforce-setup.com/lightning/setup/McpServer/<SERVER-ID>/view?setup__source=PLATFORM_STANDARD_MCP_SERVER
```

The server ID is the server name with `/` replaced by `.`:

| Server name | Server ID |
|---|---|
| `platform/sobject-reads` | `platform.sobject-reads` |
| `platform/sobject-mutations` | `platform.sobject-mutations` |
| `platform/sobject-all` | `platform.sobject-all` |
| `platform/sobject-deletes` | `platform.sobject-deletes` |
| `platform/api-catalog` | `platform.api-catalog` |
| `platform/flows` | `platform.flows` |
| `platform/invocable-actions` | `platform.invocable-actions` |
| `platform/prompt-builder` | `platform.prompt-builder` |
| `platform/data-360` | `platform.data-360` |
| `platform/tableau-next` | `platform.tableau-next` |

After deploying the ECA, always provide the user the direct Setup link for each server they requested, using the org's `instanceUrl` (substituting `.my.salesforce.com` → `.my.salesforce-setup.com`).

Ask the user which servers they need. Present this menu:

| Server name (use in URL) | What it exposes | Risk level |
|---|---|---|
| `platform/sobject-all` | Full CRUD + query across all SObjects | High (includes delete) |
| `platform/sobject-reads` | Read + query only, no mutations | Low |
| `platform/sobject-mutations` | Create + update, no delete | Medium |
| `platform/sobject-deletes` | Delete operations only | High |
| `platform/api-catalog` | REST API endpoints as tools | Medium |
| `platform/flows` | Lightning Flows as callable tools | Medium |
| `platform/invocable-actions` | Apex @InvocableMethod classes as tools | Medium |
| `platform/prompt-builder` | Prompt Builder templates as MCP prompts | Low |
| `platform/data-360` | Unified customer data SQL queries | Low |
| `platform/tableau-next` | Tableau analytics / semantic models | Low |

**Recommended starting point:** `platform/sobject-reads` — read-only, safe for initial testing.

**Tools exposed by `platform/sobject-reads`:**

| Tool | What it does |
|---|---|
| `soqlQuery` | Execute arbitrary SOQL — primary way to read data; supports relationships, subqueries, aggregates |
| `getObjectSchema` | Describe SObject fields — call with no args to index all queryable objects, then pass object name for field details |
| `find` | SOSL-based cross-object text search |
| `getRelatedRecords` | Traverse parent/child relationships from a record |
| `listRecentSobjectRecords` | Return recently viewed records for an object |
| `getUserInfo` | Return the currently authenticated Salesforce user's details |

**Security guidance:**
- Prefer scoped servers (`sobject-reads`, `sobject-mutations`) over `sobject-all`
- All servers enforce per-user FLS, object permissions, and sharing rules automatically
- The ECA consumer key acts as the client-level gate; Salesforce permissions act as the data-level gate

---

## Step 4 — Connect the MCP Client

Construct the server URL from the org type and chosen server name:

```
Production:      https://api.salesforce.com/platform/mcp/v1/<SERVER-NAME>
Sandbox/scratch: https://api.salesforce.com/platform/mcp/v1/sandbox/<SERVER-NAME>
```

Example for `sobject-reads` on a scratch org:
```
https://api.salesforce.com/platform/mcp/v1/sandbox/platform/sobject-reads
```

### Claude Code CLI

Claude Code's native `"type": "sse"` connector works with Salesforce Hosted MCP. The `callbackPort` in the config controls which localhost port Claude Code uses for the OAuth redirect.

**Required ECA callback URL:** `http://localhost:8080/callback`

Add to your Claude Code MCP config file (`.mcp.json` in project root or `~/.claude/mcp.json` globally):

```json
{
  "mcpServers": {
    "salesforce-sobject-reads": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote@latest",
        "https://api.salesforce.com/platform/mcp/v1/platform/sobject-reads",
        "8080",
        "--static-oauth-client-info",
        "{\"client_id\":\"<CONSUMER-KEY>\",\"client_secret\":\"<CONSUMER-SECRET>\"}"
      ]
    }
  }
}
```

For sandbox/scratch orgs, change the URL to `https://api.salesforce.com/platform/mcp/v1/sandbox/platform/sobject-reads`.

If port 8080 is already in use by another MCP server, increment to 8081 and update both the `mcp-remote` args and the ECA callback URL to match.

**Checking connection status in Claude Code:** Run `/mcp` in the Claude Code prompt to see which servers are connected and reconnect any that have dropped. If a server shows as disconnected, `/mcp` will reconnect it without requiring a full restart.

### Cursor

Cursor has built-in OAuth support for Salesforce Hosted MCP using the `auth` block with `CLIENT_ID`. No `mcp-remote` or extra callback URL configuration is needed.

1. Open **Cursor Settings** → **Tools & MCP** → **New MCP Server**
2. Cursor creates or opens `~/.cursor/mcp.json`
3. Add your server entries using this format. **Critical: the key must be `CLIENT_ID` (all caps) — `clientId` (camelCase) is silently ignored and the OAuth flow will not start.**

Production org:

```json
{
  "mcpServers": {
    "salesforce-sobject-reads": {
      "url": "https://api.salesforce.com/platform/mcp/v1/platform/sobject-reads",
      "auth": {
        "CLIENT_ID": "YOUR_CONSUMER_KEY_HERE"
      }
    }
  }
}
```

Sandbox or scratch org (add `sandbox/` after `v1/`):

```json
{
  "mcpServers": {
    "salesforce-sobject-reads": {
      "url": "https://api.salesforce.com/platform/mcp/v1/sandbox/platform/sobject-reads",
      "auth": {
        "CLIENT_ID": "YOUR_CONSUMER_KEY_HERE"
      }
    }
  }
}
```

4. Replace `YOUR_CONSUMER_KEY_HERE` with the Consumer Key from Setup (ECA → Settings → OAuth Settings), then save.
5. Cursor automatically opens a browser for the OAuth login. **Do NOT click "Connect" in Cursor settings while the browser is already open** — doing so regenerates the PKCE verifier mid-flow and causes `invalid code verifier`. See multi-server authentication below.

#### Authenticating multiple servers that share the same `CLIENT_ID`

When two or more entries share the same `CLIENT_ID`, Cursor starts an OAuth flow automatically at startup (opening a browser in the background) AND again when you click "Connect." If you click Connect while the auto-triggered browser is in flight, Cursor regenerates a new PKCE verifier but the browser returns an auth code tied to the old verifier — causing `Failed to complete OAuth exchange invalid code verifier`.

**Safe procedure:**

1. Start with only ONE server entry in `mcp.json` and authenticate it first
2. Fully quit and reopen Cursor — the browser opens automatically in the background; check your taskbar or browser tabs for the Salesforce login page
3. Complete the login **without clicking Connect in Cursor**
4. Once that server shows as connected, add the next server entry and repeat from step 2

**If you need to add a server to an already-running setup:**

1. Temporarily remove the already-authenticated server entry from `mcp.json`
2. Fully quit and reopen Cursor
3. Complete the login for the new server in the auto-opened browser (do not click Connect)
4. Re-add the previously removed server — it reconnects silently using its cached token

### Postman (recommended for testing first)

Use Postman to verify the connection before configuring full AI clients:
- Authorization type: **OAuth 2.0**
- Auth URL: `https://<your-instance>.my.salesforce.com/services/oauth2/authorize`
- Token URL: `https://<your-instance>.my.salesforce.com/services/oauth2/token`
- Client ID: consumer key from Step 2
- Scope: `mcp_api refresh_token`
- PKCE: enabled
- Callback URL: `https://oauth.pstmn.io/v1/callback`

---

## Step 5 — Test the Connection

After connecting, test with a simple prompt in Claude or Cursor:

```
List the first 5 Account records in my Salesforce org.
```

Or for `sobject-reads`:
```
What SObject types are available to query?
```

If authentication fails:
- **"JWT Token is required"**: one of three causes: (a) `isNamedUserJwtEnabled` is `false` on the ECA — set it to `true` in `ExtlClntAppGlobalOauthSettings` and redeploy; (b) the `mcp.json` `auth` key is `clientId` (camelCase) instead of `CLIENT_ID` (all caps) — fix the casing; (c) the OAuth flow completed but the wrong PKCE verifier was used — follow the multi-server authentication procedure in Step 4
- **"Failed to complete OAuth exchange invalid code verifier"**: you clicked "Connect" while the auto-triggered browser OAuth was already in progress — follow the multi-server authentication procedure in Step 4
- Confirm the consumer key is correct
- Confirm PKCE is enabled in the ECA (`isPkceRequired = true`)
- Confirm `MCP` and `RefreshToken` scopes are present
- Wait 30 minutes if the ECA was just created
- Verify the org type (production vs sandbox) matches the URL

---

## Output Format

When finishing, report in this order:

```text
Org:         <alias> (<instanceUrl>)
ECA name:    <AppName>
ECA detail:  https://<instance>.my.salesforce-setup.com/lightning/setup/ManageExternalClientApplication/<15-CHAR-ECA-ID>/detail
             → Settings → OAuth Settings → copy Consumer Key → paste into mcp.json
Servers:     <list of server names configured>
Activate:    https://<instance>.my.salesforce-setup.com/lightning/setup/McpServer/<SERVER-ID>/view?setup__source=PLATFORM_STANDARD_MCP_SERVER  (one line per server)
Client:      Claude Code | Cursor | Postman
Config file: <path to mcp.json if written, with YOUR_CONSUMER_KEY_HERE placeholder>
Next step:   1. Copy Consumer Key from ECA Setup page
             2. Paste into mcp.json replacing YOUR_CONSUMER_KEY_HERE
             3. Activate each server in Setup
             4. Reload client and authenticate
```

---

## Cross-Skill Integration

| Need | Delegate to | Reason |
|---|---|---|
| Non-MCP ECA / Connected App OAuth | [sf-connected-apps](../sf-connected-apps/SKILL.md) | full OAuth app design |
| Deploy metadata to org | [sf-deploy](../sf-deploy/SKILL.md) | deployment validation |
| Custom Apex @InvocableMethod for MCP tools | [sf-apex](../sf-apex/SKILL.md) | Apex implementation |
| Salesforce docs lookup | [sf-docs](../sf-docs/SKILL.md) | authoritative doc retrieval |

---

## Known Limitations

- **Scratch orgs**: ECA cannot be created via Setup UI — must use DevHub org + 2GP package + install
- **Activation delay**: Up to 30 minutes after ECA creation before MCP connections succeed
- **Multiple servers sharing one `CLIENT_ID` in Cursor**: Cursor has an OAuth race condition when two or more servers in `mcp.json` share the same `CLIENT_ID`. Authenticate servers one at a time using the procedure in Step 4 — do not click "Connect" while the auto-triggered browser is already open
- **No `claude_desktop_config.json` for Salesforce MCP**: Claude's native connector UI is the recommended path; the JSON config approach uses `.mcp.json`
- **No Setup toggle for individual servers**: Server availability is governed by the platform; you select servers in the client config, not in Salesforce Setup
