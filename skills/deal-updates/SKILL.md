---
name: deal-updates
description: >-
  Keeps a live DocJacket deal accurate from chat: reads what is wrong with it, changes the deal
  record (price, financing, side, type, address, numbers, custom fields), manages key dates and
  their outcomes, records task outcomes with a 7-day undo, adds team notes and task or document
  comments, changes who holds a seat on the deal, restores an archived deal, and approves or
  rejects new deals waiting in the review queue. Use when the TC says something on a deal is
  wrong or changed, a deadline was met or waived, tasks are done or not needed, wants a note
  recorded, or asks what was said, sent or changed on a deal.
license: MIT
metadata:
  docjacket-title: "Change and repair a live deal, with confirmation and undo"
  docjacket-tools: "find_transaction_by_property review_deal_health get_transaction_context search_transaction_context get_transaction_knowledge get_deal_section get_custom_fields update_deal get_key_dates update_key_date manage_key_date set_key_date_outcome get_open_tasks set_task_outcome undo_task_outcome manage_note manage_deal_contact add_contact_to_transaction restore_deal list_new_deal_intake approve_new_deal_proposal reject_new_deal_proposal list_held_emails get_held_email discard_held_email list_team_members"
  docjacket-scopes: "read draft actions"
  docjacket-version: "1.0.0"
---

# Deal Updates

For changes to a deal that already exists. Sending emails is in `follow-up-drafting` and `execution-workflow`; this skill covers the deal's own record, dates, tasks, notes and people.

Every write here needs the user's confirmation first, and uses only values the user gave or confirmed. Never infer a property's state, a price or a date.

## When this fires

- "Something's off on the Oak St deal", "why is this overdue?", "who is this agent on my deal?"
- "The price changed to 512,000", "it's FHA now", "fix the APN", "we're on the buyer side"
- "Inspection was met", "waive the appraisal", "HOA review doesn't apply"
- "Those three tasks are done", "skip the survey task", "undo that"
- "Add a note that the lender is out until Monday", "what did the title company say?"
- "Take Bob off the deal", "make Jane the primary buyer"
- "Bring back the Elm St deal", "approve the contract I emailed in"

## Step 1 — Read before you change

1. Find the deal (`find_transaction_by_property` or the deal in context).
2. When the user says something looks wrong, overdue, duplicated, missing or stuck, call **`review_deal_health`** first. Each finding says what is wrong, which rows (with ids), where the user sees them, and which tool fixes it, or that there is nothing to do. A contract-start date such as the Effective Date is never overdue and never something to complete.
3. Current values: **`get_transaction_context`** (one call for parties, status, key dates, tasks and missing documents; pass a `purpose`). For a specific question about the deal's documents, **`search_transaction_context`** with a free-text query.
4. Why a deadline exists or what it means: **`get_transaction_knowledge`** (the rule, its source and paragraph, and a verdict such as applies, estimated or needs_fact).
5. What was said, sent, scheduled, changed, agreed or billed: **`get_deal_section`** (notes, task and document comments, email, texts, scheduled sends, change log, key-date history, agreements, fees and invoices, Documents Needed). Pass `itemId` to read one item in full.

## Changing the deal record

**`update_deal`** changes up to 10 fields on the deal's own record: MLS, escrow and APN numbers, notes, price, earnest money, commission and broker-fee terms, side, transaction type, financing, address and state, office, Waiting On, and custom field values (ids from `get_custom_fields`). Fields it doesn't name are never touched. Report each field's old and new value; the result carries an undo handle good for 7 days.

Not for: key dates or the closing date (`update_key_date`), status, close-out or archive, the people on the deal (`manage_deal_contact`), the TC fee or portal settings.

## Key dates

- Move a date: **`update_key_date`**.
- Add key dates, rename, notes, time or location, reset a hand-set date to its calculated date, recalculate from anchors, reorder, link or unlink a task, apply or dismiss a pending date-change proposal: **`manage_key_date`** (one change per call). Dependent dates, linked tasks, reminders and scheduled emails move as they do on the deal page. It never deletes a key date.
- A deadline was met, waived, doesn't apply, or should be reopened: **`set_key_date_outcome`** with `complete`, `waive`, `not_applicable` or `reopen` (keyDateId from `review_deal_health` or `get_key_dates`). Completing stops that date's pending reminders and scheduled emails.

