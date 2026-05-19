# Optional connectors that pair well with DocJacket

The DocJacket plugin works **standalone** — paste the server URL, click Allow on the OAuth consent screen, and the 10 read tools work. But several connectors compose with DocJacket to unlock richer workflows. None of these are required.

| Connector | What it unlocks for DocJacket | How |
|---|---|---|
| **Gmail** | Match incoming agent / lender / title-company emails to DocJacket transactions | Skills like `daily-triage` can pivot from "your inbox" → "what does each email mean for which deal?" |
| **Google Calendar** | See closings + inspections + signings alongside Calendar events | Cross-reference `get_upcoming_key_dates` with Calendar entries |
| **Google Drive** | Treat Drive folders as document source for `prepare_document_request` (when drafting tools ship in v0.3) | Drafting an "missing inspection report" email can reference the actual Drive folder path |
| **Slack** | Notify a TC team channel when Prepared Work artifacts hit needs-review state (v0.3) | Wire Cowork Cloud Routines to call `list_prepared_work` and post summaries |

The DocJacket plugin **does NOT re-implement** any of these. We rely on Claude's connectors for Gmail / Calendar / Drive / Slack and DocJacket exposes only the DocJacket-specific tools. That's the divide-and-conquer pattern: each side owns what only it can do.
