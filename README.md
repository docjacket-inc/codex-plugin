<p align="left">
  <img src="assets/logo-mark-128.png" alt="DocJacket" width="96" height="96" />
</p>

# DocJacket — Transaction Coordination plugin for Codex

[![Plugin](https://img.shields.io/badge/Codex-plugin-purple)]()
[![Version](https://img.shields.io/badge/version-0.6.2-green)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey)]()

Connect DocJacket transactions, tasks, deadlines, contacts, and document checklists to OpenAI Codex. Triage your pipeline, draft and send follow-up emails from your connected Gmail, classify inbound attachments against the right deal — all from inside Codex.

**Works with:** Codex CLI · Codex for Work · ChatGPT (with MCP Connectors enabled)

## What you get

- **Full read + send tool suite** via the DocJacket MCP server at `https://mcp.docjacket.com/mcp`. Call `mcp_catalog` after install for the live inventory + per-tool gotchas and example calls.
- **`mcp_health_check`** — first call when integrating; verifies auth, scopes, and DB reachability.
- **8 skills**: `daily-triage`, `execution-workflow`, `tc-context`, `email-triage`, `document-filing`, `follow-up-drafting`, `closing-prep`, `contract-intake`. Each ships as a directory under `skills/<name>/SKILL.md`.

Three scope tiers — Read (search, summarize), Draft (compose without sending), Actions (send emails, create tasks, update dates). You authorize once at OAuth consent; the chat is the per-call approval gate.

## Tool catalog

Selected highlights — for the **live and complete** list with per-tool gotchas, pairing hints, and example calls, run `mcp_catalog` after install.

**Diagnostics + introspection**: `mcp_health_check`, `mcp_catalog`.

**Reads**: `search_transactions`, `get_transaction`, `get_key_dates`, `get_open_tasks`, `get_contacts`, `get_upcoming_key_dates`, `list_open_contingencies`, **`get_next_required_actions`** (bundled-judgment flagship), `find_transaction_by_property`, `get_missing_documents`, `list_active_transactions`, `search_contacts`, `get_contact`, `list_email_templates`, `get_email_template`.

**Direct execution**: `send_client_update`, `send_agent_followup`, `send_document_request`, `send_email_to_agent`, `create_tasks`, `update_key_date`, `complete_task`, `save_status_summary`, `render_email_template`, `add_contact_to_transaction`, `create_contact`.

## Install

**No bearer tokens. No manual config.** Paste the server URL, click Allow on the consent screen, you're done. OAuth 2.1 + Dynamic Client Registration end-to-end.

### Prerequisites

A DocJacket account on the **Pro plan**. Connecting is free; loading tools requires Pro.

### Codex CLI / Codex for Work

```bash
codex plugin marketplace add docjacket-inc/codex-plugin
codex plugin install docjacket
```

On first tool call, your browser opens to complete OAuth consent. Codex stores access + refresh tokens in your local keychain. No secrets in config files.

### ChatGPT (with MCP Connectors enabled — Plus / Team / Enterprise)

1. **Settings** → **Connectors** → **+ Add MCP server**
2. Paste: `https://mcp.docjacket.com/mcp`
3. Click **Continue**, then **Allow** on the DocJacket consent screen

The tool list loads. Run `mcp_health_check` once to confirm scopes; then `mcp_catalog` for the full inventory.

### Verify

```bash
codex chat
> Use DocJacket to summarize my active transactions.
```

The MCP icon should show `docjacket` connected. Run `mcp_catalog` for the current tool count.

## Starter prompts

| Prompt | What happens |
|---|---|
| `Use DocJacket to find transactions that need attention today.` | Daily Triage skill — grouped overdue / due-72h / due-2w list |
| `Use DocJacket to summarize my active transactions.` | `search_transactions(status: Active)` |
| `Use DocJacket to list deadlines this week.` | `get_upcoming_key_dates` |
| `Use DocJacket to show open tasks on the 1234 Main St deal.` | `find_transaction_by_property` → `get_open_tasks` |
| `Use DocJacket to look up the listing agent on 5678 Oak Ave.` | `find_transaction_by_property` → `get_transaction` (inline `listingAgent` block) |

## How attribution + revocation work

Every call carries `X-DocJacket-Source-App: codex` + `X-DocJacket-Plugin-Version: 0.6.1`. Audit in [Activity Log](https://app.docjacket.com/settings/ai-access/activity). Revoke any connected OAuth client from `/settings/ai-access` without affecting other AI assistants.

## Optional connectors

See [`CONNECTORS.md`](CONNECTORS.md) — Gmail / Calendar / Drive / Slack compose with DocJacket via Codex's existing connector layer.

## Compliance + disclaimer

This plugin does NOT provide legal advice. See [`DISCLAIMER.md`](DISCLAIMER.md) for the full scope and limitations.

## Version

`0.6.1` (2026-05-20) — Version parity bump alongside the cowork plugin (docjacket-v3 PR #565). No codex content changes — the slash-commands surface added in this release is Claude-plugin-only; Codex skills are invoked via `$docjacket-*` patterns in CLI or plain English in ChatGPT.

`0.6.0` (2026-05-19) — Catches up to the v0.6 internal source: 7 skills (was 1), inbox-workflow tool set including Draft + Actions scopes. New diagnostic tools `mcp_health_check` and `mcp_catalog`. Every tool response now includes `structuredContent` + opt-in `breadcrumbs` per MCP spec 2025-06-18 §Tool Result. Tracks DocJacket MCP server PRs #549–#551.

`0.3.0` (2026-05-18) — OAuth 2.1 + Dynamic Client Registration. Paste-URL-and-go install — no bearer tokens, no manual config. Tracks DocJacket MCP server PR #494 (HTTP 401 + WWW-Authenticate + `initialize` handshake).

`0.2.0` (2026-05-18) — 10 read tools; Daily Triage collapsed to single `get_next_required_actions` call.
`0.1.0` (2026-05-17) — Initial release: 5 read tools.

## Read more on help.docjacket.com

- **[Connect Codex / ChatGPT](https://help.docjacket.com/docs/mcp/codex)** — install walkthrough
- **[Contract Intake](https://help.docjacket.com/docs/mcp/contract-intake)** — drop a PDF in chat, get a fully-set-up transaction back
- **[Tool Catalog (`mcp_catalog`)](https://help.docjacket.com/docs/mcp/mcp-catalog)** — what `mcp_catalog` returns + how to read it
- **[AI Access overview](https://help.docjacket.com/docs/ai-access)** — the umbrella feature, OAuth model, scope tiers, audit

## Support

- Issues: <https://github.com/docjacket-inc/codex-plugin/issues>
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
