# GitHub REST API Reference

## Authentication

Use token authentication:

```bash
Authorization: Bearer ${GITHUB_TOKEN}
```

Or (legacy):

```bash
Authorization: token ${GITHUB_TOKEN}
```

Also include:

```bash
Accept: application/vnd.github+json
```

## Base URL

Default: `https://github.sie.sony.com/api/v3`

Use: `${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}`

**Note:** GitHub Enterprise uses `/api/v3` path for API endpoints.

## Common Operations

### Get Authenticated User

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/user" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Response:** JSON object with user details

### Get Repository

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Response:** JSON object with repository details

### List Repository Issues

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/issues" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Query parameters:**
- `state`: `open`, `closed`, or `all` (default: `open`)
- `assignee`: Filter by assignee username
- `labels`: Comma-separated list of label names
- `sort`: `created`, `updated`, or `comments` (default: `created`)
- `direction`: `asc` or `desc` (default: `desc`)
- `per_page`: Results per page (max: 100)
- `page`: Page number

**Response:** JSON array of issues

### Get Issue

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/issues/${ISSUE_NUMBER}" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Response:** JSON object with issue details

### Create Issue

```bash
curl -X POST "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/issues" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{
    "title": "Issue title",
    "body": "Issue description in Markdown",
    "assignees": ["username1", "username2"],
    "labels": ["bug", "urgent"]
  }'
```

**Response:** 201 Created with issue object

### Update Issue

```bash
curl -X PATCH "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/issues/${ISSUE_NUMBER}" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{
    "title": "Updated title",
    "body": "Updated description",
    "state": "closed"
  }'
```

**Response:** JSON object with updated issue

### Add Issue Comment

```bash
curl -X POST "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/issues/${ISSUE_NUMBER}/comments" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{
    "body": "Comment text in Markdown format"
  }'
```

**Response:** 201 Created with comment object

### List Pull Requests

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/pulls" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Query parameters:**
- `state`: `open`, `closed`, or `all` (default: `open`)
- `head`: Filter by head branch (`user:branch-name`)
- `base`: Filter by base branch
- `sort`: `created`, `updated`, `popularity`, or `long-running`
- `direction`: `asc` or `desc`
- `per_page`: Results per page (max: 100)
- `page`: Page number

**Response:** JSON array of pull requests

### Get Pull Request

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/pulls/${PR_NUMBER}" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Response:** JSON object with pull request details

### Create Pull Request

```bash
curl -X POST "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/pulls" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{
    "title": "PR title",
    "body": "PR description in Markdown",
    "head": "branch-name",
    "base": "main"
  }'
```

**Response:** 201 Created with pull request object

### List Commits

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/commits" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Query parameters:**
- `sha`: SHA or branch name to start listing commits from
- `path`: Only commits containing this file path
- `author`: GitHub username or email address
- `since`: ISO 8601 timestamp
- `until`: ISO 8601 timestamp
- `per_page`: Results per page (max: 100)
- `page`: Page number

**Response:** JSON array of commits

### Get File Contents

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/contents/${PATH}" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json"
```

**Query parameters:**
- `ref`: Branch, tag, or commit SHA (default: default branch)

**Response:** JSON object with file metadata and base64-encoded content

### Create or Update File

```bash
curl -X PUT "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/repos/${OWNER}/${REPO}/contents/${PATH}" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -d '{
    "message": "Commit message",
    "content": "BASE64_ENCODED_CONTENT",
    "branch": "branch-name",
    "sha": "EXISTING_FILE_SHA"
  }'
```

**Note:** 
- `content` must be base64-encoded
- `sha` is required when updating an existing file
- Omit `sha` when creating a new file

**Response:** 201 Created (new file) or 200 OK (updated file)

## Error Handling

### Common Status Codes

- **200 OK**: Successful GET/PATCH request
- **201 Created**: Successful POST request
- **204 No Content**: Successful DELETE request
- **304 Not Modified**: Resource not modified (when using conditional requests)
- **400 Bad Request**: Invalid request format
- **401 Unauthorized**: Invalid or expired token
- **403 Forbidden**: Insufficient permissions or rate limit exceeded
- **404 Not Found**: Resource doesn't exist
- **422 Unprocessable Entity**: Validation failed

### Error Response Format

```json
{
  "message": "Error message",
  "documentation_url": "https://docs.github.com/...",
  "errors": [
    {
      "resource": "Issue",
      "field": "title",
      "code": "missing_field"
    }
  ]
}
```

## Rate Limiting

GitHub API has rate limits:

- **Authenticated requests**: 5,000 requests per hour
- **Unauthenticated requests**: 60 requests per hour

Check rate limit status:

```bash
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/rate_limit" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}"
```

Response headers:
- `X-RateLimit-Limit`: Maximum requests per hour
- `X-RateLimit-Remaining`: Remaining requests
- `X-RateLimit-Reset`: Unix timestamp when limit resets

## Pagination

For paginated responses, GitHub uses Link headers:

```
Link: <https://github.sie.sony.com/api/v3/repos/owner/repo/issues?page=2>; rel="next",
      <https://github.sie.sony.com/api/v3/repos/owner/repo/issues?page=5>; rel="last"
```

Parse these headers to navigate pages.

## Tips

1. **Use Personal Access Tokens**: Create fine-grained tokens with minimal required permissions
2. **Check API version**: Always include `Accept: application/vnd.github+json` header
3. **Handle pagination**: Use `per_page` and `page` parameters, or parse Link headers
4. **Respect rate limits**: Monitor `X-RateLimit-Remaining` header
5. **Use conditional requests**: Include `If-None-Match` or `If-Modified-Since` headers to save quota
