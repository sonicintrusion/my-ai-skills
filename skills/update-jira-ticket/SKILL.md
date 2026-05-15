---
name: update-jira-ticket
description: Post a comment on any Jira ticket. Use when the user says something like "update 2965: investigation ok", "comment on DSOSYS-1234: fixed", or "please update <ticket-id>: <message>". Extracts the ticket ID and comment from the user's message and posts it to Jira.
---

# Update Jira Ticket

## Goal

Parse a ticket ID and message from the user's input and post the message as a comment on the corresponding Jira ticket.

## Step 1: Parse the user's input

The user's message follows one of these patterns:

- `update 2965: investigation ok`
- `please update DSOSYS-2965: investigation ok`
- `comment on 2965: investigation ok`

**Ticket ID extraction:**

- If the user provides a bare number (e.g. `2965`), expand it to a full Jira key by prepending the default project prefix `DSOSYS-`.
- If the user provides a full key (e.g. `DSOSYS-2965` or `ABC-123`), use it as-is.

**Message extraction:**

- Everything after the first `:` in the user's message is the comment body.
- Strip leading/trailing whitespace from the comment.

If no ticket ID is found, ask the user: "Which Jira ticket should I update?"
If no message is found after the colon, ask the user: "What comment should I add to <ticket-key>?"

## Step 2: Post the comment

Call `jira_add_issue_comment` with:
- `key`: the resolved ticket key (e.g. `DSOSYS-2965`).
- `comment`: the extracted message text, formatted using Jira wiki markup (see below).

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

If the call fails with a network/auth error:
- Ask the user to confirm they are connected to VPN.
- Suggest reloading/restarting MCP servers after reconnecting.

If Jira returns any other error, report it to the user and do not retry.

## Step 3: Confirm

Report:
- The ticket key that was updated.
- Whether the comment was posted successfully.
- A short preview of the comment that was added.
