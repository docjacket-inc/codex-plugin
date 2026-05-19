<!--
  Generated from plugins/shared/skills/execution-workflow.md — do not hand-edit.
  Update the canonical file and re-sync. See plugins/README.md for the rule.
  Generator: hand-curated v1 (per spec §3.3). Last sync: 2026-05-19.
-->

---
name: execution-workflow
title: Send emails, create tasks, and update key dates directly in DocJacket
purpose: Compose work artifacts upstream in chat. Confirm with the user. Then call the matching action tool — DocJacket executes immediately. No second approval in DocJacket.
applicable_to: [claude-desktop, codex, cowork, chatgpt, gemini]
tools_used:
  - send_client_update
  - send_document_request
  - send_agent_followup
  - send_email_to_agent
  - create_tasks
  - update_key_date
  - save_status_summary
expected_output: A status (sent / created / updated / saved) plus a one-line summary of what was done. The user sees the result on the transaction's activity feed.
required_scopes: [draft]
version: 1.0.0
---

# Execution workflow

> **What changed (2026-05-19):** the prior version of this skill (`drafting-workflow`) routed every tool call through a Pending Review tray in DocJacket. That tray has been retired — the chat conversation between you and the user **is** the approval gate. By the time you call a tool, the user has read your proposal and said yes, so DocJacket executes immediately. The old `prepare_*` / `propose_*` / `draft_*` / `suggest_*` tool names are deprecated; use the names in this skill.

## Architecture

You compose. The user confirms in chat. DocJacket executes.

Every `send_*` / `create_*` / `update_*` / `save_*` tool in this catalog takes a **finished** subject + body / payload and performs the action on call. DocJacket does not call an LLM on its side. The customer is paying for your composition through their Claude / ChatGPT / Codex / Cowork / Gemini subscription; DocJacket gets one cheap API call per action.

Tools return a uniform shape:

```json
{
  "status":        "sent" | "created" | "updated" | "saved",
  "transactionId": "uuid",
  "summary":       "Email sent to Jane Doe (buyer) — subject: \"Weekly update\".",
  "outcome":       "enqueued for jane@example.com via Gmail, outbox id=…",
  …tool-specific fields…
}
```

Show the `summary` to the user inline. The action is already done — no review step needed.

## When to use which tool

| User intent | Tool |
|---|---|
| "Send a weekly update to the buyer / seller on this deal" | `send_client_update` |
| "Ask the lender / inspector / title rep for a missing document" | `send_document_request` |
| "Email the listing agent about [topic]" (or any agent-to-agent on a specific topic) | `send_agent_followup` |
| "Draft an email to the [agent] covering these talking points" | `send_email_to_agent` |
| "Update / extend / move the [key date] to [new date]" | `update_key_date` |
| "Add a task to follow up on X" or "remind me to Y by Z" | `create_tasks` |
| "Summarize this transaction for me" or "save a status recap on the deal" | `save_status_summary` |

Three calls to keep close:

1. **Client update vs agent follow-up.** `send_client_update` is for buyer / seller communications — the recipient is the customer's client. `send_agent_followup` is for the network — listing agent, buyer agent, lender, title rep, referral source. Different tone, different liability, different default recipients.

2. **Document request vs agent follow-up.** If the email's primary purpose is *"please send me X"*, use `send_document_request` (it captures `documentTypes`, which feeds document tracking). Anything else agent-to-agent → `send_agent_followup`.

3. **Create tasks vs update key date.** "Follow up with the lender on Friday" → `create_tasks`. "Move the financing deadline to Friday" → `update_key_date`. The first creates work; the second moves a contractual milestone (cascades to scheduled reminders + FUB sync).

## Confirm before you call

Because there's no second approval step, you **must** confirm in chat before firing a `send_*` / `create_*` / `update_*` / `save_*` tool. Acceptable patterns:

