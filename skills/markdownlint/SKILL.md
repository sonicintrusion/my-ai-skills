---
name: markdownlint
description: Enforce standard Markdown formatting rules compatible with markdownlint. Use when creating or editing .md files, when the user mentions markdownlint, or when the IDE shows MD### lint errors. Prefer tooling configs under e.g. `config/markdownlint/` with `-c` (mdl/Docker), VS Code `extends`, and explicit CLI `--config`. Validate with Docker (`markdownlint/markdownlint`, bind-mount + `-w` + optional `-c`) when no local CLI is installed. Rules source of truth: markdownlint RULES.md on GitHub (DavidAnson/markdownlint).
---

# Markdownlint formatting

## Source of truth

Follow the rules defined in DavidAnson's `markdownlint` rules reference:

- <https://github.com/DavidAnson/markdownlint/blob/main/doc/Rules.md>

## Workflow

1. **Pre-lint review** — check the document against all rules (see below) before invoking Docker.
2. **Fix** — correct any violations found in step 1.
3. **Run Docker** — use the Docker command to confirm zero violations.

---

## Pre-lint review: all rules

Before running the Docker container, review the document against each rule below.
Skip rules that are excluded in the project's `style.rb` / `.markdownlint.json`.

### Headings

| Rule | Alias | Check |
| --- | --- | --- |
| MD001 | `heading-increment` | Heading levels increment by one only (no skipping from H1 to H3) |
| MD003 | `heading-style` | Heading style is consistent throughout (ATX `#` vs Setext `===`) |
| MD018 | `no-missing-space-atx` | Space after `#` in ATX headings (`# Title` not `#Title`) |
| MD019 | `no-multiple-space-atx` | Only one space after `#` (`# Title` not `#  Title`) |
| MD020 | `no-missing-space-closed-atx` | Spaces inside hashes for closed ATX (`# Title #` not `#Title#`) |
| MD021 | `no-multiple-space-closed-atx` | Only one space inside hashes for closed ATX |
| MD022 | `blanks-around-headings` | Blank line before and after every heading |
| MD023 | `heading-start-left` | Headings start at column 1 (not indented) |
| MD024 | `no-duplicate-heading` | No two headings have identical text (within scope) |
| MD025 | `single-title` | Only one H1 per document |
| MD026 | `no-trailing-punctuation` | Headings do not end with `.`, `!`, `?`, `,`, etc. |
| MD036 | `no-emphasis-as-heading` | Bold/italic text is not used as a substitute for a real heading |
| MD041 | `first-line-heading` | First content line is a top-level heading (often excluded for files with YAML frontmatter) |

### Lists

| Rule | Alias | Check |
| --- | --- | --- |
| MD004 | `ul-style` | Unordered list marker is consistent (`-`, `*`, or `+` — not mixed) |
| MD005 | `list-indent` | Items at the same level have consistent indentation |
| MD007 | `ul-indent` | Unordered list items indented by the configured number of spaces |
| MD029 | `ol-prefix` | Ordered list prefixes follow the configured style (`1.` throughout, or `1. 2. 3.`) |
| MD030 | `list-marker-space` | Correct number of spaces after list marker |
| MD032 | `blanks-around-lists` | Blank line before and after every list block |

### Code blocks

