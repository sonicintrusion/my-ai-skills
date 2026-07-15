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

### 1a — Ping internal host

```bash
ping -c 1 -t 3 jira.sie.sony.com > /dev/null 2>&1 && echo "VPN_OK" || echo "VPN_FAIL"
```

### 1b — Handle result

- **VPN_OK** — proceed to Step 2.
- **VPN_FAIL** — stop and report:

  > VPN does not appear to be connected. Please connect to the VPN and then
  > re-run `/morning-start`.

  Do not proceed until VPN is confirmed.

---

## Step 2: Parallel Startup Tasks

Once VPN is confirmed, launch all three tasks **in parallel** (do not wait for
one before starting the next):

- **Task A** — AWS login via toka
- **Task B** — MCP server reconnect
- **Task C** — Split-tunnel routing

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

### Task B: Reconnect Claude MCP Servers

Run the reconnect command for each configured MCP server in sequence:

```bash
claude mcp reconnect sie-jira-mcp
claude mcp reconnect sie-github-mcp
claude mcp reconnect sie-confluence-mcp
```

**Result handling for each server:**

- **Success** — record as connected.
- **Auth error** — attempt the MCP auth flow: extract the URL, open it in
  Google Chrome Dev (`open -a "Google Chrome Dev" "<url>"`), wait ~30s,
  retry once automatically.
- **Other error** — record as failed with reason, continue to the next server.

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

## Step 3: Final Report

Print a concise startup summary:

```text
Morning startup complete.

  VPN:            Connected
  AWS (toka):     OK
  MCP servers:    3/3 connected
  Split-tunnel:   Routes configured

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
| MCP auth error | Open auth URL in Chrome Dev, wait 30s, retry |
| MCP network error | Note as failed, continue with next server |
| vpn-split-tunnel.bash not found | Report path not found, skip Step 4 |
| Split-tunnel fails | Report error, offer retry with `mobile` or `direct` mode |
