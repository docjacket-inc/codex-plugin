<!--
  Generated from plugins/shared/skills/daily-triage.md — do not hand-edit.
  Update the canonical file and re-sync. See plugins/README.md for the rule.
  Generator: hand-curated v1 (per spec §3.3). Last sync: 2026-05-18.
-->

---
name: daily-triage
title: Daily Triage
purpose: Find every transaction that needs attention today and surface the next action per deal, grouped by urgency.
tools_used:
  - docjacket.get_next_required_actions
required_scopes: [read]
version: 0.2.0
---

# Daily Triage

## When to invoke

Trigger on any variation of:

- "what needs my attention today?"
- "give me a TC briefing"
- "what's overdue?"
- "what's at risk this week?"
- "morning briefing" / "morning triage"
- "what should I work on first?"

## Workflow

**One call. The tool returns a pre-ranked, pre-bucketed feed.**

1. Call `docjacket.get_next_required_actions` with `limit: 75`. (Optional: pass `transactionId` to scope to a single deal.)
2. The response is already sorted overdue-first and tagged with `urgency` per row — group by the `urgency` field as-is.
3. (Once `docjacket.list_prepared_work` ships, add a second call to surface pending-approval items as a 4th bucket. Skip for now.)

**Do not fan out per transaction** — the previous v0.1 workflow called `search_transactions` then `get_open_tasks` + `get_key_dates` per active deal then sorted client-side. That's been replaced by this single call, and re-introducing the old pattern wastes 50+ tool calls per triage on a busy org.

## Output shape

```
🔥 Overdue (3)
  • 1234 Main St — Inspection Deadline was due 2 days ago.
  • 5678 Oak Ave — Task "Order title" is 5 days overdue.
  • 9012 Pine Rd — Financing Deadline was due 1 day ago.

⏰ Due within 72 hours (5)
  • 2468 Elm — Closing Date due in 2 days.
  • ...

📅 Due within 2 weeks (12)
  • ...
```

Use the `urgency` field to decide the section header:

| `urgency` | Section header | Emoji |
|---|---|---|
| `overdue` | Overdue | 🔥 |
| `due_72h` | Due within 72 hours | ⏰ |
| `due_2w`  | Due within 2 weeks | 📅 |
| `due_4w`  | Due within 4 weeks | 📋 |

Render the `rationale` string verbatim — it's already phrased for human reading. Prefix each row with `transactionAddress` (or "—" if null) followed by an em-dash.

If `summary.totalCount === 0`, render: `✅ All clear — no items need attention right now.`

For each non-empty bucket, include the count from `summary` (e.g. `🔥 Overdue (3)`).

## Anti-patterns (do not do)

- **Don't** call `docjacket.search_transactions` + `docjacket.get_open_tasks` + `docjacket.get_key_dates` separately and re-sort. That was the v0.1 workflow; `get_next_required_actions` does it server-side now in one shot.
- **Don't** call `docjacket.get_transaction` to look up the address — `transactionAddress` is already on each row.
- **Don't** call `docjacket.get_next_required_actions` per transaction in a loop. One org-wide call covers everything; the `transactionId` arg is only for the rare "scope to this deal" case.
- **Don't** summarize what was already completed or approved.
- **Don't** propose any writes from this skill — triage is read-only.
- **Don't** flatten the buckets into one mixed list.

## Skill version

`v0.2.0` — collapsed from 4-call workflow to single `get_next_required_actions` call.
