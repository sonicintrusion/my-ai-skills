---
name: compliance-docs
description: Load compliance rules from the local repo and apply them to all future tasks in this session. Use when the user wants to enforce compliance, governance, or policy rules found in docs/ or compliance/ folders. Scans the repo for compliance documents, loads their content, then acts as a compliance-aware assistant for all subsequent queries.
---

# Compliance docs

## Goal

Scan the local repository for compliance and policy documents, load their
contents, and apply those rules as constraints on all subsequent tasks and
queries in the session.

## Step 1: Scan for compliance documents

1. Determine `workspaceRoot` (the repository root for the current workspace).
2. Search for compliance documents in the following locations (in priority order):
   - `compliance/` folder at the workspace root
   - `docs/compliance/` subfolder
   - `docs/` folder at the workspace root
   - Any folder whose name contains "compliance", "policy", "governance", or
     "regulatory" (case-insensitive) at the first two levels of the repo tree
3. Within each found folder, collect all files with these extensions:
   - `.md`, `.txt`, `.rst`, `.pdf` (if text-extractable), `.yaml`, `.json`
4. If no compliance folder or documents are found, report this to the user and
   ask if they'd like to specify a path manually. Do not proceed silently.

## Step 2: Load and parse compliance documents

For each file found in Step 1:

1. Read the full content of the file.
2. Extract the following if present:
   - Rules or requirements (numbered or bulleted lists, sections headed with
     "Rule", "Requirement", "Policy", "Standard", "Must", "Shall", "Prohibited")
   - Scope statements (who or what the rules apply to)
   - Exceptions or exemptions
   - Version or effective date metadata
3. Build an in-memory **Compliance Rule Set** summarising the loaded rules as a
   flat, deduplicated list with source file attribution.

## Step 3: Report what was loaded

After scanning and loading, always report to the user:

- Which folders were searched
- Which files were loaded (relative paths)
- A concise summary of the rule categories found (e.g. "Data handling: 4 rules,
  Access control: 2 rules, …")
- Any files that could not be read or parsed (with reason)

If no rules were found in the loaded files, say so explicitly.

## Step 4: Apply compliance rules to all subsequent tasks

Once the Compliance Rule Set is loaded, treat every subsequent user request in
this session as subject to those rules. For each task:

1. **Pre-flight check** — before generating code, content, or taking an action,
   silently verify it against the loaded rules.
2. **Flag violations** — if a requested action would violate a rule, stop and
   explain which rule(s) are breached and why, citing the source file and rule
   text.
3. **Suggest compliant alternatives** — where a violation is found, offer a
   modified approach that satisfies both the user's goal and the compliance
   constraints.
4. **Annotate outputs** — when producing code or documents, note any compliance
   considerations that informed decisions (e.g. "Field X is not logged per
   `compliance/data-handling.md` §3.2").
5. **Do not silently suppress** — never quietly omit or alter output to satisfy
   compliance without telling the user. Always be transparent about what was
   constrained and why.

## Step 5: Refreshing compliance documents

If the user asks to **reload** or **refresh** compliance docs (e.g. "reload
compliance", "refresh rules"), repeat Steps 1–3 from scratch and replace the
current Compliance Rule Set with the newly loaded one. Confirm the refresh to
the user.

If the user asks to **show compliance rules** or **list rules**, display the
full Compliance Rule Set with source attribution.

## Conflict resolution

If the user's request conflicts with a loaded compliance rule:

1. State the conflict clearly: "This request conflicts with [rule summary] from
   `[source file]`."
2. Ask whether the user wants to:
   a. Proceed with a compliant alternative (recommend one)
   b. Acknowledge the conflict and proceed anyway (note this as a conscious
      compliance exception in the output)
   c. Review and amend the compliance document first
3. Do not block the user unilaterally — surface the conflict and let them decide
   with full information.

## Scope of compliance enforcement

The loaded rules apply to all tasks in the session including:

- Code generation and modification
- Documentation and markdown output
- Configuration files
- Data queries and transformations
- Any tool calls (e.g. Jira comments, GitHub PRs, file writes)

## Edge cases

- If two compliance documents contain contradictory rules, flag the contradiction
  and ask the user which takes precedence before proceeding.
- If a compliance document references external standards by name (e.g. "GDPR
  Article 17", "ISO 27001 A.8.2"), note the reference but do not attempt to
  fetch or interpret external documents unless the user provides them.
- If the repo contains a `.complianceignore` file at the root, treat its
  contents as a glob-pattern exclusion list for compliance document scanning
  (same syntax as `.gitignore`).

## Markdown quality rules

Apply the rules defined in the `markdownlint` skill for all files created or modified.
