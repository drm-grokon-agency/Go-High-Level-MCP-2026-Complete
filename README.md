# GoHighLevel MCP Server

Model Context Protocol server for GoHighLevel. Exposes GHL API operations as MCP tools over stdio, Streamable HTTP, and legacy SSE.

## Current API Coverage

- Official GHL endpoints parsed: `576`
- Official endpoint coverage: `576 / 576`
- Generated official endpoint tools: `238`
- MCP tools in registry: `834` (`802` raw API tools plus `32` curated agent workflow tools)

Coverage artifacts:

- `docs/GHL-API-COVERAGE-REPORT.md`
- `docs/GHL-LOCAL-ENDPOINT-CLASSIFICATION.md`
- `docs/ghl-api-coverage.json`
- `docs/API-DASHBOARD.md`
- `docs/tool-inventory.json`

---

## MCP Config Setup Guide

This section walks through setting up your Claude Desktop (or Claude Code) config correctly. Read this before running anything.

### Step 1 — Clone and build

```bash
git clone https://github.com/drm-grokon-agency/Go-High-Level-MCP-2026-Complete.git
cd Go-High-Level-MCP-2026-Complete
npm install
npm run build
```

### Step 2 — Get your GHL credentials

You need a **Private Integration Token (PIT)** from GoHighLevel for each sub-account you want to connect.

In GHL: **Settings → Integrations → Private Integrations → Create**

- Give it a name (e.g. "Claude MCP")
- Enable **all scopes** (no scope limitations)
- Copy the generated token — this is your `GHL_API_KEY`

Also grab your **Location ID**:
- In GHL: **Settings → Business Profile** — the Location ID is in the URL or displayed on that page

And your **Company ID**:
- In GHL: open any sub-account → check the URL for the `companyId` parameter, or ask your agency admin
- The Company ID is required for user search tools (`get_users`, `filter_users_by_email`, `search_users`) to work correctly

### Step 3 — Configure Claude Desktop

Open your Claude Desktop config file:
- **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`
- **Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`

Add an entry under `mcpServers` for each GHL sub-account you want to connect. Use a distinct name for each (e.g. `hl-gom`, `hl-gor`):

```json
{
  "mcpServers": {
    "hl-your-account-name": {
      "command": "node",
      "args": ["C:/path/to/Go-High-Level-MCP-2026-Complete/dist/server.js"],
      "env": {
        "GHL_API_KEY": "pit-your-private-integration-token",
        "GHL_LOCATION_ID": "your-location-id",
        "GHL_COMPANY_ID": "your-company-id",
        "GHL_BASE_URL": "https://services.leadconnectorhq.com",
        "GHL_API_VERSION": "2023-02-21"
      }
    }
  }
}
```

> **Critical:** Use `GHL_API_VERSION=2023-02-21` — not `2021-07-28`. The older version causes many tools to silently fail or return empty results.

> **Windows paths:** Use forward slashes or escaped backslashes in the `args` path (e.g. `C:/Users/YourName/...` or `C:\\Users\\YourName\\...`).

### Step 4 — Restart Claude Desktop

Fully quit and relaunch Claude Desktop after any config change. The MCP server does not hot-reload.

### Step 5 — Verify

Ask Claude: *"List all workflows for location [your-location-id]"* using `ghl_list_workflows`. If you get results back, everything is working.

---

### Multiple sub-accounts

Add one block per sub-account with a unique server name:

```json
{
  "mcpServers": {
    "hl-account-one": {
      "command": "node",
      "args": ["C:/path/to/dist/server.js"],
      "env": {
        "GHL_API_KEY": "pit-token-for-account-one",
        "GHL_LOCATION_ID": "location-id-one",
        "GHL_COMPANY_ID": "company-id",
        "GHL_BASE_URL": "https://services.leadconnectorhq.com",
        "GHL_API_VERSION": "2023-02-21"
      }
    },
    "hl-account-two": {
      "command": "node",
      "args": ["C:/path/to/dist/server.js"],
      "env": {
        "GHL_API_KEY": "pit-token-for-account-two",
        "GHL_LOCATION_ID": "location-id-two",
        "GHL_COMPANY_ID": "company-id",
        "GHL_BASE_URL": "https://services.leadconnectorhq.com",
        "GHL_API_VERSION": "2023-02-21"
      }
    }
  }
}
```

