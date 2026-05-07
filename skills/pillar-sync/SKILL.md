---
name: pillar-sync
description: Bidirectional sync between a pillar's local workspace files and Confluence/Jira. Use when the user says "pillar-sync", "sync pho pillar", "sync pillar roadmap", or "run pillar-sync for <pillar>". Reads the Confluence roadmap page and Jira, updates local files — and pushes local changes (status, comments, description, assignee) back up to Jira.
---

# Pillar Sync

## Goal

Bidirectional sync for a pillar workspace under `pillar/<pillar-code>/`:

1. **Pull** — fetch the current state from Confluence and Jira into local markdown files
2. **Push** — detect local changes that differ from Jira and write them back up to Jira

## Prerequisites

- Confluence MCP access (requires Okta/VPN auth — if it fails with an auth error, open the sign-in URL provided and retry)
- Jira MCP access (requires VPN — if it fails, ask the user to confirm VPN is connected)

## Inputs

The skill needs two things. Extract them from the user's message or ask:

1. **Pillar code** — short lowercase identifier (e.g. `pho`)
2. **Confluence page URL or page ID** — e.g. `https://confluence.sie.sony.com/pages/viewpage.action?pageId=3116384383`

Extract the numeric page ID from the URL's `pageId=` query parameter if a URL is given.

## Step 1: Load pillar config

Check for a config file at `pillar/<pillar-code>/config.md`. If it exists, read it for:

- `confluence_page_id`
- `jira_project_key` (default: `DSOSYS`)
- `pillar_name` (display name)

If the config doesn't exist yet, create it using the inputs provided (see config template below).

## Step 2: Fetch the Confluence page

Call `confluence_get_page` with `id: <page_id>` and `includeBody: true`.

Parse the returned storage-format body to extract:

- The roadmap table rows: each row typically has columns like Epic, Description, Status, Owner, Notes
- Any epics listed in section headings or bullet lists if no table is present

If the page body is in Confluence storage XML format, look for `<table>` elements and extract `<td>` cell text. Strip XML/HTML tags to get plain text values.

## Step 3: Determine epics

For each row/item found on the Confluence page:

- Extract: epic name/title, Jira epic key (if present, e.g. `DSOSYS-XXXX`), status, owner, description/notes
- If no Jira key is present, flag it as `[NO_JIRA_KEY]` — do NOT create the Jira epic automatically; record it in ROADMAP.md with a note that it needs a Jira epic

## Step 4: Fetch epic details from Jira

For each epic that has a Jira key:

1. Call `jira_get_issue` with the epic key and `additionalFields: ["description", "status", "assignee", "summary"]`
2. Call `jira_search_issues` with JQL: `"Epic Link" = <EPIC_KEY> OR parentEpic = <EPIC_KEY> ORDER BY created ASC` (or `issueFunction in subtasksOf("key = <EPIC_KEY>")` if that fails) to get child tickets
   - Simpler fallback JQL: `project = DSOSYS AND "Epic Link" = <EPIC_KEY>`

## Step 5: Push local changes to Jira

Before writing updated local files, compare what is already on disk against the freshly-fetched Jira data for each epic and child ticket that has a Jira key. For each file found at `pillar/<pillar-code>/<KEY>/EPIC.md` or `pillar/<pillar-code>/<KEY>/<TICKET_KEY>.md`:

### 5a: Read the existing local file

Extract the current values from the `## Status` section:

- `Jira status:` line → local status
- `Assignee:` line → local assignee (strip `_(unassigned)_` as empty)

Extract the `## Description` section body → local description (as plain text).

Extract any entries in `## Notes / Updates` or `## Notes` that are **not** already reflected in Jira comments (see 5d below).

### 5b: Detect a status change

If `local status` differs from the Jira status fetched in Step 4:

1. Call `jira_get_transitions` for the ticket key to list available transitions
2. Find the transition whose name case-insensitively matches `local status` (e.g. "In Progress", "Done", "Not Ready")
3. If a matching transition exists, call `jira_transition_issue` to move the ticket
4. Log: `[PUSH] <KEY> status: <old> → <new>`
5. If no matching transition is found, log a warning and skip — do not error

### 5c: Detect a description change

If `local description` (stripped of markdown formatting) differs meaningfully from the Jira description:

1. Call `jira_update_issue` with `fields: { description: "<local description as plain text>" }`
2. Log: `[PUSH] <KEY> description updated`

Only push if the local description section has substantive content (not just `-` placeholder).

### 5d: Detect new comments

Read the `### Comments` subsection under `## Notes / Updates` (or `## Notes`) in the local file.
Each comment entry is a line formatted as: `- YYYY-MM-DD — <comment text>`

Fetch existing Jira comments by calling `jira_get_issue` with `additionalFields: ["comment"]`.

