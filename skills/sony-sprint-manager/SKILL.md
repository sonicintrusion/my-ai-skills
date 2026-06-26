---
name: sony-sprint-manager
description: >-
  Manage sprint work using Jira. Use when the user asks to manage their sprint,
  start sprint planning, create a sprint work file, or sync sprint context from
  Jira. First action: retrieve the current active sprint via the Jira MCP and
  create/update a markdown file under sprints/SPRINT/ in the current workspace.
---

# Sony sprint manager

## Goal

Maintain a per-sprint markdown workspace file driven by Jira.

## Prerequisite: VPN connectivity

Jira MCP access may require being connected to the corporate VPN.

If Jira MCP calls fail with network/connection errors (or show "server errored" / "no servers available"):

- Ask the user to confirm they're connected to VPN.
- Suggest reloading/restarting MCP servers after VPN is connected.

## Sprint workspace location

This skill must work in any IDE and any repository. All paths are **relative to the current workspace/repo root**.

1. Determine `workspaceRoot` (the repository root for the current workspace).
2. Determine `sprintsRoot`:
   - If the repo already has a `sprints/` folder at the root, use it.
   - Otherwise, create and use `sprints/` at the root.

Each sprint gets its own subfolder using a short sprint key (e.g. `94-3`): `<sprintsRoot>/<sprintKey>/`

Only **active and future** sprints live directly under `<sprintsRoot>/`. Closed sprints are moved to the archive:

- Active/future: `<sprintsRoot>/<sprintKey>/`
- Closed: `<sprintsRoot>/archive/<sprintKey>/`
- Archive index: `<sprintsRoot>/archive/index.md`

Each ticket gets its own subfolder inside the sprint directory: `<sprintDir>/<ISSUE_KEY>/`

The ticket markdown file lives inside the ticket subfolder: `<sprintDir>/<ISSUE_KEY>/<ISSUE_KEY>.md`

Any additional files created for a ticket (diagrams, scripts, specs, etc.) are stored in the same ticket subfolder alongside the markdown file.

## Step 0: auto-archive closed sprints

Before doing anything else, scan `<sprintsRoot>/` for sprint folders that belong in the archive.

1. For each subdirectory of `<sprintsRoot>/` that is **not** `archive/`:
   - Read its `sprint.md` frontmatter.
   - If `state: closed` (or if the `end_date` is in the past and the sprint is no longer active in Jira), the sprint is closed.
2. For each closed sprint found:
   - Ensure `<sprintsRoot>/archive/` exists.
   - Move the entire sprint folder:

     ```bash
     mv <sprintsRoot>/<sprintKey>/ <sprintsRoot>/archive/<sprintKey>/
     ```

   - Update `<sprintsRoot>/archive/index.md` (see **Archive index** section below).
3. If no sprint.md exists in a top-level subfolder (e.g. it was manually created), leave it in place — do not move unknown folders.
4. Report to the user how many sprints were archived (or "no sprints archived" if none).

## Mandatory first step: retrieve the current sprint (Jira MCP or REST API)

Check if `JIRA_TOKEN` environment variable is set to determine access method.
See `../mcp-api-access/jira-rest-api.md` for REST API details.

**If JIRA_TOKEN is available (REST API path):**

1. Fetch candidate boards:

   ```bash
   curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/agile/1.0/board?name=Sony&maxResults=100" \
     -H "Authorization: Bearer ${JIRA_TOKEN}"
   ```

2. Select the board (prefer `type == "scrum"` and name contains "sony" or "sonic")
3. Retrieve the current sprint:

   ```bash
   curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/agile/1.0/board/${BOARD_ID}/sprint?state=active" \
     -H "Authorization: Bearer ${JIRA_TOKEN}"
   ```

4. If no active sprint, retry with `state=future`
5. Choose the first returned sprint as "current"

**If JIRA_TOKEN is not available (MCP path):**

1. Before calling any Jira MCP tool, check its schema/descriptor first
   (whatever mechanism the runtime provides for tool schema inspection).

