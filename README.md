# DocJacket — Transaction Coordination plugin for Codex

[![Plugin](https://img.shields.io/badge/Codex-plugin-purple)]()
[![Version](https://img.shields.io/badge/version-0.2.0-green)]()
[![License](https://img.shields.io/badge/license-MIT-lightgrey)]()

Connect DocJacket transactions, tasks, deadlines, contacts, and document checklists to OpenAI Codex. Triage your pipeline, brief on any active deal, and check missing documents — all from inside Codex.

**Works with:** Codex CLI · Codex for Work · ChatGPT (when Codex plugins reach the official directory)

## What you get

- **10 read tools** exposed via the DocJacket MCP server at `https://mcp.docjacket.com/mcp`
- **Daily Triage** skill — `$docjacket-daily-triage` returns every transaction needing attention, pre-ranked

Read-only. No writes, no autonomous actions. Drafting tools (`prepare_*` / `propose_*`) ship in **v0.3** once the upstream MCP server reaches Phase 8.

## Tool catalog

`search_transactions`, `get_transaction`, `get_key_dates`, `get_open_tasks`, `get_contacts`, `get_upcoming_key_dates`, `list_open_contingencies`, **`get_next_required_actions`** (bundled-judgment flagship), `find_transaction_by_property`, `get_missing_documents`.

## Install

### Prerequisites

1. A DocJacket account with Owner or Admin role.
2. A bearer token from [`app.docjacket.com/settings/ai-access`](https://app.docjacket.com/settings/ai-access) — mint a new one, label it "Codex Plugin".

### Steps

```bash
codex plugin marketplace add DocJacket-LLC/codex-plugin
codex plugin install docjacket
codex secret set docjacket.token <paste-your-bearer-token>
```

Verify:

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

Every call carries `X-DocJacket-Source-App: codex` + `X-DocJacket-Plugin-Version: 0.2.0`. Audit in [Activity Log](https://app.docjacket.com/settings/ai-access/activity).

## Optional connectors

See [`CONNECTORS.md`](CONNECTORS.md) — Gmail / Calendar / Drive / Slack compose with DocJacket via Codex's existing connector layer.

## Compliance + disclaimer

Read-only. See [`DISCLAIMER.md`](DISCLAIMER.md) for the full scope and limitations.

## Version

`0.2.0` (2026-05-18) — 10 read tools; Daily Triage collapsed to single `get_next_required_actions` call.
`0.1.0` (2026-05-17) — Initial release: 5 read tools.

## Support

- Docs: <https://docs.docjacket.com/mcp/codex>
- Issues: <https://github.com/DocJacket-LLC/codex-plugin/issues>
- Email: support@docjacket.com
