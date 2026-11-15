---
name: playbook-builder
description: >-
  Builds a new DocJacket Playbook (a deal checklist of key dates, tasks, task emails and document
  lists) or changes one of the organization's own Playbooks, through the Playbook builder's draft
  tools. Everything is drafted first and saved only when the user approves the finished draft.
  Also explains why a Playbook task is or isn't on a deal. Use when the TC wants to set up their
  process, build or edit a checklist or Playbook, copy tasks between Playbooks, or asks why a task
  showed up on a deal.
license: MIT
metadata:
  docjacket-title: "Build or change a Playbook through a reviewed draft"
  docjacket-tools: "get_customer_process list_checklist_templates start_playbook_draft list_playbook_drafts get_draft list_vocabulary read_playbook draft_playbook propose_key_dates draft_key_date propose_tasks add_tasks draft_task edit_tasks copy_tasks propose_task_emails draft_task_email draft_email_template propose_document_list draft_document_list copy_documents draft_finish preview_dates validate_draft commit_playbook_draft undo_playbook_setup explain_task"
  docjacket-scopes: "read draft actions"
  docjacket-version: "1.0.0"
---

# Playbook Builder

A **Playbook** is how an organization works a kind of deal: its key dates, its tasks (with timing, phase, assignee and conditions), the emails a task sends when it is completed, and the documents the deal needs. DocJacket applies it to every matching deal.

The Playbook builder works on a **draft**. Every `draft_*`, `propose_*`, `copy_*`, `add_tasks` and `edit_tasks` call changes only the draft; nothing reaches the account until `commit_playbook_draft`, and that call needs the user's approval in chat.

## When this fires

- "Set up our buyer checklist", "build a Playbook for Colorado listings", "add these tasks to my Playbook"
- "Copy the inspection tasks from our buyer Playbook into the listing one"
- "Make this task only show up on FHA loans", "add an email when the EMD task is done"
- "Why is this task on the deal?" / "Why didn't the HOA task get added?"
- "What is our normal process for this?" (start with `get_customer_process`; build only if they ask)

## Before you start

1. **`get_customer_process`** answers "how do we normally work a deal like this?". It says whether the content is the organization's own or a DocJacket starter; a starter is a suggestion, never the firm's policy. If several Playbooks fit, it returns candidates: ask which one.
2. **`list_checklist_templates`** lists checklist templates (filter by state, side, transactionType, category, complexity) when the user wants to compare starters.
3. Building a Playbook needs the Playbook builder turned on and permission to edit Playbooks. If `start_playbook_draft` refuses, tell the user what it says.

## The recipe

### Step 1 — Open the draft

- **`start_playbook_draft`** with mode `build_playbook` (new: blank, from a DocJacket starter, or from one of the account's own Playbooks) or `improve_playbook` (edit one of the account's own). It resumes the user's open draft if there is one (`list_playbook_drafts` lists them). Keep the returned **`sessionId`**: every other call in this skill needs it.
- **`get_draft`** reads the draft: the current step in full and the other steps as counts. Pass `step` for another step, or `ref` for one item in full (an email's subject and body, a task's text).

### Step 2 — Header

**`draft_playbook`** sets name, category, transaction type, state and side. Pass the whole row. Take every value from the user; never infer a state from an address.

### Step 3 — Key dates

- Bulk: **`propose_key_dates`** from what the user described. It returns counts and refs, not rows; read them back with `get_draft`.
- One at a time: **`draft_key_date`** (whole row on create and update; row null on delete). Prefer a library key date (`list_vocabulary` → `library_key_dates`) when one fits.

### Step 4 — Tasks

- A list the user typed or pasted: **`add_tasks`** (up to 100 per call). Send each name exactly as the user wrote it and only the fields you have a reason to set.
- A whole set from a description: **`propose_tasks`**.
- One task: **`draft_task`**. Keep a name the user wrote exactly and set `nameIsVerbatim`. For a checklist under a task, add each check as its own task with `subtaskOf` set to the parent's ref (one level only). Never put a list of checks in a description.
- Several existing tasks changing the same way (move under another key date, new phase or due rule): **`edit_tasks`**.
- Tasks from another of the account's own Playbooks: **`read_playbook`** first, then **`copy_tasks`** (by `all`, `keyDates`, `phases` or `names`). Never retype another Playbook's tasks with `draft_task`.
- **Conditions** ("only on FHA loans", "only in Texas"): build an Add-when rule from `list_vocabulary` → `rule_conditions` and set it as `conditionRule` with `edit_tasks`. A new task with a rule takes two calls: `draft_task`, then `edit_tasks`. If nothing in `rule_conditions` fits, say so in one sentence and suggest a Dropdown custom field under Settings > Custom Fields; never leave the condition in a note.

