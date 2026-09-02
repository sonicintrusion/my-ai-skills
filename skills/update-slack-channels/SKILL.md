---
name: update-slack-channels
description: >-
  Fetches the full list of Slack channels the user is a member of and updates
  the reference_slack_channels.md memory file. Use when the user says "update
  slack channels", "refresh my channel list", "sync slack channels to memory",
  or "update channels memory".
---

# Update Slack Channels Memory

## Goal

Fetch all Slack channels and group DMs the current user is a member of, then
overwrite `memory/reference_slack_channels.md` with an up-to-date snapshot.

---

## Step 1: Resolve the Slack User ID

First, read the user profile memory:

```text
Read: ./memory/user_profile.md
```

Look for a `Slack` row in the Identifiers table. If it exists and the value is
not `TBD` or a placeholder, extract it as `SLACK_USER_ID` and derive
`SLACK_DISPLAY_NAME` from context (e.g. the local username `KNguyen`). Skip to
Step 2.

If the Slack row is missing or set to `TBD`, call:

```text
get_my_user()
```

Extract `user_id` and `display_name` (or `full_name`). Store as `SLACK_USER_ID`
and `SLACK_DISPLAY_NAME`.

Then update the `user_profile.md` Identifiers table — replace the `TBD` value
(or add a new row) with the resolved `SLACK_USER_ID`.

If this call returns an auth error, follow the standard auth-retry policy:
open the auth URL in Google Chrome Dev, wait 30 seconds, then retry once.

---

## Step 2: Fetch All Channels

Call `list_my_channels` in pages until exhausted. Use `limit=200` per page and
follow `next_cursor` until it is empty.

```text
list_my_channels(limit=200, types=["public_channel", "private_channel"])
```

Collect every channel object across all pages. Deduplicate by `channel_id`.

---

## Step 3: Classify Channels

Split the collected list into three groups based on Slack API flags:

| Group | Criteria |
| --- | --- |
| Public Channels | `is_private == false`, `is_mpim == false` |
| Private Channels | `is_private == true`, `is_mpim == false` |
| Private Groups (MPIM) | `is_mpim == true` |

Sort each group alphabetically by channel name.

---

## Step 4: Write the Memory File

Overwrite `memory/reference_slack_channels.md` in the project root of
`my-assistant` (`/Users/KNguyen/dev/sie/github/d-knguyen/my-assistant/`).

Use this exact format:

```markdown
---
name: reference_slack_channels
description: Full list of Slack channels and group DMs the user is a member of, with IDs and purposes — exported <YYYY-MM-DD>
metadata:
  type: reference
---

Exported: <YYYY-MM-DD> — Total: <N> (<pub> public, <priv> private, <mpim> private groups)
Slack user: <SLACK_DISPLAY_NAME> (<SLACK_USER_ID>)

## Public Channels (<pub>)

| Channel | ID | Purpose |
| --- | --- | --- |
| #<name> | <id> | <purpose> |
...

## Private Channels (<priv>)

| Channel | ID | Purpose |
| --- | --- | --- |
| #<name> | <id> | <purpose> |
...

## Private Groups (<mpim>)

| Group | ID | Purpose |
| --- | --- | --- |
| <name> | <id> | <purpose> |
...
```

- Replace `<YYYY-MM-DD>` with today's date.
- Use the channel `purpose.value` field for Purpose; leave blank if empty.
- Private groups do not get a `#` prefix.
- Update counts to match actual results.

---

## Step 5: Report

Print a one-line summary:

```text
Slack channels updated: <N> total (<pub> public, <priv> private, <mpim> groups) — memory/reference_slack_channels.md
```
