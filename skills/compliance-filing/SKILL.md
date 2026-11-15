---
name: compliance-filing
description: "Gets a DocJacket deal into the brokerage's compliance system (such as Dotloop, SkySlope or KW Command): finds which system the deal files in, prepares the deal's fields, parties and document names for that system, checks signed PDFs for blank signature, initials and date lines, and emails chosen documents to a Dotloop or SkySlope email-in address after the user approves a preview. Use when the TC asks to file, submit or set up a deal in Dotloop, SkySlope or another compliance system, or to check a document for missing signatures."
license: MIT
---

<!--
docjacket-title: File a deal in the brokerage's compliance system and check signatures
docjacket-tools: find_transaction_by_property get_compliance_system set_compliance_system prepare_compliance_handoff list_documents review_signatures get_signature_review send_compliance_documents confirm_email_send undo_setting_change
docjacket-scopes: read actions
docjacket-version: 1.0.0
-->

# Compliance Filing

Brokerages make their agents file each deal in a compliance system. DocJacket can prepare everything that system asks for and, for Dotloop and SkySlope, email the documents in. It never logs in to those systems and never stores their passwords.

## When this fires

- "File the Oak St deal in Dotloop", "set this deal up in SkySlope", "get this ready for compliance"
- "Which system does this agent use?", "Mike's brokerage uses KW Command"
- "Check the purchase agreement for missing signatures", "did everyone initial every page?"
- "Send the signed docs to the loop"

## Step 1 — Which system

1. Find the deal (`find_transaction_by_property` or the deal in context).
2. **`get_compliance_system`** reads the system from the deal's agent first, then the agent's brokerage. Status is `set`, `not_set` or `conflict`.
3. On `not_set` or `conflict`, ask the user which system they file in. Save only their answer with **`set_compliance_system`**: pass `contactId` (the agent) or `companyId` (the brokerage), exactly one, and `system` (null clears it). Save it on the agent unless the user says the whole brokerage uses it. Never save DocJacket's suggestion on your own. The change can be taken back for 7 days with `undo_setting_change`.

## Step 2 — Prepare the handoff

**`prepare_compliance_handoff`** returns the deal arranged for its system: fields under plain names (county, earnest money holder, commissions, parcel id), parties with roles, each document with the name it should carry there, how documents can be delivered, and for Dotloop short tested steps that stop before anything is shared or submitted.

- With no saved system it returns `system_not_set` and no packet: go back to Step 1.
- Pass `system` only to try a different system for one call; nothing is saved.
- It returns no document links or file contents and sends nothing.

Present the packet as a short checklist the TC can work through in the other system.

## Step 3 — Check signatures (optional, costs money)

1. Pick the PDF (`list_documents`). Use this only for a signed or partly signed form.
2. Tell the user each new review counts against the organization's signature review allowance and costs money (an unchanged file that was already reviewed returns the earlier result at no cost). Call **`review_signatures`** only on their yes. It returns a `runId` with status `queued`.
3. **`get_signature_review`** with that `runId` (or the transactionId and documentId). Poll every 20 to 30 seconds while the status is `queued` or `running`. Don't call `review_signatures` again to refresh.
4. Report each flag in plain words: page, party, what is missing (signature, initials, date) and whether it is missing or only a note. Confirming or dismissing a flag happens in DocJacket.

## Step 4 — Email documents to Dotloop or SkySlope

Only when `get_compliance_system` says the deal files in Dotloop or SkySlope.

1. Ask the user which documents go and the exact email-in address. For Dotloop, each folder of a loop has its own `@upload.dotloop.com` address and the user must give it to you; SkySlope uses the file's `@skyslope.com` address. **Never invent or guess an address.**
2. **`send_compliance_documents`**. It sends nothing on its own:
   - `awaiting_confirmation`: show the COMPLETE preview, with every attachment under its new name and the address. Call **`confirm_email_send`** with the `confirmCode` only when the user explicitly says yes.
   - `parked_for_review`: the email has NOT been sent; it is waiting for approval in DocJacket. Say so.
3. The email goes from the user's connected Gmail or Outlook mailbox. Report it as sent only when `confirm_email_send` returns `queued` (see `execution-workflow` for the other results).

Never use `send_compliance_documents` for any other recipient or system.

## Guardrails

- Ask before saving a compliance system, before a paid signature review, and before any send.
- DocJacket does not submit or share anything inside the compliance system; the TC does that step.
- Never look up or ask for logins or passwords for compliance systems.

## Tool reference

| Tool | Scope | Use |
|---|---|---|
| `get_compliance_system` | Read | Which system the deal files in |
| `set_compliance_system` | Actions | Save the user's answer on the agent or brokerage (7-day undo) |
| `prepare_compliance_handoff` | Read | Fields, parties and document names for that system |
| `review_signatures` | Actions | Start a paid signature and initials review of a PDF |
| `get_signature_review` | Read | Poll the review and read its flags |
| `send_compliance_documents` | Actions | Preview an email of chosen PDFs to a Dotloop or SkySlope address |
| `confirm_email_send` | Actions | Send it after the user's explicit yes |