For each local comment entry:
- If no Jira comment exists with matching text (substring match), post it via `jira_add_issue_comment`
- Log: `[PUSH] <KEY> comment posted: "<first 60 chars>…"`

### 5e: Detect an assignee change

If `local assignee` differs from the Jira assignee:

1. Call `jira_get_user_profile` with the local assignee name to resolve their `accountId`
2. Call `jira_update_issue` with `fields: { assignee: { accountId: "<id>" } }`
3. Log: `[PUSH] <KEY> assignee: <old> → <new>`
4. If the user profile lookup fails, log a warning and skip

### Push safety rules

- **Never push** to tickets in `Closed` or `Done` status — log a warning instead
- **Never push** if the local file's `## Status` section still contains placeholder text (e.g. `_(pending`, `_(no Jira`)
- Only push fields where local content is genuinely different from Jira — do not push no-ops
- Confirm with the user before pushing if more than 5 tickets would be affected in a single run

## Step 6: Write ROADMAP.md

Write or update `pillar/<pillar-code>/ROADMAP.md` with:

```markdown
# <Pillar Name> — Roadmap

- Confluence: [<page title>](<confluence_url>)
- Last synced: <YYYY-MM-DD>

## Epics

| Epic | Summary | Status | Owner | Jira |
| --- | --- | --- | --- | --- |
| DSOSYS-XXXX | ... | In Progress | ... | [DSOSYS-XXXX](https://jira.sie.sony.com/browse/DSOSYS-XXXX) |
| _(no key)_ | Some planned epic | Planned | ... | — needs Jira epic |
```

Rules:
- Never delete existing content below a `## Notes` heading if one exists — preserve it
- Update the `Last synced` date on every run

## Step 7: Create/update epic folders and files

For each epic with a Jira key:

1. Create directory `pillar/<pillar-code>/<EPIC_KEY>/` if it doesn't exist
2. Create/update `pillar/<pillar-code>/<EPIC_KEY>/EPIC.md` using the template below
3. For each child ticket found in Step 4, create/update `pillar/<pillar-code>/<EPIC_KEY>/<TICKET_KEY>.md`

### EPIC.md template

```markdown
# <EPIC_KEY>: <Epic Summary>

## Status

- Jira status: <status>
- Assignee: <assignee>
- Jira: [<EPIC_KEY>](https://jira.sie.sony.com/browse/<EPIC_KEY>)
- Confluence: [Roadmap](<confluence_url>)

## Description

<epic description as markdown bullets>

## Child tickets

| Key | Summary | Status | Assignee |
| --- | --- | --- | --- |
| [DSOSYS-XXXX](https://jira.sie.sony.com/browse/DSOSYS-XXXX) | ... | ... | ... |

## Notes

-
```

### Ticket markdown template (child tickets)

```markdown
# <TICKET_KEY>: <Summary>

## Status

- Jira status: <status>
- Assignee: <assignee>
- Jira: [<TICKET_KEY>](https://jira.sie.sony.com/browse/<TICKET_KEY>)
- Epic: [<EPIC_KEY>](https://jira.sie.sony.com/browse/<EPIC_KEY>)

## Description

-

## Plan

-

## Notes / Updates

-
```

### Preserving user content

- If a file already exists, NEVER overwrite content under `## Plan` or `## Notes` / `## Notes / Updates` headings
- Only update the `## Status` section and regenerate the `## Child tickets` table in EPIC.md
- Append new Jira history as a `### Jira history` subsection under Notes if changes are detected

## Step 8: Config file template

If `pillar/<pillar-code>/config.md` doesn't exist, create it:

```markdown
# Pillar config: <pillar-code>

- pillar_code: <pillar-code>
- pillar_name: <Pillar Name>
- confluence_page_id: <page_id>
- confluence_url: <full_url>
- jira_project_key: DSOSYS
```

## Step 9: Report

After completing, output a summary with two sections:

**Pull (Confluence/Jira → local):**

- How many epics were found on the Confluence page
- How many had Jira keys vs needed one
- How many epic folders were created vs updated
- How many child tickets were processed

**Push (local → Jira):**

- How many status transitions were applied
- How many descriptions were updated
- How many comments were posted
- How many assignees were changed
- Any warnings (no matching transition, user not found, skipped closed tickets, etc.)

## Markdown output quality

All files must conform to markdownlint rules:

- One blank line before and after every heading (MD022)
- Lists surrounded by blank lines (MD032)
- No trailing spaces (MD009)
- No multiple consecutive blank lines (MD012)
- No bare URLs — use `<https://…>` or `[text](url)` (MD034)
- Every file ends with a single newline (MD047)
- Fenced code blocks must specify a language (MD040)
- Table separator rows must use `| --- |` style, not `|---|` (MD060)
