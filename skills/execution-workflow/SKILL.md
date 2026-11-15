---
name: execution-workflow
description: >-
  Composes work in chat and carries it out in DocJacket. Emails are previewed to the user and sent
  only after an explicit yes via confirm_email_send (or wait in the DocJacket review queue for a
  team member); internal changes (tasks, key dates, reminders, contacts, Playbooks) run on call.
  Reports a status such as awaiting_confirmation, parked_for_review, queued, created or updated,
  plus a one-line summary of what was done or what is waiting; an email is never reported as on
  its way unless confirm_email_send returned status queued. Use when the TC asks to send an email,
  create or change tasks, update key dates, set reminders, add or update contacts, send calendar
  invites, or apply a Playbook.
license: MIT
metadata:
  docjacket-title: "Send emails, create tasks, and update key dates in DocJacket"
  docjacket-tools: "send_client_update send_document_request send_agent_followup send_email_to_agent confirm_email_send create_tasks complete_task update_task reschedule_task reassign_task delete_tasks update_key_date add_key_dates_batch create_reminder update_reminder cancel_reminder get_reminders save_status_summary search_contacts create_contact update_contact add_contact_to_transaction create_deal_calendar_event send_key_date_invites cancel_key_date_invites list_playbooks get_playbook preview_playbook apply_playbook undo_playbook start_playbook_draft list_playbook_drafts commit_playbook_draft undo_playbook_setup"
  docjacket-scopes: "draft"
  docjacket-version: "1.2.0"
---

# Execution workflow

> **Retired:** the old `prepare_*` / `propose_*` / `draft_*` / `suggest_*` tools and the Pending Review tray are gone. Use the tool names in this skill.

## Architecture

You compose. The user confirms in chat. Then there are two tiers.

