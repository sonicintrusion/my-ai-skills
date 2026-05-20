# Quick Reference: API Access Methods

This document provides quick examples of how to check which access method will be used and how to test each service.

## Check Environment Status

```bash
# Quick status check
env | grep -E '(JIRA|GITHUB|CONFLUENCE)_(TOKEN|BASE_URL)'
```

## Test Each Service

### Jira

```bash
# Test Jira access (get current user)
curl -X GET "${JIRA_BASE_URL:-https://jira.sie.sony.com}/rest/api/2/myself" \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -s | jq -r '.displayName // "Error: " + .errorMessages[0]'
```

### GitHub

```bash
# Test GitHub access (get current user)
curl -X GET "${GITHUB_API_URL:-https://github.sie.sony.com/api/v3}/user" \
  -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  -H "Accept: application/vnd.github+json" \
  -s | jq -r '.login // "Error: " + .message'
```

### Confluence

```bash
# Test Confluence access (get current user)
curl -X GET "${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com}/rest/api/user/current" \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  -s | jq -r '.username // "Error"'
```

## Troubleshooting

### Token Issues

**Symptom**: 401 Unauthorized
- **Cause**: Token is invalid or expired
- **Fix**: Regenerate token in the service's settings

**Symptom**: 403 Forbidden
- **Cause**: Token lacks required permissions
- **Fix**: Update token permissions/scopes

### Network Issues

**Symptom**: Connection timeout or refused
- **Cause**: VPN required or firewall blocking
- **Fix**: Connect to VPN or check network settings

**Symptom**: Certificate errors
- **Cause**: Self-signed certificates or corporate proxy
- **Fix**: Add `-k` flag to curl (insecure, use cautiously) or configure CA certificates

### MCP Fallback

If REST API fails but MCP is configured:
1. Skill will automatically attempt to use MCP
2. Check MCP server status: Look for MCP server logs
3. Restart MCP servers if needed

## Decision Flow

```
┌─────────────────────────┐
│  Skill needs to access  │
│   external service      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ Check for *_TOKEN env   │
│ variable                │
└───────────┬─────────────┘
            │
        ┌───┴───┐
        │Token? │
        └───┬───┘
            │
    ┌───────┴───────┐
    │               │
    ▼               ▼
┌───────┐      ┌────────┐
│  Yes  │      │   No   │
└───┬───┘      └────┬───┘
    │               │
    ▼               ▼
┌───────────┐  ┌─────────┐
│  Use REST │  │ Use MCP │
│    API    │  │ Server  │
└───────────┘  └─────────┘
```

## Current Environment Status

Run this to see your current configuration:

```bash
echo "=== API Access Configuration ==="
echo ""
echo "Jira:"
echo "  Token: $([ -n "$JIRA_TOKEN" ] && echo "✓ Available (will use REST API)" || echo "✗ Not set (will use MCP)")"
echo "  Base URL: ${JIRA_BASE_URL:-https://jira.sie.sony.com} (default)"
echo ""
echo "GitHub:"
echo "  Token: $([ -n "$GITHUB_TOKEN" ] && echo "✓ Available (will use REST API)" || echo "✗ Not set (will use MCP)")"
echo "  API URL: ${GITHUB_API_URL:-https://github.sie.sony.com/api/v3} (default)"
echo ""
echo "Confluence:"
echo "  Token: $([ -n "$CONFLUENCE_TOKEN" ] && echo "✓ Available (will use REST API)" || echo "✗ Not set (will use MCP)")"
echo "  Base URL: ${CONFLUENCE_BASE_URL:-https://confluence.sie.sony.com} (default)"
```

## Performance Comparison

| Method | Latency | Reliability | VPN Required | Setup Complexity |
|--------|---------|-------------|--------------|------------------|
| REST API | Lower (~100-500ms) | High | Maybe* | Low (just token) |
| MCP Server | Higher (~200-1000ms) | Medium | Maybe* | Medium (server + config) |

*Depends on corporate network configuration

## Best Practices

1. **Set tokens for best performance**: REST API is typically faster
2. **Keep tokens secure**: Store in secure credential managers
3. **Rotate tokens regularly**: Follow security best practices
4. **Monitor rate limits**: Both REST API and MCP have limits
5. **Use VPN when required**: Some services require VPN regardless of method
6. **Test both paths**: Ensure fallback works if primary method fails
