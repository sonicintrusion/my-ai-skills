---
name: pull-request
description: >-
  Manage the full GitHub pull request lifecycle: create, update, review, check
  CI, and merge. Use when the user asks to create a PR, open a pull request,
  update PR title or description, mark ready for review, check PR status, review
  code changes, investigate CI failures, or merge a branch.
disable-model-invocation: true
---

# GitHub Pull Request Skill

## Goal

Manage the full pull request lifecycle on GitHub: create, update, review,
check, and merge.

For all GitHub access, follow the tier order defined in
`../mcp-api-access/access-strategy.md` (MCP → gh CLI → REST API).

For REST API details, see `../mcp-api-access/github-rest-api.md`.

---

## Tool Mapping

Use the first available tier for each operation.

| Operation | MCP tool | gh CLI | REST API endpoint |
|-----------|----------|--------|-------------------|
| Create PR | `github_create_pull_request` | `gh pr create` | `POST /repos/{owner}/{repo}/pulls` |
| Get PR | `github_get_file_contents` (via PR number) | `gh pr view {number}` | `GET /repos/{owner}/{repo}/pulls/{number}` |
| Update PR | `github_update_pull_request` | `gh pr edit {number}` | `PATCH /repos/{owner}/{repo}/pulls/{number}` |
| List PRs | `github_list_pull_requests` | `gh pr list` | `GET /repos/{owner}/{repo}/pulls` |
| Add comment | `github_add_issue_comment` | `gh pr comment {number}` | `POST /repos/{owner}/{repo}/issues/{number}/comments` |
| Get checks | `github_get_workflow_run` | `gh pr checks {number}` | `GET /repos/{owner}/{repo}/commits/{sha}/check-runs` |
| Get job logs | `github_get_job_logs` | `gh run view {run-id} --log-failed` | `GET /repos/{owner}/{repo}/actions/jobs/{job_id}/logs` |
| Merge PR | _(not available)_ | `gh pr merge {number} --squash` | `PUT /repos/{owner}/{repo}/pulls/{number}/merge` |

---

## Operation 1: Create Pull Request

Intent examples:

- create a PR
- open pull request from current branch
- create draft PR

Required behavior:

1. Verify repository state before PR creation.
1. Check for local changes and ask the user how to handle them:
   - untracked files
   - tracked but uncommitted changes
1. Confirm there are no outstanding local commits to resolve before push.
1. Push the working branch to origin.
1. Build PR title with this exact format:
   - `<TICKET-KEY> - <readable title>`
   - Example branch: `dsosys-3080/fix-dep-e1-30day-monitor`
   - Example PR title: `DSOSYS-3080 - fix: 30 day monitor for DEP-E1 cluster`
1. Derive `<TICKET-KEY>` from branch or user-provided ticket.
1. Derive readable title from branch name by converting dashes and
   abbreviations into clear human text.
1. Check for a PR template in the repository before building the description.
   Look in this order:
   - `.github/pull_request_template.md`
   - `.github/PULL_REQUEST_TEMPLATE/pull_request_template.md`
   - `pull_request_template.md` (repo root)
   Use the first one found. If none exist, skip this step.
1. Build PR description:
   - If a template was found, populate each section of the template with
     relevant content (task summary, commit list, test plan, etc.).
     Preserve the template's headings and structure exactly.
   - If no template was found, use a **simple** description by default
     unless the user has explicitly requested a detailed one. A simple
     description is a short plain-text summary and a commit list in a
     style similar to gh-generated commit lists.
     If the user explicitly requests a **detailed** description, build a
     structured description with these four sections in this order:
       - `## Summary` — plain-English bullet points describing what
         changed; maximum 3 bullets.
       - `## Jira` — the ticket key as a markdown link to the Jira issue
         (derive the URL from the ticket key using the project's known
         Jira base URL, or ask the user if unknown).
       - `## Changes` — a markdown table listing every file touched in
         this PR. Columns: `File`, `Lines changed`, `% of total`. Compute
         percentages from `git diff --stat` output; round to one decimal
         place. Sort rows by percentage descending.
       - `## Related PRs` — a list of any PRs referenced in commit
         messages, branch names, or ticket links. Leave the section
         heading with a placeholder note if none are found.
1. Create the PR in draft state.
1. Ask the user to review the draft PR.
1. Keep PR in draft until the user asks to mark it ready for review, or they
   do it themselves.
1. Return PR number, URL, final title, and description preview.
1. If the PR is marked ready for review, proceed to Operation 5.

### Op 1: gh CLI example

```bash
gh pr create \
  --title "DSOSYS-3080 - fix: 30 day monitor for DEP-E1 cluster" \
  --body "$(cat pr-body.md)" \
  --draft \
  --base main
```

### Op 1: REST API example

```bash
curl -X POST "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/pulls" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{
    "title": "DSOSYS-3080 - fix: 30 day monitor for DEP-E1 cluster",
    "body": "PR description here",
    "head": "dsosys-3080/fix-dep-e1-30day-monitor",
    "base": "main",
    "draft": true
  }'
```

