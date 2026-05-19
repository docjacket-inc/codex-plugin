<p align="left">
  <img src="assets/logo-mark-128.png" alt="DocJacket" width="96" height="96" />
</p>

# DocJacket — Transaction Coordination plugin for Codex

[![Plugin](https://img.shields.io/badge/Codex-plugin-purple)]()
[![Version](https://img.shields.io/badge/version-0.3.0-green)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey)]()

Connect DocJacket transactions, tasks, deadlines, contacts, and document checklists to OpenAI Codex. Triage your pipeline, brief on any active deal, and check missing documents — all from inside Codex.

**Works with:** Codex CLI · Codex for Work · ChatGPT (when Codex plugins reach the official directory)

## What you get

- **10 read tools** exposed via the DocJacket MCP server at `https://mcp.docjacket.com/mcp`
- **Daily Triage** skill — `$docjacket-daily-triage` returns every transaction needing attention, pre-ranked

Read-only. No writes, no autonomous actions. Drafting tools (`prepare_*` / `propose_*`) ship in **v0.4** once the upstream MCP server reaches Phase 8.

## Tool catalog

`search_transactions`, `get_transaction`, `get_key_dates`, `get_open_tasks`, `get_contacts`, `get_upcoming_key_dates`, `list_open_contingencies`, **`get_next_required_actions`** (bundled-judgment flagship), `find_transaction_by_property`, `get_missing_documents`.

## Install

**No bearer tokens. No manual config.** Paste the server URL, click Allow on the consent screen, you're done. v0.3.0 uses OAuth 2.1 + Dynamic Client Registration end-to-end.

### Prerequisites

A DocJacket account on the **Pro plan**. Connecting is free; loading tools requires Pro.

### Codex CLI / Codex for Work

```bash
codex plugin marketplace add DocJacket-LLC/codex-plugin
codex plugin install docjacket
```

On first tool call, your browser opens to complete OAuth consent. Codex stores access + refresh tokens in your local keychain. No secrets in config files.

### ChatGPT (with MCP Connectors enabled — Plus / Team / Enterprise)

1. **Settings** → **Connectors** → **+ Add MCP server**
2. Paste: `https://mcp.docjacket.com/mcp`
3. Click **Continue**, then **Allow** on the DocJacket consent screen

10 tools load. You're done.

### Verify

```bash
codex chat
> Use DocJacket to summarize my active transactions.
```

The MCP icon should show `docjacket` with 10 tools.

## Starter prompts

| Prompt | What happens |
|---|---|
| `Use DocJacket to find transactions that need attention today.` | Daily Triage skill — grouped overdue / due-72h / due-2w list |
| `Use DocJacket to summarize my active transactions.` | `search_transactions(status: Active)` |
| `Use DocJacket to list deadlines this week.` | `get_upcoming_key_dates` |
| `Use DocJacket to show open tasks on the 1234 Main St deal.` | `find_transaction_by_property` → `get_open_tasks` |
| `Use DocJacket to look up the listing agent on 5678 Oak Ave.` | `find_transaction_by_property` → `get_transaction` (inline `listingAgent` block) |

## How attribution + revocation work

Every call carries `X-DocJacket-Source-App: codex` + `X-DocJacket-Plugin-Version: 0.3.0`. Audit in [Activity Log](https://app.docjacket.com/settings/ai-access/activity). Revoke any connected OAuth client from `/settings/ai-access` without affecting other AI assistants.

## Optional connectors

See [`CONNECTORS.md`](CONNECTORS.md) — Gmail / Calendar / Drive / Slack compose with DocJacket via Codex's existing connector layer.

## Compliance + disclaimer

Read-only. See [`DISCLAIMER.md`](DISCLAIMER.md) for the full scope and limitations.

## Version

`0.3.0` (2026-05-18) — OAuth 2.1 + Dynamic Client Registration. Paste-URL-and-go install — no bearer tokens, no manual config. Tracks DocJacket MCP server PR #494 (HTTP 401 + WWW-Authenticate + `initialize` handshake).

`0.2.0` (2026-05-18) — 10 read tools; Daily Triage collapsed to single `get_next_required_actions` call.
`0.1.0` (2026-05-17) — Initial release: 5 read tools.

## Support

- Docs: <https://help.docjacket.com/docs/mcp/codex>
- Issues: <https://github.com/DocJacket-LLC/codex-plugin/issues>
- Email: support@docjacket.com

## Brand assets

| File | Size | Use |
|---|---|---|
| [`assets/logo-mark.svg`](assets/logo-mark.svg) | vector | source / scalable |
| [`assets/logo-mark-32.png`](assets/logo-mark-32.png) | 32×32 | favicon, dense list rows |
| [`assets/logo-mark-64.png`](assets/logo-mark-64.png) | 64×64 | small directory cards |
| [`assets/logo-mark-128.png`](assets/logo-mark-128.png) | 128×128 | plugin manifest icon |
| [`assets/logo-mark-256.png`](assets/logo-mark-256.png) | 256×256 | medium cards |
| [`assets/logo-mark-512.png`](assets/logo-mark-512.png) | 512×512 | marketplace tile |
| [`assets/logo-mark-1024.png`](assets/logo-mark-1024.png) | 1024×1024 | hero / future-proof |
