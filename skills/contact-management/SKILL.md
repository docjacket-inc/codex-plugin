---
name: contact-management
description: "Manages DocJacket contacts and links them to transactions: search, get, create and link. Use when the TC mentions contacts, parties, adding someone to a transaction, looking up an agent / lender / inspector / title company, or when a new party needs to be added during intake or email triage."
license: MIT
---

<!--
docjacket-title: Manage DocJacket contacts and link them to transactions
docjacket-tools: search_contacts get_contacts find_contact_by_email create_contact create_contact_role add_contact_to_transaction search_transactions find_transaction_by_property render_email_template
docjacket-scopes: read actions
-->

# Contact Management

Four building blocks: search, get, create, link. Combine them for the common
flows below. Never duplicate a contact — always search first. Get every `contactId` from `search_contacts`, `get_contacts` or `find_contact_by_email`; never guess one. Send a request id (`X-DocJacket-Client-Request-Id`) on writes and retry with the same one (see `execution-workflow`).

## Looking up a contact

- **Free-text search** (name, email fragment, company, phone digits):
  `search_contacts(query: "...", contactType: "Agent" | "Lender" | ...)`.
  Returns the compact picker list.
- **Exact email** (definitive single result from inbound mail):
  `find_contact_by_email(email: "...")`. Use this first when matching an
  inbound email to a transaction.
- **Full detail** (after you have a `contactId`):
  `search_contacts(contactId: "...")`. The `contact` field has identity,
  address, license, every active transaction the contact is a party on, and a
  recent activity timeline (`activityLimit`, default 25).

## Adding a new contact

Always search first. The system will deduplicate on email even if you don't,
but the search-first flow gives the TC visibility into the match.

1. `search_contacts(query: "Jane Buyer")` — confirm no match.
2. `create_contact(firstName, lastName, email, phone, company, contactType,
   kind)`. Either `email` OR (`firstName` + `lastName`) is required;
   `kind: "Organization"` lets you create a company-only contact (title co,
   lender entity) with `company` alone.
3. If `created: false` and `reason: "existing_email_match"` comes back, the
   contact was already in the system — `contactId` points at the existing row.
   Use it as-is.

## Linking an existing contact to a transaction

```
add_contact_to_transaction(
  contactId: "<from search_contacts / find_contact_by_email / create_contact>",
  transactionId: "<from search_transactions / find_transaction_by_property>",
  role: "<see roles below>",
  isPrimary: <true|false>
)
```

**Idempotent.** Calling with the same `(transaction, contact, role)` returns
the existing link with `alreadyLinked: true`. Safe to retry.

### Role names

Accepts both canonical names and extraction-slug aliases (auto-mapped):

| Canonical                              | Extraction slug |
|---|---|
| Buyer                                  | Buyer           |
| Seller                                 | Seller          |
| Buying Agent                           | BuyerAgent      |
| Listing Agent                          | SellerAgent     |
| Title Company                          | TitleCompany    |
| Lender                                 | Lender          |
| Loan Officer                           | —               |
| Home Inspector                         | —               |
| Appraiser                              | —               |
| Buyer's Attorney / Seller's Attorney   | —               |
| Transaction Coordinator                | —               |

There is no plain "Attorney" or "Inspector" role — use the side-specific
attorney roles and "Home Inspector". Custom roles defined in the TC's
organization (e.g. Photographer, Stager) also resolve via `contact_roles.Name`.
Returns `ROLE_NOT_FOUND` listing every role actually available to the org
when no match — pick from that list rather than guessing.

## Creating a new role

When `ROLE_NOT_FOUND` shows nothing that fits and the TC agrees a new role is
needed:

```
create_contact_role(
  name: "Closing Attorney",
  category: "Legal",          // Client | Professional | Agent | Service Provider |
                              // Legal | Lending | Property | Rental | Services | Other
  partyKey: "buyer_side",     // optional grouping on the Assign Contacts dropdown
  allowMultiple: false
)
```

**Idempotent on name** — an exact match returns the existing role with
`created: false`. A *similar* name (asking for "Attorney" when "Buyer's
Attorney" exists) fails with `SIMILAR_ROLES_EXIST` and the near-matches;
prefer one of those — system roles work better across the app (overview
cards, contact blocks, smart fields) than custom ones. Only re-emit with
`ignoreSimilar: true` after the TC confirms the new role is genuinely
distinct.

## During email triage

When an unknown sender appears:

1. `find_contact_by_email(email)` — returns empty.
2. Ask the TC: "This email is from jane@titleco.com — is this someone on
   one of your transactions?"
3. If yes:
   - `search_transactions` or `find_transaction_by_property` to identify the
     deal.
   - `create_contact(...)` with the info you have.
   - `add_contact_to_transaction(contactId, transactionId, role, isPrimary)`.
4. If no, leave it alone — don't create speculative contacts.

## Context for drafting emails

When drafting a templated email (`render_email_template`), the contact's
`company`, `licenseNumber`, and role on the transaction influence tone.
Pull `search_contacts(contactId: "...")` for the recipient before drafting so the merge fields
have everything available.

## Errors you should expect

| Code                    | What to do                                                  |
|---|---|
| `VALIDATION_FAILED`     | Re-emit with the field the message identifies.              |
| `CONTACT_NOT_FOUND`     | Call `search_contacts`; the ID is wrong or cross-org.       |
| `TRANSACTION_NOT_FOUND` | Call `search_transactions`; the ID is wrong or cross-org.   |
| `ROLE_NOT_FOUND`        | Re-emit with one of the available roles the error lists. If nothing fits, offer `create_contact_role`. |
| `SIMILAR_ROLES_EXIST`   | Prefer one of the listed near-matches; only re-emit `create_contact_role` with `ignoreSimilar: true` after the TC confirms. |
| `CREATE_FAILED`         | A write race; retry `create_contact` or fall through to `search_contacts` to find the parallel-created row. |