---

## Operation 2: Update Pull Request

Intent examples:

- update PR title/body
- mark PR ready for review
- convert to draft

Required behavior:

1. Identify PR number.
1. Apply requested metadata/state changes.
1. Confirm updated fields and final PR state.

State transition rule:

- If user explicitly requests, convert draft PR to ready-for-review.
- Do not convert draft to ready automatically.
- After converting to ready-for-review, proceed to Operation 5.

### Op 2: gh CLI examples

```bash
# Update title and body
gh pr edit {number} --title "new title" --body "new description"

# Mark ready for review
gh pr ready {number}

# Convert back to draft
gh pr edit {number} --draft
```

### Op 2: REST API example

```bash
curl -X PATCH "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{"draft": false}'
```

---

## Operation 3: Review and Comments

Intent examples:

- add review comment
- request changes
- approve PR
- respond to review threads

Required behavior:

1. Identify PR and target thread/file/line where relevant.
1. Post review or comment action.
1. Report thread status and outstanding items.

### Op 3: gh CLI examples

```bash
# Add a general comment
gh pr comment {number} --body "comment text"

# Submit a review
gh pr review {number} --approve
gh pr review {number} --request-changes --body "feedback here"
```

### Op 3: REST API example

```bash
# Add comment (uses issues endpoint — PR comments are issues comments)
curl -X POST "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/issues/${PR_NUMBER}/comments" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{"body": "comment text"}'
```

---

## Operation 4: Checks and Merge Readiness

Intent examples:

- check PR status
- why is CI failing
- is this PR ready to merge

Required behavior:

1. Collect status checks and review requirements.
1. Summarize blockers (failed checks, missing approvals, requested changes).
1. Return merge readiness verdict.

### Op 4: gh CLI example

```bash
gh pr checks {number}
gh pr view {number} --json reviewDecision,mergeable,statusCheckRollup
```

---

## Operation 5: CI Check After Ready for Review

This operation runs automatically after a PR transitions to ready-for-review,
whether triggered by Operation 1 or Operation 2.

### Step 1: Check for running CI jobs

Query the PR's status checks to determine if any CI jobs are in progress.

```bash
gh pr checks {number} --watch
# or via MCP: github_get_workflow_run with the run ID from the PR
```

If no CI jobs are running or queued, report that and stop — no further action
needed.

### Step 2: Inform the user and ask how to proceed

If CI jobs are running, inform the user:

> CI jobs are currently running on this PR. Would you like me to wait for
> them to complete and check the outcomes?

The user has two choices:

- **Yes, wait** — proceed to Step 3.
- **No, I'll handle it** — stop here; do not monitor further.

### Step 3: Wait for CI jobs to complete

Monitor the PR's status checks until all jobs reach a terminal state
(success, failure, cancelled, skipped).

Report each job result as it completes, then provide a final summary once all
jobs are done.

### Step 4: Assess outcomes

If all jobs passed, report success and stop.

If any jobs failed or were cancelled:

1. Identify the failing job(s) by name.
1. Fetch the relevant log output for each failure.

   ```bash
   gh run view {run-id} --log-failed
   # or via MCP: github_get_job_logs with failed_only=true
   ```

1. Analyse the failure — determine whether it is:
   - a code issue in this PR
   - a flaky or pre-existing failure unrelated to this PR
   - a configuration or environment issue
1. For failures caused by this PR, suggest the specific changes needed to fix
   them (file paths, what to change, and why).
1. For failures not caused by this PR, note this clearly so the user can
   decide whether to investigate or proceed.

---

## Operation 6: Merge Pull Request

Intent examples:

- merge PR
- squash and merge

Merge strategy rules:

- **Default: squash merge.** Use squash merge unless the user explicitly
  requests a merge commit.
- **Merge commit: only if explicitly requested** by the user.
- **Rebase merge: never.** Do not use rebase merge under any circumstances,
  even if the user asks. If asked, explain the preference and offer squash
  merge or merge commit instead.

Required behavior:

1. Verify merge prerequisites.
1. Execute the merge using squash merge unless the user has explicitly
   requested a merge commit.
1. Confirm merge result and resulting branch state.
1. Check for outstanding automations/workflows after merge.
1. If automation is running, wait until completion.
1. After automation completes (or if none are running), ask the user whether
   there is any further activity needed.
1. If no further activity is needed:
   - update notes
   - close the related ticket

### Op 6: gh CLI example

```bash
# Squash merge (default)
gh pr merge {number} --squash --delete-branch

# Merge commit (only if explicitly requested)
gh pr merge {number} --merge --delete-branch
```

### Op 6: REST API example

```bash
curl -X PUT "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}/merge" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{
    "merge_method": "squash",
    "commit_title": "DSOSYS-3080 - fix: 30 day monitor for DEP-E1 cluster (#42)"
  }'
```
