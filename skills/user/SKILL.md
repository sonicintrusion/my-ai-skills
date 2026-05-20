---
name: user
description: >-
  Discovers the OS user and home directory for the environment where commands
  run (shell username, HOME). Use when the user asks who is running the session,
  needs paths under their home, tilde expansion context, or "my user" /
  "this machine's user" for scripts, config, or file locations.
disable-model-invocation: true
---

# Current user context

## When to apply

Use this skill whenever you need **authoritative** identity or home-directory facts for the **same environment** where terminal commands execute—not assumed from chat metadata.

## How to gather facts

Run in the project shell (no extra permissions unless the task already needs them):

```bash
printf 'username=%s\n' "$(whoami)"
printf 'home=%s\n' "${HOME-}"
printf 'user=%s\n' "${USER-}"
printf 'uid=%s\n' "$(id -u 2>/dev/null || echo unknown)"
```

- **username**: `whoami` is the login name for the current shell user.
- **home**: `$HOME` is the standard home directory for that user (quote when passing to other commands if paths may contain spaces).

If any command fails, say which failed and what you got from the others.

## What to return

Reply with at least:

- **Username** (from `whoami`)
- **Home directory** (from `$HOME`)

Optionally include `$USER` and UID if useful for the task.

## Limits

- This reflects the **terminal session user**, not necessarily the Cursor account email or Git author unless you also read git config.
- Sandboxed or restricted environments may hide or alter some values; report what the shell actually prints.
