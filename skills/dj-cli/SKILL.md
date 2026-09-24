<!--
  Generated from plugins/shared/skills/dj-cli.md — do not hand-edit.
  Update the canonical file and re-sync. See plugins/README.md for the rule.
  Generator: hand-curated v1 (per spec §3.3). Last sync: 2026-09-24.
-->

---
name: dj-cli
title: Use DocJacket from a shell with the dj CLI
purpose: Read DocJacket transactions, deadlines, tasks, documents and contacts through the `dj` command-line tool when the agent has a shell. Chat clients without a shell use the DocJacket MCP tools instead.
applicable_to: [claude-code, codex, cowork]
tools_used: [dj]
expected_output: Answers grounded in dj's JSON output, with transaction ids carried between commands.
required_scopes: [read]
version: 0.1.1
---

# DocJacket via the `dj` CLI

`dj` reads DocJacket through its public API. It is **read-only**: it cannot send, change, or delete anything. **Use it only when you can run shell commands and `dj --version` works; otherwise use the DocJacket MCP tools** (for example `find_transaction_by_property`, `get_next_required_actions`). Setup: `curl -fsSL https://cdn.docjacket.com/dj/install.sh | sh`, then `export DJ_API_KEY=<key from Settings › API & AI Access>`. Check with `dj auth status`.

## The core loop

```bash
dj transactions resolve "412 Oak"            # a property the user mentioned → one transaction id
dj transactions get <id>                     # status, parties, price, key dates
dj next-actions --transaction <id>           # what needs attention, ranked by DocJacket
```

- **Ambiguous resolve (exit 7):** show the user the `candidates` and ask which one. Never pick one yourself.
- **No match (exit 6):** resolve only knows addresses. For a person's name, use `dj transactions list --query "Johnson"`.
- **Ranking:** trust `next-actions`' ranking and urgency tiers; do not re-rank.

## Other reads

```bash
dj transactions list --status active --limit 10      # --status is case-insensitive; `dj transactions statuses` lists names
dj transactions missing-docs <id>                     # required documents not yet on file
dj transactions contingencies <id>
dj transactions communications <id> --channel email --limit 5
dj key-dates list --transaction <id>
dj key-dates upcoming --days 7
dj tasks list --transaction <id>                      # every open task plus checklist counts
dj documents list --transaction <id>
dj documents url <document-id>                        # signed link, ~1 hour; never paste it anywhere public
dj contacts search "Jerry Mitchell"
dj contacts by-email jerry@example.com
dj contacts get <contact-id>
```

Every command has `--help`. `--full` (where offered) prints the raw API response; prefer the compact default.

## Reading results

- **stdout:** one JSON document per command.
- **stderr, on failure:** `{"error":{"code","message","hint"}}`. Follow the `hint`.
- **Exit codes:**

| Code | Meaning |
|---|---|
| 0 | ok |
| 2 | bad arguments or config |
| 3 | invalid input |
| 4 | DJ_API_KEY missing or rejected |
| 5 | key lacks the scope |
| 6 | not found |
| 7 | ambiguous; ask the user |
| 8 | rate limited; wait `retryAfterSeconds` |
| 9 | server error |
| 10 | network error |

- **An unknown transaction id is exit 6**, never an empty "nothing to do".
- **Paging:** lists say `hasMore` / `truncated`. Page or raise `--limit` only if the user needs more.
