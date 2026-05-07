---
name: pillar-status
description: Sync current epic and ticket statuses from Jira into local pillar workspace files. Use when the user says "pillar-status", "update pho status", "sync pho epics", or "check pillar status". Detects epics removed from Confluence and archives them locally.
---

# Pillar Status

## Goal

Keep the local pillar workspace in sync with the current state of Jira epics and tickets. Also detect epics that have been removed from the Confluence roadmap page and archive them.

## Prerequisites

- Jira MCP access (requires VPN)
- Confluence MCP access (requires Okta/VPN auth) — needed to detect removed epics

## Inputs

The skill needs the **pillar code** (e.g. `pho`). Extract from the user's message or ask.

## Step 1: Load pillar config

Read `pillar/<pillar-code>/config.md` to get:

- `confluence_page_id`
- `jira_project_key`
- `confluence_url`

If the config doesn't exist, tell the user to run `pillar-sync` first.

## Step 2: Discover known epics

Scan `pillar/<pillar-code>/` for subdirectories that contain an `EPIC.md` file. These are the known epics in the local workspace. Extract the epic key from the directory name (e.g. `DSOSYS-2936`).

Skip the `_archived/` subdirectory.

## Step 3: Fetch current state from Confluence

Call `confluence_get_page` with `id: <confluence_page_id>` and `includeBody: true`.

Extract the list of epic keys currently referenced on the Confluence page (same parsing logic as `pillar-sync`).

Build two sets:
- **Active epics**: keys present on Confluence page AND in local workspace
- **Removed epics**: keys in local workspace but NO LONGER on Confluence page

## Step 4: Archive removed epics

For each removed epic:

1. Create `pillar/<pillar-code>/_archived/` if it doesn't exist
2. Move the entire epic folder: `pillar/<pillar-code>/<EPIC_KEY>/` → `pillar/<pillar-code>/_archived/<EPIC_KEY>/`
3. Add a note at the top of `_archived/<EPIC_KEY>/EPIC.md`:

```markdown
> **Archived <YYYY-MM-DD>**: This epic was removed from the PHO pillar Confluence roadmap page.
> It may have been reassigned to another pillar or deprioritised.
```

4. Update `ROADMAP.md` to move the epic row to an `## Archived epics` section at the bottom

## Step 5: Update active epics from Jira

For each active epic key:

1. Call `jira_get_issue` with `additionalFields: ["description", "status", "assignee", "summary", "comment"]`
2. Re-fetch child tickets with JQL: `project = <PROJECT_KEY> AND "Epic Link" = <EPIC_KEY>`
3. Update `EPIC.md`:
   - Update `## Status` section (status, assignee)
   - Regenerate `## Child tickets` table
   - If status changed since last sync, append to `### Jira history` under Notes
4. For each child ticket:
   - Update the `## Status` section in its markdown file
   - If new comments exist in Jira since the last sync date, append them under `### Jira history`
   - Never touch `## Plan` or user-written notes

## Step 6: Update ROADMAP.md

Regenerate the epics table in `ROADMAP.md` with the latest statuses and update `Last synced` date.

Preserve any user-written content below a `## Notes` heading.

## Step 7: Report

Output a summary:

- How many epics were updated
- How many epics were archived (with their keys)
- Any tickets with status changes (key + old status → new status)
- Any errors fetching from Jira

## Markdown output quality

All files must conform to markdownlint rules:

- One blank line before and after every heading (MD022)
- Lists surrounded by blank lines (MD032)
- No trailing spaces (MD009)
- No multiple consecutive blank lines (MD012)
- No bare URLs — use `<https://…>` or `[text](url)` (MD034)
- Every file ends with a single newline (MD047)
- Fenced code blocks must specify a language (MD040)
