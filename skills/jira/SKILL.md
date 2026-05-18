---
name: jira
description: Interact with Jira tickets. Supports creating, updating, and closing tickets. Use when the user says "update 2965: comment", "close 2965", "create jira ticket", or "update bau: note". Parses ticket IDs, handles BAU tickets specially, and works with both MCP and REST API.
---

# Jira Skill

## Goal

Unified interface for Jira ticket operations: create, update, close, and special BAU ticket handling.

## Supported Operations

1. **Update** - Add a comment to any Jira ticket
2. **Close** - Transition a ticket to Done and update local markdown
3. **Update BAU** - Special handling for BAU tickets in sprints
4. **Create** - Create a new Jira ticket (future)

## Operation Detection

Parse the user's message to determine which operation:

- `update 2965: comment` → **Update ticket**
- `comment on DSOSYS-2965: message` → **Update ticket**
- `close 2965` → **Close ticket**
- `close 2965: closing note` → **Close ticket with comment**
- `update bau: daily notes` → **Update BAU ticket**
- `create ticket: summary` → **Create ticket** (future)

## Common: Ticket ID Expansion

If the user provides a bare number (e.g. `2965`), expand it to `DSOSYS-<number>`.
If a full key is provided (e.g. `DSOSYS-2965` or `ABC-123`), use it as-is.

## Common: API Access Method

Priority order for Jira access:

1. **Try MCP first** - Use Jira MCP tools if available
2. **Fallback to REST API** - If MCP unavailable, check for `JIRA_TOKEN` and use REST API

**REST API base URL:** `${JIRA_BASE_URL:-https://jira.sie.sony.com}`

See `../api-fallback/jira-rest-api.md` for REST API details.

---

# Operation 1: Update Ticket

Add a comment to any Jira ticket.

## Input Patterns

- `update 2965: investigation ok`
- `please update DSOSYS-2965: investigation ok`
- `comment on 2965: investigation ok`

## Step 1: Parse input

- Extract ticket ID (expand if needed)
- Extract comment text (everything after first `:`)
- If no ticket ID: ask "Which Jira ticket should I update?"
- If no comment: ask "What comment should I add to <ticket-key>?"

## Step 2: Post the comment

**Try MCP first:**

Call `jira_add_issue_comment` with:
- `key`: the resolved ticket key
- `comment`: the message text in Jira wiki markup

If MCP is available and succeeds, done.

**If MCP fails or unavailable (REST API fallback):**

Check if `JIRA_TOKEN` environment variable is set. If yes:

```bash
curl -X POST "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/comment" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"${COMMENT_TEXT}\"}"
```

Check response status (201 = success).

**If both fail:**

Suggest VPN check, MCP restart, or verify REST API token.

## Step 3: Confirm

Report:
- The ticket key updated
- Success/failure status
- Preview of the comment added

## Jira Wiki Markup Reference

- Bold: `*text*`
- Italic: `_text_`
- Code: `{{text}}`
- Code block: `{code}...{code}`
- Link: `[text|url]` or `[url]`
- List: `* item`
- Numbered: `# item`
- Heading: `h2. Title`
- Do NOT use markdown syntax

---

# Operation 2: Close Ticket

Transition a Jira ticket to Done and update local markdown file.

## Input Patterns

- `close 2965`
- `close DSOSYS-2965`
- `mark 2965 as done`
- `close 2965: all dashboards delivered` (with closing note)

## Step 1: Parse input

- Extract ticket ID (expand if needed)
- Extract optional closing note (everything after `:`)
- If no ticket ID: ask "Which Jira ticket should I close?"

## Step 2: Locate local markdown file

1. Determine workspace root
2. Search `sprints/` recursively for `<TICKET-KEY>.md`
3. If found, remember path for Step 5
4. If not found, skip markdown update

## Step 3: Transition to Done

**Try MCP first:**

1. Call `jira_get_transitions` for available transitions
2. Find transition named `Done` (fallback: `Closed`, then `Resolved`)
3. Call `jira_transition_issue` with transition ID and resolution field
4. If `resolution` cannot be set during transition, make separate update call

If MCP succeeds, proceed to Step 4. Do NOT update markdown until Jira succeeds.

**If MCP fails or unavailable (REST API fallback):**

Check if `JIRA_TOKEN` is set. If yes:

```bash
# Get transitions
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/transitions" \
  -H "Authorization: Bearer ${JIRA_TOKEN}"

# Execute transition
curl -X POST "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/transitions" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "transition": { "id": "TRANSITION_ID" },
    "fields": { "resolution": { "name": "Done" } }
  }'
```

