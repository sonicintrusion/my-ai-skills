---
name: pillar
description: Manage pillar workspaces synced with Confluence and Jira. Supports init (bootstrap new pillar), sync (bidirectional sync), and status (quick update). Use when the user says "pillar-init pho", "pillar-sync pho", "pillar-status pho", or similar. Automatically detects operation from user input.
disable-model-invocation: true
---

# Pillar Skill

## Goal

Unified interface for managing pillar workspaces that sync with Confluence roadmap pages and Jira epics/tickets.

## Supported Operations

1. **Init** - Bootstrap a new pillar workspace from scratch
2. **Sync** - Full bidirectional sync (Confluence/Jira ↔ local files)
3. **Status** - Quick status update (sync Jira statuses, archive removed epics)

## Operation Detection

Parse the user's message to determine which operation:

- `pillar-init pho` → **Init**
- `init pho pillar` → **Init**
- `set up new pillar for pho` → **Init**
- `pillar-sync pho` → **Sync**
- `sync pho pillar` → **Sync**
- `pillar-status pho` → **Status**
- `update pho status` → **Status**
- `check pho pillar status` → **Status**

## Common: Pillar Code

All operations need a **pillar code** — short lowercase identifier (e.g. `pho`, `dxe`). Extract from user's message or ask.

## Common: API Access

**MCP Primary (Confluence & Jira):**

- Confluence MCP: Requires Okta/VPN auth (if auth error, open sign-in URL and retry)
- Jira MCP: Requires VPN connection
- If MCP fails, suggest VPN check or MCP server restart

**REST API Fallback:**

- Not typically used for pillar operations (MCP is primary)
- Confluence and Jira tokens available if needed

## Common: Workspace Structure

```text
pillar/
└── <pillar-code>/
    ├── config.md           # Pillar configuration
    ├── ROADMAP.md          # Epic overview table
    ├── _archived/          # Archived epics
    └── <EPIC-KEY>/         # Epic folders
        ├── EPIC.md         # Epic details
        └── <TICKET-KEY>.md # Child ticket files
```

---

# Operation 1: Init

Bootstrap a new pillar workspace from scratch.

## Input Patterns

- `pillar-init pho`
- `init pho pillar`
- `set up a new pillar for pho`
- `create pillar workspace for pho`

## Inputs Required

Ask the user for (or extract from their message):

1. **Pillar code** — short lowercase identifier (e.g. `pho`)
2. **Pillar name** — display name (e.g. `Platform Health and Observability`)
3. **Confluence page URL** — the roadmap page URL (e.g. `https://confluence.sie.sony.com/pages/viewpage.action?pageId=3116384383`)
4. **Jira project key** — defaults to `DSOSYS` if not specified

## Step 1: Create folder structure

Create directories:

```text
pillar/<pillar-code>/
pillar/<pillar-code>/_archived/
```

## Step 2: Create config file

Write `pillar/<pillar-code>/config.md`:

```markdown
# Pillar config: <pillar-code>

- pillar_code: <pillar-code>
- pillar_name: <Pillar Name>
- confluence_page_id: <extracted_page_id>
- confluence_url: <full_url>
- jira_project_key: <PROJECT_KEY>
```

Extract the numeric page ID from the `pageId=` query parameter in the Confluence URL.

## Step 3: Run sync operation

Internally invoke the **Sync** operation (see Operation 2 below) using the same inputs. The sync will populate ROADMAP.md, epic folders, and ticket files.

## Step 4: Report

Confirm:

- Workspace created at `pillar/<pillar-code>/`
- Config file written
- Initial sync completed (show sync summary)

---

# Operation 2: Sync

Full bidirectional sync between local workspace and Confluence/Jira.

## Op 2: Input Patterns

- `pillar-sync pho`
- `sync pho pillar`
- `sync pillar roadmap for pho`

## Op 2: Goal

Two-way sync:

1. **Pull** — Fetch current state from Confluence and Jira into local files
2. **Push** — Detect local changes and write them back to Jira

## Op 2: Inputs Required

1. **Pillar code** — e.g. `pho`
2. **Confluence page URL or page ID** (optional if config exists)

## Step 1: Load pillar config

Read `pillar/<pillar-code>/config.md` for:

- `confluence_page_id`
- `jira_project_key` (default: `DSOSYS`)
- `pillar_name`
- `confluence_url`

