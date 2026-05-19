<!--
  Generated from plugins/shared/skills/follow-up-drafting.md — do not hand-edit.
  Update the canonical file and re-sync. See plugins/README.md for the rule.
  Generator: hand-curated v1 (per spec §3.3). Last sync: 2026-05-19.
-->

---
name: follow-up-drafting
title: Draft and send follow-up emails for missing documents, approaching deadlines, and unanswered communications
purpose: Compose chase emails with tone calibrated for urgency + recipient; confirm in chat; fire the matching send_* tool.
applicable_to: [claude-desktop, codex, cowork, chatgpt, gemini]
tools_used:
  - get_transaction
  - get_key_dates
  - get_upcoming_key_dates
  - get_missing_documents
  - get_open_tasks
  - get_contacts
  - find_contact_by_email
  - send_client_update
  - send_document_request
  - send_agent_followup
  - send_email_to_agent
expected_output: A drafted email shown inline, an explicit confirmation prompt, then the matching send_* tool fires. The result includes the recipient + subject so the TC has a clear record.
required_scopes: [draft]
version: 1.0.0
---

# Follow-Up Drafting

## When this fires

The TC says any of:
- "Follow up with [party] about [topic]"
- "Chase [recipient] on [missing doc / deadline]"
- "Nudge the [agent / lender / inspector]"
- "Remind [recipient]"
- "Send a check-in to [buyer / seller]"
- "Draft a weekly update to [buyer / seller]"
- "Reply to [thread]"

Also fires from `email-triage` when the TC approves drafting a chase.

## The recipe

### Step 1 — Pull context

1. **`get_transaction`** — full state of the deal (status, parties, financials)
2. **`get_key_dates`** — deadline proximity for tone calibration in step 3
3. **`get_missing_documents`** — if this is about a missing doc
4. **`get_open_tasks`** — what else is in flight
5. **`get_contacts`** or **`find_contact_by_email`** — resolve the recipient

### Step 2 — Pick the right tool

| User intent | Tool |
|---|---|
| Weekly update to buyer or seller | `send_client_update` |
| Specific missing-doc chase | `send_document_request` |
| Agent-to-agent on a topic | `send_agent_followup` |
| Generic email with talking points | `send_email_to_agent` |

(Detailed mapping in the `execution-workflow` skill.)

### Step 3 — Calibrate tone

#### By urgency (days from deadline)

| Days out | Tone | Example opening |
|---|---|---|
| 7+ | Friendly check-in | "Just wanted to touch base on..." |
| 3-7 | Polite, specific | "We have [item] due by [date], could you please..." |
| 1-2 | Firm, action required | "This is due tomorrow — please send..." |
| Overdue | Direct, professional | "This was due on [date]. Please provide at your earliest convenience to avoid delays." |

#### By recipient role

| Recipient | Tone |
|---|---|
| Buyer / Seller | Warm, reassuring, simple. No jargon. |
| Listing Agent / Buyer Agent | Professional peer. Assume process knowledge. |
| Lender | Formal. Reference loan number when known. |
| Title Company | Direct. Reference file number when known. |
| Inspector / Appraiser | Brief, scheduling-focused. |

### Step 4 — Compose

You (the model) compose the subject + body using the context. Include:

- **Property address in subject line** (always)
- **The relevant deadline in the body** when applicable
- **Specific ask** (not "let me know" — "send the survey by Friday")
- **Brief** — TCs value brevity; aim for < 150 words

### Step 5 — Confirm in chat

Per `execution-workflow` — the chat is the approval gate. Show the draft inline:

```
Drafted for Mike Park (buyer agent):

Subject: Survey needed for 412 Oak St — closing 5/22

Hi Mike,
Quick reminder — we need the property survey for 412 Oak St in hand by
Wednesday 5/20 to keep our closing on track. Can you send it over today?

Thanks,
Casey

Want me to send it?
```

Wait for the user's "yes" before firing the tool.

### Step 6 — Fire the right `send_*` tool

Pass `transactionId`, `recipientContactId` (or `recipientRole`), `subject`, `bodyHtml`, `bodyPlainText`. The tool returns a result like:

```json
{
  "status": "sent",
  "transactionId": "<guid>",
  "summary": "Email sent to Mike Park (mike@parkrealty.com) — subject: 'Survey needed for 412 Oak St — closing 5/22'",
  "outcome": "enqueued for mike@parkrealty.com via Gmail, outbox id=…"
}
```

### Step 7 — Report to the user

```
✅ Email sent to Mike Park (mike@parkrealty.com)
   Subject: "Survey needed for 412 Oak St — closing 5/22"

You can see this on the Smith deal's activity feed.
```

## Email rules (in the composed body)

- Always include property address in subject line
- Include the relevant deadline date when applicable
- Keep emails under 150 words
- One ask per email (combine related items on the same party into one email)
- Never threaten or use adversarial language

## Examples

**Missing-doc chase to listing agent, 2 days from deadline:**

```
Subject: Survey needed for 412 Oak St — closing 5/22

Hi Mike,

Quick reminder — we need the property survey for 412 Oak St
in hand by Wednesday 5/20 to keep our closing on track.
Can you send it over today?

Thanks,
Casey
```

**Weekly update to buyer:**

```
Subject: Weekly update — 412 Oak St

Hi Jane,

Quick update on 412 Oak St. The inspection cleared Tuesday
with only minor items the seller agreed to address. We're
tracking to close on 6/15, and I'll have the CD timeline
locked by Friday.

Let me know if anything comes up.

Casey
```

## Don't do

- **Don't fire `send_*` without an explicit user yes.** Per `execution-workflow`.
- **Don't impersonate the TC's personal voice perfectly on day 1.** Use a professional default; the TC can ask you to adjust before sending.
- **Don't fabricate facts.** Only state things the tools confirmed.
- **Don't pile on.** If multiple items need follow-up with the same party, bundle them into one email.
- **Don't say "Pending Review" or link `/prepared-work`.** That UI is retired. Say _"sent"_ when the action completes.
