---
name: markdownlint
description: Enforce standard Markdown formatting rules compatible with markdownlint. Use when creating or editing .md files, when the user mentions markdownlint, or when the IDE shows MD### lint errors. Prefer tooling configs under e.g. `config/markdownlint/` with `-c` (mdl/Docker), VS Code `extends`, and explicit CLI `--config`. Validate with Docker (`markdownlint/markdownlint`, bind-mount + `-w` + optional `-c`) when no local CLI is installed. Rules source of truth: https://raw.githubusercontent.com/markdownlint/markdownlint/main/docs/RULES.md
---

# Markdownlint formatting

## Source of truth

Follow the rules defined in `markdownlint`’s official rules reference:

- <https://raw.githubusercontent.com/markdownlint/markdownlint/main/docs/RULES.md>

## Running checks via Docker

Use the official image **`markdownlint/markdownlint`**, which runs Ruby **`mdl`**. You **must** bind-mount the documents (and config, if any) into the container and set the working directory so paths line up.

From the **repository root** (or adjust `$(pwd)` / `$PWD`), lint the tree:

```shell
docker run --rm -it \
  -v "$PWD:/work:ro" \
  -w /work \
  markdownlint/markdownlint .
```

If **`mdl` config lives in a subfolder** (recommended — see below), pass **`-c`** before paths:

```shell
docker run --rm -it \
  -v "$PWD:/work:ro" \
  -w /work \
  markdownlint/markdownlint \
  -c config/markdownlint/.mdlrc .
```

- **`-v "$PWD:/work:ro"`** — host repo at `/work` inside the container (`:ro` is optional; drop it if something needs write access).
- **`-w /work`** — `mdl` runs with that as cwd so `.` and relative paths resolve correctly.
- **Arguments after the image name** are passed to `mdl` (see `docker run --rm markdownlint/markdownlint --help`): `.` lints the tree; you can pass files or dirs instead (e.g. `README.md skills/`).

For scripts or CI where no TTY is available, omit **`-it`** (otherwise Docker may warn or fail).

**Toolchains:** Ruby **`mdl`** (this image) uses **`.mdlrc`** + style `.rb` files (see [configuration](https://github.com/markdownlint/markdownlint/blob/main/docs/configuration.md)). Node **`markdownlint-cli` / `markdownlint-cli2`** and VS Code’s **`markdownlint`** extension use **`.markdownlint.json`** / **`.markdownlint.yaml`** — same MD rule IDs, different config files. Use **`## Config layout (subfolder)`** below to keep both families under one directory instead of the repo root.

When asked to **run markdownlint** or **verify `.md` files** without a local `mdl` install, use the mounted `docker run` pattern above unless the user specifies another toolchain.

## Config layout (subfolder)

Keep **`mdl`** and **Node / VS Code** configs together under one folder (example: **`config/markdownlint/`**) so the repository root stays clean.

Suggested layout:

```text
config/markdownlint/
  .mdlrc              # Ruby mdl
  style.rb            # rule toggles for mdl (name can vary)
  .markdownlint.json  # Node markdownlint + VS Code (via extends)
```

**Ruby `mdl`:** `mdl` normally discovers `.mdlrc` in the **current working directory**. With a subfolder config, always pass **`-c path/to/.mdlrc`** (Docker example above; locally `mdl -c config/markdownlint/.mdlrc .`). Inside `.mdlrc`, point the style file relative to **that** `.mdlrc` so it works no matter what cwd is:

```ruby
style "#{File.dirname(__FILE__)}/style.rb"
```

([Official recommendation](https://github.com/markdownlint/markdownlint/blob/main/docs/configuration.md) — avoids resolving `style.rb` relative to the shell cwd.)

**VS Code (`vscode-markdownlint`):** auto-load only applies to `.markdownlint.json` / `.markdownlint.yaml` in standard locations (often workspace root). For a subfolder file, add **`.vscode/settings.json`** (workspace or checked in) so the extension extends it:

```json
{
  "markdownlint.config": {
    "extends": "${workspaceFolder}/config/markdownlint/.markdownlint.json"
  }
}
```

Paths in **`extends`** resolve relative to the workspace folder when set there.

**Node CLIs:** pass an explicit config path if your tool does not search subfolders (examples: `markdownlint-cli2 --config`, `markdownlint-cli -c`), pointing at `config/markdownlint/.markdownlint.json` or the generator’s expected filename.

**Migrating from the repo root:** move `.mdlrc`, `style.rb`, and `.markdownlint.json` (or `.yaml`) into this folder; set **`style "#{File.dirname(__FILE__)}/style.rb"`** in `.mdlrc`; add **`extends`** in **`.vscode/settings.json`**; add **`mdl -c …`** (or Docker **`docker run … -c config/markdownlint/.mdlrc`**) everywhere you invoke **`mdl`**.

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

**Preferred:** define the setting once under **`config/markdownlint/`** (see **`## Config layout (subfolder)`**) so editors and CLIs stay aligned.

For **Node / VS Code** (`config/markdownlint/.markdownlint.json`):

```json
{
  "MD013": false
}
```

Same rule in YAML (`.markdownlint.yaml`, same folder):

```yaml
MD013: false
```

For **Ruby `mdl`** (`style.rb` referenced from `.mdlrc`), disable line length with **`exclude_rule`** after **`all`** per [creating styles](https://github.com/markdownlint/markdownlint/blob/main/docs/creating_styles.md):

```ruby
all

exclude_rule 'MD013'
```

Alternatively keep MD013 enabled but relax it with **`rule 'MD013', :line_length => N`** if you only want a higher limit.

The VS Code extension needs **`extends`** (or a root config) as described in **`## Config layout (subfolder)`**. Node CLIs may need an explicit **`--config`** / **`-c`** path.

**Single file or region:** HTML comments understood by markdownlint CLI/extension, for example:

`<!-- markdownlint-disable MD013 -->` … `<!-- markdownlint-enable MD013 -->`, or `<!-- markdownlint-disable-next-line MD013 -->` on the line above a long line.

When the target repo disables MD013, **do not reflow prose** only to satisfy line-length warnings.

## When fixing existing markdown

- Make the minimal changes required to satisfy markdownlint.
- Preserve the author’s content; don’t rewrite prose unless asked.