2. If Jira MCP authentication is required, complete the interactive sign-in flow
   and then retry the same Jira call.

3. Fetch candidate boards.
   - Call the Jira MCP tool `jira_get_agile_boards` with a name filter if the user
     mentioned one (for example "Sony", "Sonic", or a known team/board name);
     otherwise omit the filter and request up to 100 boards.

4. Select the board.
   - If exactly one board matches, use it.
   - If multiple match, prefer (in order):
      - `type == "scrum"`
      - board name contains "sony" or "sonic" (case-insensitive)
   - If still ambiguous, ask the user to pick the board name/id before proceeding.

5. Retrieve the current sprint.
   - Call `jira_get_sprints_from_board` with `state: "active"` for the selected `boardId`.
   - If no active sprint is returned, retry with `state: "future"` and explain that
     there is no active sprint.
   - Choose the first returned sprint as "current" (Jira returns active/future sprints
     ordered; if more than one active is returned, pick the one with the latest
     `startDate` if present).

## Create/update the sprint markdown file

Once you have the sprint object:

1. Compute:
   - `sprintId`: Jira sprint id
   - `sprintName`: Jira sprint name
   - `sprintKey`: a short, filesystem-safe key derived from `sprintName`. Prefer
     extracting the numeric sprint marker (e.g. `94.3` → `94-3`). If extraction
     fails, fall back to a short slug of the sprint name.
   - `sprintDir`: `<sprintsRoot>/<sprintKey>/`
   - (no `work.md` file)

2. Ensure the directory exists.

3. Create or update `<sprintDir>/sprint.md` with sprint metadata (see template below).
   - If the file already exists, update the frontmatter, **Sprint details** section,
     and **Sprint tickets** table; preserve any other user-written content below.

4. Populate the **Sprint tickets** table in `sprint.md`:
   - **REST API path:** call the sprint issues endpoint and iterate all returned issues.
   - **MCP path:** call `jira_get_sprint_issues` for the `sprintId` (use default `fields`).
   - For each issue write one row: `| <ISSUE_KEY> | <SUMMARY> | <STATUS> | <ASSIGNEE> |`
   - Link the key to its ticket file when it exists: `[<ISSUE_KEY>](./<ISSUE_KEY>/<ISSUE_KEY>.md)`
   - If the table already exists in the file, replace it in full (it reflects live Jira state).

5. Do not create a sprint-level `work.md` unless the user explicitly asks for one.

### `sprint.md` template

```markdown
---
sprint_id: <SPRINT_ID>
sprint_name: "<SPRINT_NAME>"
sprint_key: <SPRINT_KEY>
board_id: <BOARD_ID>
board_name: "<BOARD_NAME>"
project_key: <PROJECT_KEY>
project_id: <PROJECT_ID>
state: <active|future|closed>
start_date: <YYYY-MM-DD or empty>
end_date: <YYYY-MM-DD or empty>
goal: "<SPRINT_GOAL or empty>"
jira_sprint_url: https://jira.sie.sony.com/secure/GHViewer.jspa?rapidViewId=<BOARD_ID>&selectedSprintId=<SPRINT_ID>
synced_at: <ISO-8601 timestamp of when this file was last written>
---

# Sprint <SPRINT_NAME>

## Sprint details

| Field      | Value                                                                 |
| ---------- | --------------------------------------------------------------------- |
| Sprint ID  | `<SPRINT_ID>`                                                         |
| Sprint key | `<SPRINT_KEY>`                                                        |
| Board      | <BOARD_NAME> (`<BOARD_ID>`)                                           |
| Project    | <PROJECT_KEY> (`<PROJECT_ID>`)                                        |
| State      | <active/future/closed>                                                |
| Start date | <YYYY-MM-DD>                                                          |
| End date   | <YYYY-MM-DD>                                                          |
| Goal       | <SPRINT_GOAL>                                                         |
| Jira link  | [Open in Jira](https://jira.sie.sony.com/secure/GHViewer.jspa?rapidViewId=<BOARD_ID>&selectedSprintId=<SPRINT_ID>) |

## Sprint tickets

| Key | Summary | Status | Assignee |
| --- | ------- | ------ | -------- |
```

