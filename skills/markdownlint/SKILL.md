---
name: markdownlint
description: Enforce standard Markdown formatting rules compatible with markdownlint. Use when creating or editing .md files, when the user mentions markdownlint, or when the IDE shows MD### lint errors. Rules source of truth: https://raw.githubusercontent.com/markdownlint/markdownlint/main/docs/RULES.md
---

# Markdownlint formatting

## Source of truth

Follow the rules defined in `markdownlint`’s official rules reference:

- <https://raw.githubusercontent.com/markdownlint/markdownlint/main/docs/RULES.md>

## Practical checklist (covers the most common MD### errors)

- Ensure **one blank line** before and after headings (MD022).
- Ensure **lists are surrounded by blank lines** (MD032).
- Remove **trailing spaces** (MD009).
- Avoid **multiple consecutive blank lines** (MD012).
- Avoid **bare URLs**; use `<https://…>` or `[text](https://…)` (MD034).
- End every file with **a single newline** (MD047).
- Start files with a top-level heading when appropriate (MD041/MD002).
- Do **not** use a standalone line that is **only** bold or italic as a section title; use a real heading (`#` … `######`) instead (MD036 / `no-emphasis-as-header`).
- Use fenced code blocks with a language when applicable (MD031/MD040).

## When fixing existing markdown

- Make the minimal changes required to satisfy markdownlint.
- Preserve the author’s content; don’t rewrite prose unless asked.