Each entry is a completely independent MCP server instance pointing to the same `dist/server.js` binary but with different credentials.

---

### Known API Behaviors

These are confirmed behaviors discovered through live testing against the GHL API — not assumptions:

| Tool | Behavior |
|------|----------|
| `ghl_list_workflows` | GHL API only accepts `locationId` — any other query param returns empty. `status`, `limit`, and `skip` filters are applied client-side after fetching all workflows. |
| `get_users` / `filter_users_by_email` | Requires both `locationId` AND `GHL_COMPANY_ID` in config. Without `GHL_COMPANY_ID`, user search returns zero results. |
| `search_contacts` | May not be pre-loaded in Claude Desktop sessions with large MCP servers. If the tool appears unresponsive, use `tool_search` to load it first. |

---

### Troubleshooting

**Tools returning empty results or 422 errors**
- Verify `GHL_API_VERSION` is `2023-02-21`
- Restart Claude Desktop after any config change

**User search returning zero results**
- Ensure `GHL_COMPANY_ID` is set in your config for that server entry

**A tool says "not loaded"**
- This MCP exposes 800+ tools; Claude Desktop loads a subset per session
- Use `tool_search` with the tool name to load it on demand, then retry

**Build errors after git pull**
- Run `npm install` first, then `npm run build`
- Node 18+ required

---

## Companion Tooling

```bash
npm run tools:doctor       # Check build output, env, and API coverage health
npm run tools:list         # Browse the registered MCP tool inventory
npm run tools:report       # Regenerate the API dashboard and tool inventory JSON
npm run tools:configure    # Print a Claude-compatible MCP config snippet
npm run tools:update-api   # Refresh official GHL API coverage and generated tools
```

Direct CLI usage:

```bash
npx ghl-mcp doctor
npx ghl-mcp list-tools --search ads
npx ghl-mcp configure claude
npx ghl-mcp test-tool search_contacts '{"locationId":"your_location_id","pageLimit":1}'
```

---

## Agent Tool Profiles

```bash
GHL_TOOL_PROFILE=curated npm run start:stdio
```

- `full` — default; all `834` tools
- `curated` — `32` high-level agent workspace tools only
- `raw` — `802` raw endpoint tools only

---

## Transports

- `npm run start:stdio` — stdio for desktop MCP clients
- `npm run start:http` — Streamable HTTP at `/mcp`
- `npm run start:legacy` — legacy SSE at `/sse`

---

## Project Layout

```text
src/
  clients/       GHL API clients
  tools/         MCP tool modules
  types/         shared TypeScript types
  main.ts        Streamable HTTP MCP server
  server.ts      stdio MCP server
  http-server.ts legacy SSE MCP server
scripts/         API scanner, generator, build, smoke test
docs/            generated API coverage reports
examples/        MCP recipes and starter agent templates
mcp-apps/        companion MCP Apps server and bundled UI
tests/           Jest tests
```

## Scripts

```bash
npm run build              # Compile server files to dist/
npm run lint               # TypeScript syntax check
npm test                   # Jest tests
npm run scan:ghl-api       # Refresh official GHL API coverage
npm run tools:doctor       # Check local MCP setup
npm run tools:report       # Generate API dashboard and tool inventory
```

## Notes

- `src/tools/official-spec-tools.ts` and `src/tools/official-spec-endpoints.json` are generated. Do not edit by hand.
- Run `npm run scan:ghl-api` after GHL API docs change.
- The live smoke test is opt-in and only runs when `GHL_API_KEY` and `GHL_LOCATION_ID` are present.