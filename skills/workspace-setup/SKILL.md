---
name: workspace-setup
description: >-
  Sets up and maintains a DocJacket organization from chat: the setup checklist, client agents
  and their TC fees and coordinators, organization settings, contact roles, document categories,
  custom fields, email templates and starter packs, online form templates, and the plan and
  upgrade link. Every change is confirmed with the user first, and most can be undone for 7 days.
  Use when the TC asks what to set up next, wants to set up their agents or fees, change a
  setting, manage templates or forms, or asks about their plan, limits or upgrading.
license: MIT
metadata:
  docjacket-title: "Set up and maintain the organization: agents, settings, templates, forms and plan"
  docjacket-tools: "get_setup_status propose_setup_from_evidence set_agent_defaults change_org_setting get_default_fees undo_setting_change get_contact_roles get_document_categories save_setup_item archive_setup_item get_custom_fields list_team_members list_email_templates get_email_template render_email_template get_merge_fields create_email_template save_email_template install_template_pack undo_template_change list_form_templates get_form_definition create_form_template clone_form_template add_form_template_fields get_plan get_checkout_link get_recent_support_history"
  docjacket-scopes: "read actions"
  docjacket-version: "1.0.0"
---

# Workspace Setup

Organization-level setup and housekeeping. These tools change how the whole organization works, so the rule is the same for every write: **use only a value the user stated, confirm it with them before the call, and never invent a fee, a state or an address.**

## When this fires

- "What should I set up next?", "how far along is our setup?", "help me get started"
- "Set up my agents", "Sarah's TC fee is $395", "I coordinate all of Mike's deals"
- "Change our primary state", "set the default TC fee", "why did this deal get a fee?"
- "Do we have a role for a transaction attorney?", "why are documents going to Uncategorized?"
- "Install the buyer emails", "change our under-contract email", "what merge fields can I use?"
- "Build me a listing intake form", "add the compensation fields to my intake form"
- "What plan are we on?", "why can't I add another deal?", "how do I upgrade?"

## Step 1 — Where the setup stands

**`get_setup_status`** first (no arguments). It is computed live from the real configuration: what is done, what needs attention, the next required step, and for each item the tool that resolves it or the page to open (`resolve.navigate`, `resolve.url`). Lead with the next required step.

## Client agents

1. **`propose_setup_from_evidence`** (no arguments) lists agents the organization has represented on real deals who are not set up as client agents yet, the deals that show it, any TC fee already charged on them, and a suggested fee when all those fees agree. Every suggestion needs the user's yes.
2. **`set_agent_defaults`** makes one change per call: setting `tc_fee` (amount, 0 for no fee, null to clear) or setting `coordinator` (`assignToMe: true`, a `profileId` from `list_team_members`, or null).

## Organization settings

- **`get_default_fees`** reads the default TC fee added to every new deal.
- **`change_org_setting`**, one setting per call: `primary_state` (2-letter code the user stated; never taken from an address, contract or deal), `default_tc_fee` (amount, 0 or null) or `default_tc_fee_description`. An agent's own fee or coordinator goes through `set_agent_defaults` instead.
- **`undo_setting_change`** takes back a `set_agent_defaults`, `change_org_setting` or `set_compliance_system` change within 7 days, by the `preparedWorkId` it returned. Use it only when the user asks to take back a change you made; to set a different value, call the setting tool again.

## Organization lists

