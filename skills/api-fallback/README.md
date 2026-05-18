# API Fallback Pattern

## Overview

Skills that interact with external services (Jira, GitHub, Confluence, etc.) should support both MCP server access and direct REST API access. This allows skills to work in environments where:

- MCP servers are not configured or unavailable
- VPN requirements make MCP access unreliable
- Direct API access is preferred for performance or simplicity

## Decision Logic

When a skill needs to access an external service, follow this priority order:

1. **Check for API token** in environment variables
2. **If token exists**: Use REST API (see service-specific guides below)
3. **If no token**: Use MCP server tools
4. **If MCP fails**: Suggest the user provide an API token or fix MCP configuration

## Environment Variable Naming

Service tokens should follow this pattern:

- Jira: `JIRA_TOKEN`
- GitHub: `GITHUB_TOKEN`
- Confluence: `CONFLUENCE_TOKEN`
- Generic: `<SERVICE_NAME>_TOKEN`

Additional configuration variables:

- `JIRA_BASE_URL` - Base URL for Jira instance (default: `https://jira.sie.sony.com`)
- `CONFLUENCE_BASE_URL` - Base URL for Confluence instance (default: `https://confluence.sie.sony.com`)
- `GITHUB_API_URL` - GitHub API URL (default: `https://github.sie.sony.com/api/v3`)

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

## Implementation Pattern

### Step 1: Detect available access method

```markdown
1. Check if `<SERVICE>_TOKEN` environment variable exists and is non-empty
2. If yes, set `useRestApi = true`
3. If no, set `useRestApi = false` (will use MCP)
```

### Step 2: Execute operation

```markdown
**If using REST API:**
- Get base URL from environment or use Sony default
  - Jira: `${JIRA_BASE_URL:-https://jira.sie.sony.com}`
  - GitHub: `${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}`
  - Confluence: `${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}`
- Construct the appropriate REST API request (see service-specific guide)
- Use `curl` via the Bash tool with appropriate headers
- Parse the JSON response

**If using MCP:**
- Call the appropriate MCP tool (e.g., `jira_add_issue_comment`)
- Handle MCP-specific errors (VPN, authentication, etc.)
```

### Step 3: Error handling

```markdown
**REST API errors:**
- 401 Unauthorized: Token is invalid or expired
- 403 Forbidden: Token lacks required permissions
- 404 Not Found: Resource doesn't exist
- 429 Too Many Requests: Rate limit exceeded
- Network errors: Check connectivity

**MCP errors:**
- Server unavailable: MCP server not running or not configured
- Authentication required: Interactive sign-in needed
- Network/connection: VPN may be required
```

## Example Implementation

```markdown
## Step 2: Post the comment

1. Check if `JIRA_TOKEN` environment variable is set.

**If JIRA_TOKEN is available (REST API path):**

1. Get the token: `JIRA_TOKEN` from environment
2. Get the base URL: `JIRA_BASE_URL` from environment (default: `https://jira.sie.sony.com`)
3. Make a REST API request:
   ```bash
   curl -X POST "${JIRA_BASE_URL}/rest/api/2/issue/${ISSUE_KEY}/comment" \
     -H "Authorization: Bearer ${JIRA_TOKEN}" \
     -H "Content-Type: application/json" \
     -d "{\"body\": \"${COMMENT_TEXT}\"}"
   ```
4. Check response status code (200/201 = success)

**If JIRA_TOKEN is not available (MCP path):**

1. Call `jira_add_issue_comment` with:
   - `key`: the ticket key
   - `comment`: the comment text
2. Handle MCP-specific errors (VPN, authentication, etc.)
```

## Adding New Services

To add support for a new service:

1. Define the token environment variable name: `<SERVICE>_TOKEN`
2. Create a REST API guide under `skills/api-fallback/<service>-rest-api.md`
3. Document the API endpoints, authentication, and request/response formats
4. Update skills that use the service to include the fallback pattern
