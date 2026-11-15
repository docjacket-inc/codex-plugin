---
name: contract-intake
description: >-
  Walks a TC from a signed contract PDF to a fully set-up DocJacket transaction without leaving
  the chat: extract, review, apply, schedule, confirm. The result is a new transaction with
  parties, key dates, a checklist, intro emails previewed and confirmed (or waiting for a team
  member's approval), reminders scheduled, and a portal link the TC can share. Use when the TC has
  a new signed contract to intake or asks to build a deal from a contract.
license: MIT
metadata:
  docjacket-title: "Intake a contract PDF and build a complete transaction in one conversation"
  docjacket-tools: "list_documents extract_existing_document create_upload_link get_upload_link_status list_new_deal_intake request_upload_url kick_off_extraction upload_document_for_extraction get_extraction_results apply_extraction create_transaction add_key_dates_batch apply_checklist create_reminder get_intake_status render_email_template send_email_to_agent send_client_update confirm_email_send search_contacts create_contact add_contact_to_transaction create_deal_calendar_event send_key_date_invites preview_playbook apply_playbook get_portal_link"
  docjacket-scopes: "actions"
  docjacket-version: "1.3.0"
---

# Contract Intake

## No document? Use `create_transaction`

If the user wants a deal opened with **no contract to read** — "set up blank listings for units 1-4 at 412 Harbor View", a list of addresses from a spreadsheet or screenshot — skip extraction entirely and call `create_transaction`:

1. Build the list (street, unit, city, state, zip, type — default `Listing` — side, optional status/price/agent).
2. **Show the user the exact list and get a yes.** The deals are created the moment you call.
3. Call `create_transaction({ transactions: [...] })` — at most 25 per call.
4. Report each item back: `created` (new `transactionId`), `skipped_duplicate` (the deal already exists — `existingTransactionId`), or `error`. A repeat call is safe: already-created deals come back as `skipped_duplicate`.

A matching Playbook is auto-applied exactly as `apply_extraction` does.

## When this fires

Trigger on any variation of:

- "Intake this contract"
- "Set up a new deal for [address]" (with a PDF attached)
- "Open this file" / "Start this transaction" (when a PDF is in the conversation)
- "Build out this purchase agreement"
- "Process this listing agreement"

If the user names a state or side ("Florida buyer-side", "Nevada seller"), capture it — pass to `state` and `side` on the upload call so state-specific extraction tunes itself.

## The recipe (canonical end-to-end)

The flow has five phases. Each ends with the LLM confirming the result before the next call. Don't batch the whole thing — the TC needs to see and approve at each step.

### Phase 1 — Upload + extract

There are five paths, in this order. Pick by where the PDF is and what the MCP client supports.

**Path A — Already in DocJacket (canonical, try this first).** The TC uploaded the PDF earlier via DocDrop, email intake, the web wizard, or a prior chat call. No bytes move; DocJacket already has them. Works in every MCP client (Cowork, Codex, ChatGPT, Claude.ai free or paid, Gemini).

```
1a. list_documents({ hasExtraction: false, limit: 20 })   // discovery
    → { documents: [{ documentId, filename, uploadDate, transactionAddress, ... }] }

1b. (Confirm with the TC: "Found 3 documents uploaded today — extract 'fl-buyer-purchase.pdf'?")

1c. extract_existing_document({
      documentId,           // from list_documents
      state,                // "FL" | "NV" | ... (optional; overrides stored)
      side,                 // "Buyer" | "Seller" (optional)
      documentTypeHint      // optional; falls back to stored DocumentType
    })
    → { extractionJobId, status: "queued", estimatedSecondsToComplete, cacheHit }
```

When the TC says "extract the contract I uploaded last night" or "process the file from this morning," this is the path. Always call `list_documents` first to surface candidates rather than guessing.

**Path B — Not in DocJacket yet: an upload link (default in every client).** No code sandbox, no paid plan, and no bytes in chat.

```
2a. create_upload_link({ filename?, state?, side?, documentTypeHint?, transactionId? })
    → { uploadLinkId, url }: tell the user to open the url in the browser where they use DocJacket and drop the PDF there
2b. get_upload_link_status({ uploadLinkId })   // waits on the server up to waitSeconds (default 30, max 45)
    → waiting | uploading → call again straight away
    → submitted → { extractionJobId } → continue with Phase 2
    → failed (relay errorCode) | expired (create a new link)
```

**Path C — Not in DocJacket yet: email it.** Ask the user to email or forward the PDF, from their own email address, to their new-deal address: `get_organization_summary` → `intake_addresses.new_deal.address`. Email from anyone else waits in the Review Queue. Then poll `list_new_deal_intake` to see what became of it. No bytes pass through chat, and it needs no paid plan or code execution.

**Path D — Attached to chat, code sandbox that can reach `uploadHost`.** TC drops a PDF directly in chat that isn't in DocJacket yet. Two-step presigned upload: bytes go from the LLM's sandbox to blob storage via curl, never touching chat tokens.

```
2a. request_upload_url({ filename: "contract.pdf" })
    → { uploadId, uploadUrl, method: "PUT", headers, expiresAt }

2b. (In the code-execution sandbox:)
    bash> curl -X PUT -T /mnt/user-data/uploads/contract.pdf \
            -H "Content-Type: application/pdf" \
            "<uploadUrl>"

2c. kick_off_extraction({ uploadId, filename, state?, side?, documentTypeHint? })
    → { extractionJobId, status: "queued", estimatedSecondsToComplete, cacheHit }
```

Requires a code-execution sandbox whose network allowlist includes the host returned as `uploadHost` (an R2 storage host, not mcp.docjacket.com). If 2b fails with a 403 / network / DNS error, the sandbox cannot reach `uploadHost` — stop and use Path B (upload link) or Path C (email) instead; do not retry.

**Path E — Attached to chat, anywhere else (last resort).** Tiny PDFs (~under 1 MB) attached to chat with no other option. Bytes ride in chat context for the rest of the conversation; compounds with later turns, so only use when paths A to D don't apply.

```
3. upload_document_for_extraction({ fileBase64, filename, state?, side?, documentTypeHint? })
   → { extractionJobId, status: "queued", estimatedSecondsToComplete, cacheHit }
```

If `cacheHit` comes back true (any path), the same PDF was extracted before — the response includes the prior job's complete results immediately. Skip Phase 2's wait.

### Phase 2 — Wait until extraction completes

```
2. get_extraction_results({ extractionJobId })
   → (the server waits up to 30 seconds) { status: "processing", progressPercent: 35, ... }
3. get_extraction_results({ extractionJobId })   ← straight away, no pause
   → { status: "complete", fullResult: { parties, dates, financials, ... } }
```

**The call waits on the server** (`waitSeconds`, default 30, max 45) and answers as soon as the job finishes. While the status is still pending or processing, call it again straight away; don't sleep between calls. **Stop after 10 minutes in total** (typical job is 50-350s; the cap is for hung-job safety). Between calls, update the user with a one-line "still working…" — don't go silent.

If `status: "failed"`, surface the `errorMessage` and ask whether to retry the upload (Phase 1 again, same file) or escalate.

### Phase 3 — Present extracted fields, get confirmation, apply

Show the TC the key fields from `fullResult` in a compact form. Don't dump JSON — paraphrase.

```
Here's what I extracted:
  Property:    1234 Main St, Tampa FL 33602
  Buyer:       Sarah Miller (sarah@example.com)
  Seller:      John Doe
  Closing:     June 15, 2026
  EMD:         $5,000 due 3 days after contract
  Inspection:  10-day period
  Financing:   Conventional, $340k loan
  Buyer agent: Mike Park (mike@parkrealty.com)

Look right, or anything to change?
```

The TC may say "change closing to June 20" or "Sarah's email is sarah@gmail.com not sarah@example.com" or "looks good."

```
5. apply_extraction({
     extractionJobId,
     overrides: { closingDate: "2026-06-20", buyerEmail: "sarah@gmail.com" },
     transactionType: "Purchase",  // optional; defaults to Purchase
     side: "Buyer"                  // optional
   })
   → { transactionId, applied: true, linkedContacts: 4,
       autoAppliedChecklist: { name: "Florida Buyer Purchase", id: "..." },
       summary: "..." }
```

The response includes `transactionId` — **pin every subsequent tool call to this id.**

Overrides are a flat camelCase map; only fields you list replace extracted values. Absent fields flow through unchanged. Known keys: `propertyAddress`, `propertyCity`, `propertyState`, `propertyZip`, `county`, `propertyType`, `contractDate`, `closingDate`, `inspectionDeadline`, `financingDeadline`, `appraisalDeadline`, `titleDeadline`, `dueDiligenceDeadline`, `possessionDate`, `earnestMoneyDueDate`, `effectiveDate`, `surveyDeadline`, `buyerName`, `buyerEmail`, `buyerPhone`, `buyer2*`, `sellerName`, …, `purchasePrice`, `earnestMoney`, `loanAmount`, `downPayment`, `financingType`, `buyerAgentName`, `buyerAgentEmail`, `buyerAgentPhone`, `buyerAgentLicense`, `buyerAgentBrokerage`, `sellerAgent*`, `titleCompanyName`, `titleContactName`, `titlePhone`, `titleEmail`.

### Phase 4 — Fill in the timeline + send intros + schedule reminders

`apply_extraction` already created any key dates the contract stated explicitly + auto-applied the default checklist. Now offer to fill in computed dates and intro communications.

Build a timeline using contract knowledge:

```
Based on this contract, here's the timeline I'd suggest:
  EMD due:           May 22 (3 business days from contract)
  Inspection ends:   May 29 (10 days from acceptance)
  Appraisal due:     June 5
  Loan approval:     June 13 (2 days before CD)
  CD must arrive:    June 17 (3 business days before closing)
  Final walkthrough: June 19
  Closing:           June 20

Want me to add these as key dates and send intro emails to the buyer + listing agent?
```

TC says yes (or edits — *"move walkthrough to June 18"*):

```
6. add_key_dates_batch({
     transactionId,
     keyDates: [
       { keyDateType: "EarnestMoneyDue",      dateValue: "2026-05-22" },
       { keyDateType: "InspectionDeadline",   dateValue: "2026-05-29" },
       { keyDateType: "AppraisalDeadline",    dateValue: "2026-06-05" },
       { keyDateType: "FinancingDeadline",    dateValue: "2026-06-13" },
       { keyDateType: "FinalWalkthrough",     dateValue: "2026-06-18" }
     ]
   })
   → { created: 5, failed: [] }
```

Per-row failures (e.g. unknown key date type) come back in the `failed` array — surface them to the TC: *"Added 5 of 6. 'survey_deadline' isn't recognized — want me to add it as a custom key date?"*

Then send the intro emails. Use `render_email_template` FIRST to show the TC drafts before sending:

```
7. render_email_template({ templateId: "intro-buyer", transactionId })
   → { subject, bodyHtml, unresolvedFields: [] }
8. render_email_template({ templateId: "intro-listing-agent", transactionId })
   → { subject, bodyHtml, unresolvedFields: [] }
```

Present both drafts to the TC. On approval, call the send tools. Nothing is sent by these calls. Each returns either `status: "awaiting_confirmation"` with a `preview` and `confirmCode`, or `status: "parked_for_review"` (waiting for a team member in the DocJacket review queue; a normal outcome, not an error). For each `awaiting_confirmation`, show the TC the complete preview and call `confirm_email_send` with the code only on their explicit yes (`decision: "discard"` on no). Never tell the TC an intro is on its way unless `confirm_email_send` returned `queued`. Details and error codes are in `execution-workflow`:

```
9.  send_client_update({ transactionId, audience: "buyer", ... })
10. send_email_to_agent({ transactionId, recipientRole: "listing_agent", ... })
```

If the TC says "set up the standard reminders" or "remind the agent before EMD":

```
11. create_reminder({
      transactionId,
      deadlineType: "EarnestMoneyDue",
      deadlineDate: "2026-05-22",
      daysBeforeReminder: 2,
      remindWho: "party",
      partyRole: "buyer_agent",
      channel: "email"
    })
    → { reminderId, scheduledForUtc, summary }
```

Reminders fire via DocJacket's Hangfire scheduler, whether the chat is open or not. A reminder created here does not send on its own: when it comes due, its email waits in the DocJacket review queue until a team member approves it. Tell the TC to approve it there.

### Phase 5 — Confirm + share portal

```
12. get_intake_status({ transactionId })
    → { propertyAddress, parties, keyDates: { count, types[] },
        checklist: { applied, name, itemsTotal },
        reminders: { count, nextScheduledFor },
        missingRecommendedSteps: ["no_inspection_deadline"]  // empty when complete
      }

13. get_portal_link({ transactionId })
    → { url, calendarFeedUrl, ... }
```

Show the TC the final summary:

```
✅ Done. Transaction created for 1234 Main St (Tampa FL).
   • 4 contacts linked (buyer, seller, both agents)
   • 7 key dates scheduled
   • 24 checklist tasks applied (Florida Buyer Purchase)
   • 2 intro emails approved and on their way (or: waiting for approval in DocJacket, not sent yet)
   • 2 reminders scheduled (EMD nudge for buyer's agent + TC)
   • Portal link: <url>
```

## Recovery — when something goes wrong mid-flow

If a follow-up call partially fails (`add_key_dates_batch` returns a non-empty `failed` array; `apply_checklist` returns errors; `create_reminder` returns `INVALID_CHANNEL_FOR_ORG`), don't restart. Call:

```
get_intake_status({ transactionId })
```

The response's `missingRecommendedSteps[]` tells you exactly what to retry:

| Step code | Fix tool |
|---|---|
| `no_checklist_applied` | `apply_checklist` |
| `no_closing_date` / `no_inspection_deadline` | `add_key_dates_batch` |
| `no_buyer_agent` / `no_listing_agent` / `no_title_company` | `search_contacts` first; `add_contact_to_transaction` if found, `create_contact` then `add_contact_to_transaction` if not |
| `no_reminders_scheduled` | `create_reminder` |

If the conversation resumes on a different surface (TC switched from Claude iOS to Claude Desktop, or moved to Codex), the `transactionId` is your durable handle. Use `search_transactions` or `find_transaction_by_property` to re-locate, then `get_intake_status` to re-orient.

## Idempotency

- **`extract_existing_document`**, **`kick_off_extraction`**, and **`upload_document_for_extraction`** all share the same content-hash dedup (90-day cache). A retry of the same PDF — via any path — returns `cacheHit: true` with the prior job's ID.
- **`apply_extraction`** is race-safe — a second call after success returns the same `transactionId` (response shows `status: "already_applied"`).
- **`create_reminder`** dedups on `(deadlineType, daysBeforeReminder)` per transaction; duplicate call returns `DUPLICATE_REMINDER`.
- **`add_key_dates_batch`** doesn't dedup at the batch level but each row is idempotent — re-adding the same `keyDateType` updates the existing slot.

- **Request ids.** Send a request id on every write: over MCP set the header `X-DocJacket-Client-Request-Id` to a fresh UUID per intended action, and if a call times out retry with the SAME id so DocJacket replays the first result instead of acting twice. A new id means a new action. Details in `execution-workflow` (Retries and ids). If your client cannot set the header, rely on the natural-key dedup above.

If you suspect a partial-failure state, call `get_intake_status` first and only retry the specific gaps. Get contact, key date and task ids from lookup reads (`search_contacts`, `get_key_dates`, `get_open_tasks`); never guess them.

## Anti-patterns (don't)

- **Don't skip Phase 3 confirmation.** Even when extraction looks clean, the TC sees fields the LLM doesn't — names spelled wrong, an emailed-from-counsel addendum the contract refers to, etc. Always present and confirm before `apply_extraction`.
- **Don't paste the entire extracted PDF text** into chat as part of the confirmation. Paraphrase. The TC wants the 8-10 key fields, not 30 pages.
- **Don't pre-populate `overrides` with every field** "just to be safe." Only include keys the user actually edited. Per-field merge means absent keys flow through unchanged — extra noise wastes tokens and slows the LLM's reasoning on the next round.
- **Don't fabricate dates the contract doesn't state.** If the contract has no walkthrough date, leave it absent. The TC can add it later via `add_key_dates_batch` once they've negotiated it.
- **Don't call `apply_extraction` more than once** for the same `extractionJobId` expecting a different result. It's idempotent — same job, same transaction. If the TC wants a NEW transaction from the same PDF, that's a workflow gap (escalate).
- **Don't wait forever.** Cap at 10 minutes wall-clock. If extraction hasn't completed by then, something's wrong — surface the issue.
- **Don't apply the checklist twice.** `apply_extraction` auto-applies the default state/type/side template. Calling `apply_checklist` again with the same template creates duplicates. Use `replaceExisting: true` if the TC explicitly wants to swap templates.

## Migration from the wizard

If a TC's prior workflow was the web app's upload wizard, this chat flow lands them at the same final state — same transaction row, same parties, same key dates, same auto-applied checklist. The wizard runs DocDrop auto-create + doc rename + listing-data parsing as extra bells that this flow doesn't (today). If the TC asks why their MCP-built transaction lacks DocDrop, file it as a future enhancement and offer to set up DocDrop manually via the web app's Settings.

## Tool-by-tool reference

| Tool | Scope | When |
|---|---|---|
| `list_documents` | Read / `documents:read` | Phase 1a — Path A — find docs already in DocJacket |
| `extract_existing_document` | Actions / `upload_document` | Phase 1c — Path A — canonical: extract a doc already in DocJacket |
| `create_upload_link` | Actions / `upload_document` | Phase 1 — Path B — default for a PDF not in DocJacket: the user drops it on a link |
| `get_upload_link_status` | Read / `documents:read` | Phase 1 — Path B — wait for the upload; returns `extractionJobId` |
| `list_new_deal_intake` | Read / `transactions:read` | Phase 1 — Path C — what became of an emailed contract |
| `request_upload_url` | Actions / `upload_document` | Phase 2a — Path D — code sandbox that can reach uploadHost only |
| `kick_off_extraction` | Actions / `upload_document` | Phase 2c — Path D — start pipeline after sandbox PUT |
| `upload_document_for_extraction` | Actions / `upload_document` | Phase 3 — Path E — base64 last resort for tiny chat-attached PDFs |
| `get_extraction_results` | Read / `documents:read` | Phase 2 — wait for completion (the call waits on the server) |
| `apply_extraction` | Actions / `transactions:create` | Phase 3 — confirm and create the transaction |
| `create_transaction` | Actions / `transactions:create` | No-document path — blank deals from an address list (1-25) |
| `add_key_dates_batch` | Draft / `key_dates:propose` | Phase 4 — batch-add timeline dates |
| `apply_checklist` | Draft / `prepared_work:create` | Phase 4 (rare) — swap or add a checklist |
| `render_email_template` | Read / `emails:prepare` | Phase 4 — preview drafts before send |
| `send_client_update` / `send_email_to_agent` | Actions / `send_email` | Phase 4 — preview the intros (nothing is sent by the call) |
| `confirm_email_send` | Actions | Phase 4 — send (or discard) a previewed intro, only after the TC's explicit yes |
| `search_contacts` / `create_contact` / `add_contact_to_transaction` | Read / Draft | Phase 4 / recovery — find, create only if none found, link to the deal |
| `create_deal_calendar_event` | Draft | Optional — closing or walkthrough on the TC's calendar (invites nobody) |
| `send_key_date_invites` | Actions | Optional — calendar invites to deal contacts; preview first, then confirm with the `confirmCode` |
| `preview_playbook` / `apply_playbook` | Read / Draft | Optional — preview then apply a Playbook to the deal |
| `create_reminder` | Actions / `reminders:create` | Phase 4 — schedule durable nudges |
| `get_intake_status` | Read / `transactions:read` | Phase 5 / recovery — final check or re-orientation |
| `get_portal_link` | Read / `transactions:read` | Phase 5 — share with parties |