### Step 5 — Task emails

- **`propose_task_emails`** suggests emails for the external-facing tasks (intro, next steps, under contract, check-in, invoice, thank you) from the account's own templates, or DocJacket's when none fit.
- **`draft_task_email`** adds, replaces or removes one email sent when a task is completed. Use a template id from `list_vocabulary` → `email_templates` and role names from `list_vocabulary` → `contact_roles` (or DealAgent / CoopAgent). These emails always wait for a person's review; they cannot be set to send on a date or to a typed-in address.
- A new email the account doesn't have: **`draft_email_template`** with source `drafted` (leave subject and body empty to have the builder write them, or send your own). The subject starts `{{property_full_address}} - `, and the body may use only merge fields from `list_vocabulary` → `merge_fields`. Then pass its ref as the template of a `draft_task_email`.

### Step 6 — Document lists

- **`propose_document_list`** first when the draft has none: it attaches the list that fits the state and side. With no side set, it asks which side's list to use.
- **`draft_document_list`** changes one list (conditions only from `list_vocabulary` → `condition_catalog`, never as text). Changing an existing list makes the account's own copy at Create; the original is never changed.
- Lists from another own Playbook: **`copy_documents`** (mode `attach` or `merge_into`). Never retype them.

### Step 7 — Finish, check and review

1. **`draft_finish`**: signature, which agents the Playbook applies to, and whether it becomes the default. Replacing an existing default is the user's choice on the review card, not yours.
2. **`preview_dates`** with a sample effective date shows the dates the Playbook would produce, holiday aware.
3. **`validate_draft`** lists what blocks Create, what is a warning and what was fixed. Fix the blockers.
4. **Show the user the draft** (`get_draft`) in plain words: the key dates, the tasks grouped by key date, the emails and the document lists. Ask for an explicit yes.

### Step 8 — Commit

**`commit_playbook_draft`** with the `draftVersion` you last read. This is the call that changes the account. A draft changed since you read it is refused (`stale_draft`): read it again, show what changed, and ask again. Calling it again with the same `draftVersion` returns the first result.

The result carries `undoUntil`: for 7 days **`undo_playbook_setup`** removes what the commit created and restores what it replaced. Use it only when the user confirms they want the setup taken back.

## Explaining a task on a deal

When a draft session was opened from a deal, **`explain_task`** says why a Playbook task is or isn't on that deal: the Playbook, the condition and what the deal records for it, the timing, and what would fix it. Explain it in plain words; change the Playbook only if the user asks.

## Guardrails

- Never call `commit_playbook_draft` or `undo_playbook_setup` without the user's explicit yes in chat.
- Use real ids and names from `list_vocabulary`; never invent a role, merge field, condition or template.
- Prefer the bulk tools (`add_tasks`, `edit_tasks`, `copy_tasks`, `propose_*`) over many single calls.
- Applying a Playbook to a live deal is a different job (`preview_playbook` / `apply_playbook` in `execution-workflow`).

## Tool reference

| Tool | Scope | Use |
|---|---|---|
| `get_customer_process` | Read | How the organization normally works this kind of deal |
| `list_checklist_templates` | Read | Compare checklist templates |
| `start_playbook_draft` / `list_playbook_drafts` | Draft / Read | Open or resume a draft; returns `sessionId` |
| `get_draft` / `list_vocabulary` / `read_playbook` | Read | Read the draft, its picklists, another own Playbook |
| `draft_playbook` | Draft | Header |
| `propose_key_dates` / `draft_key_date` | Draft | Key dates |
| `add_tasks` / `propose_tasks` / `draft_task` / `edit_tasks` / `copy_tasks` | Draft | Tasks |
| `propose_task_emails` / `draft_task_email` / `draft_email_template` | Draft | Task emails |
| `propose_document_list` / `draft_document_list` / `copy_documents` | Draft | Document lists |
| `draft_finish` | Draft | Signature, agents, default |
| `preview_dates` / `validate_draft` | Read | Check before commit |
| `commit_playbook_draft` | Actions | Create the Playbook (user's yes first) |
| `undo_playbook_setup` | Actions | Take back a commit within 7 days |
| `explain_task` | Read | Why a task is or isn't on a deal |
