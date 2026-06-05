---
name: ready-project
description: >-
  Prepare a local code repository for development work against a Jira ticket.
  Locates or clones the repo, pulls latest changes on the default branch, saves
  any uncommitted work, and creates a correctly-named feature branch. Use when
  the user says "ready project", "set up the repo", "start work on TICKET-123",
  "clone and branch", or any request to prepare a local repo before coding.
---

# Ready Project

## Goal

Given a Jira ticket number, ensure the correct repository is available locally,
up-to-date on its default branch, and ready on a new ticket-prefixed branch.

---

## Step 1: Locate the Repository

### 1a — Check current working directory

Run `git rev-parse --show-toplevel` in the current directory. If it succeeds,
the user is already inside a repo — use that repo.

### 1b — Check known local roots

If not inside a repo, search these directories for a matching repo:

```text
~/dev/sie/github/
~/dev/sie/gitlab/
~/dev/sonic/github/
~/dev/sonic/gitlab/
~/dev/adhoc/
~/dev/opensource/
```

List subdirectories in each root. Match against the repo name the user provided
or implied from the ticket.

### 1c — Repo not found locally

If no match is found in any known root, ask:

> I couldn't find the repository locally. Do you have a link to the repo (e.g.
> GitHub/GitLab URL)?

- **If yes** — note the URL and proceed to Step 2.
- **If no** — ask the following questions to locate the repo:
  1. What is the name (or partial name) of the repository?
  1. Is it on the internal GitHub instance (`github.sie.sony.com`) or
     `github.com`?
  1. What org or owner owns the repo?

  Use these answers to construct the clone URL and proceed to Step 2.

---

## Step 2: Clone the Repository (if not local)

Determine the target clone directory based on the host and org.

### Internal GitHub (`github.sie.sony.com`)

Repos are grouped by org under `~/dev/sie/github/<org>/`. Known orgs:

| Org slug | Directory |
|----------|-----------|
| `d-bis` | `~/dev/sie/github/d-bis/` |
| `d-sie` | `~/dev/sie/github/d-sie/` |
| `d-data-platform` | `~/dev/sie/github/d-data-platform/` |
| `d-knguyen` | `~/dev/sie/github/d-knguyen/` |

If the org is unknown or not listed, check `~/dev/sie/github/` for a matching
subdirectory name, or ask the user which org owns the repo.

Clone target: `~/dev/sie/github/<org>/<repo-name>`

### GitHub.com / Sonic repos

Repos are stored directly without an org subdirectory.

Clone target: `~/dev/sonic/github/<repo-name>`

### GitLab (internal)

Clone target: `~/dev/sie/gitlab/<repo-name>`

### Other host

Ask the user where to clone.

Clone example (internal GitHub with org):

```bash
git clone <url> ~/dev/sie/github/<org>/<repo-name>
```

Clone example (sonic/public):

```bash
git clone <url> ~/dev/sonic/github/<repo-name>
```

If clone fails, report the error and stop. Do not proceed to Step 3.

---

## Step 3: Switch to Default Branch and Pull

### 3a — Identify default branch

```bash
git remote show origin | grep "HEAD branch" | awk '{print $NF}'
```

Common values: `main`, `master`, `develop`.

### 3b — Check for uncommitted changes

```bash
git status --short
```

If there are **no** uncommitted changes, skip to 3d.

### 3c — Save uncommitted changes

If uncommitted changes exist, choose how to handle them:

- If currently **not on the default branch** and changes are logically part of
  in-progress work, commit them:

  > There are uncommitted changes on branch `<branch-name>`. Should I commit
  > them here so they are saved before I switch branches?

  If yes:

  ```bash
  git add -A
  git commit -m "wip: save in-progress changes before branch switch"
  ```

- If unclear (e.g. on default branch with unexpected changes), stash instead:

  > There are uncommitted changes on `<branch-name>`. I'll stash them so
  > they are preserved.

  ```bash
  git stash push -m "ready-project: stash before branch switch"
  ```

  Inform the user of the stash reference so they can restore it later.

### 3d — Switch to default branch and pull

```bash
git checkout <default-branch>
git pull origin <default-branch>
```

If the pull fails (e.g. diverged history, merge conflicts), stop and report
the full error. Do not proceed to Step 4 until the user resolves it.

---

## Step 4: Create the Feature Branch

### 4a — Derive the ticket key

If the user provided a full key (e.g. `DSOSYS-3088`), use it as-is.
If they provided a bare number (e.g. `3088`), expand to `DSOSYS-<number>`.

### 4b — Determine the branch type prefix

Use the conventional commits type that best matches the ticket or user intent:

| Type | When to use |
|------|-------------|
| `feat` | New feature or functionality |
| `fix` | Bug fix |
| `chore` | Maintenance, dependencies, tooling |
| `refactor` | Code restructure without behaviour change |
| `docs` | Documentation only |
| `test` | Adding or updating tests |
| `ci` | CI/CD pipeline changes |
| `perf` | Performance improvement |

If the ticket title or user description is available, infer the type from it.
When ambiguous, default to `feat` and mention the assumption.

### 4c — Build the branch slug

From the ticket title or user description:

1. Lowercase all words.
1. Remove articles, conjunctions, and filler words (`a`, `an`, `the`, `for`,
   `and`, `of`, `to`, `in`, `with`, `on`).
1. Replace spaces and special characters with hyphens.
1. Truncate so the full branch name is ≤ 60 characters total.

Examples:

| Ticket | Type | Slug | Branch |
|--------|------|------|--------|
| DSOSYS-3088: Add 30-day monitor for DEP-E1 | `feat` | `add-30day-monitor-dep-e1` | `dsosys-3088/feat-add-30day-monitor-dep-e1` |
| DSOSYS-1234: Fix login timeout bug | `fix` | `login-timeout` | `dsosys-1234/fix-login-timeout` |
| DSOSYS-9999: Update dependencies | `chore` | `update-deps` | `dsosys-9999/chore-update-deps` |

### 4d — Create and switch to the branch

```bash
git checkout -b <branch-name>
```

Confirm the branch was created and is active:

```bash
git branch --show-current
```

---

## Step 5: Confirm and Report

Report a summary:

- Repository path (local)
- Default branch pulled (with latest commit SHA or message)
- Any stashed or committed changes (with stash ref if applicable)
- New branch name and active status

Example:

> Repository ready at `~/dev/sie/github/platform-tools`.
> Pulled latest `main` (abc1234 — "chore: bump alpine to 3.19").
> Branch created: `dsosys-3088/feat-add-30day-monitor-dep-e1`.

---

## Error Handling

| Situation | Action |
|-----------|--------|
| Clone fails (auth, network) | Report error, check VPN/SSH key, stop |
| Pull fails (conflicts) | Report exact error, ask user to resolve, stop |
| Branch already exists | Ask: create new unique branch or switch to existing? |
| No ticket number provided | Ask: "What ticket are we working on?" |
| Ambiguous repo match (multiple results) | List matches and ask user to choose |
