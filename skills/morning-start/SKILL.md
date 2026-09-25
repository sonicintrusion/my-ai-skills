---
name: morning-start
description: >-
  Morning startup routine. Verifies VPN is connected, logs into all AWS profiles
  via toka, reconnects all configured Claude MCP servers, and runs the
  vpn-split-tunnel.bash script to configure split-tunnel routing. Use when the
  user says "morning start", "start my day", "morning routine", "run startup",
  or "set up for the day".
---

# Morning Start

## Goal

Run the morning startup sequence: verify VPN, then run AWS login, MCP reconnect,
and split-tunnel routing in parallel, and report a combined result.

---

## Step 1: Verify VPN Connection

Check that the VPN is connected before proceeding with any other steps.

### 1a — Curl internal host

```bash
curl -s -o /dev/null -w "%{http_code}" --max-time 3 https://jira.sie.sony.com | grep -q "^[0-9]" && echo "VPN_OK" || echo "VPN_FAIL"
```

### 1b — Handle result

- **VPN_OK** — proceed to Step 2.
- **VPN_FAIL** — stop and report:

  > VPN does not appear to be connected. Please connect to the VPN and then
  > re-run `/morning-start`.

  Do not proceed until VPN is confirmed.

---

## Step 2: Parallel Startup Tasks

Once VPN is confirmed, launch all four tasks **in parallel** (do not wait for
one before starting the next):

- **Task A** — AWS login via toka
- **Task B** — MCP server reconnect
- **Task C** — Split-tunnel routing
- **Task D** — Homebrew update

Collect all results before proceeding to Step 3 (Final Report).

---

### Task A: AWS Login via toka

Run `toka` to log into all configured AWS profiles.

```bash
toka
```

`toka` is aliased to `~/scripts/sony/sony-aws-login.bash aloy`. It logs into
all Sony AWS profiles non-interactively. The command may take 10–30 seconds.

**Result handling:**

- **Exit 0** — record as `OK`.
- **Non-zero exit or error output** — record as `FAILED: <error>`.

---

### Task B: Verify MCP Server Configuration

Run the list command to confirm all expected MCP servers are registered:

```bash
claude mcp list
```

Check that all three servers appear in the output:

- `sie-jira-mcp`
- `sie-github-mcp`
- `sie-confluence-mcp`

**Result handling:**

- **All three present** — record as `OK`.
- **Any missing** — record as `MISSING: <server-name>` and include in the final report.

After reporting, instruct the user:

> MCP servers connect and authenticate on first use in each session. If a
> server fails with an auth error during the session, run `/mcp` in the Claude
> Code chat to see server status and follow any auth prompts that appear.

---

### Task C: Configure Split-Tunnel Routing

```bash
~/dev/sonic/github/bunch-o-scripts/vpn-split-tunnel.bash
```

Run without arguments (uses the default router `192.168.12.1`).

**Result handling:**

- **Exit 0** — record as `OK`.
- **Non-zero exit** — record as `FAILED: <error>`. After the parallel phase,
  ask the user if they want to retry with a different mode (`mobile` or
  `direct`):

  ```bash
  ~/dev/sonic/github/bunch-o-scripts/vpn-split-tunnel.bash mobile
  # or
  ~/dev/sonic/github/bunch-o-scripts/vpn-split-tunnel.bash direct
  ```

---

### Task D: Homebrew Update

Run the brew update script to keep packages current.

```bash
~/dev/sonic/github/bunch-o-scripts/brew_update.sh
```

Runs `brew update && brew upgrade && brew autoremove && brew cleanup` with
`HOMEBREW_NO_INTERACTIVE=1` to suppress prompts and proceed non-interactively.
This may take a few minutes depending on how many packages need updating. Run in
background so it does not block the parallel phase from completing.

**Result handling:**

- **Exit 0, no `CLAUDE_PENDING` line** — record as `OK: N package(s) upgraded`.
- **Exit 0, `CLAUDE_PENDING` line present** — record as
  `OK: N package(s) upgraded` and include a note:
  > ⚠️ `claude-code` has an update available. Run `brew upgrade claude-code`
  > after restarting your session to avoid breaking a live session.
- **Non-zero exit** — falls back to `brew doctor`; record result as
  `FAILED: <error>`.

---

## Step 3: Connectivity Checks (Parallel)

Once Step 2 is complete, launch all five checks **in parallel** using MCP tools:

- **Task D** — GitHub access check
- **Task E** — Jira access check
- **Task F** — Confluence access check
- **Task G** — Slack unread count
- **Task H** — ServiceNow unresolved ticket count

Collect all results before proceeding to Step 4 (Final Report).

### Auth error handling (applies to all Step 3 tasks)

When any MCP call returns an auth error:

1. Extract the auth URL from the error response.
2. Open it automatically — **do not prompt the user**:

   ```bash
   open -a "Google Chrome Dev" "<auth-url>"
   ```

3. Wait 20 seconds without interrupting the user:

   ```bash
   sleep 20
   ```

4. Retry the call once.
5. **Retry succeeds** — record as `OK` and continue.
6. **Retry fails** — record as `AUTH_FAILED: <server-name>` and include in
   the final report. Do not prompt the user at this point.

---

### Task D: GitHub Access Check

Use the `sie-github-mcp` MCP server to check the last commit on two repos.
Run lookups in series.

**Repo 1:** `bis/cloud-docs`

```text
github_list_commits(owner="bis", repo="cloud-docs", perPage=1)
```

**Repo 2:** `data-platform/cloud-docs`