If config doesn't exist, create it using provided inputs.

## Step 2: Fetch Confluence page (PULL)

**Try MCP first:**

Call `confluence_get_page` with:

- `id: <page_id>`
- `includeBody: true`

Parse the storage-format body to extract:

- Roadmap table rows (Epic, Description, Status, Owner, Notes columns)
- Epic keys (e.g. `DSOSYS-XXXX`)
- Epic titles, statuses, owners

If page is XML/HTML, look for `<table>` elements and extract `<td>` cell text. Strip tags to get plain text.

## Step 3: Determine epics

For each row/item on Confluence page:

- Extract: epic name, Jira key (if present), status, owner, description
- If no Jira key: flag as `[NO_JIRA_KEY]` — do NOT auto-create, note in ROADMAP.md

## Step 4: Fetch Jira data (PULL)

For each epic with a Jira key:

**Try MCP first:**

1. Call `jira_get_issue` with:
   - `key: <EPIC_KEY>`
   - `additionalFields: ["description", "status", "assignee", "summary"]`

2. Call `jira_search_issues` with JQL:
   - `"Epic Link" = <EPIC_KEY> OR parentEpic = <EPIC_KEY> ORDER BY created ASC`
   - Fallback: `project = DSOSYS AND "Epic Link" = <EPIC_KEY>`

## Step 5: Push local changes to Jira (PUSH)

Before updating local files, check existing local files for changes to push.

For each file at `pillar/<pillar-code>/<KEY>/EPIC.md` or `pillar/<pillar-code>/<KEY>/<TICKET_KEY>.md`:

### 5a: Read existing local file

Extract from local file:

- `## Status` section: `Jira status:` and `Assignee:` lines
- `## Description` section: full description text
- `## Notes / Updates` → `### Comments`: dated comment entries

### 5b: Detect status change

If local status differs from Jira status:

**Try MCP:**

1. Call `jira_get_transitions` for ticket key
2. Find transition matching local status (case-insensitive)
3. Call `jira_transition_issue` to apply transition
4. Log: `[PUSH] <KEY> status: <old> → <new>`

If no matching transition, log warning and skip.

### 5c: Detect description change

If local description differs meaningfully from Jira:

**Try MCP:**

1. Call `jira_update_issue` with `fields: { description: "<plain text>" }`
2. Log: `[PUSH] <KEY> description updated`

Only push if local description has substantive content (not placeholder `-`).

### 5d: Detect new comments

Parse `### Comments` entries: `- YYYY-MM-DD — <comment text>`

**Try MCP:**

1. Call `jira_get_issue` with `additionalFields: ["comment"]`
2. For each local comment not in Jira (substring match):
   - Call `jira_add_issue_comment`
   - Log: `[PUSH] <KEY> comment posted`

### 5e: Detect assignee change

If local assignee differs from Jira:

**Try MCP:**

1. Call `jira_get_user_profile` to resolve `accountId`
2. Call `jira_update_issue` with `fields: { assignee: { accountId: "<id>" } }`
3. Log: `[PUSH] <KEY> assignee: <old> → <new>`

### Push safety rules

- **Never push** to tickets in `Closed` or `Done` status
- **Never push** if local file has placeholder text
- Only push genuine differences (no no-ops)
- Confirm with user if >5 tickets affected

## Step 6: Write ROADMAP.md

Create/update `pillar/<pillar-code>/ROADMAP.md`:

```markdown
# <Pillar Name> — Roadmap

- Confluence: [<page title>](<confluence_url>)
- Last synced: <YYYY-MM-DD>

## Epics

| Epic | Summary | Status | Owner | Jira |
| --- | --- | --- | --- | --- |
| DSOSYS-XXXX | ... | In Progress | ... | [DSOSYS-XXXX](https://jira.sie.sony.com/browse/DSOSYS-XXXX) |
| _(no key)_ | Planned epic | Planned | ... | — needs Jira epic |
```

Preserve content below `## Notes` heading if exists.

## Step 7: Create/update epic files

For each epic with Jira key:

1. Create `pillar/<pillar-code>/<EPIC_KEY>/` directory
2. Create/update `EPIC.md` (template below)
3. Create/update child ticket files: `<TICKET_KEY>.md`

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

### Child ticket template

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

- **NEVER** overwrite `## Plan` or `## Notes` sections
- Only update `## Status` and regenerate `## Child tickets` table
- Append Jira changes as `### Jira history` under Notes

