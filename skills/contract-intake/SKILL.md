<!--
  Generated from plugins/shared/skills/contract-intake.md — do not hand-edit.
  Update the canonical file and re-sync. See plugins/README.md for the rule.
  Generator: hand-curated v1 (per spec §3.3). Last sync: 2026-05-19.
-->

---
name: contract-intake
title: Intake a contract PDF and build a complete transaction in one conversation
purpose: Walk a TC from "I just got a signed contract" to "the deal is fully set up in DocJacket" without leaving the chat — extract, review, apply, schedule, confirm.
applicable_to: [claude-desktop, codex, cowork, chatgpt, gemini]
tools_used:
  - upload_document_for_extraction
  - get_extraction_results
  - apply_extraction
  - add_key_dates_batch
  - apply_checklist
  - create_reminder
  - get_intake_status
  - render_email_template
  - send_email_to_agent
  - send_client_update
  - get_portal_link
expected_output: A new transaction in DocJacket with parties, key dates, a checklist, intro emails sent, reminders scheduled, and a portal link the TC can share. The TC never opened the web app.
required_scopes: [actions]
version: 1.0.0
---

# Contract Intake

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

```
1. upload_document_for_extraction({
     fileBase64,           // from the attached PDF
     filename,             // user-provided or "contract.pdf"
     state,                // "FL" | "NV" | ... (optional but boosts accuracy)
     side,                 // "Buyer" | "Seller" (optional; inferred from extraction)
     documentTypeHint      // "purchase_agreement" | "listing_agreement" | ...
   })
   → { extractionJobId, status: "queued", estimatedSecondsToComplete, cacheHit }
```

If `cacheHit` comes back true, the same PDF was uploaded before — the response includes the prior job's complete results immediately. Skip Phase 2's polling loop.

### Phase 2 — Poll until extraction completes

```
2. get_extraction_results({ extractionJobId })
   → { status: "processing", progressPercent: 35, ... }
3. (wait 2-3 seconds)
4. get_extraction_results({ extractionJobId })
   → { status: "complete", fullResult: { parties, dates, financials, ... } }
```

**Polling cadence: every 2-3 seconds. Max wait: 10 minutes** (typical job is 50-350s; the max-wait is for hung-job safety). While polling, update the user with a one-line "still working…" if more than ~15 seconds pass — don't go silent.

If `status: "failed"`, surface the `errorMessage` and ask whether to retry with `upload_document_for_extraction` (same file) or escalate.

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

Present both drafts to the TC. On approval:

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

Reminders fire via DocJacket's Hangfire scheduler — they go out whether the chat is open or not. The TC can set them and forget them.

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
   • 2 intro emails sent
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
| `no_buyer_agent` / `no_listing_agent` / `no_title_company` | `add_contact_to_transaction` |
| `no_reminders_scheduled` | `create_reminder` |

If the conversation resumes on a different surface (TC switched from Claude iOS to Claude Desktop, or moved to Codex), the `transactionId` is your durable handle. Use `search_transactions` or `find_transaction_by_property` to re-locate, then `get_intake_status` to re-orient.

## Idempotency

- **`upload_document_for_extraction`** is content-hash deduped (90-day cache). A retry returns `cacheHit: true` with the prior job's ID.
- **`apply_extraction`** is race-safe — a second call after success returns the same `transactionId` (response shows `status: "already_applied"`).
- **`create_reminder`** dedups on `(deadlineType, daysBeforeReminder)` per transaction; duplicate call returns `DUPLICATE_REMINDER`.
- **`add_key_dates_batch`** doesn't dedup at the batch level but each row is idempotent — re-adding the same `keyDateType` updates the existing slot.

So retries are safe across all the write tools. If you suspect a partial-failure state, call `get_intake_status` first and only retry the specific gaps.

## Anti-patterns (don't)

- **Don't skip Phase 3 confirmation.** Even when extraction looks clean, the TC sees fields the LLM doesn't — names spelled wrong, an emailed-from-counsel addendum the contract refers to, etc. Always present and confirm before `apply_extraction`.
- **Don't paste the entire extracted PDF text** into chat as part of the confirmation. Paraphrase. The TC wants the 8-10 key fields, not 30 pages.
- **Don't pre-populate `overrides` with every field** "just to be safe." Only include keys the user actually edited. Per-field merge means absent keys flow through unchanged — extra noise wastes tokens and slows the LLM's reasoning on the next round.
- **Don't fabricate dates the contract doesn't state.** If the contract has no walkthrough date, leave it absent. The TC can add it later via `add_key_dates_batch` once they've negotiated it.
- **Don't call `apply_extraction` more than once** for the same `extractionJobId` expecting a different result. It's idempotent — same job, same transaction. If the TC wants a NEW transaction from the same PDF, that's a workflow gap (escalate).
- **Don't loop polling forever.** Cap at 10 minutes wall-clock. If extraction hasn't completed by then, something's wrong — surface the issue.
- **Don't apply the checklist twice.** `apply_extraction` auto-applies the default state/type/side template. Calling `apply_checklist` again with the same template creates duplicates. Use `replaceExisting: true` if the TC explicitly wants to swap templates.

## Migration from the wizard

If a TC's prior workflow was the web app's upload wizard, this chat flow lands them at the same final state — same transaction row, same parties, same key dates, same auto-applied checklist. The wizard runs DocDrop auto-create + doc rename + listing-data parsing as extra bells that this flow doesn't (today). If the TC asks why their MCP-built transaction lacks DocDrop, file it as a future enhancement and offer to set up DocDrop manually via the web app's Settings.

## Tool-by-tool reference

| Tool | Scope | When |
|---|---|---|
| `upload_document_for_extraction` | Actions / `upload_document` | Phase 1 — kick off the pipeline |
| `get_extraction_results` | Read / `documents:read` | Phase 2 — poll for completion |
| `apply_extraction` | Actions / `transactions:create` | Phase 3 — confirm and create the transaction |
| `add_key_dates_batch` | Draft / `key_dates:propose` | Phase 4 — batch-add timeline dates |
| `apply_checklist` | Draft / `prepared_work:create` | Phase 4 (rare) — swap or add a checklist |
| `render_email_template` | Read / `emails:prepare` | Phase 4 — preview drafts before send |
| `send_client_update` / `send_email_to_agent` | Actions / `send_email` | Phase 4 — send intros |
| `create_reminder` | Actions / `reminders:create` | Phase 4 — schedule durable nudges |
| `get_intake_status` | Read / `transactions:read` | Phase 5 / recovery — final check or re-orientation |
| `get_portal_link` | Read / `transactions:read` | Phase 5 — share with parties |