## Tasks

- **`set_task_outcome`** records `complete`, `skip` or `reopen` for up to 25 tasks on one deal (taskIds from `get_open_tasks`). Completing unblocks and re-dates dependent tasks and starts the task's follow-up emails; skipping sends nothing.
- DocJacket sends or holds the emails a completion triggers; never send them yourself. When the result has `completionEmailsHeld: true` they are held, not sent: follow `execution-workflow` ("Releasing a held email"), and when `heldFor` is `"person"` tell the user they are waiting for their approval in DocJacket.
- **`undo_task_outcome`** with the `preparedWorkId` takes back a complete or skip within 7 days. Emails already sent stay sent.
- Rename, re-date, reassign or delete a task: see `execution-workflow` (`update_task`, `reschedule_task`, `reassign_task`, `delete_tasks`). Teammate ids come from `list_team_members`.

## Notes and comments

**`manage_note`** adds, edits, pins, unpins, removes or restores an internal note on a deal, or a comment on one of its tasks or documents. They are for the team only: never on the client or agent portal and never emailed. Remove archives; restore brings it back. Edit, remove and restore work only on notes the user wrote (noteId from `get_deal_section`). Write only what the user asked you to record.

## People on the deal

- Add someone: `add_contact_to_transaction` (see `contact-management`).
- **`manage_deal_contact`**: `remove` takes a person off the deal (undoable for 7 days; the contact record is kept), `change_role` moves them to another existing role, `set_primary` makes them the primary holder of their role. Use it only after the user names the person and the change.

## Archived deals

**`restore_deal`** brings back one archived deal with its documents, text threads and review-queue drafts, and turns its portal links back on. Skipped tasks stay skipped and cancelled scheduled emails don't come back. Pass `address` to find an archived deal (normal deal search doesn't return them). Never use it to reopen a closed or cancelled deal.

## New deals waiting for review

1. **`list_new_deal_intake`** shows contracts emailed to the new-deal address and what became of each. Reading takes 1 to 6 minutes; poll no more than every 30 seconds.
2. For a proposal held for review, show the user the proposal and ask which side they represent. **`approve_new_deal_proposal`** with `actionId` and `side` (`Buyer`, `Seller` or `NotSure`) creates the deal; pass `createDuplicate: true` only if the user agreed to a second deal at the same address, and `fileAsHistorical` only if they chose. Call `list_new_deal_intake` again for the new transactionId.
3. **`reject_new_deal_proposal`** removes it from the Review Queue without creating a deal.

Only owners, admins, brokers and transaction coordinators can approve or reject.

## Held emails the user doesn't want

Show the email first (`list_held_emails`, `get_held_email`). On the user's no, **`discard_held_email`** with its `actionId`: it is never sent, and whatever waited behind it (a scheduled email, a reminder) stops too. It cannot approve or send anything.

## Guardrails

- Confirm before every write and say what will happen; afterwards, report what changed in one line.
- Removing a person, rejecting a proposal and discarding an email are marked destructive: name the person, proposal or email in your confirmation question.
- Offer the 7-day undo when it exists (`update_deal`, `set_task_outcome` via `undo_task_outcome`, `manage_deal_contact` remove).

## Tool reference

| Tool | Scope | Use |
|---|---|---|
| `review_deal_health` | Read | What is wrong with the deal and which tool fixes it |
| `get_transaction_context` / `search_transaction_context` / `get_transaction_knowledge` | Read | Current state, document search, why a deadline exists |
| `get_deal_section` | Read | Notes, comments, email, texts, change log, fees |
| `update_deal` | Actions | Deal record fields (7-day undo) |
| `update_key_date` / `manage_key_date` / `set_key_date_outcome` | Draft | Move, manage, complete or waive key dates |
| `set_task_outcome` / `undo_task_outcome` | Draft | Complete, skip or reopen tasks; undo |
| `manage_note` | Draft | Team notes and task or document comments |
| `manage_deal_contact` | Actions | Remove, change role, set primary |
| `restore_deal` | Actions | Bring back an archived deal |
| `list_new_deal_intake` / `approve_new_deal_proposal` / `reject_new_deal_proposal` | Read / Actions | Emailed-in contracts waiting for review |
| `discard_held_email` | Actions | Drop a held email the user doesn't want |
