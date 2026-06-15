---
name: pr-review
description: >-
  Review GitHub pull requests for correctness, project coding standards, test
  evidence, CI status, documentation, maintainability, scope, and readiness.
  Use when the user asks to review a PR, inspect a pull request, validate code
  changes before approval or merge, check whether tests/examples/docs are
  sufficient, or produce a review findings summary.
---

# PR Review

## Goal

Perform a careful pull request review that waits for automated checks when
needed, understands the purpose of the change, evaluates the implementation
against project standards, and reports concise findings in chat.

Do not post a GitHub review, approve, request changes, or add PR comments unless
the user explicitly asks for that action. By default, provide the review report
only in chat.

## Access Order

Use the repository's GitHub access strategy when available:
`../mcp-api-access/access-strategy.md`.

Prefer MCP tools first for GitHub, Jira, CI, Confluence, or other external
systems. If MCP is unavailable, use `gh` CLI next, then documented REST APIs.
If authentication is required, surface the login URL or missing credential and
retry after the user confirms access is ready.

## Review Workflow

### 1. Identify the PR

Resolve the repository, PR number or URL, base branch, head branch, commit SHA,
title, description, linked tickets, changed files, and current check status.

If the PR is local-only or the remote PR cannot be read, use the local branch
diff against the intended base branch. Ask for the missing PR identifier only
when it cannot be inferred safely.

### 2. Wait for Automated Checks

Before deep review, query CI and status checks for the PR's current head commit.

- If checks are pending, queued, or running, monitor them until they reach a
  terminal state before calling the review complete.
- If all checks pass, include that result in the final report.
- If checks fail, are cancelled, or are skipped unexpectedly, fetch useful logs
  or failure details, determine whether the failure appears related to the PR,
  and treat related failures as blocking findings.
- If checks cannot be queried because of authentication or service access, say
  so clearly and do not imply CI passed.

When the repository exposes obvious local validation commands, run the relevant
low-risk tests or linters if the PR branch is available locally. Otherwise,
evaluate the CI results and the test evidence included in the PR.

### 3. Understand Intent

Read the PR description, linked ticket or issue, commit messages, and relevant
surrounding code before judging the implementation.

Summarize the intended behavior in your own words. Decide whether the code
change is appropriate for that purpose:

- Scope matches the stated ticket or product need.
- The change does not include unrelated churn.
- The implementation preserves existing behavior unless the PR explains the
  behavior change.
- New behavior has clear user, operational, or maintainer value.

If the purpose is unclear, make that an open question or review finding instead
of guessing.

### 4. Inspect the Code

Review the diff with enough surrounding context to understand how the changed
code runs in production, tests, and developer workflows. Prefer specific,
line-referenced findings over broad style commentary.

Check for:

- Correctness, edge cases, error handling, null or empty inputs, and rollback
  behavior.
- Project coding standards, established patterns, architecture boundaries,
  naming, formatting, and dependency choices.
- Security, privacy, secrets, permissions, injection risks, unsafe logging, and
  supply-chain concerns.
- Performance, concurrency, reliability, observability, and operational impact.
- Backward compatibility, migrations, configuration, environment assumptions,
  and deploy or release implications.
- Concision: avoid over-engineering, duplicated logic, dead code, excessive
  abstraction, and unrelated rewrites.

Do not treat personal preference as a finding unless it maps to a project
standard, maintainability risk, correctness risk, or user impact.

### 5. Verify Tests and Examples

Confirm that the PR includes appropriate proof for the risk and behavior of the
change.

- Automated tests should cover the new or changed behavior, including important
  negative and edge cases.
- CI results should be visible and successful before the review is complete.
- User-facing, API, data, or infrastructure changes should include examples,
  screenshots, command output, fixtures, migration dry runs, or other evidence
  that demonstrates the behavior.
- If tests are missing but the change is genuinely trivial, say why the residual
  risk is low.

Treat missing or weak test evidence as a finding when the change affects shared,
user-facing, operational, security-sensitive, or hard-to-roll-back behavior.

### 6. Check Documentation

Verify documentation updates match the change's blast radius.

Look for updates to README files, API docs, runbooks, configuration examples,
Jira or release notes, architecture notes, migration notes, comments, or inline
docs when the change introduces behavior that future maintainers need to
understand.

Prefer concise documentation near the audience that needs it. Do not require
documentation for self-explanatory internal refactors unless they alter a
contract, workflow, or operational procedure.

## Final Report

Always end with a bullet-form summary in chat. Lead with findings, ordered by
severity, using file and line references when possible.

Include:

- `Findings`: blocking issues first, then important non-blocking issues. Say
  "No blocking findings" when appropriate.
- `CI/tests`: final CI status, local checks run, and whether test evidence is
  sufficient.
- `Purpose/fit`: one bullet summarizing what the PR appears to do and whether
  the implementation fits that purpose.
- `Docs/examples`: whether documentation or examples are adequate.
- `Open questions`: unclear intent, missing context, or assumptions.
- `Verdict`: one of `ready`, `ready with nits`, `needs changes`, or `blocked`.

Keep the report concise and actionable. If there are no meaningful issues, say
so directly and call out any residual risk or test gaps.
