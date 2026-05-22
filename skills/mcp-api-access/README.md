# API Fallback Pattern

## Overview

Skills that interact with external services (Jira, GitHub, Confluence, etc.) should support both MCP server access and direct REST API access. This allows skills to work in environments where:

- MCP servers are not configured or unavailable
- VPN requirements make MCP access unreliable
- Direct API access is preferred for performance or simplicity

## Decision Logic

When a skill needs to access an external service, follow this priority order:

1. **Try MCP first** - Use MCP server tools if available
2. **Fallback to REST API** - If MCP unavailable or fails, check for API token in environment variables
3. **If both fail** - Suggest the user configure MCP servers or provide an API token

## Environment Variable Naming

API tokens are used as fallback when MCP is unavailable. Service tokens should follow this pattern:

- Jira: `JIRA_TOKEN`
- GitHub: `GITHUB_TOKEN`
- Confluence: `CONFLUENCE_TOKEN`
- Jenkins: `JENKINS_TOKEN_<ALIAS>` (one per instance; default falls back to `JENKINS_TOKEN_PROD`)
- Generic: `<SERVICE_NAME>_TOKEN`

Additional configuration variables:

- `JIRA_BASE_URL` - Base URL for Jira instance (default: `https://jira.sie.sony.com`)
- `CONFLUENCE_BASE_URL` - Base URL for Confluence instance (default: `https://confluence.sie.sony.com`)
- `GITHUB_API_URL` - GitHub API URL (default: `https://github.sie.sony.com/api/v3`)
- `JENKINS_USER` - Jenkins username (shared across instances)

**Note:** MCP is the primary access method. Tokens are only checked if MCP is unavailable.

## Checking for Tokens

```bash
# Check if token exists (returns non-empty string if present)
echo $JIRA_TOKEN
echo $GITHUB_TOKEN
echo $CONFLUENCE_TOKEN
```

## Service-Specific Guides

See the following files for REST API implementation details:

- [Jira REST API](jira-rest-api.md)
- [GitHub REST API](github-rest-api.md)
- [Confluence REST API](confluence-rest-api.md)
- [Jenkins REST API](jenkins-rest-api.md)

## Implementation Pattern

### Step 1: Detect available access method

```markdown
1. Try to use MCP server tools first
2. If MCP unavailable or fails, check if `<SERVICE>_TOKEN` environment variable exists
3. If token exists, set `useRestApi = true`
4. If neither available, report error
```

### Step 2: Execute operation

```markdown
**Try MCP first:**
- Call the appropriate MCP tool (e.g., `jira_add_issue_comment`)
- Handle MCP-specific errors (VPN, authentication, etc.)
- If successful, operation complete

**If MCP fails or unavailable (REST API fallback):**
- Check if token exists in environment
- Get base URL from environment or use service default
  - Jira: `${JIRA_BASE_URL:-https://jira.sie.sony.com}`
  - GitHub: `${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}`
  - Confluence: `${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}`
- Construct the appropriate REST API request (see service-specific guide)
- Use `curl` via the Bash tool with appropriate headers
- Parse the JSON response
```

### Step 3: Error handling

```markdown
**MCP errors:**
- Server unavailable: MCP server not running or not configured
- Authentication required: Interactive sign-in needed
- Network/connection: VPN may be required
- If MCP fails, try REST API fallback

**REST API errors (fallback):**
- 401 Unauthorized: Token is invalid or expired
- 403 Forbidden: Token lacks required permissions
- 404 Not Found: Resource doesn't exist
- 429 Too Many Requests: Rate limit exceeded
- Network errors: Check connectivity
```

## Example Implementation

````markdown
## Step 2: Post the comment

**Try MCP first:**

Call `jira_add_issue_comment` with:
- `key`: the ticket key
- `comment`: the comment text

If MCP succeeds, done. If MCP fails or unavailable, proceed to fallback.

**If MCP fails or unavailable (REST API fallback):**

1. Check if `JIRA_TOKEN` environment variable is set.
1. If yes, get base URL: `JIRA_BASE_URL` from environment (default: `https://jira.sie.sony.com`)
1. Make a REST API request:

   ```bash
   curl -X POST "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/comment" \
     -H "Authorization: Bearer ${JIRA_TOKEN}" \
     -H "Content-Type: application/json" \
     -d "{\"body\": \"${COMMENT_TEXT}\"}"
   ```

1. Check response status code (200/201 = success)

**If both fail:**

Suggest VPN check, MCP restart, or verify REST API token.

## Adding New Services

To add support for a new service:

1. Define the token environment variable name: `<SERVICE>_TOKEN`
2. Create a REST API guide under `skills/mcp-api-access/<service>-rest-api.md`
3. Document the API endpoints, authentication, and request/response formats
4. Update skills that use the service to include the fallback pattern

**Note:** Services with multiple instances (e.g. Jenkins) use per-instance
tokens named `<SERVICE>_TOKEN_<ALIAS>`. The guide for that service must document
how to infer the alias from the user-provided URL and which instance token to
fall back to when no alias matches.
