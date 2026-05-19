<!--
  Generated from plugins/shared/skills/email-triage.md — do not hand-edit.
  Update the canonical file and re-sync. See plugins/README.md for the rule.
  Generator: hand-curated v1 (per spec §3.3). Last sync: 2026-05-19.
-->

---
name: email-triage
title: Match Gmail / Outlook emails to DocJacket transactions and identify required actions
purpose: Cross-reference the TC's inbox against active deals, group matched email by transaction, and surface what needs the TC's attention.
applicable_to: [claude-desktop, codex, cowork, chatgpt, gemini]
tools_used:
  - list_active_transactions
  - find_contact_by_email
  - search_transactions
  - classify_document
  - get_missing_documents
  - get_checklist_status
  - get_upcoming_key_dates
expected_output: A per-transaction briefing grouping matched email + attachments + deadline proximity, with explicit asks for TC decisions.
required_scopes: [read]
version: 1.0.0
---

# Email-to-Transaction Triage

## When this fires

The TC says any of:
- "Triage my inbox"
- "Check my email"
- "Match incoming emails to my active deals"
- "What's in my inbox related to my transactions?"
- "Find any email related to a deal"

## The recipe

### Step 1 — Cache the active-deals roster (one call)

Call **`list_active_transactions`** with `include_parties: true` and `include_key_dates: true`. This gives you every active deal in the org plus the email address of every party plus upcoming deadlines. **Cache this for the conversation** — don't re-fetch unless the user adds or closes a deal.

### Step 2 — Read the inbox

Use the agent platform's Gmail / Outlook / Superhuman connector (the customer's own — DocJacket doesn't have inbox access).

- Pull recent unread threads from the last 24-48 hours
- For each thread: sender, subject, snippet, attachment names

### Step 3 — Match each email to a transaction (priority order)

For every email:

1. **Compare sender email against the parties cached in step 1.** Exact match → definitive transaction match.
2. **If no sender match**, scan subject + first paragraph of body for:
   - Property addresses → call **`search_transactions`** with the address
   - Buyer / seller names → call **`search_transactions`** with the name
3. **Still no match** → group separately as "unmatched."

You can also call **`find_contact_by_email`** as a more thorough sender lookup if the cached parties don't cover the whole address book.

### Step 4 — For each attachment

- Call **`classify_document`** with the filename + a short content preview (first ~500 chars if you have OCR; otherwise filename only)
- If the matched transaction is known, call **`get_missing_documents`** to see if the classified type fills a gap

### Step 5 — Group results by transaction and present

```
✉  412 Oak St — Smith deal (closing 2026-06-15, 27 days)
   • Inspection contingency expires 2026-05-22 (3 days)
   • Email from Mike Park (buyer agent): "Inspection report attached"
   • Attachment: InspectionReport_412Oak.pdf — looks like an inspection report
   • Fills a missing-doc gap on this deal ✓
   • Suggested action: thank Mike + ask about access for any noted repairs
```

For unmatched emails:
```
✉  Unmatched
   • From: lender@bigbank.com — "Loan #12345 conditions" — no party match
   • Ask the TC: which deal does this relate to?
```

For deadline-proximity context — if a matched deal has key dates in the next 7 days, surface them.

### Step 6 — Ask the TC for decisions

Bundle related decisions per transaction:

> _"Three deals have inbound mail. For Smith (412 Oak): inspection report is here and looks complete — want me to send a thank-you to Mike + a status note to the buyer? For Johnson (88 Maple): lender sent loan conditions — want me to draft a follow-up about Friday's commitment deadline? Chen looks like an unrelated newsletter."_

## After the TC says yes — actions

The `execution-workflow` skill handles the action layer. Use:

| User intent | Action tool |
|---|---|
| Email the buyer / seller a status update | `send_client_update` |
| Email an agent / lender / title rep about a topic | `send_agent_followup` |
| Email asking for a missing document | `send_document_request` |
| Free-form email with talking points | `send_email_to_agent` |
| Create a task to follow up later | `create_tasks` |
| Save a status recap on the deal | `save_status_summary` |

**Filing the actual attachment** is not yet automated — DocJacket doesn't have an `upload_document` tool. Tell the TC: _"I'd file the inspection report to the Smith deal, but file-on-approval isn't available yet — upload it manually in DocJacket → Documents → Smith."_ A `create_tasks` call can capture the upload as a reminder.

## Presentation rules

- **Lead with what needs attention.** Skip newsletters / promotions / auto-notifications.
- **Group by transaction**, not chronologically.
- **Mention deadline proximity** for any deal with key dates in the next 7 days.
- **One ask per deal when possible.** Don't make the TC answer five questions for one deal.
- **Confirm before any `send_*` / `create_*` / `update_*` call** — per `execution-workflow`, the chat is the approval gate. _"Want me to send it?"_ → wait for yes → fire.

## Don't do

- **Don't auto-match low-confidence emails.** If you only have a first name in the subject, ask which one.
- **Don't summarize emails the TC didn't ask about.** This is triage, not analysis.
- **Don't repeat the same deal across multiple groupings.** One section per transaction.
- **Don't fire `send_*` without an explicit user yes.** Even if the email is obvious, get the green light.