- *"Here's the draft email to Jane: [subject + body]. Want me to send it?"* (user replies "yes" → call `send_client_update`)
- *"I'll move the inspection deadline from May 25 to June 1 based on the amendment. Sound right?"* (user replies "yes" → call `update_key_date`)
- *"I'll add a task: 'Follow up with lender about appraisal — due Friday.' OK?"* (user replies "yes" → call `create_tasks`)

If the user's instruction is unambiguous and self-contained (*"send the buyer the weekly update we just drafted"*), you can fire the tool without a separate confirmation turn. Use judgment.

## Required upstream context

Before composing, gather what you need. The customer's mail-MCP (Gmail / Outlook / Superhuman) handles their inbox; DocJacket handles deal state.

### Recipe — weekly buyer update

```
1. get_transaction({ id })                 — parties, key dates, current state
2. get_contacts({ transactionId })         — find the buyer contact
3. get_key_dates({ transactionId })        — what's coming up
4. get_open_tasks({ transactionId })       — what's still pending
5. (compose subject + body inline using the above)
6. (show user the draft, ask for confirmation)
7. send_client_update({
     transactionId, audience: "buyer", focus: "weekly",
     recipientContactId, subject, bodyHtml, bodyPlainText
   })
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

- **Don't fire a `send_*` / `update_*` tool without an explicit user yes.** The chat is the gate. If the user said "draft an email" without saying "send it", show the draft and wait. If the user said "send the email", you can call the tool.

- **Don't paste full external email content into `bodyHtml` or `sourceContextSummary`.** Summarize upstream context in one or two sentences. Quoting the original message inline in the new body is fine; archiving someone else's inbox in our database is not.

- **Don't call any tool without the `transactionId` being verifiably for the caller's org.** If you got the id from `search_transactions`, you're fine — that tool is org-scoped. If the user typed a UUID, validate with `get_transaction({ id })` first.

- **Don't fabricate dollar amounts or contractual terms.** Ground every number in `get_transaction` data or an explicit user statement.

- **Don't compose without grounding.** A draft to "Sarah" with no transaction lookup gets details wrong. Always call at least `get_transaction` + `get_contacts` first — they're cheap.

- **Don't say "Pending Review" or link `/prepared-work`.** That UI is retired. After a successful action, say "Done — sent to Jane" or "Done — moved the inspection deadline to June 1."

## Bundling

If the user asks for "an email to the buyer AND a task to follow up Friday," that's two tool calls — `send_client_update` + `create_tasks`. Two confirmations (or one combined "Sound good?") before firing. Don't try to combine them in a single tool.

## Output to the user

After a successful tool call:

```
✅ <summary>

You can see this on the transaction's activity feed.
```

For emails specifically, mention the recipient + subject so the user has a clear record of what just went out from their account:

```
✅ Email sent to Jane Doe (jane@example.com)
   Subject: "Weekly update — 8824 Clear View St."
```

## Send mode is decided by the org

When the customer's org is in **user-mailbox mode** (Option C), `send_*` tools route the email through the user's connected Gmail or Outlook account. The email leaves from the customer's actual address. Otherwise, the email goes via Postmark with the customer's brand. You don't control this — it's an org setting. If a `send_*` call fails because the user has user-mailbox mode but hasn't connected a mailbox, surface the error message and point them at Settings → Integrations.

## Migration from the old `prepare_*` / `propose_*` / `draft_*` tools

If your manifest still references the old names, the server returns HTTP 410 Gone with the new tool name in the error body. Treat that as a hard rename — call the new tool with the same arguments and the action succeeds.

| Old name | New name |
|---|---|
| `prepare_client_update` | `send_client_update` |
| `prepare_document_request` | `send_document_request` |
| `prepare_agent_followup` | `send_agent_followup` |
| `draft_email_to_agent` | `send_email_to_agent` |
| `suggest_task` / `propose_task` | `create_tasks` |
| `propose_key_date_update` | `update_key_date` |
| `draft_status_summary` | `save_status_summary` |
