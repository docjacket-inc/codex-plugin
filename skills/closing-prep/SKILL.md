<!--
  Generated from plugins/shared/skills/closing-prep.md — do not hand-edit.
  Update the canonical file and re-sync. See plugins/README.md for the rule.
  Generator: hand-curated v1 (per spec §3.3). Last sync: 2026-05-19.
-->

---
name: closing-prep
title: Run a pre-closing checklist for a transaction and surface what's missing or at risk
purpose: Walk the standard pre-closing checks against a transaction and report what's done, what needs attention, and what's missing — so the TC arrives at closing day with no surprises.
applicable_to: [claude-desktop, codex, cowork, chatgpt, gemini]
tools_used:
  - get_transaction
  - get_key_dates
  - get_missing_documents
  - get_checklist_status
  - list_open_contingencies
  - get_open_tasks
  - find_contact_by_email
  - send_document_request
  - send_agent_followup
  - create_tasks
expected_output: A structured pre-closing report grouped by status (complete / needs attention / overdue), with offers to send follow-ups or create tasks for any gaps.
required_scopes: [read, draft]
version: 1.0.0
---

# Closing Preparation

## When this fires

- The TC says "closing prep" / "pre-closing" / "final review" / "ready for closing?"
- A transaction has a closing date in the next 7 days
- The TC asks "what's left on [transaction]?" with a deal that's near closing

## The recipe

### Step 1 — Pull the deal state

1. **`get_transaction`** — full record (address, parties, side, price, status)
2. **`get_key_dates`** — verify closing date and any pre-closing dates (final walkthrough, CD delivery)
3. **`get_missing_documents`** — what documents are still expected
4. **`get_checklist_status`** — task counts + overdue list
5. **`list_open_contingencies`** — inspection / appraisal / financing release status
6. **`get_open_tasks`** — anything specific assigned

### Step 2 — Apply the pre-closing checklist

For each item below, report status using the data from step 1:

| # | Item | How to check |
|---:|---|---|
| 1 | **Closing Disclosure (CD) received** | Missing-docs list shouldn't show `closing_statement` |
| 2 | **CD received ≥ 3 business days before closing** | Use the CD's filed-on date if available; flag if too tight |
| 3 | **Final walkthrough scheduled** | Look for a key date or task referencing walkthrough |
| 4 | **Wire instructions received and verified** | Missing-docs list + a task/note confirming verification — flag if unconfirmed |
| 5 | **Title commitment received** | Missing-docs list |
| 6 | **All contingencies released or satisfied** | `list_open_contingencies` should be empty or marked satisfied |
| 7 | **Loan approved / Clear to Close** (if financed) | Look for `loan_approval` doc or task |
| 8 | **Homeowner's insurance binder received** | Missing-docs list — `insurance` |
| 9 | **HOA transfer documents** (if applicable) | Missing-docs list — `hoa_document` |
| 10 | **Amendments / addenda all signed** | Look for unsigned amendments — heuristic via task state |
| 11 | **Commission details confirmed** | Look for related task |
| 12 | **Closing location and time confirmed** | Look for a key date with time/location + task |

### Step 3 — Group results by status

```
🎯 412 Oak St — Closing Friday 6/15 (3 days)

✅ Complete (8)
  • CD received 6/9 (timing OK — 6 days before close)
  • Inspection contingency released 5/22
  • Financing contingency released 6/5
  • Appraisal contingency released 6/3
  • Title commitment received 5/30
  • Insurance binder received 6/8
  • Wire instructions received 6/10 (verified by phone per Casey's note)
  • Walkthrough scheduled Thu 6/13 at 5pm

⚠️ Needs Attention (2)
  • Buyer's HOA estoppel — requested 6/1, not received
    → Suggested: chase the HOA management company today
  • Loan commitment letter — verbal clear-to-close but no written letter
    → Suggested: ask the lender to send the CTC letter for the file

❌ Overdue / Missing (1)
  • Property survey — was due 6/5, not received
    → Suggested: chase the title company today (overdue 6 days)
```

### Step 4 — Offer to act

Bundle related actions:

> _"Want me to send chases for the HOA estoppel + loan letter + survey? Three emails, one confirmation."_

On user yes, fire:
- **`send_document_request`** for each missing-doc chase (one call per recipient — bundle items going to the same person into one email)
- **`send_agent_followup`** for non-doc-specific follow-ups (e.g. asking the agent about a contingency status)
- **`create_tasks`** for items the TC wants to track but not chase yet

Per `execution-workflow`, **one confirmation can authorize multiple bundled calls** — but only if the user clearly said yes to the bundle. If unclear, confirm each.

## State-specific awareness

Different states have different required disclosures, addendum forms, and timing rules. The classifier + missing-docs baseline cover federal items (LBP for pre-1978 properties, CD timing, etc.); state-specific compliance is handled by per-state skill files in DocJacket's QA review system (see Phase 6.5 substrate).

Treat the checklist above as the universal floor. If the user's brokerage / state requires additional items, ask the user and remember them within the conversation.

## Don't do

- **Don't speculate about what's missing without checking the tools.** Use `get_missing_documents`, not assumptions.
- **Don't claim a contingency is released without checking.** Use `list_open_contingencies`.
- **Don't bury done items.** Only call out completed items if they're compliance-relevant (CD timing) or the user asked for a full report.
- **Don't recommend actions that need a signature or wire transfer.** Users make those calls; you surface the gap.
- **Don't fire `send_*` / `create_*` without an explicit user yes** — per `execution-workflow`. Bundle confirmations when sensible.