```text
github_list_commits(owner="data-platform", repo="cloud-docs", perPage=1)
```

For each, record the most recent commit's short SHA, author, and date.

**Result handling:**

- **Success** — record as `OK: <sha> by <author> on <date>`.
- **Auth error** — follow Step 3 auth policy above.
- **Other error** — record as `FAILED: <error>`.

---

### Task E: Jira Access Check

Use the `sie-jira-mcp` MCP server to find the most recently created Jira issue
by the current user.

**Step 1:** Resolve the current user's identity.

```text
jira_get_me()
```

Extract the `emailAddress` (or `name`) field from the response.

**Step 2:** Search for their most recent issue.

```text
jira_search_issues(jql="reporter = '<emailAddress>' ORDER BY created DESC", maxResults=1)
```

Record the issue key, summary, and creation date of the top result.

**Result handling:**

- **Success** — record as `OK: <issue-key> - <summary> (<date>)`.
- **Auth error** — follow Step 3 auth policy above.
- **No results** — record as `OK: no issues found`.
- **Other error** — record as `FAILED: <error>`.

---

### Task F: Confluence Access Check

Use the `sie-confluence-mcp` MCP server to check the last-updated time for two
pages. Run lookups in series.

**Page 1:** DSOSYS space overview — fetch by page ID `1244322093`.

```text
confluence_get_page(id="1244322093", expand=["version","history"])
```

**Page 2:** Pillar Roadmap page — fetch by page ID `3116384383`.

```text
confluence_get_page(id="3116384383", expand=["version","history"])
```

For each, record the page title and last-modified date.

**Result handling:**

- **Success** — record as `OK: "<title>" last updated <date>`.
- **Auth error** — follow Step 3 auth policy above.
- **Other error** — record as `FAILED: <error>`.

### Task G: Slack Mentions Check

Use the `sie-slack-mcp` MCP server to verify Slack connectivity and check for
any @mentions in the last 24 hours.

```text
list_mentions(since="1d", count=20)
```

**Result handling:**

- **Mentions found** — record as `OK: <N> mention(s) in the last 24h`.
- **No mentions** — record as `OK: no mentions in the last 24h`.
- **Auth error** — follow Step 3 auth policy above.
- **Other error** — record as `FAILED: <error>`.

---

### Task H: ServiceNow Unresolved Ticket Count

Use the `sie-servicenow-mcp` MCP server to count open/in-progress incidents
assigned to the current user.

**Step 1:** Resolve the current user's identity.

```text
servicenow_get_user()
```

Extract the `email` field from the response.

**Step 2:** Query their unresolved incidents.

```text
servicenow_query_table(
  table="incident",
  sysparm_query="assigned_to.email=<email>^state!=6^state!=7",
  sysparm_fields="number,short_description,state",
  sysparm_limit=100
)
```

State values `6` (Resolved) and `7` (Closed) are excluded so only active
tickets are counted. Count the returned records.

**Result handling:**

- **Success** — record as `OK: <N> unresolved incidents`.
- **Zero results** — record as `OK: no unresolved incidents`.
- **Auth error** — follow Step 3 auth policy above.
- **Other error** — record as `FAILED: <error>`.

---

## Step 4: Final Report

Print a concise startup summary:

```text
Morning startup complete.

  VPN:                  Connected
  AWS (toka):           OK
  MCP servers:          3/3 configured
  Split-tunnel:         Routes configured
  Homebrew:             OK (claude-code update pending — see note below)

  GitHub (bis):         OK: abc1234 by author on 2026-07-22
  GitHub (data-platform): OK: def5678 by author on 2026-07-21
  Jira:                 OK: DPS-123 - Some ticket (2026-07-20)
  Confluence (DSOSYS):  OK: "Overview" last updated 2026-07-22
  Confluence (Pillar):  OK: "Pillar Roadmap - Platform Health O11y" last updated 2026-07-21
  Slack:                OK: no mentions in the last 24h
  ServiceNow:           OK: 3 unresolved incidents

Ready.
```

Adjust counts and statuses to reflect actual results. If any step failed but
the user chose to continue, mark that step as `FAILED` in the summary.

---

## Error Handling

| Situation | Action |
|---|---|
| VPN not connected | Stop immediately, prompt user to connect VPN |
| `toka` not found | Report "`toka` alias not found — check shell profile" |
| toka fails | Report error, ask user whether to continue |
| `claude` CLI not found | Report "claude CLI not found — check PATH" |
| MCP server missing from list | Report as missing, remind user to re-add with `claude mcp add` |
| vpn-split-tunnel.bash not found | Report path not found, skip split-tunnel step |
| brew_update.sh not found | Report path not found, skip Homebrew step |
| Homebrew update fails | Record `FAILED: <error>` in summary |
| `CLAUDE_PENDING` in brew output | Record `OK` but surface note: "claude-code update available — run `brew upgrade claude-code` after restarting session" |
| Split-tunnel fails | Report error, offer retry with `mobile` or `direct` mode |
| Any MCP auth error (Step 3) | Auto: open auth URL in Chrome Dev, `sleep 20`, retry once — no user prompt |
| MCP auth still fails after retry | Record as `AUTH_FAILED: <server>` in summary |
| GitHub repo not found / access denied | Record as `FAILED: <error>` in summary |
| Jira returns no results | Record as `OK: no issues found` — not a failure |
| Confluence page not found | Record as `FAILED: page not found` in summary |
| Slack returns no mentions | Record as `OK: no mentions in the last 24h` — not a failure |
| ServiceNow returns no results | Record as `OK: no unresolved incidents` — not a failure |