## Step 8: Report

Output summary with two sections:

**Pull (Confluence/Jira → local):**

- Epics found on Confluence page
- Epics with/without Jira keys
- Epic folders created/updated
- Child tickets processed

**Push (local → Jira):**

- Status transitions applied
- Descriptions updated
- Comments posted
- Assignees changed
- Warnings (no transition, user not found, skipped closed tickets)

---

# Operation 3: Status

Quick status update — sync Jira statuses and archive removed epics.

## Op 3: Input Patterns

- `pillar-status pho`
- `update pho status`
- `check pho pillar status`
- `sync pho epics`

## Op 3: Goal

Keep local workspace in sync with current Jira state and detect removed epics.

## Op 3: Inputs Required

**Pillar code** — e.g. `pho`

## Op 3: Step 1: Load pillar config

Read `pillar/<pillar-code>/config.md` for:

- `confluence_page_id`
- `jira_project_key`
- `confluence_url`

If config doesn't exist, tell user to run sync first.

## Step 2: Discover known epics

Scan `pillar/<pillar-code>/` for subdirectories containing `EPIC.md`. Extract epic key from directory name (e.g. `DSOSYS-2936`).

Skip `_archived/` subdirectory.

## Step 3: Fetch current Confluence state

**Try MCP:**

Call `confluence_get_page` with:

- `id: <confluence_page_id>`
- `includeBody: true`

Extract list of epic keys currently on Confluence page.

Build two sets:

- **Active epics**: keys on Confluence AND in local workspace
- **Removed epics**: keys in local workspace but NOT on Confluence

## Step 4: Archive removed epics

For each removed epic:

1. Create `pillar/<pillar-code>/_archived/` if needed
2. Move folder: `<EPIC_KEY>/` → `_archived/<EPIC_KEY>/`
3. Add note at top of `_archived/<EPIC_KEY>/EPIC.md`:

```markdown
> **Archived <YYYY-MM-DD>**: This epic was removed from the <pillar> pillar Confluence roadmap page.
> It may have been reassigned to another pillar or deprioritised.
```

4. Update `ROADMAP.md`: move epic row to `## Archived epics` section

## Step 5: Update active epics from Jira

For each active epic:

**Try MCP:**

1. Call `jira_get_issue` with `additionalFields: ["description", "status", "assignee", "summary", "comment"]`
2. Re-fetch child tickets with JQL: `project = <PROJECT_KEY> AND "Epic Link" = <EPIC_KEY>`
3. Update `EPIC.md`:
   - Update `## Status` section
   - Regenerate `## Child tickets` table
   - If status changed, append to `### Jira history`
4. For each child ticket:
   - Update `## Status` section
   - If new Jira comments since last sync, append to `### Jira history`
   - Never touch `## Plan` or user notes

## Step 6: Update ROADMAP.md

Regenerate epics table with latest statuses and update `Last synced` date.

Preserve user-written content below `## Notes` heading.

## Step 7: Report

Output summary:

- Epics updated (count)
- Epics archived (count, list keys)
- Tickets with status changes (key + old → new)
- Any Jira fetch errors

---

# Error Handling

## MCP Errors

**Confluence MCP:**

- Authentication required: Open sign-in URL and retry
- Network error: Check VPN connection
- Page not found: Verify page ID in config

**Jira MCP:**

- Network error: Check VPN connection
- Authentication required: MCP interactive sign-in
- Ticket not found: Epic may have been deleted
- Transition not found: Status name doesn't match available transitions

## File System Errors

- Config not found: Run init first (for sync/status)
- Permission denied: Check file/folder permissions
- Invalid pillar code: Use lowercase alphanumeric

## Validation Errors

- Missing required inputs: Ask user for missing info
- Invalid Confluence URL: Must contain `pageId=` parameter
- Invalid Jira project key: Must be uppercase (e.g. `DSOSYS`)

---

## Markdown quality rules

Apply the rules defined in the `markdownlint` skill for all files created or modified.

---

# Integration Notes

This skill manages pillar workspaces independently. Other skills that may reference pillars:

- `sony-sprint-manager` — Sprint management (separate from pillars)
- `jira` — Direct Jira operations (pillars use Jira data but don't invoke jira skill)

Pillars use Confluence and Jira MCP APIs directly, not via skill invocation.
