# Jira REST API Reference

## Authentication

Use Bearer token authentication:

```bash
Authorization: Bearer ${JIRA_TOKEN}
```

## Base URL

Default: `https://jira.sie.sony.com`

Use: `${JIRA_BASE_URL:-https://jira.sie.sony.com}`

API endpoints are under `/rest/api/2/` (Jira REST API v2) or `/rest/agile/1.0/` (Jira Agile/Software API).

## Common Operations

### Get Issue

```bash
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json"
```

**Query parameters:**
- `fields`: Comma-separated list of fields to return (default: all)
  - Example: `fields=summary,status,assignee,description`
- `expand`: Comma-separated list of entities to expand
  - Example: `expand=changelog,renderedFields`

**Response:** JSON object with issue details

### Add Comment

```bash
curl -X POST "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/comment" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "body": "Comment text in Jira wiki markup format"
  }'
```

**Jira wiki markup format:**
- Bold: `*text*`
- Italic: `_text_`
- Code: `{{text}}`
- Code block: `{code}...{code}`
- Link: `[text|url]`
- List: `* item`

**Response:** 201 Created with comment object

### Update Issue

```bash
curl -X PUT "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "fields": {
      "summary": "New summary",
      "description": "New description"
    }
  }'
```

**Response:** 204 No Content on success

### Transition Issue

```bash
# Step 1: Get available transitions
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/transitions" \
  -H "Authorization: Bearer ${JIRA_TOKEN}"

# Step 2: Execute transition
curl -X POST "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/issue/${ISSUE_KEY}/transitions" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "transition": {
      "id": "TRANSITION_ID"
    },
    "fields": {
      "resolution": {
        "name": "Done"
      }
    }
  }'
```

**Response:** 204 No Content on success

### Get Current User

```bash
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/myself" \
  -H "Authorization: Bearer ${JIRA_TOKEN}"
```

**Response:** JSON object with user details (accountId, displayName, emailAddress)

### Get Agile Boards

```bash
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/agile/1.0/board" \
  -H "Authorization: Bearer ${JIRA_TOKEN}"
```

**Query parameters:**
- `name`: Filter by board name
- `type`: Filter by board type (`scrum` or `kanban`)
- `startAt`: Pagination start (default: 0)
- `maxResults`: Page size (default: 50, max: 100)

**Response:** JSON object with array of boards

### Get Board Sprints

```bash
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/agile/1.0/board/${BOARD_ID}/sprint" \
  -H "Authorization: Bearer ${JIRA_TOKEN}"
```

**Query parameters:**
- `state`: Filter by sprint state (`active`, `future`, `closed`)
- `startAt`: Pagination start
- `maxResults`: Page size

**Response:** JSON object with array of sprints

### Get Sprint Issues

```bash
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/agile/1.0/sprint/${SPRINT_ID}/issue" \
  -H "Authorization: Bearer ${JIRA_TOKEN}"
```

**Query parameters:**
- `fields`: Comma-separated list of fields
- `startAt`: Pagination start
- `maxResults`: Page size

**Response:** JSON object with array of issues

### Search Issues (JQL)

```bash
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/search" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  --data-urlencode "jql=project=DSOSYS AND assignee=currentUser()"
```

**Query parameters:**
- `jql`: JQL query string (URL-encoded)
- `fields`: Comma-separated list of fields
- `startAt`: Pagination start
- `maxResults`: Page size (max: 100)

**Response:** JSON object with array of issues

## Error Handling

### Common Status Codes

- **200 OK**: Successful GET request
- **201 Created**: Successful POST request
- **204 No Content**: Successful PUT/DELETE request
- **400 Bad Request**: Invalid request format or parameters
- **401 Unauthorized**: Invalid or expired token
- **403 Forbidden**: Insufficient permissions
- **404 Not Found**: Resource doesn't exist
- **429 Too Many Requests**: Rate limit exceeded

### Error Response Format

```json
{
  "errorMessages": ["Error message"],
  "errors": {
    "field": "Error details"
  }
}
```

## Rate Limiting

Jira Cloud/Server may impose rate limits. If you receive 429 responses:

- Wait before retrying (check `Retry-After` header if present)
- Reduce request frequency
- Batch operations when possible

## Tips

1. **Use `--data-urlencode` for JQL**: JQL queries must be URL-encoded
2. **Check field names**: Custom fields use IDs like `customfield_10001`
3. **Pagination**: Always use `startAt` and `maxResults` for large result sets
4. **Expanding data**: Use `expand` parameter sparingly to avoid large responses
