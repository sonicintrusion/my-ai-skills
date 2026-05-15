---
name: update-bau-ticket
description: Update the BAU (Business As Usual) ticket in the current sprint. Use when the user wants to add a comment, note, or update to their BAU ticket. Traverses the sprints/ folder to find the BAU ticket file, appends the user's update to the Comments section under Notes / Updates, and posts the same update as a comment on the Jira ticket.
---

# Update BAU ticket

## Goal

Find the BAU ticket markdown file in the current sprint, append a dated update to it, and post the same update as a comment on the corresponding Jira ticket.

## Step 1: Locate the BAU ticket file

1. Determine `workspaceRoot` (the repository root for the current workspace).
2. Locate the `sprints/` folder at the workspace root.
3. Find the most recent sprint subfolder:
   - List all subdirectories under `sprints/`.
   - Select the one that sorts last (highest sprint number, e.g. `94-3` before `94-4`).
   - If the user specifies a sprint key, use that folder instead.
4. Within the sprint folder, find the BAU ticket file:
   - A BAU ticket is identified by its title containing "BAU" (case-insensitive) in the `# Heading` on line 1.
   - Read each `.md` file's first line to check.
   - If exactly one match is found, use it.
   - If multiple matches are found, prefer the one whose title contains "BAU:" followed by a sprint number or the user's name.
   - If no match is found in the most recent sprint, search all other sprint subfolders (newest first) and use the first BAU ticket found.
   - If still no match, tell the user no BAU ticket was found and stop.

## Step 2: Parse the user's input

The user's message after invoking the skill is the update content. It may be:

- A single sentence or short paragraph describing what was done.
- A URL or Slack link to attach.
- A combination (description + link).

If the user provides no content (invokes the skill with no arguments), ask: "What would you like to add to the BAU ticket?"

## Step 3: Append the update

1. Open the BAU ticket file.
2. Locate the `### Comments` section under `## Notes / Updates`.
   - If `### Comments` does not exist, append it at the end of the file (before the final newline).
3. Append a new bullet at the end of the `### Comments` list:
   - Format: `- <TODAY_DATE> — <update_content>`
   - `TODAY_DATE` is today's date in `YYYY-MM-DD` format.
   - Format any bare URLs as `<https://…>` to comply with markdown lint rules.
   - Do not modify any existing content.

## Step 4: Post a comment on the Jira ticket

1. Derive the Jira issue key from the BAU ticket filename (e.g. `DSOSYS-2886.md` → `DSOSYS-2886`).
2. Call `jira_add_issue_comment` with:
   - `key`: the issue key derived above.
   - `comment`: the update content provided by the user, formatted using Jira wiki markup (see below).
3. If the Jira MCP call fails with a network/auth error:
   - Ask the user to confirm they are connected to VPN.
   - Suggest reloading/restarting MCP servers after reconnecting.
   - Still complete the local markdown update even if Jira is unreachable.
4. If Jira returns an error other than a network/auth issue, report it to the user but do not retry.

**Jira wiki markup rules for comments:**

- Bold: `*text*`
- Italic: `_text_`
- Inline code: `{{text}}`
- Code block: `{code}...{code}`
- Link: `[text|https://url]` or `[https://url]`
- Bullet list: `* item` (one `*` per indent level)
- Numbered list: `# item`
- Heading: `h2. Title`
- Horizontal rule: `----`
- Do **not** use markdown syntax (`**`, `##`, backticks, `[text](url)`) — Jira renders it as raw text.
- Plain prose with no special formatting needs no markup at all.

## Step 5: Confirm

After completing both steps, report:

- The file path that was updated and the exact bullet that was appended.
- Whether the Jira comment was posted successfully (include the issue key).

## Markdown output quality

All edits must conform to markdownlint rules:

- No bare URLs — use `<https://…>` or `[text](url)` (MD034).
- No trailing spaces (MD009).
- Lists surrounded by blank lines (MD032).
- Every file ends with a single newline (MD047).