**How to derive `project_key` / `project_id`:**

- REST API path: pull `projectKey` from the first issue returned by the sprint issues
  endpoint, or call `/rest/api/2/project` and match by board.
- MCP path: if `jira_get_sprint_issues` returns at least one issue, read
  `fields.project.key` and `fields.project.id` from the first result. If no issues
  yet, call `jira_get_agile_boards` and inspect the board's `location.projectKey` /
  `location.projectId` fields.
- If the project key cannot be determined, leave the field empty and note `unknown`
  in the table.

## Optional next step: pull sprint issues into the workspace

If the user asks to sync issues, or if the workspace file is newly created:

**If JIRA_TOKEN is available (REST API path):**

1. Call the sprint issues endpoint:

   ```bash
   curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/agile/1.0/sprint/${SPRINT_ID}/issue" \
     -H "Authorization: Bearer ${JIRA_TOKEN}"
   ```

2. Parse the JSON response for issues array
3. Append a section to `work.md`:

```markdown
## Sprint issues

- TODO: list issues here (key, summary, status, assignee)
```

**If JIRA_TOKEN is not available (MCP path):**

1. Check the `jira_get_sprint_issues` schema/descriptor (MANDATORY before calling).
2. Call `jira_get_sprint_issues` for the `sprintId` (use default `fields`; do not set
   `expand` unless requested).
3. Append a section to `work.md`:

```markdown
## Sprint issues

- TODO: list issues here (key, summary, status, assignee)
```

## Create per-ticket markdown files (assigned to me)

When the user asks to generate markdown files for each ticket assigned to them in the
current sprint:

1. Ensure you have:
   - The current `sprintId`, `sprintName`, and `sprintDir` from the steps above
   - The current user identity from Jira

**If JIRA_TOKEN is available (REST API path):**

1. Get current user:

   ```bash
   curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/myself" \
     -H "Authorization: Bearer ${JIRA_TOKEN}"
   ```

2. Retrieve sprint issues:

   ```bash
   curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/agile/1.0/sprint/${SPRINT_ID}/issue" \
     -H "Authorization: Bearer ${JIRA_TOKEN}"
   ```

3. Filter to issues assigned to the current user (match on accountId)
4. For each filtered issue, get full details:

   ```bash
   curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}?fields=description,comment" \
     -H "Authorization: Bearer ${JIRA_TOKEN}"
   ```

**If JIRA_TOKEN is not available (MCP path):**

1. Get current user identity (use `jira_get_me` if available)
2. Retrieve sprint issues with `jira_get_sprint_issues`.
3. Filter to issues assigned to the current user.
   - Prefer matching on immutable user identity (e.g. account id) when present;
     fall back to display name/email only if that's all Jira returns.
4. For each filtered issue, call `jira_get_issue` with `additionalFields: ["description"]`
   for full details

**For both paths:**

### Carried-over ticket detection (scan all previous sprints, including archive)

Before creating any ticket file, check whether it already exists in a previous sprint:

1. Build the full list of candidate sprint directories:
   - All subdirectories of `<sprintsRoot>/` **except** `archive/` and the current `<sprintKey>` folder.
   - All subdirectories of `<sprintsRoot>/archive/`.
2. For each candidate directory (scan all of them, not just the most recent),
   check whether `<prevSprintDir>/<ISSUE_KEY>/` exists.
3. Take the match from the **most recent** previous sprint if the same ticket
   appears in more than one (this means it was carried over multiple times).

**If a previous folder is found for `<ISSUE_KEY>`:**

1. Move the entire ticket subfolder into the current sprint:

   ```bash
   mv <prevSprintDir>/<ISSUE_KEY>/ <sprintDir>/<ISSUE_KEY>/
   ```

2. After moving, update the ticket file's `## Status` section to reflect the
   current Jira status (do not overwrite user notes below that section).
