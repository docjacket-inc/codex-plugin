# Common Real Estate Document Types

The canonical list of document categories DocJacket's classifier recognizes. The classifier (`Features/Extraction/Services/DocumentTypeClassifier.cs`) maps incoming filenames + content to one of these types via keyword matching (fast path) or LLM fallback (slow path). The `classify_document` MCP tool wraps it.

Use this reference when:
- Explaining a `classify_document` result to the user
- Disambiguating an ambiguous filename
- Mapping a TC's vocabulary to the typed system ("the CD" → `closing_statement`)

## Contract documents

| Slug | Common names | Notes |
|---|---|---|
| `purchase_agreement` | Purchase Agreement, Residential Contract, Real Estate Contract | The main contract |
| `amendment` | Amendment, First Amendment, Second Amendment | Modifies the original contract |
| `counterproposal` | Counter Offer, Counterproposal | Response to an offer |
| `amend_extend` | Amend & Extend, Extension Addendum | Specifically extends deadlines |
| `addendum` | Addendum, Lead Paint Addendum, HOA Addendum | Adds new terms |
| `listing_agreement` | Exclusive Listing Agreement, Seller Listing | Listing-side contract |

## Inspections & valuations

| Slug | Common names | Notes |
|---|---|---|
| `inspection_report` | Home Inspection, Pest Inspection, Radon Report | The inspector's report |
| `inspection_resolution` | Repair Response, Inspection Resolution | Seller's response to findings |
| `appraisal` | Appraisal Report, Property Appraisal | The appraiser's report |
| `appraisal_waiver` | Appraisal Waiver, Appraisal Contingency Waiver | Waives the appraisal contingency |

## Title & survey

| Slug | Common names | Notes |
|---|---|---|
| `title_commitment` | Title Commitment, Preliminary Title Report | Title company's clean-title agreement — **compliance-sensitive** |

## Lending

| Slug | Common names | Notes |
|---|---|---|
| `loan_approval` | Pre-Approval Letter, Loan Commitment, Clear to Close | Lender confirmation — **compliance-sensitive** |
| `closing_statement` | Closing Disclosure (CD), Settlement Statement, HUD-1, ALTA Statement | Final accounting — **compliance-sensitive** + 3-business-day rule |

## Disclosures

| Slug | Common names | Notes |
|---|---|---|
| `disclosure` | Seller's Property Disclosure, Lead-Based Paint Disclosure | State-specific requirements vary — **compliance-sensitive** |

## HOA & insurance

| Slug | Common names | Notes |
|---|---|---|
| `hoa_document` | HOA Documents, CC&Rs, HOA Estoppel, Resale Certificate | Required for HOA properties |
| `insurance` | Insurance Binder, Homeowner's Policy, Proof of Insurance | Lender requirement for funding |

## MLS / listing side

| Slug | Common names | Notes |
|---|---|---|
| `mls_sheet` | MLS Listing Sheet, MLS Data Sheet | Property facts pulled from MLS |
| `seller_intake` | Seller Listing Intake, Seller Information Form | Pre-listing seller info |

## Catch-all

| Slug | Notes |
|---|---|
| `unknown` | Classifier couldn't determine. Ask the user. |

## Compliance-sensitive types (require explicit user confirmation regardless of classifier confidence)

- `closing_statement` — CD has a 3-business-day delivery rule
- `title_commitment` — affects clear-to-close
- `loan_approval` — affects funding
- `disclosure` — required disclosure window varies by state

For these, the agent should always say _"looks like a [type] — is that right?"_ before treating it as final.

## Common TC vocabulary → slug mapping

| TC says | Slug |
|---|---|
| "the contract" | `purchase_agreement` |
| "the CD" | `closing_statement` |
| "the HUD" / "the HUD-1" | `closing_statement` |
| "the title work" | `title_commitment` |
| "the appraisal" | `appraisal` |
| "the inspection" | `inspection_report` |
| "the survey" | (no direct slug — track as task, not document type) |
| "lead paint" | `disclosure` or `addendum` (depending on form) |
| "MLS sheet" / "MLS print" | `mls_sheet` |
| "wire instructions" | (no direct slug — fraud-sensitive; track separately) |
| "HOA docs" / "HOA package" | `hoa_document` |
