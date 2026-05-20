---
name: pull-request
description: >-
  Pull request skill for GitHub. Covers creating, updating, reviewing, and
  merging pull requests. Follows the access strategy in mcp-api-access.
---

# GitHub Pull Request Skill

## Goal

Manage the full pull request lifecycle on GitHub: create, update, review,
check, and merge.

For all GitHub access, follow the tier order defined in
`../mcp-api-access/access-strategy.md` (MCP → gh CLI → REST API).

For REST API details, see `../mcp-api-access/github-rest-api.md`.

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
1. Build PR description with:
   - short task summary based on ticket description
   - commit list in a style similar to gh-generated commit lists
1. Create the PR in draft state.
1. Ask the user to review the draft PR.
1. Keep PR in draft until the user asks to mark it ready for review, or they
   do it themselves.
1. Return PR number, URL, final title, and description preview.
1. If the PR is marked ready for review, proceed to Operation 6.

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
- After converting to ready-for-review, proceed to Operation 6.

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

---

## Operation 6: CI Check After Ready for Review

This operation runs automatically after a PR transitions to ready-for-review,
whether triggered by Operation 1 or Operation 2.

### Step 1: Check for running CI jobs

Query the PR's status checks to determine if any CI jobs are in progress.

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
1. Analyse the failure — determine whether it is:
   - a code issue in this PR
   - a flaky or pre-existing failure unrelated to this PR
   - a configuration or environment issue
1. For failures caused by this PR, suggest the specific changes needed to fix
   them (file paths, what to change, and why).
1. For failures not caused by this PR, note this clearly so the user can
   decide whether to investigate or proceed.

---

## Operation 7: Merge Pull Request

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

---

## Next Content To Add

- Concrete MCP tool mapping per operation
- Concrete gh command mapping per operation
- Concrete REST API endpoint mapping per operation
- Input parsing rules and defaults
- Structured success/failure response templates
