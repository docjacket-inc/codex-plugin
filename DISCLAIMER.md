# Disclaimer

The DocJacket plugin surfaces DocJacket transaction data inside Claude. It does **not** provide legal, real-estate, lending, or compliance advice.

## What this plugin is

- A read-only window into your DocJacket organization's transactions, key dates, contacts, and document checklists.
- A skill library (Daily Triage, Status Reporter sub-agent) that helps your AI assistant surface what needs attention.

## What this plugin is NOT

- **Legal advice.** Real-estate transactions involve contractual obligations, state-specific disclosures, and licensure requirements that vary by jurisdiction. This plugin's output is data from your DocJacket account — interpret it with your own professional judgment or consult an attorney for legal questions.
- **A replacement for your broker, attorney, or compliance officer.** Sign-off authority for closings, contingency releases, and disclosure adequacy lives with licensed humans, not with this plugin.
- **An auto-executor.** This plugin is read-only as of v0.2. Future versions will add drafting tools that prepare emails / task updates / key-date changes for human approval — they will **never** auto-send or auto-execute. Every write is gated on your explicit review.

## Per-state coverage

The `get_missing_documents` tool's baseline (Purchase Agreement, Seller Disclosure, EM Receipt, Lead-Based Paint Disclosure, Wire Fraud Advisory) is intentionally universal — it does NOT cover state-specific requirements like California TDS / SPQ / Wood Destroying Pest report, Florida HOA Disclosure, Texas Seller's Disclosure, etc. Per-state customization lands as a future tool revision. The response from this tool always includes a `disclaimer` field flagging this scope.

## Liability

You are responsible for what your AI assistant does with DocJacket data. Audit the [Activity Log](https://app.docjacket.com/settings/ai-access/activity) regularly. Revoke tokens you no longer use. Don't share AI access tokens.

For support: support@docjacket.com.
