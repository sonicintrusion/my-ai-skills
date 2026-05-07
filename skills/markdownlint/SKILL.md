---
name: markdownlint
description: Enforce standard Markdown formatting rules compatible with markdownlint. Use when creating or editing .md files, when the user mentions markdownlint, or when the IDE shows MD### lint errors. Validate with Docker (`markdownlint/markdownlint`, bind-mount + `-w`) when no local CLI is installed. Rules source of truth: https://raw.githubusercontent.com/markdownlint/markdownlint/main/docs/RULES.md
---

# Markdownlint formatting

## Source of truth

Follow the rules defined in `markdownlint`’s official rules reference:

- <https://raw.githubusercontent.com/markdownlint/markdownlint/main/docs/RULES.md>

## Running checks via Docker

Use the official image **`markdownlint/markdownlint`**, which runs Ruby **`mdl`**. You **must** bind-mount the documents (and config, if any) into the container and set the working directory so paths line up.

From the **repository root** (or adjust `$(pwd)` / `$PWD`):

```shell
docker run --rm -it \
  -v "$PWD:/work:ro" \
  -w /work \
  markdownlint/markdownlint .
```

- **`-v "$PWD:/work:ro"`** — host repo at `/work` inside the container (`:ro` is optional; drop it if something needs write access).
- **`-w /work`** — `mdl` runs with that as cwd so `.` and relative paths resolve correctly.
- **Arguments after the image name** are passed to `mdl` (see `docker run --rm markdownlint/markdownlint --help`): `.` lints the tree; you can pass files or dirs instead (e.g. `README.md skills/`).

For scripts or CI where no TTY is available, omit **`-it`** (otherwise Docker may warn or fail).

**Config:** `mdl` uses **`.mdlrc`** and style files / `-c`, `-s` (see [configuration](https://github.com/markdownlint/markdownlint/blob/main/docs/configuration.md)). It does **not** read `.markdownlint.json` / `.markdownlint.yaml` (those target the Node-based markdownlint CLIs and VS Code’s default rule set).

When asked to **run markdownlint** or **verify `.md` files** without a local `mdl` install, use the mounted `docker run` pattern above unless the user specifies another toolchain.

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

## Disabling rules when they don’t fit the repo

MD013 (**line length**, alias `line-length`) forces wrapping at a max column count (default 80). Many teams turn it off for long prose, tables, URLs, or generated Markdown—disabling it project-wide is valid.

**Preferred:** add or extend config at the repo root so editors and CLI agree:

`.markdownlint.json`:

```json
{
  "MD013": false
}
```

`.markdownlint.yaml` (same effect):

```yaml
MD013: false
```

The VS Code `markdownlint` extension and `markdownlint-cli` / `markdownlint-cli2` pick up these files when present (see each tool’s docs for search paths).

**Single file or region:** HTML comments understood by markdownlint CLI/extension, for example:

`<!-- markdownlint-disable MD013 -->` … `<!-- markdownlint-enable MD013 -->`, or `<!-- markdownlint-disable-next-line MD013 -->` on the line above a long line.

When the target repo disables MD013, **do not reflow prose** only to satisfy line-length warnings.

## When fixing existing markdown

- Make the minimal changes required to satisfy markdownlint.
- Preserve the author’s content; don’t rewrite prose unless asked.
