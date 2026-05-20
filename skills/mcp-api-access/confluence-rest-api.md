# Confluence REST API Reference

## Authentication

Use Bearer token authentication:

```bash
Authorization: Bearer ${CONFLUENCE_TOKEN}
```

## Base URL

Default: `https://confluence.sie.sony.com`

Use: `${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}`

API endpoints are under `/rest/api/` (Confluence REST API).

## Common Operations

### Get Current User

```bash
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/user/current" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json"
```

**Response:** JSON object with user details

### Get Space

```bash
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/space/${SPACE_KEY}" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json"
```

**Query parameters:**

- `expand`: Comma-separated list of entities to expand
  - Example: `expand=description,homepage,metadata.labels`

**Response:** JSON object with space details

### List Spaces

```bash
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/space" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json"
```

**Query parameters:**

- `type`: Filter by space type (`global` or `personal`)
- `status`: Filter by status (`current` or `archived`)
- `limit`: Results per page (max: 500)
- `start`: Pagination offset

**Response:** JSON object with array of spaces

### Get Page

```bash
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content/${PAGE_ID}" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json"
```

**Query parameters:**

- `expand`: Comma-separated list of entities to expand
  - Common values: `body.storage,body.view,version,space,history,ancestors,metadata.labels`
- `status`: Content status (`current`, `trashed`, `draft`, `historical`)

**Response:** JSON object with page details

### Get Page by Title

```bash
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  --data-urlencode "spaceKey=${SPACE_KEY}" \
  --data-urlencode "title=${PAGE_TITLE}"
```

**Query parameters:**

- `spaceKey`: Space key (required)
- `title`: Exact page title (required)
- `expand`: Entities to expand

**Response:** JSON object with array of matching pages

### Create Page

```bash
curl -X POST "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "page",
    "title": "Page Title",
    "space": {
      "key": "SPACEKEY"
    },
    "body": {
      "storage": {
        "value": "<p>Page content in storage format</p>",
        "representation": "storage"
      }
    },
    "ancestors": [
      {
        "id": "PARENT_PAGE_ID"
      }
    ]
  }'
```

**Note:**

- `ancestors` is optional (if omitted, page is created at space root)
- Content must be in Confluence storage format (XHTML-like)

**Response:** 201 Created with page object

### Update Page

```bash
curl -X PUT "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content/${PAGE_ID}" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "version": {
      "number": CURRENT_VERSION + 1
    },
    "title": "Updated Title",
    "type": "page",
    "body": {
      "storage": {
        "value": "<p>Updated content</p>",
        "representation": "storage"
      }
    }
  }'
```

**Note:**

- `version.number` must be current version + 1
- Get current version from a GET request first

**Response:** JSON object with updated page

### Add Comment

```bash
curl -X POST "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "comment",
    "container": {
      "id": "PAGE_ID",
      "type": "page"
    },
    "body": {
      "storage": {
        "value": "<p>Comment text</p>",
        "representation": "storage"
      }
    }
  }'
```

**Response:** 201 Created with comment object

### List Comments

```bash
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content/${PAGE_ID}/child/comment" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json"
```

**Query parameters:**

- `expand`: Entities to expand (e.g., `body.storage,version`)
- `limit`: Results per page
- `start`: Pagination offset

**Response:** JSON object with array of comments

### Add Label

```bash
curl -X POST "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content/${PAGE_ID}/label" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "prefix": "global",
      "name": "label-name"
    }
  ]'
```

**Response:** JSON object with label details

### Search Content (CQL)

```bash
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/content/search" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  --data-urlencode "cql=space=SPACEKEY AND type=page AND title~\"search term\""
```

**Query parameters:**

- `cql`: Confluence Query Language query (URL-encoded)
- `expand`: Entities to expand
- `limit`: Results per page (max: 200)
- `start`: Pagination offset

**CQL Examples:**

- `space=SPACEKEY AND type=page`: All pages in a space
- `creator=currentUser()`: Content created by current user
- `lastModified >= "2026-01-01"`: Recently modified content
- `label="important" AND space=SPACEKEY`: Labeled content

**Response:** JSON object with array of content

## Storage Format

Confluence uses a storage format (XHTML-like) for content:

### Common Elements

- Paragraph: `<p>Text</p>`
- Bold: `<strong>Text</strong>`
- Italic: `<em>Text</em>`
- Code: `<code>Text</code>`
- Link: `<a href="https://url">Text</a>`
- Heading: `<h1>Heading</h1>` (h1-h6)
- List (unordered): `<ul><li>Item</li></ul>`
- List (ordered): `<ol><li>Item</li></ol>`
- Code block: `<ac:structured-macro ac:name="code"><ac:plain-text-body><![CDATA[code]]></ac:plain-text-body></ac:structured-macro>`

### Macros

Confluence macros use structured-macro elements:

```xml
<ac:structured-macro ac:name="info">
  <ac:rich-text-body>
    <p>Info message</p>
  </ac:rich-text-body>
</ac:structured-macro>
```

Common macro names:

- `info`, `note`, `warning`, `tip`: Colored panels
- `code`: Code block
- `toc`: Table of contents
- `excerpt`: Page excerpt

## Error Handling

### Common Status Codes

- **200 OK**: Successful GET request
- **201 Created**: Successful POST request
- **204 No Content**: Successful PUT/DELETE request
- **400 Bad Request**: Invalid request format
- **401 Unauthorized**: Invalid or expired token
- **403 Forbidden**: Insufficient permissions
- **404 Not Found**: Resource doesn't exist
- **409 Conflict**: Version conflict (page was modified)

### Error Response Format

```json
{
  "statusCode": 400,
  "message": "Error message",
  "reason": "Bad Request"
}
```

## Rate Limiting

Confluence may impose rate limits. If you receive 429 responses:

- Wait before retrying
- Reduce request frequency
- Contact administrator for rate limit increase

## Tips

1. **Version management**: Always increment version number when updating pages
2. **Storage format**: Use proper XHTML for content (no markdown)
3. **Pagination**: Use `limit` and `start` parameters for large result sets
4. **CQL queries**: Must be URL-encoded
5. **Expand sparingly**: Large expand parameters can slow requests
6. **Get before update**: Always fetch current version before updating
7. **Ancestors**: To nest pages, include parent page ID in `ancestors` array