1. **Messages that reach other people** (the four `send_*` email tools, `send_key_date_invites`, `cancel_key_date_invites`). Nothing is sent by the first call. An email send tool returns a preview and a `confirmCode` (or parks the email for a team member's review); you send it with `confirm_email_send` only after the user says yes to the preview. Calendar invites work the same way with their own status and code.
2. **Internal writes** (tasks, key dates, reminders, contacts, calendar events, Playbooks, status summaries). The tool performs the action on call, so confirm intent in chat first.

Every write tool takes a **finished** subject + body / payload. DocJacket does not call an LLM on its side. The customer is paying for your composition through their Claude / ChatGPT / Codex / Cowork / Gemini subscription; DocJacket gets one cheap API call per action.

### What the result means

Read `status` before you tell the user anything.

| `status` | Meaning | Tell the user |
|---|---|---|
| `awaiting_confirmation` | An email send tool returned a `preview` and a `confirmCode` (starts `djc_`). NOTHING has been sent. | Show the COMPLETE preview, then ask. On an explicit yes call `confirm_email_send`; on no, call it with `decision: "discard"`. |
| `parked_for_review` | The email has NOT been sent. It is waiting in the DocJacket review queue for a team member (the organization reviews AI-written emails, or something forced a review). No `confirmCode` is issued. | "It is waiting for approval in DocJacket, not sent yet." Give `reviewReason` (or `heldBecause`) in plain English if present. |
| `already_approved` | A reviewer already approved a draft with this exact subject and recipient and it went out from the queue. Nothing new was sent. | Say that, and that a different subject is needed to send something new. |
| `queued` | `confirm_email_send` approved the email and queued it to send. It has not left yet. | Report recipient and subject as approved and on its way. Only `get_held_email` with its `actionId` showing `state: "sent"` means it went out; `failed` means it did not. |
| `discarded` | `confirm_email_send` with `decision: "discard"` removed the email from the review queue. | Say it was discarded and nothing was sent. |
| `preview_pending_confirmation` | A calendar-invite tool returned a preview and a `confirmCode`. Nothing has gone out yet. | Show the attendees and dates. On an explicit yes, call the SAME tool again with the `confirmCode`. |
| `created` / `updated` / `saved` | An internal write ran. | Report the one-line `summary`. |

```json
{
  "status":        "awaiting_confirmation",
  "confirmCode":   "djc_...",
  "expiresAt":     "2026-11-06T18:00:00Z",
  "recipient":     "Jane Doe (jane@example.com)",
  "preview":       { "to": [...], "cc": [...], "bcc": [...], "subject": "...", "bodyHtml": "...", "bodyText": "...", "attachments": [...] },
  "summary":       "NOT sent yet. Show the user the complete email above ..."
}
```

Show the `summary` to the user. For an email, the action is not done until `confirm_email_send` returns `queued` or a team member approves it.

## When to use which tool

| User intent | Tool |
|---|---|
| "Send a weekly update to the buyer / seller on this deal" | `send_client_update` |
| "Ask the lender / inspector / title rep for a missing document" | `send_document_request` |
| "Email the listing agent about [topic]" (or any agent-to-agent on a specific topic) | `send_agent_followup` |
| "Draft an email to the [agent] covering these talking points" | `send_email_to_agent` |
| "Update / extend / move the [key date] to [new date]" | `update_key_date` |
| "Add several key dates to this transaction" (intake or post-amendment) | `add_key_dates_batch` |
| "Add a task to follow up on X" or "remind me to Y by Z" | `create_tasks` |
| "Set a reminder for [X days] before [deadline]" or "nudge the agent before EMD" | `create_reminder` |
| "Summarize this transaction for me" or "save a status recap on the deal" | `save_status_summary` |
| "Mark that task done" | `complete_task` (taskId from `get_open_tasks`). If its result has `completionEmailsHeld: true`, the emails the task triggers are held, NOT sent (listed in `heldEmails`, or by `list_held_emails` when that is empty): never send them yourself, and do not render or send `triggeredEmailTemplates`; when `heldFor` is `"assistant"`, release each like a held email (below) or discard it; when it is `"person"`, tell the user they are waiting for their approval in DocJacket, because no assistant can release them |
| "Change that task's title or notes" | `update_task` |
| "Push that task to Friday" | `reschedule_task` |
| "Give that task to Dana" | `reassign_task` |
| "Delete these tasks" | `delete_tasks` |
| "Change / cancel that reminder" | `update_reminder` / `cancel_reminder` (reminderId from `get_reminders`) |
| "What reminders are set on this deal?" | `get_reminders` |
| "Find / add / fix a contact; put them on this deal" | `search_contacts`, then `create_contact` only if none found, `update_contact`, `add_contact_to_transaction` |
| "Put the walkthrough on my calendar" | `create_deal_calendar_event` (invites nobody) |
| "Invite the lender and title rep to the inspection date" / "cancel those invites" | `send_key_date_invites` / `cancel_key_date_invites` |
| "What would this Playbook add to the deal?" / "Apply it" / "Undo that" | `preview_playbook` / `apply_playbook` / `undo_playbook` |
| "Set up a new Playbook for my team" | `start_playbook_draft`, `list_playbook_drafts`, `commit_playbook_draft`, `undo_playbook_setup` |

Four calls to keep close:

1. **Client update vs agent follow-up.** `send_client_update` is for buyer / seller communications — the recipient is the customer's client. `send_agent_followup` is for the network — listing agent, buyer agent, lender, title rep, referral source. Different tone, different liability, different default recipients.

2. **Document request vs agent follow-up.** If the email's primary purpose is *"please send me X"*, use `send_document_request` (it captures `documentTypes`, which feeds document tracking). Anything else agent-to-agent → `send_agent_followup`.

3. **Create tasks vs update key date vs create reminder.** "Follow up with the lender on Friday" → `create_tasks` (manual todo for the TC). "Move the financing deadline to Friday" → `update_key_date` (moves the contractual milestone; cascades). "Nudge the lender 2 days before financing deadline" → `create_reminder` (durable Hangfire job; fires server-side even when the chat is closed).

4. **`update_key_date` vs `add_key_dates_batch`.** Single-row date moves (TC amends one deadline) → `update_key_date`. Bulk seeding (intake just produced 5-7 timeline dates that all need to land) → `add_key_dates_batch` — one transactional call instead of N round trips, with per-row failure reporting so the LLM can retry just the misses.

## Confirm before you call

### Emails: preview, confirm, send

Compose the email, then call the `send_*` tool. It sends nothing. The recipient must be a contact already on the deal (`CONTACT_NOT_ON_DEAL` otherwise); add them with `add_contact_to_transaction` first, after the user agrees.

1. **`status: "awaiting_confirmation"`.** Show the user the COMPLETE email exactly as returned in `preview`: recipients (to, cc, bcc), subject, full body, attachments. Call `confirm_email_send` with the `confirmCode` ONLY after the user explicitly says yes to that preview. If they say no, call it with `decision: "discard"`. Never confirm on your own, on a vague reply, or on a yes given to an earlier draft: if the preview changed, show it again. A `confirmCode` works once, for 30 minutes, for the same user and connection, and for exactly the content previewed. An email nobody confirms waits in the DocJacket review queue.
2. **`status: "parked_for_review"`.** The email has NOT been sent. It is waiting in the DocJacket review queue for a team member. Tell the user in plain words: "It is waiting for approval in DocJacket, not sent yet", and give `reviewReason` (or `heldBecause`) if present. Never say sent, delivered or done. There is no code to confirm and nothing more for you to call; do not call the send tool again to push it through. The organization's setting (`review_auto_emails_before_send`) decides between this path and the preview, and you cannot override it.
3. **`confirm_email_send` returns `status: "queued"`** (approved and queued to send; `get_held_email` shows `sent` or `failed` once it is done) or `"discarded"`, or `"already_approved"` if a reviewer got there first. Report exactly that.

**Releasing a held email.** Only some organizations let you release an email DocJacket is holding (`list_held_emails`). There, `prepare_held_email_approval` with its `actionId` prepares the release and sends nothing: `status: "awaiting_confirmation"` works exactly like step 1 (show the COMPLETE preview, confirm only on an explicit yes, `decision: "discard"` on a no). `status: "cannot_release"` means a blocking hold reason must be fixed first; tell the user what `holdReasons` says. `person_must_approve` cannot be fixed from here: a team member approves that email in DocJacket. `NOT_ENABLED` means the organization approves held emails in DocJacket; tell the user it is waiting there. A held email whose requested send time has passed (`requested_send_time_passed`, or `SEND_TIME_PASSED` from confirm) can only be rescheduled or sent now in DocJacket's web app: tell the user, and do not prepare it again.

If `confirm_email_send` returns an error, read the code:

| Code | What to do |
|---|---|
| `CONFIRM_CODE_EXPIRED`, `CONTENT_CHANGED`, `SUPERSEDED` | Call the send tool again for a fresh preview and show the whole new preview. (An older preview was replaced, the 30 minutes ran out, or the email changed since you previewed it.) |
| `REVIEW_REQUIRED`, `NOT_ALLOWED_TO_CONFIRM`, `RECIPIENT_NOT_ALLOWED`, `NO_LONGER_PENDING` | Tell the user the email is waiting in the DocJacket review queue for a team member. Do not retry. |
| `HOLD_NOT_CLEARED` | A held email gained a blocking hold reason after you prepared it. Nothing was sent; call `get_held_email` and tell the user what holds it. |
| `CONFIRM_IN_PROGRESS` | Wait a moment and call `confirm_email_send` again with the same code. |
| `CONFIRM_CODE_INVALID`, `CONFIRM_CODE_NOT_YOURS` | Check the code with the user, or call the send tool again for a fresh preview. |
| `TRANSACTION_NOT_FOUND`, `CONFIRM_FAILED` | Tell the user it did not go out; the email stays in the review queue. |

Calling `confirm_email_send` again with the same code returns the same answer and never sends twice.

### Calendar invites

Google emails every attendee, so these are preview-then-confirm.

1. Call `send_key_date_invites` (or `cancel_key_date_invites`) with the deal, `keyDateIds` (from `get_key_dates`) and `attendeeContactIds` (contacts on the deal with an email who receive transaction updates).
2. The first call sends nothing. It returns `status: "preview_pending_confirmation"` with the final attendees, dates and a one-time `confirmCode`. If the organization reviews outgoing messages it returns `parked_for_review` instead: tell the user it is waiting for approval.
3. Show the user the preview exactly as returned. Only on an explicit yes, call the SAME tool again with that `confirmCode`. The code works once, for 30 minutes. If the user changes anything, call without the code to get a fresh preview.

`create_deal_calendar_event` is different: it adds an event to the user's own calendar immediately and invites nobody. Confirm the details first.

### Internal writes

Confirm intent in chat before a `create_*` / `update_*` / `save_*` / `apply_*` call. Acceptable patterns:

- *"I'll move the inspection deadline from May 25 to June 1 based on the amendment. Sound right?"* (user replies "yes" -> call `update_key_date`)
- *"I'll add a task: 'Follow up with lender about appraisal — due Friday.' OK?"* (user replies "yes" -> call `create_tasks`)

If the user's instruction is unambiguous and self-contained (*"move the inspection deadline to June 1"*), you can call the tool without a separate confirmation turn. Use judgment.

`delete_tasks` is a hard delete with no undo. List the exact tasks (titles and deals) and get an explicit yes first, even when the user already asked for the deletion.

## Retries and ids

Send a request id on every write. Over MCP set the header `X-DocJacket-Client-Request-Id` to a fresh unique value per intended action (a UUID). If a call times out or you are unsure it landed, retry with the SAME id: DocJacket replays the first result instead of running the action twice. A new id means a new action. Over REST use `Idempotency-Key` the same way. Without the header a retry is not de-duplicated. If your client cannot set the header, call the matching read first (`get_open_tasks`, `get_reminders`, `get_key_dates`, `get_contacts`) and retry only what is missing. When you confirm a send (`confirm_email_send`) or a calendar invite, retrying the confirm with the same request id replays the first answer; an id that was already spent on a different action must be replaced with a new one.

## Ids come from lookups

Get every id from a lookup read; never guess or invent one, and never reuse an id from an earlier conversation without re-reading.

- taskId: `get_open_tasks` (use `search` for words in the title; `scope: "org"` to search beyond one deal) or `get_checklist_status`.
- contactId: `search_contacts`, `get_contacts`, `find_contact_by_email`.
- keyDateId: `get_key_dates`.
- reminderId: `get_reminders` (states: `sent`, `awaiting_review`, `cancelled`, `held`, `scheduled`). `update_reminder` and `cancel_reminder` only work on reminders that have not gone out yet (`REMINDER_SENT` otherwise).
- Playbook id: `list_playbooks`, then `get_playbook`.
- transactionId: `search_transactions` or `find_transaction_by_property`.

If a lookup returns several matches, show them and ask. If it returns none, say so; do not create a duplicate (`create_contact` only after `search_contacts` finds no match).

## Required upstream context

Before composing, gather what you need. The customer's mail-MCP (Gmail / Outlook / Superhuman) handles their inbox; DocJacket handles deal state.

### Recipe — weekly buyer update

```
1. get_transaction({ id })                 — parties, key dates, current state
2. get_contacts({ transactionId })         — find the buyer contact
3. get_key_dates({ transactionId })        — what's coming up
4. get_open_tasks({ transactionId })       — what's still pending
5. (compose subject + body inline using the above)
6. (show user the full draft, ask for confirmation)
7. send_client_update({
     transactionId, audience: "buyer", focus: "weekly",
     recipientContactId, subject, bodyHtml, bodyPlainText
   })
   → "awaiting_confirmation": show the full preview; on the user's yes, confirm_email_send({ confirmCode })
   → "parked_for_review": tell the user it is waiting for approval in DocJacket
```

### Recipe — chase a missing document

```
1. get_missing_documents({ transactionId }) — see what's actually missing
2. get_contacts({ transactionId })          — find the contact for the role
3. (compose subject + body referencing the missing doc)
4. (confirm with user)
5. send_document_request({
     transactionId, recipientRole: "listing_agent",
     recipientContactId, documentTypes: ["seller_disclosure"],
     subject, bodyHtml, bodyPlainText
   })
```

### Recipe — schedule a durable reminder

```
User: "Remind the buyer's agent 2 days before EMD is due."

1. get_key_dates({ transactionId })          — find the EMD key date (or get it from the user)
2. get_contacts({ transactionId })           — confirm the buyer's agent is on the deal
3. (confirm timing + recipient with user)
4. create_reminder({
     transactionId,
     deadlineType: "EarnestMoneyDue",
     deadlineDate: "2026-05-22",
     daysBeforeReminder: 2,
     remindWho: "party",
     partyRole: "buyer_agent",
     channel: "email"
   })
```

The reminder fires via DocJacket's Hangfire scheduler, whether the chat is open or not. It does not send on its own: when it comes due, its email waits in the DocJacket review queue until a team member approves it. Tell the TC to approve it there. "Scheduled" means it will wait for a team member's approval when due, not that the email goes out. If the same `(deadlineType, daysBeforeReminder)` pair already has a reminder, the call returns `DUPLICATE_REMINDER` (idempotent).

For conditional "only nudge if they haven't already responded" logic, that's a Cowork Scheduled Task or Codex Automation that wraps `create_reminder` + a Gmail thread check — out of scope for the deterministic-server-side reminder.

### Recipe — batch-add timeline key dates (post-extraction or post-amendment)

```
User: "Add EMD May 22, inspection May 29, appraisal June 5, financing June 13, walkthrough June 19."

1. (paraphrase back for confirmation)
2. add_key_dates_batch({
     transactionId,
     keyDates: [
       { keyDateType: "EarnestMoneyDue",     dateValue: "2026-05-22" },
       { keyDateType: "InspectionDeadline",  dateValue: "2026-05-29" },
       { keyDateType: "AppraisalDeadline",   dateValue: "2026-06-05" },
       { keyDateType: "FinancingDeadline",   dateValue: "2026-06-13" },
       { keyDateType: "FinalWalkthrough",    dateValue: "2026-06-19" }
     ]
   })
   → { created: 5, failed: [] }
```

Up to 50 entries per batch. Per-row failures (unknown `keyDateType`, invalid date) come back in the `failed` array — surface them: *"Added 4 of 5. The fifth ('survey_deadline') isn't a recognized type — want me to add it as a custom date instead?"*

For intake-time bulk seeding, prefer this over a loop of `update_key_date` calls — one DB transaction, one round trip, per-row failures don't abort the batch.

### Recipe — follow up on a prior email thread

```
User: "Reply to Sarah's note about the inspection extension"

1. (Use your mail-MCP — Gmail, Outlook, etc.) search_threads("sarah inspection")
2. (Use your mail-MCP) get_thread(threadId)  — read her actual message
3. get_transaction({ ... })                  — get the deal context
4. (compose a reply grounded in BOTH)
5. (show user the draft, confirm)
6. send_agent_followup({ …, sourceContextSummary: "Sarah asked Wed about …" })
```

DocJacket does not have an inbox-read tool. Your mail-MCP does.

## Anti-patterns (don't)

- **Don't fire a `send_*` tool without an explicit user yes.** If the user said "draft an email" without saying "send it", show the draft and wait. Even after that yes, the send tool only previews or parks; `confirm_email_send` is what sends.

- **Don't call `confirm_email_send` without the user's explicit yes to the full preview**, and don't say an email is on its way unless it returned `queued`. For `parked_for_review`, don't re-call the send tool to force it through.

- **Don't guess ids.** Look them up (see Ids come from lookups).

- **Don't paste full external email content into `bodyHtml` or `sourceContextSummary`.** Summarize upstream context in one or two sentences. Quoting the original message inline in the new body is fine; archiving someone else's inbox in our database is not.

- **Don't call any tool without the `transactionId` being verifiably for the caller's org.** If you got the id from `search_transactions`, you're fine — that tool is org-scoped. If the user typed a UUID, validate with `get_transaction({ id })` first.

- **Don't fabricate dollar amounts or contractual terms.** Ground every number in `get_transaction` data or an explicit user statement.

- **Don't compose without grounding.** A draft to "Sarah" with no transaction lookup gets details wrong. Always call at least `get_transaction` + `get_contacts` first — they're cheap.

- **Don't say "Pending Review" or link `/prepared-work`.** That UI is retired. After a preview, ask. After `parked_for_review`, say it is waiting for approval in DocJacket. After `confirm_email_send` returns `queued`, report it as approved and on its way. After an internal write, say "Done — moved the inspection deadline to June 1."

## Bundling

If the user asks for "an email to the buyer AND a task to follow up Friday," that's two tool calls — `send_client_update` + `create_tasks`. Two confirmations (or one combined "Sound good?") before firing. Each email still gets its own preview and its own `confirm_email_send` call; the task half is done on call. One yes can authorise several previews only if the user saw every one. Don't try to combine them in a single tool.

## Output to the user

After an internal write:

```
Done: <summary>

You can see this on the transaction's activity feed.
```

After an email preview (status `awaiting_confirmation`): show the whole email and ask "Send this?" Nothing has gone out.

After an email is parked (status `parked_for_review`), mention the recipient and subject and say plainly that nothing has gone out yet:

```
Waiting for approval in DocJacket, not sent yet.
   To: Jane Doe (jane@example.com)
   Subject: "Weekly update — 8824 Clear View St."
```

Only after `confirm_email_send` returns `queued`:

```
Approved and on its way to Jane Doe (jane@example.com)
   Subject: "Weekly update — 8824 Clear View St."
```

## Send mode is decided by the org

When the customer's org is in **user-mailbox mode** (Option C), an approved email leaves through the user's connected Gmail or Outlook account, so it comes from their actual address. Otherwise it goes via Postmark with the customer's brand. You don't control this. The organization's `review_auto_emails_before_send` setting decides whether an AI-written email waits for a team member (`parked_for_review`) or comes back as a preview for the user to confirm (`awaiting_confirmation`). If a confirmed send fails because the user has user-mailbox mode but hasn't connected a mailbox, point them at Settings -> Integrations.

## Migration from the old `prepare_*` / `propose_*` / `draft_*` tools

If you encounter the old names in older skill files, transcripts, or third-party manifests, treat them as a straight rename — same arguments, just the new tool name. The server no longer registers the old names; a call to one returns an "unknown tool" error.

| Old name | New name |
|---|---|
| `prepare_client_update` | `send_client_update` |
| `prepare_document_request` | `send_document_request` |
| `prepare_agent_followup` | `send_agent_followup` |
| `draft_email_to_agent` | `send_email_to_agent` |
| `suggest_task` / `propose_task` | `create_tasks` |
| `propose_key_date_update` | `update_key_date` |
| `draft_status_summary` | `save_status_summary` |