**If both fail:**

Ask user to check VPN. Suggest MCP restart or verify REST API token.

## Step 4: Post closing comment (optional)

If closing note was provided:

**Try MCP first:**

Call `jira_add_issue_comment` with the closing note in Jira wiki markup.

**If MCP fails (REST API fallback):**

If `JIRA_TOKEN` is set:

```bash
curl -X POST "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/comment" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"${COMMENT_TEXT}\"}"
```

## Step 5: Update local markdown

If markdown file was found in Step 2:

1. In `## Status` section, update `- Jira status:` to `Done`
2. Add/update `- Closed:` line with today's date (YYYY-MM-DD)
3. If closing note provided, append to `### Comments` under `## Notes / Updates`:
   - Format: `- <TODAY_DATE> — <closing_note>`
4. Ensure proper markdown formatting (no bare URLs, trailing spaces, etc.)

## Step 6: Confirm

Report:
- Ticket key closed
- Jira transition success/failure
- Resolution status
- Closing comment posted (if applicable)
- Markdown file updated (path and changes)

---

# Operation 3: Update BAU Ticket

Special handling for BAU (Business As Usual) tickets in the current sprint.

## Input Patterns

- `update bau: daily standup notes`
- `update bau ticket: completed dashboard review`

## Step 1: Locate BAU ticket file

1. Determine workspace root
2. Locate `sprints/` folder
3. Find most recent sprint subfolder (highest sort order, e.g. `94-4` > `94-3`)
   - User can specify sprint if needed
4. Search for BAU ticket:
   - Read first line of each `.md` file
   - Match if heading contains "BAU" (case-insensitive)
   - Prefer file with "BAU:" followed by sprint number or user name
5. If no match in recent sprint, search all sprint folders (newest first)
6. If still no match, report "No BAU ticket found" and stop

## Step 2: Parse update content

The user's message after "update bau:" is the update content. Can be:
- Description of work done
- URL or Slack link
- Combination of both

If no content provided, ask: "What would you like to add to the BAU ticket?"

## Step 3: Append to local markdown

1. Open the BAU ticket file
2. Locate `### Comments` under `## Notes / Updates`
   - If section doesn't exist, create it at end of file
3. Append new bullet: `- <TODAY_DATE> — <update_content>`
   - Use YYYY-MM-DD format for date
   - Wrap bare URLs in `<...>` for markdown compliance

## Step 4: Post to Jira

1. Derive Jira key from filename (e.g. `DSOSYS-2886.md` → `DSOSYS-2886`)

**Try MCP first:**

Call `jira_add_issue_comment` with the update in Jira wiki markup.

**If MCP fails or unavailable (REST API fallback):**

Check if `JIRA_TOKEN` is set. If yes:

```bash
curl -X POST "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/comment" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"${COMMENT_TEXT}\"}"
```

If Jira fails (network/auth), still complete local markdown update. Suggest VPN check, MCP restart, or verify REST API token.

## Step 5: Confirm

Report:
- BAU ticket file path updated
- Exact bullet appended
- Jira comment posted (success/failure with issue key)

---

# Operation 4: Create Ticket (Future)

Placeholder for creating new Jira tickets. To be implemented.

---

# Error Handling

## Network/VPN Errors

- Ask user to confirm VPN connection
- Suggest reloading/restarting MCP servers
- For REST API: verify token and base URL

## Authentication Errors

- REST API (401/403): Token invalid or insufficient permissions
- MCP: Interactive sign-in may be required

## Not Found Errors

- Ticket doesn't exist
- Invalid ticket key format
- BAU ticket file not found in sprints

## Validation Errors

- Missing required fields
- Invalid transition (ticket already in Done state)
- Invalid comment format

---

# Markdown Quality Rules

All markdown file edits must follow these rules:

- No bare URLs — use `<https://…>` or `[text](url)` (MD034)
- No trailing spaces (MD009)
- Lists surrounded by blank lines (MD032)
- One blank line before/after headings (MD022)
- No multiple blank lines (MD012)
- File ends with single newline (MD047)
- Fenced code blocks specify language (MD040)

---

# Integration with Other Skills

This skill is called by:
- Direct user commands (`update 2965: comment`)
- Potentially other skills that need Jira operations

Other skills (sprint-manager, pillar-sync) use Jira MCP/REST API directly and do NOT invoke this skill.
