# Optional connectors that pair well with DocJacket

The DocJacket plugin works **standalone** — paste the server URL, click Allow on the OAuth consent screen, and the DocJacket tools work. But several connectors compose with DocJacket to unlock richer workflows. None of these are required.

| Connector | What it unlocks for DocJacket | How |
|---|---|---|
| **Gmail** | Match incoming agent / lender / title-company emails to DocJacket transactions | Skills like `daily-triage` can pivot from "your inbox" → "what does each email mean for which deal?" |
| **Google Calendar** | See closings + inspections + signings alongside Calendar events | Cross-reference `get_upcoming_key_dates` with Calendar entries |
| **Google Drive** | Point at the Drive folder where a missing document lives when chasing it | `get_missing_documents`, then `send_document_request` can mention the Drive folder path (the email is previewed and sent only on your yes) |
| **Slack** | Tell a TC team channel what is waiting for review or due soon | A scheduled routine calls `list_held_emails` and `get_next_required_actions` and posts a summary |

The DocJacket plugin **does NOT re-implement** any of these. We rely on Claude's connectors for Gmail / Calendar / Drive / Slack and DocJacket exposes only the DocJacket-specific tools. That's the divide-and-conquer pattern: each side owns what only it can do.