| Rule | Alias | Check |
| --- | --- | --- |
| MD031 | `blanks-around-fences` | Blank line before and after fenced code blocks (including inside list items) |
| MD040 | `fenced-code-language` | Every fenced code block specifies a language (`bash`, `text`, etc. — no bare fences) |
| MD046 | `code-block-style` | Code block style is consistent throughout (fenced vs indented) |
| MD048 | `code-fence-style` | Code fence character is consistent (`` ` `` vs `~`) |

### Links and images

| Rule | Alias | Check |
| --- | --- | --- |
| MD011 | `no-reversed-links` | Link syntax not reversed (`[text](url)` not `(text)[url]`) |
| MD034 | `no-bare-urls` | URLs enclosed in `<…>` or `[text](url)` — no raw `https://…` in prose |
| MD039 | `no-space-in-links` | No spaces inside link brackets (`[text](url)` not `[ text ](url)`) |
| MD042 | `no-empty-links` | No links with an empty destination `[text]()` |
| MD044 | `proper-names` | Proper nouns use the correct configured capitalisation (e.g. `JavaScript` not `javascript`) |
| MD045 | `no-alt-text` | Images have alt text (`![alt](img.png)` not `![](img.png)`) |
| MD051 | `link-fragments` | Anchor fragments (`#heading-id`) point to an existing heading |
| MD052 | `reference-links-images` | Reference-style links/images use a defined label |
| MD053 | `link-image-reference-definitions` | Every reference definition is used at least once |
| MD054 | `link-image-style` | Link/image style matches configured allowed formats |
| MD059 | `descriptive-link-text` | Link text is descriptive (not "click here", "here", "link", etc.) |

### Whitespace

| Rule | Alias | Check |
| --- | --- | --- |
| MD009 | `no-trailing-spaces` | No trailing spaces at end of lines |
| MD010 | `no-hard-tabs` | No hard tab characters for indentation (use spaces) |
| MD012 | `no-multiple-blanks` | No more than one consecutive blank line |
| MD013 | `line-length` | Lines within configured max length (often excluded project-wide) |

### Inline markup

| Rule | Alias | Check |
| --- | --- | --- |
| MD033 | `no-inline-html` | No raw HTML tags (use markdown equivalents) |
| MD037 | `no-space-in-emphasis` | No spaces inside emphasis markers (`*text*` not `* text *`) |
| MD038 | `no-space-in-code` | No unnecessary spaces inside code backticks (`` `text` `` not `` ` text ` ``) |
| MD049 | `emphasis-style` | Emphasis marker style consistent (`*` vs `_`) |
| MD050 | `strong-style` | Strong marker style consistent (`**` vs `__`) |

### Blockquotes and horizontal rules

| Rule | Alias | Check |
| --- | --- | --- |
| MD027 | `no-multiple-space-blockquote` | Only one space after `>` in blockquotes |
| MD028 | `no-blanks-blockquote` | No blank lines inside blockquotes |
| MD035 | `hr-style` | Horizontal rule style consistent (`---`, `***`, `___` — not mixed) |

### Tables

| Rule | Alias | Check |
| --- | --- | --- |
| MD055 | `table-pipe-style` | Leading/trailing pipe style consistent across all tables |
| MD056 | `table-column-count` | Every row in a table has the same number of cells |
| MD058 | `blanks-around-tables` | Blank line before and after every table |
| MD060 | `table-column-style` | Table column pipe alignment consistent |

### File-level

| Rule | Alias | Check |
| --- | --- | --- |
| MD014 | `commands-show-output` | `$` before shell commands only when output is also shown |
| MD043 | `required-headings` | Headings match the configured required structure (if configured) |
| MD047 | `single-trailing-newline` | File ends with exactly one newline character |

---

## Running checks via Docker

Use the official image **`markdownlint/markdownlint`**, which runs Ruby **`mdl`**. Bind-mount
the documents (and config, if any) into the container and set the working directory so paths
line up.

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

- **`-v "$PWD:/work:ro"`** — host repo at `/work` inside the container.
- **`-w /work`** — `mdl` runs with that as cwd so `.` and relative paths resolve correctly.
- Arguments after the image name are passed to `mdl`. Use `.` for the whole tree or pass specific files/dirs (e.g. `README.md skills/`).

For scripts or CI without a TTY, omit **`-it`**.

**Toolchains:** Ruby **`mdl`** (this image) uses **`.mdlrc`** + style `.rb` files. Node
**`markdownlint-cli` / `markdownlint-cli2`** and VS Code's **`markdownlint`** extension use
**`.markdownlint.json`** / **`.markdownlint.yaml`** — same MD rule IDs, different config format.

## Config layout (subfolder)

Keep **`mdl`** and **Node / VS Code** configs together under one folder
(example: **`config/markdownlint/`**) so the repository root stays clean.

Suggested layout:

```text
config/markdownlint/
  .mdlrc              # Ruby mdl
  style.rb            # rule toggles for mdl (name can vary)
  .markdownlint.json  # Node markdownlint + VS Code (via extends)
```

**Ruby `mdl`:** pass **`-c path/to/.mdlrc`** always. Inside `.mdlrc`, point the style file
relative to **that** `.mdlrc`:

```ruby
style "#{File.dirname(__FILE__)}/style.rb"
```

**VS Code (`vscode-markdownlint`):** add **`.vscode/settings.json`** to point at the subfolder config:

```json
{
  "markdownlint.config": {
    "extends": "${workspaceFolder}/config/markdownlint/.markdownlint.json"
  }
}
```

**Node CLIs:** pass an explicit config path (`markdownlint-cli2 --config` / `markdownlint-cli -c`)
pointing at `config/markdownlint/.markdownlint.json`.

## Disabling rules when they don't fit the repo

For **Ruby `mdl`** (`style.rb`):

```ruby
all

exclude_rule 'MD013'
```

For **Node / VS Code** (`config/markdownlint/.markdownlint.json`):

```json
{
  "MD013": false
}
```

**Single file or region** (Node/VS Code only — HTML comments are ignored by Ruby `mdl`):

```markdown
<!-- markdownlint-disable MD013 -->
Long line here...
<!-- markdownlint-enable MD013 -->
```

## When fixing existing markdown

- Make the minimal changes required to satisfy markdownlint.
- Preserve the author's content; don't rewrite prose unless asked.
- After fixing, re-run the Docker check to confirm zero violations.
