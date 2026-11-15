---
name: document-filing
description: Helps the TC identify what each attachment is and which DocJacket transaction it belongs to. DocJacket's classifier suggests a document type; the TC decides where it is filed. Produces a "[filename] looks like [type] for [transaction]" prompt that notes when it fills a gap on the missing-documents list, with an optional follow-up task so the document gets uploaded. Use when the TC has attachments or documents to identify, classify or file to a deal.
license: MIT
---

<!--
docjacket-title: Classify documents and propose filing them to the right transaction
docjacket-tools: classify_document get_missing_documents search_transactions find_contact_by_email create_tasks
docjacket-scopes: read
docjacket-version: 1.0.0
-->

# Document Filing

## When this fires

The TC says any of:
- "What kind of document is this?"
- "Where should I file [filename]?"
- "File this attachment"
- "Save this PDF to [transaction]"

Also fires as a follow-up to `email-triage` when the TC asks about a matched attachment.

## The recipe

### Step 1 — Identify the target transaction

If the TC didn't name it:
1. Ask. Or, if you came from email-triage, use the matched transaction from there.
2. If they say "the Smith deal" — call **`search_transactions`** with "Smith" and confirm.

### Step 2 — Classify the document

Call **`classify_document`** with:
- `filename` — the original filename
- `content_preview` — first ~500 chars of OCR/extracted text if available

The classifier returns a `suggestedType`, a confidence value, and `isHighConfidence: true/false`.

### Step 3 — Check whether it fills a gap

If you have a transaction, call **`get_missing_documents`**. Compare the classifier's `suggestedType` to the missing-docs list.

### Step 4 — Present the suggestion

Show the TC:

```
InspectionReport_412Oak.pdf looks like an Inspection Report for 412 Oak St (Smith).
This fills a gap on the missing-docs list.
```

If `isHighConfidence` is FALSE OR the type is compliance-sensitive (see list below), say so explicitly:

```
The filename / content is ambiguous — could be Title Commitment or Title Insurance Policy. Which is correct?
```

### Step 5 — Filing path (today)

**No MCP tool files a document onto a deal.** `create_upload_link` and the other upload tools are only for a contract going through extraction (see `contract-intake`). Filing requires the TC to upload the file in the DocJacket UI. Tell them:

> _"Upload it in DocJacket → Documents → 412 Oak St. I'll create a task to track that — okay?"_

If they say yes, call **`create_tasks`** with a task like:

```json
{
  "transactionId": "<guid>",
  "tasks": [{
    "name": "Upload InspectionReport_412Oak.pdf — Inspection Report",
    "priority": "Medium",
    "dueDate": "<today + 1 day>"
  }]
}
```

That gives the TC a tracked item and an audit trail.

### Step 6 — If the attachment came from email-triage

Bundle the filing-task with the follow-up email. One confirmation: _"I'll thank Mike for the inspection report + add a task to upload it to DocJacket. Sound good?"_ → on yes, call `send_agent_followup` + `create_tasks`. Show the email preview and send it with `confirm_email_send` only on the user's explicit yes (or tell them it is waiting for approval if it was `parked_for_review`). The task is created right away.

## Compliance-sensitive types — always confirm

Even when `isHighConfidence: true`, always confirm before treating these as final:

- **Closing Disclosure (CD) / closing_statement** — federal 3-business-day rule
- **Title Commitment / title_commitment** — affects clear-to-close
- **Wire Instructions** — fraud target; verify by phone, never AI-only
- **Settlement Statement (HUD-1)** — final accounting
- **Loan Approval / loan_approval / Clear-to-Close letter** — affects funding
- **Seller's Disclosure / disclosure** — state-specific required disclosure window

For these, classification is a SUGGESTION the TC ratifies — never an assumption.

## Ambiguous filenames — examples

| Filename | Action |
|---|---|
| `scan001.pdf` | Ask the TC. No keyword signal. |
| `signed.pdf` | Ask which document. |
| `final.pdf` | Ask which document. |
| `addendum.pdf` | Could be any addendum — ask which. |
| `InspectionReport.pdf` | Classify as `inspection_report`; confirm. |
| `TitleCommitment_412Oak.pdf` | Classify as `title_commitment`; ALWAYS confirm (compliance-sensitive). |

## Common real-estate document types

See [`references/document-types.md`](references/document-types.md) for the canonical list of types `classify_document` recognizes.

## Don't do

- **Don't guess on compliance-sensitive types.** Ask.
- **Don't claim a document was filed.** No upload tool yet. Use language like _"looks like a [type] — please upload manually."_
- **Don't classify a document the TC didn't ask about.** Triage shows what's matched; classification happens on request.
- **Don't tell the TC the confidence score.** _"Looks like an inspection report"_ — not _"95% confident."_
