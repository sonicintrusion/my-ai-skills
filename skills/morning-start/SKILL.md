---
name: morning-start
description: >-
  Morning startup routine. Verifies VPN is connected, logs into all AWS profiles
  via toka, verifies configured service integrations, configures split-tunnel
  routing, updates Homebrew packages, and checks work services. Use when the user
  says "morning start", "start my day", "morning routine", "run startup", or
  "set up for the day". The workflow is runtime- and model-independent.
---

# Morning Start

## Goal

Run the morning startup sequence: verify VPN, then run AWS login, integration
discovery, split-tunnel routing, and Homebrew update in parallel. Check service
connectivity and report a combined result.

Use the tools and integration-discovery mechanism provided by the current AI
runtime. Do not assume a particular model, vendor, CLI, slash command, tool
prefix, or exact MCP function name. Match available tools to the required
operation by their documented capability.

The MCP calls below are fast-path examples, not required interfaces. Use an
example directly when the runtime exposes that operation; otherwise select the
equivalent available tool and preserve the same inputs, limits, and result.

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
- **Task B** — Service integration verification
- **Task C** — Split-tunnel routing
- **Task D** — Homebrew update

Collect all results before proceeding to Step 3 (Connectivity Checks).

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

### Task B: Verify Service Integration Configuration

Use the current runtime's native tool or integration discovery mechanism to
confirm that integrations for these services are available:

- Jira
- Confluence
- Slack

Some integrations are **disabled by default** to save context. Do not report
them as missing:

- **GitHub** — the MCP server is disabled; GitHub access uses the `gh` CLI
  (verified in Step 3, Task D).
- **ServiceNow** — the MCP server is disabled; it is only checked when the user
  has enabled it for the session (Step 3, Task H).

If the runtime already lists available tools in the session context, inspect
that list directly instead of making a separate discovery call.

Integration names and tool prefixes vary between runtimes. Match by capability,
not by a hardcoded server or function name. If the runtime does not expose a
configuration-listing operation, record this task as `DEFERRED` and use the
connectivity checks in Step 3 as the authoritative verification.

**Result handling:**

- **All three available** — record as `OK`.
- **Discovery unavailable** — record as `DEFERRED: verified by connectivity checks`.
- **Any missing** — record as `MISSING: <service>` and include it in the final
  report.

Do not direct the user to a vendor-specific CLI or slash command. If an
integration needs authentication, follow the portable auth handling in Step 3.

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

The script updates Homebrew packages non-interactively. Before running it,
identify whether the current AI client is installed through Homebrew and whether
the script protects that package from a mid-session upgrade. If it does not,
exclude that package using supported script options or defer the Homebrew task;
never replace the client executable that is running this workflow. This task may
take a few minutes, so run it in the background.

**Result handling:**

- **Exit 0** — record as `OK: N package(s) upgraded`.
- **Output reports a deferred update for the active AI client** — keep the
  update deferred and include the script's package-specific follow-up command in
  the final report. Do not upgrade the client that is executing this workflow.
- **Non-zero exit** — falls back to `brew doctor`; record result as
  `FAILED: <error>`.

---

## Step 3: Connectivity Checks (Parallel)

Once Step 2 is complete, launch all five checks **in parallel** using the best
available service-integration tools:

- **Task D** — GitHub access check (`gh` CLI)
- **Task E** — Jira access check
- **Task F** — Confluence access check
- **Task G** — Slack mentions check
- **Task H** — ServiceNow unresolved ticket count (only if enabled)

Collect all results before proceeding to Step 4 (Final Report).

### Auth error handling (applies to all Step 3 tasks)

> **Session auth gate.** Step 3 authenticates every service integration in the
> current session. If the runtime supports delegation, delegated agents may not
> inherit the current session's auth tokens. Complete auth before delegating any
> later work.
>
> **MCP first.** Prefer an available MCP or native connector. If it remains
> unavailable or lacks the required operation after authentication is handled,
> use the runtime's documented API or local-tool fallback and identify the
> fallback in the final report.

When a service integration returns an auth error:

1. Extract the auth URL from the error response and show it clearly to the user.
2. If the runtime can safely open URLs, open it in the user's default browser.
3. Ask the user to complete login and confirm when it is finished.
4. Retry the call once after confirmation.
5. **Retry succeeds** — record as `OK` and continue.
6. **Retry fails** — do not move on silently. Report the error and ask whether
   to retry or skip that service.
7. **Third attempt succeeds** — record as `OK` and continue.
8. **Third attempt fails** — record as `AUTH_FAILED: <service>` and **pause
   Step 3**. Do not delegate work for that service. Proceed only after every
   service is `OK` or the user explicitly chooses a fallback or skip.

---

### Task D: GitHub Access Check

Use the `gh` CLI to check the last commit on two repos. The GitHub MCP server
is disabled by default, so do not wait for or authenticate it. Run each lookup
as its own separate Bash call, in series. Do not wrap them in a shell loop
(`for ...; do ...; done`): a loop needs a wildcard allow rule that approves any
command, so a plain `gh api` call per repo is used instead.

```bash
gh api --hostname github.sie.sony.com \
  "repos/bis/cloud-docs/commits?per_page=1" \
  --jq '.[0] | "\(.sha[0:7]) by \(.commit.author.name) on \(.commit.author.date[0:10])"'
```

