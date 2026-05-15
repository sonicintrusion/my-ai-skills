---
name: close-jira-ticket
description: Close a Jira ticket and mark its local markdown file as done. Use when the user says "close 2965", "close DSOSYS-1234", "mark 2965 as done", or "close ticket 2965: optional closing note". Sets the Jira ticket to Done, updates the resolution, and updates the Status section in the local markdown file.
---

# Close Jira Ticket

## Goal

Transition a Jira ticket to Done, set its resolution, optionally post a closing comment, and update the local markdown file's Status section to reflect the closed state.

## Step 1: Parse the user's input

The user's message follows one of these patterns:

- `close 2965`
- `close DSOSYS-2965`
- `mark 2965 as done`
- `close 2965: all dashboards delivered`

**Ticket ID extraction:**

- If the user provides a bare number (e.g. `2965`), expand it to `DSOSYS-<number>`.
- If the user provides a full key (e.g. `DSOSYS-2965` or `ABC-123`), use it as-is.

**Optional closing note:**

- If the message contains a `:` after the ticket ID, everything after the first `:` is a closing comment to post on the ticket.
- If no note is given, skip posting a comment.

If no ticket ID can be found, ask: "Which Jira ticket should I close?"

## Step 2: Locate the local markdown file

1. Determine `workspaceRoot` (the repository root for the current workspace).
2. Recursively search `sprints/` for a file named `<TICKET-KEY>.md` (e.g. `DSOSYS-2965.md`).
3. If found, use it in Step 5. If not found, skip the markdown update and tell the user.

## Step 3: Transition the ticket to Done

1. Call `jira_get_transitions` with the ticket key to retrieve available transitions.
2. Find the transition whose name matches `Done` (case-insensitive). If no exact match, prefer `Closed`, then `Resolved`.
3. Call `jira_transition_issue` with:
   - `key`: the ticket key.
   - `transitionId`: the ID found above.
   - `fields`: `{ "resolution": { "name": "Done" } }` — include this only if the transition accepts field updates (check the transition's `fields` from step 1; if `resolution` is listed, include it).
4. If `resolution` cannot be set during transition, call `jira_update_issue` afterwards with `fields: { "resolution": { "name": "Done" } }`.

If the transition call fails with a network/auth error:

- Ask the user to confirm they are connected to VPN.
- Suggest reloading/restarting MCP servers after reconnecting.
- Do not proceed with the markdown update until Jira succeeds.

If no `Done`/`Closed`/`Resolved` transition is found, list the available transitions and ask the user which one to use.

## Step 4: Post a closing comment (optional)

If the user provided a closing note in Step 1, call `jira_add_issue_comment` with:

- `key`: the ticket key.
- `comment`: the closing note text, formatted using Jira wiki markup (see below).

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

## Step 5: Update the local markdown file

1. Open the markdown file found in Step 2.
2. In the `## Status` section, find the line starting with `- Jira status:` and change its value to `Done`.
3. Add or update a `- Closed:` line immediately below `- Jira status:` with today's date in `YYYY-MM-DD` format.
4. If a closing note was provided, append a new bullet to `### Comments` under `## Notes / Updates`:
   - Format: `- <TODAY_DATE> — <closing_note>`
5. Do not modify any other content.

**Markdown quality rules:**

- No bare URLs — use `<https://…>` or `[text](url)` (MD034).
- No trailing spaces (MD009).
- Lists surrounded by blank lines (MD032).
- Every file ends with a single newline (MD047).

## Step 6: Confirm

Report:

- The ticket key that was closed.
- Whether the Jira transition succeeded and what the new status is.
- Whether the resolution was set.
- Whether a closing comment was posted (if applicable).
- The markdown file path that was updated and the exact changes made.
