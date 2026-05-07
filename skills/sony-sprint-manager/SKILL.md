---
name: sony-sprint-manager
description: Manage sprint work using Jira. Use when the user asks to manage their sprint, start sprint planning, create a sprint work file, or sync sprint context from Jira. First action: retrieve the current active sprint via the Jira MCP and create/update a markdown file under shared-agent-skills/sony/sprints/<sprint>/.
---

# Sony sprint manager

## Goal

Maintain a per-sprint markdown workspace file driven by Jira.

## Prerequisite: VPN connectivity

Jira MCP access may require being connected to the corporate VPN.

If Jira MCP calls fail with network/connection errors (or show “server errored” / “no servers available”):

- Ask the user to confirm they’re connected to VPN.
- Suggest reloading/restarting MCP servers after VPN is connected.

## Sprint workspace location

This skill must work in any IDE and any repository. All paths are **relative to the current workspace/repo root**.

1. Determine `workspaceRoot` (the repository root for the current workspace).
2. Determine `sprintsRoot`:
   - If the repo already has a `sprints/` folder at the root, use it.
   - Otherwise, create and use `sprints/` at the root.

Each sprint gets its own subfolder using a short sprint key (e.g. `94-3`): `<sprintsRoot>/<sprintKey>/`

Ticket markdown files live directly inside the sprint subfolder: `<sprintDir>/<ISSUE_KEY>.md`

## Mandatory first step: retrieve the current sprint (Jira MCP)

1. Before calling any Jira MCP tool, check its schema/descriptor first (whatever mechanism the runtime provides for tool schema inspection).

2. If Jira MCP authentication is required, complete the interactive sign-in flow and then retry the same Jira call.

3. Fetch candidate boards.
   - Call the Jira MCP tool `jira_get_agile_boards` with a name filter if the user mentioned one (for example “Sony”, “Sonic”, or a known team/board name); otherwise omit the filter and request up to 100 boards.

4. Select the board.
   - If exactly one board matches, use it.
   - If multiple match, prefer (in order):
     - `type == "scrum"`
     - board name contains “sony” or “sonic” (case-insensitive)
   - If still ambiguous, ask the user to pick the board name/id before proceeding.

5. Retrieve the current sprint.
   - Call `jira_get_sprints_from_board` with `state: "active"` for the selected `boardId`.
   - If no active sprint is returned, retry with `state: "future"` and explain that there is no active sprint.
   - Choose the first returned sprint as “current” (Jira returns active/future sprints ordered; if more than one active is returned, pick the one with the latest `startDate` if present).

## Create/update the sprint markdown file

Once you have the sprint object:

1. Compute:
   - `sprintId`: Jira sprint id
   - `sprintName`: Jira sprint name
   - `sprintKey`: a short, filesystem-safe key derived from `sprintName`. Prefer extracting the numeric sprint marker (e.g. `94.3` → `94-3`). If extraction fails, fall back to a short slug of the sprint name.
   - `sprintDir`: `<sprintsRoot>/<sprintKey>/`
   - (no `work.md` file)

2. Ensure the directory exists.

3. Do not create a sprint-level `work.md` unless the user explicitly asks for one.

## Optional next step: pull sprint issues into the workspace

If the user asks to sync issues, or if the workspace file is newly created:

1. Check the `jira_get_sprint_issues` schema/descriptor (MANDATORY before calling).
2. Call `jira_get_sprint_issues` for the `sprintId` (use default `fields`; do not set `expand` unless requested).
3. Append a section to `work.md`:

```markdown
## Sprint issues

- TODO: list issues here (key, summary, status, assignee)
```

## Create per-ticket markdown files (assigned to me)

When the user asks to generate markdown files for each ticket assigned to them in the current sprint:

1. Ensure you have:
   - The current `sprintId`, `sprintName`, and `sprintDir` from the steps above
   - The current user identity from Jira (use `jira_get_me` if available)

2. Retrieve sprint issues with `jira_get_sprint_issues`.

3. Filter to issues assigned to the current user.
   - Prefer matching on immutable user identity (e.g. account id) when present; fall back to display name/email only if that’s all Jira returns.

4. For each filtered issue, create a markdown file:
   - `ticketPath`: `<sprintDir>/<ISSUE_KEY>.md`
   - Never overwrite existing content; if the file exists, only update the frontmatter/header section (leave user notes intact).
   - Populate **Description** by calling `jira_get_issue` with `additionalFields: ["description"]` and formatting the returned description as markdown bullets.
   - Populate **Notes / Updates** by pulling Jira history:
     - Prefer `jira_get_issue` with `additionalFields: ["comment"]` (for comments). Optionally include `expand: "changelog"` when useful and not too large.
     - Optionally call `jira_get_worklog` for time log entries.
     - Write a concise, dated bullet list under **Notes / Updates**.
     - Always format URLs as proper Markdown links or wrap them in angle brackets (`<...>`) to avoid bare-URL lint failures.
     - If the user already has notes there, append a `**Jira history**` subsection rather than overwriting.

5. Use this file template for new files:

```markdown
# <ISSUE_KEY>: <SUMMARY>

## Status

- Jira status:
- Assignee:

## Description

-

## Plan

-

## Notes / Updates

-
```

## Markdown output quality

All markdown files generated by this skill must conform to the `markdownlint` skill rules. Key requirements:

- One blank line before and after every heading (MD022).
- Lists surrounded by blank lines (MD032).
- No trailing spaces (MD009).
- No multiple consecutive blank lines (MD012).
- No bare URLs — use `<https://…>` or `[text](url)` (MD034).
- Every file ends with a single newline (MD047).
- Fenced code blocks must specify a language (MD040).

## If Jira MCP is unavailable

If Jira MCP tools are unavailable or erroring, ask the user to paste either:

- the sprint issue list (key + summary + status + assignee), or
- just the list of issue keys

Then generate the same per-ticket markdown files from that pasted data.