```bash
gh api --hostname github.sie.sony.com \
  "repos/data-platform/cloud-docs/commits?per_page=1" \
  --jq '.[0] | "\(.sha[0:7]) by \(.commit.author.name) on \(.commit.author.date[0:10])"'
```

For each, record the most recent commit's short SHA, author, and date.

**Result handling:**

- **Success** — record as `OK: <sha> by <author> on <date>`.
- **Auth error** — run `gh auth status --hostname github.sie.sony.com` and ask
  the user to run `gh auth login --hostname github.sie.sony.com`; retry once.
- **Other error** — record as `FAILED: <error>`.

---

### Task E: Jira Access Check

Use an available Jira integration to find the most recently created Jira issue
reported by the current user.

**Step 1:** Resolve the current user's identity.

Use the integration's current-user operation and extract its stable account
identifier, email address, or username.

```text
jira_get_me()
```

**Step 2:** Search for their most recent issue.

Search for one issue using `reporter = '<current-user>' ORDER BY created DESC`,
substituting the identity format supported by the integration.

```text
jira_search_issues(
  jql="reporter = '<current-user>' ORDER BY created DESC",
  maxResults=1
)
```

Record the issue key, summary, and creation date of the top result.

**Result handling:**

- **Success** — record as `OK: <issue-key> - <summary> (<date>)`.
- **Auth error** — follow Step 3 auth policy above.
- **No results** — record as `OK: no issues found`.
- **Other error** — record as `FAILED: <error>`.

---

### Task F: Confluence Access Check

Use an available Confluence integration to check the last-updated time for two
pages. Run lookups in series.

**Page 1:** DSOSYS space overview — fetch by page ID `1244322093`.

Fetch page ID `1244322093`, including version or history data when the
integration supports expansion fields.

```text
confluence_get_page(id="1244322093", expand=["version", "history"])
```

**Page 2:** Pillar Roadmap page — fetch by page ID `3116384383`.

Fetch page ID `3116384383`, including version or history data when the
integration supports expansion fields.

```text
confluence_get_page(id="3116384383", expand=["version", "history"])
```

For each, record the page title and last-modified date.

**Result handling:**

- **Success** — record as `OK: "<title>" last updated <date>`.
- **Auth error** — follow Step 3 auth policy above.
- **Other error** — record as `FAILED: <error>`.

### Task G: Slack Mentions Check

Use an available Slack integration to verify connectivity and request up to 20
mentions from the last 24 hours.

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

The ServiceNow MCP server is **disabled by default**. If no ServiceNow
integration is available in the session, record `SKIPPED: disabled by default`
and do not attempt authentication or a REST fallback. Run the steps below only
when the user has enabled ServiceNow for the session.

Use the available ServiceNow integration to count open or in-progress incidents
assigned to the current user.

**Step 1:** Resolve the current user's identity.

Use the integration's current-user operation and extract the email address or
stable user identifier.

```text
servicenow_get_user()
```

**Step 2:** Query their unresolved incidents.

Query the `incident` table for up to 100 records. Request `number`,
`short_description`, and `state`, and use the equivalent of
`assigned_to.email=<email>^state!=6^state!=7` for the integration's query
syntax.

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

- **Not enabled** — record as `SKIPPED: disabled by default`.
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
  Integrations:         3/3 available (GitHub via gh; ServiceNow off)
  Split-tunnel:         Routes configured
  Homebrew:             OK (active-client update deferred — see note below)

  GitHub (bis):         OK: abc1234 by author on 2026-07-22
  GitHub (data-platform): OK: def5678 by author on 2026-07-21
  Jira:                 OK: DPS-123 - Some ticket (2026-07-20)
  Confluence (DSOSYS):  OK: "Overview" last updated 2026-07-22
  Confluence (Pillar):  OK: "Pillar Roadmap - Platform Health O11y" last updated 2026-07-21
  Slack:                OK: no mentions in the last 24h
  ServiceNow:           SKIPPED: disabled by default

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
| Integration discovery unavailable | Defer verification to the Step 3 connectivity checks |
| Service integration missing | Report it as missing and name the unavailable capability |
| vpn-split-tunnel.bash not found | Report path not found, skip split-tunnel step |
| brew_update.sh not found | Report path not found, skip Homebrew step |
| Homebrew update fails | Record `FAILED: <error>` in summary |
| Active AI client update deferred | Record `OK` and surface the package-specific follow-up command emitted by the script |
| Split-tunnel fails | Report error, offer retry with `mobile` or `direct` mode |
| Any integration auth error (Step 3) | Surface the auth URL, help the user open it, and retry after confirmation |
| Retry still fails | Report the error and ask whether to retry, use a documented fallback, or skip |
| Third attempt fails | Record `AUTH_FAILED: <service>`, pause Step 3, and ask the user to retry or skip |
| `gh` not authenticated | Ask the user to run `gh auth login --hostname github.sie.sony.com`, then retry once |
| GitHub repo not found / access denied | Record as `FAILED: <error>` in summary |
| Jira returns no results | Record as `OK: no issues found` — not a failure |
| Confluence page not found | Record as `FAILED: page not found` in summary |
| Slack returns no mentions | Record as `OK: no mentions in the last 24h` — not a failure |
| ServiceNow MCP not enabled | Record as `SKIPPED: disabled by default` — not a failure |
| ServiceNow returns no results | Record as `OK: no unresolved incidents` — not a failure |
