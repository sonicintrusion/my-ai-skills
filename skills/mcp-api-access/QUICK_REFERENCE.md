# Quick Reference: API Access Methods

## Select access by capability

Follow [the shared access strategy](access-strategy.md):

1. Inspect available MCP tools, including supported deferred-tool discovery.
2. Use the operation supported by the current schema; names vary by runtime.
3. If authentication is required, show the URL and retry after the user confirms
   login. Do not infer success from a delay.
4. If MCP is unavailable or lacks the operation, use the service CLI, then REST.
5. Report the method used and any capability that could not be verified.

Token presence never takes priority over an available MCP tool. GitHub MCP may
be disabled in some sessions; use `gh` when no suitable MCP tool is exposed.

## Check fallback credentials without exposing values

```bash
python3 - <<'PYTHON'
import os
names = {"JIRA_TOKEN", "GITHUB_TOKEN", "CONFLUENCE_TOKEN"}
names.update(key for key in os.environ if key.startswith("JENKINS_TOKEN_"))
for name in sorted(names):
    status = "set" if os.environ.get(name) else "not set"
    print(f"{name}: {status}")
PYTHON
```

Never dump environment variables or print tokens. Use service-specific guides
for REST authentication and endpoints only after selecting the fallback:

- [Jira](jira-rest-api.md)
- [GitHub](github-rest-api.md)
- [Confluence](confluence-rest-api.md)
- [Jenkins](jenkins-rest-api.md)

## Troubleshooting

- Authentication error: surface sign-in instructions and retry after confirmation.
- Permission denied: verify access; do not bypass restrictions with another token.
- Network timeout: check the target host and required VPN connectivity.
- Certificate error: configure the trusted CA; preserve certificate verification.
- Missing tools: report which capabilities are absent in the current session.
- Uncertain mutation result: check for the resulting object before retrying.
