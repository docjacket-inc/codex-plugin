---
name: daily-triage
description: Finds every DocJacket transaction that needs attention today and surfaces the next action per deal, as a bulleted list grouped by the urgency buckets the tool returns, most urgent first. Use when the TC asks what needs attention today, for a morning briefing, or for a triage of their active deals.
license: MIT
---

<!--
docjacket-title: Daily Triage
docjacket-tools: get_next_required_actions
docjacket-scopes: read
docjacket-version: 0.2.1
-->

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

1. Call `get_next_required_actions` with `limit: 75`. (Optional: pass `transactionId` to scope to a single deal.)
2. The response is already sorted overdue-first and tagged with `urgency` per row — group by the `urgency` field as-is.

That's it. **Do not fan out per transaction** — the previous v0.1 workflow called `search_transactions` then `get_open_tasks` + `get_key_dates` per active deal then sorted client-side. That's been replaced by this single call, and re-introducing the old pattern wastes 50+ tool calls per triage on a busy org.

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

Render the `rationale` string verbatim — it's already phrased for human reading. Prefix each row with the `transactionAddress` (or "—" if null) followed by an em-dash.

If `summary.totalCount === 0`, render: `✅ All clear — no items need attention right now.`

For each non-empty bucket, the section header should include the count from `summary` (e.g. `🔥 Overdue (3)`).

The `summary` counts cover every qualifying item, not only the rows in `actions`. When `summary.truncated` is true, say the list shows the first `summary.returned` of `summary.totalCount` items (and "at least" when `summary.countsAreLowerBound` is true).

## Anti-patterns (do not do)

- **Don't** call `search_transactions` + `get_open_tasks` + `get_key_dates` separately and re-sort. That was the v0.1 workflow; `get_next_required_actions` does it server-side now in one shot.
- **Don't** call `get_transaction` to look up the address — `transactionAddress` is already on each row.
- **Don't** call `get_next_required_actions` per transaction in a loop. One org-wide call covers everything; the `transactionId` arg is only for the rare "scope to this deal" case.
- **Don't** summarize what was already completed or approved. Triage is about what still needs a decision.
- **Don't** propose any writes from this skill. Triage is read-only. If the user follows up with "draft the emails" or "send these", that's a separate skill invocation (Phase 8+).
- **Don't** flatten the buckets into one mixed list — the urgency grouping is the whole point.

## Example invocation trace

User: _"What needs my attention today?"_

```
[1] get_next_required_actions(limit: 75)
    → 26 actions returned (1 overdue, 5 due_72h, 12 due_2w, 8 due_4w)
[2] Group by urgency.field; render grouped list per the Output shape table above.
```

Total: 1 tool call. Typical latency ~150ms.

## Skill version

`v0.2.0` — collapsed from 4-call workflow to single `get_next_required_actions` call. The substrate now provides the ranking + rationales; the agent's job is just to render.

`v0.1.0` (prior) — orchestrated `search_transactions` + `get_open_tasks` + `get_key_dates` per active transaction, then client-side bucketed + sorted. Retired 2026-05-18 when the bundled-judgment tool shipped.
