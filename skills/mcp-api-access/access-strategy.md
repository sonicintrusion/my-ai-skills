# Access Strategy: MCP → Shell → REST API

## Overview

Skills that interact with external services must follow a strict 3-tier access
order. Each tier is only attempted if the previous tier is unavailable or cannot
fulfil the request.

1. **Tier 1 — MCP**: Use the configured MCP tool.
1. **Tier 2 — Shell CLI**: Use the service's dedicated CLI tool.
1. **Tier 3 — REST API**: Fall back to direct API calls.

Do not skip tiers. If a tier requires authentication, complete that process
before moving to the next tier.

---

## Tier 1: MCP

### Try the MCP tool

Call the relevant MCP tool for the operation.

If the call succeeds, the task is complete. Do not proceed to Tier 2.

### MCP authentication required

If the MCP tool responds with an authentication or authorisation error:

1. Extract the auth URL from the MCP response.
1. Present the URL to the user:

   > MCP requires authentication. Please complete sign-in at: `<auth URL>`

1. Wait for the user to confirm they have completed authentication.
1. Retry the MCP operation.

If the retry succeeds, the task is complete. Do not proceed to Tier 2.

### When to fall through from Tier 1

Fall through when:

- MCP tool is not configured or unavailable.
- MCP tool remains unavailable after completing authentication.
- MCP tool does not support the required operation or feature.
- MCP tool returns a persistent error unrelated to authentication.

---

## Tier 2: Shell CLI

Each service may have a dedicated shell CLI. Use the appropriate tool from the
table below.

| Service    | CLI tool | Availability check | Auth check        |
|------------|----------|--------------------|-------------------|
| GitHub     | `gh`     | `command -v gh`    | `gh auth status`  |
| Jira       | —        | N/A                | N/A               |
| Confluence | —        | N/A                | N/A               |

If no CLI tool is listed for the service, skip directly to Tier 3.

### Try the shell CLI

1. Verify the CLI tool is available:

   ```bash
   command -v <tool>
   ```

   If the tool is not installed, fall through to Tier 3.

1. Verify the CLI tool is authenticated:

   ```bash
   <tool> auth status
   ```

   If unauthenticated:

   1. Inform the user:

      > `<tool>` is not authenticated. Please complete sign-in
      > (e.g. `<tool> auth login`) and confirm when ready.

   1. Wait for the user to confirm authentication is complete.
   1. Re-run the auth status check.
   1. Retry the requested operation.

   If authentication fails or the tool is still unavailable, fall through to
   Tier 3.

1. Execute the operation using the CLI tool.

If the operation succeeds, the task is complete. Do not proceed to Tier 3.

### When to fall through from Tier 2

Fall through when:

- No CLI tool exists for the service.
- CLI tool is not installed.
- CLI tool remains unauthenticated after the user completes the auth flow.
- CLI tool does not support the required operation or feature.
- CLI tool returns a persistent error unrelated to authentication.

---

## Tier 3: REST API

Use direct REST API calls via `curl` in the Bash tool.

See [README.md](README.md) for the general fallback pattern and the
service-specific guides below:

- [Jira REST API](jira-rest-api.md)
- [GitHub REST API](github-rest-api.md)
- [Confluence REST API](confluence-rest-api.md)

### REST API authentication

REST API calls require a token from the environment:

| Service    | Token variable          | Base URL variable     |
|------------|-------------------------|-----------------------|
| GitHub     | `GITHUB_TOKEN`          | `GITHUB_API_URL`      |
| Jira       | `JIRA_TOKEN`            | `JIRA_BASE_URL`       |
| Confluence | `CONFLUENCE_TOKEN`      | `CONFLUENCE_BASE_URL` |
| Jenkins    | `JENKINS_TOKEN_<ALIAS>` | extracted from URL    |

Default base URLs:

- GitHub: `https://github.sie.sony.com/api/v3`
- Jira: `https://jira.sie.sony.com`
- Confluence: `https://confluence.sie.sony.com`
- Jenkins: extracted from the user-provided URL (no fixed default)

**Jenkins token note:** The alias is inferred from the URL hostname (e.g.
`jenkins-manage.example.com` → `JENKINS_TOKEN_MANAGE`). When no alias can be
inferred, fall back to `JENKINS_TOKEN_PROD`. Also requires `JENKINS_USER`
(username, shared across instances). See [Jenkins REST API](jenkins-rest-api.md)
for full resolution logic.

See [Jenkins REST API](jenkins-rest-api.md) for full token-resolution logic.

> The `<TOKEN>` environment variable is not set. Please set it and retry.

### When all tiers fail

If all three tiers fail or are unavailable, report:

- Which tiers were attempted.
- The specific failure reason for each.
- Suggested corrective actions (VPN, token rotation, MCP restart, CLI install).

---

## Decision Flow

```text
┌─────────────────────┐
│ Operation requested │
└──────────┬──────────┘
           │
           ▼
┌──────────────────────┐
│ Tier 1: Try MCP tool │
└──────────┬───────────┘
           │
     ┌─────┴──────┐
     │ Succeeded? │
     └─────┬──────┘
           │
     ┌─────┴──────┐
     │            │
    Yes           No
     │            │
     ▼            ▼
  ┌──────┐  ┌───────────────────┐
  │ Done │  │  Auth required?   │
  └──────┘  └────────┬──────────┘
                     │
              ┌──────┴──────┐
              │             │
             Yes            No
              │             │
              ▼             ▼
       ┌─────────────┐  ┌──────────────────────┐
       │ Present URL │  │ Tier 2: Try shell CLI │
       │ Wait, retry │  └──────────┬───────────┘
       └──────┬──────┘             │
              │              ┌─────┴──────┐
         ┌────┴────┐         │ Succeeded? │
         │Succeeded│         └─────┬──────┘
         └────┬────┘               │
              │              ┌─────┴──────┐
        ┌─────┴──────┐       │            │
        │            │      Yes           No
       Yes           No      │            │
        │            │       ▼            ▼
        ▼            ▼    ┌──────┐  ┌────────────────────┐
     ┌──────┐  ┌──────────┐ Done │  │ Tier 3: REST API   │
     │ Done │  │ Tier 2   │└──────┘  │ (see mcp-api-access) │
     └──────┘  └──────────┘         └────────────────────┘
```

---

## Completion Reporting

When the operation completes, report:

1. Which tier was used (MCP, shell CLI, or REST API).
1. Whether authentication was required and how it was resolved.
1. The resulting object(s) created, updated, or fetched.
