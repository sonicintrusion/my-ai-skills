# my-ai-skills

Reusable skills for Codex and Claude. Skills use the current runtime's available
models and tools; external service access follows the
[shared MCP-first strategy](skills/mcp-api-access/access-strategy.md).

## Usage

Link individual skill folders into your runtime's skill directory without
replacing existing skills. For example, run from this repository in Codex:

```shell
mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
ln -s "$PWD/skills/pr-review" "${CODEX_HOME:-$HOME/.codex}/skills/pr-review"
ln -s "$PWD/skills/mcp-api-access" "${CODEX_HOME:-$HOME/.codex}/skills/mcp-api-access"
```

For Claude, use `~/.claude/skills` as the destination. The `mcp-api-access`
folder contains shared references rather than an invocable skill; keep it
alongside the linked skill folders so relative references resolve.