3. Do not update the old sprint's `sprint.md` — the archive index and the current
   sprint's ticket table are the authoritative navigation points.

**If no previous folder exists for `<ISSUE_KEY>`** (genuinely new ticket):

- Create a fresh ticket subfolder and markdown file as normal (see template below).
- Ensure `ticketDir` exists before writing.
- Populate **Description** by formatting the returned description as markdown bullets.
- Populate **Notes / Updates** by pulling Jira history:
   - For REST API: Include comments from the response.
   - For MCP: Use `jira_get_issue` with `additionalFields: ["comment"]`. Optionally
     include `expand: "changelog"` when useful and not too large. Optionally call
     `jira_get_worklog` for time log entries.
   - Write a concise, dated bullet list under **Notes / Updates**.
   - Always format URLs as proper Markdown links or wrap them in angle brackets
     (`<...>`) to avoid bare-URL lint failures.

### New ticket file template

```markdown
# <ISSUE_KEY>: <SUMMARY>

## Status

- Jira status:
- Assignee:
- Jira: [<ISSUE_KEY>](https://jira.sie.sony.com/browse/<ISSUE_KEY>)

## Description

-

## Plan

-

## Notes / Updates

-

## Investigation

-
```

## Additional files for a ticket

When working on a ticket requires creating files beyond the main markdown
(e.g. scripts, diagrams, specs, test data, config snippets):

- Always place them inside the ticket's subfolder: `<sprintDir>/<ISSUE_KEY>/`
- Use descriptive filenames that make the file's purpose clear at a glance.
- Reference any such files from the ticket's `<ISSUE_KEY>.md` under a
  `## Artifacts` section so they are discoverable from the markdown.

### `conversation.md` — agent chat history

Skills and agents may create a `conversation.md` file inside the ticket subfolder
to record their chat history for that ticket. Treat it exactly like any other
artifact file:

- Move it with the ticket folder when carrying over to a new sprint.
- Reference it in the `## Artifacts` section of `<ISSUE_KEY>.md` when it exists.
- Do not delete or overwrite it during sprint sync operations.

**Using `conversation.md` as a fallback:** If `<ISSUE_KEY>.md` is missing or its
content is corrupted/incomplete, read `conversation.md` (if present) to recover
context — it may contain the last known status, decisions made, and notes from
prior agent sessions. Use that content to reconstruct or re-populate the ticket
file rather than starting from scratch.

Example structure (CPT-123 carried over from 94-3 into 94-4, with 94-3 closed and archived):

```text
sprints/
  archive/
    index.md               ← one row per archived sprint
    94-3/
      sprint.md
      CPT-121/
        CPT-121.md
  95-1/
    sprint.md
    CPT-123/               ← moved from archive/94-3, work notes preserved
      CPT-123.md
      conversation.md
      migration-plan.md
      seed-data.sql
    CPT-124/               ← new ticket, fresh file
      CPT-124.md
```

## Archive index

`<sprintsRoot>/archive/index.md` is a lightweight index of all archived sprints. Create it when the first sprint is archived; update it each time a sprint is moved to archive.

### Format

```markdown
# Sprint archive

| Sprint key | Sprint name | Start date | End date | Folder |
| ---------- | ----------- | ---------- | -------- | ------ |
| 94-3 | Sony Sprint 94.3 | 2024-01-15 | 2024-01-28 | [94-3](./94-3/sprint.md) |
```

### Rules

- One row per archived sprint, sorted by `end_date` descending (most recent first).
- The **Folder** column links to the sprint's `sprint.md` inside the archive.
- When adding a new row, insert it at the top of the table (after the header).
- Do not remove rows from this index — it is the permanent history of past sprints.

## Markdown output quality

Apply the rules defined in the `markdownlint` skill for all files created or modified.

## If Jira MCP is unavailable

If Jira MCP tools are unavailable or erroring, ask the user to paste either:

- the sprint issue list (key + summary + status + assignee), or
- just the list of issue keys

Then generate the same per-ticket markdown files from that pasted data.