- **`get_contact_roles`**: the roles a deal can use (system and the organization's own), which allow several holders, and which no deal uses.
- **`get_document_categories`**: categories, the folder each maps to, which are required, how many are unmapped.
- **`get_custom_fields`**: custom field definitions, merge codes, whether extraction fills them.
- **`save_setup_item`** creates or edits a contact role or document category (`kind`, fields, and `id` to edit). There is no undo tool: to reverse an edit, save the old values again. DocJacket's system roles can't be edited; create the organization's own role instead.
- **`archive_setup_item`** makes a role or category inactive (hidden from pickers, kept on deals that use it). It never deletes; the user can reactivate it in Settings.
- **`list_team_members`** gives teammates' ids for coordinator and assignee choices.

## Email templates

1. **`list_email_templates`** first (filter by category, audience, stage, side or search). **`get_email_template`** reads one in full; **`render_email_template`** fills it in for a deal.
2. **`get_merge_fields`** without a transactionId returns the merge fields the organization can use, including contact blocks. Only use `{{field}}` names it returns.
3. **`install_template_pack`** installs a DocJacket starter pack: `buyer`, `seller` or `essentials`. Templates the organization already has by the same name are skipped.
4. **`save_email_template`** creates a template, changes one of the organization's own, or saves the organization's own copy of a DocJacket starter (the original is never changed). Its next automatic email waits in the Review Queue once. `create_email_template` also creates one, as a draft unless `isActive` is true.
5. **`undo_template_change`** reverses a `save_email_template` or `install_template_pack` within 7 days by its `preparedWorkId`.

None of these send an email.

## Online forms

1. **`list_form_templates`** first whenever the user names a form: every other forms tool needs the id it returns. **`get_form_definition`** reads the fields a form already has.
2. Close to an existing form (especially a SYSTEM starter)? **`clone_form_template`** makes an editable, unpublished copy.
3. Nothing close? **`create_form_template`** authors the whole form in one call; fields with `mapsTo` save onto the deal, the rest are captured for the inbox only.
4. Extending a form: **`add_form_template_fields`** (additive only; check `get_form_definition` so you don't duplicate a question).

Forms are created as drafts; a person publishes them in the form builder. Report which answers will reach the deal and which are capture-only.

## Plan, limits and upgrading

- **`get_plan`** (no arguments): plan name, paid or not, subscription status, limit and current use for deals, seats and AI extractions, who can subscribe, the billing page and the plans available with prices. Use it when a limit stopped the user.
- **`get_checkout_link`** with the `plan_key` from `get_plan` returns a Stripe checkout link for an owner or admin. Show the link; never open it or pay yourself. Nothing is charged until they pay on Stripe's page. A member who isn't an owner or admin gets the names of who can subscribe. Changing or cancelling an existing subscription happens on the billing page.
- MCP access is included with every DocJacket plan. On Free it is limited to 100 MCP calls per month, with limited data extractions.

## Support history

**`get_recent_support_history`** (no arguments) shows the last 10 help-desk messages as triage verdicts (never the message text) and open bug reports. Use it to spot a repeat question or an issue that is already open.

## Guardrails

- Confirm every write first, and say whether it can be undone (7 days with `undo_setting_change` or `undo_template_change`, or not at all for `save_setup_item` and `archive_setup_item`).
- One setting per call; report the old and new value after each change.
- Never invent a fee amount, a state, a merge field or a form field mapping.

## Tool reference

| Tool | Scope | Use |
|---|---|---|
| `get_setup_status` / `propose_setup_from_evidence` | Read | What to set up next; agents seen on real deals |
| `set_agent_defaults` / `change_org_setting` / `undo_setting_change` | Actions | Agent fee or coordinator; org setting; 7-day undo |
| `get_default_fees` / `get_contact_roles` / `get_document_categories` / `get_custom_fields` / `list_team_members` | Read | Current configuration |
| `save_setup_item` / `archive_setup_item` | Actions | Contact roles and document categories |
| `list_email_templates` / `get_email_template` / `render_email_template` / `get_merge_fields` | Read | Find and preview templates and merge fields |
| `create_email_template` / `save_email_template` / `install_template_pack` / `undo_template_change` | Actions | Change templates; 7-day undo |
| `list_form_templates` / `get_form_definition` | Read | Find a form and its fields |
| `create_form_template` / `clone_form_template` / `add_form_template_fields` | Actions | Build or extend a form (draft) |
| `get_plan` / `get_checkout_link` | Read | Plan, limits, upgrade link |
| `get_recent_support_history` | Read | Recent support questions and open bugs |
