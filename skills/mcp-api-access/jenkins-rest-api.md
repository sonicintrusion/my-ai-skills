# Jenkins REST API Reference

## Token Resolution

Jenkins uses multiple instances, each with its own token. Tokens are exported
in the shell as `JENKINS_TOKEN_<ALIAS>` where `<ALIAS>` is the uppercase
account alias for that instance (e.g. `JENKINS_TOKEN_MANAGE`,
`JENKINS_TOKEN_RANDD`, `JENKINS_TOKEN_PROD`).

### Inferring the token from a URL

Hostnames do not always embed a readable alias (e.g.
`build.randd.bis.sie.sony.com`). Use this resolution order:

1. **Scan available tokens.** List all `JENKINS_TOKEN_*` variables set in the
   environment and check whether any alias appears as a substring of the
   hostname (case-insensitive).
2. **Fall back to prod.** If no alias matches, use `JENKINS_TOKEN_PROD`.
3. **Verify the token is set.** If the resolved variable is empty, stop and
   ask the user which token to use.

### Token lookup algorithm

```bash
JENKINS_URL="https://build.randd.bis.sie.sony.com/job/SYS/job/EKS/..."
JENKINS_HOST=$(echo "$JENKINS_URL" | awk -F/ '{print $3}')

# Scan all JENKINS_TOKEN_* variables for a hostname match
JENKINS_TOKEN=""
while IFS='=' read -r var_name var_value; do
  alias="${var_name#JENKINS_TOKEN_}"          # strip prefix → e.g. RANDD
  if echo "$JENKINS_HOST" | grep -qi "$alias"; then
    JENKINS_TOKEN="$var_value"
    break
  fi
done < <(env | grep '^JENKINS_TOKEN_' | sort)

# Fall back to prod if no match found
if [ -z "$JENKINS_TOKEN" ]; then
  JENKINS_TOKEN="${JENKINS_TOKEN_PROD}"
fi

if [ -z "$JENKINS_TOKEN" ]; then
  echo "Error: no Jenkins token resolved for host ${JENKINS_HOST}."
  echo "Set JENKINS_TOKEN_PROD or a matching JENKINS_TOKEN_<ALIAS> and retry."
  exit 1
fi
```

## Authentication

Jenkins REST API uses HTTP Basic Authentication. The username is taken from the
local shell variable `$USER`. If `$USER` is empty, ask the user for their
Jenkins username before proceeding.

```bash
# Resolve username
JENKINS_USERNAME="${USER}"
if [ -z "$JENKINS_USERNAME" ]; then
  echo "Error: \$USER is not set. Please provide your Jenkins username."
  exit 1
fi
```

All API calls use:

```bash
-u "${USER}:${JENKINS_TOKEN}"
```

All write operations must also include a CSRF crumb unless the instance has
CSRF disabled. See [CSRF / Crumb](#csrf--crumb) below.

## Base URL

The base URL is the root of the Jenkins instance (no trailing path).
Extract it from the user-provided URL:

```bash
# Given: https://jenkins-manage.example.com/job/my-pipeline/build
# Base:  https://jenkins-manage.example.com
JENKINS_BASE_URL=$(echo "$JENKINS_URL" | awk -F/ '{print $1"//"$3}')
```

## CSRF / Crumb

Most Jenkins instances require a CSRF crumb header on write operations (POST).

### Fetch a crumb

```bash
CRUMB=$(curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/crumbIssuer/api/json" \
  | jq -r '"\(.crumbRequestField):\(.crumb)"')
```

### Use the crumb in write requests

```bash
curl -X POST "${JENKINS_BASE_URL}/..." \
  -u "${USER}:${JENKINS_TOKEN}" \
  -H "${CRUMB}" \
  ...
```

If the instance returns 404 on `/crumbIssuer`, CSRF is disabled and the
`-H "${CRUMB}"` header can be omitted.

## Common Operations

### Get server info

```bash
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/api/json" | jq .
```

### List jobs

```bash
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/api/json?tree=jobs[name,url,color]" | jq .
```

### Get job info

```bash
# JOB_PATH is the URL path segment(s) after /job/, slash-separated
# e.g. for https://jenkins.example.com/job/my-folder/job/my-job/
# JOB_PATH = "my-folder/job/my-job"
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/job/${JOB_PATH}/api/json" | jq .
```

### Get last build status

```bash
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/job/${JOB_PATH}/lastBuild/api/json" \
  | jq '{result: .result, building: .building, url: .url, number: .number}'
```

### Get specific build info

```bash
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/job/${JOB_PATH}/${BUILD_NUMBER}/api/json" | jq .
```

### Trigger a build (no parameters)

```bash
CRUMB=$(curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/crumbIssuer/api/json" \
  | jq -r '"\(.crumbRequestField):\(.crumb)"')

curl -X POST "${JENKINS_BASE_URL}/job/${JOB_PATH}/build" \
  -u "${USER}:${JENKINS_TOKEN}" \
  -H "${CRUMB}"
```

**Response:** 201 Created with a `Location` header pointing to the queue item.

### Trigger a parameterised build

```bash
CRUMB=$(curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/crumbIssuer/api/json" \
  | jq -r '"\(.crumbRequestField):\(.crumb)"')

curl -X POST "${JENKINS_BASE_URL}/job/${JOB_PATH}/buildWithParameters" \
  -u "${USER}:${JENKINS_TOKEN}" \
  -H "${CRUMB}" \
  --data-urlencode "BRANCH=main" \
  --data-urlencode "DEPLOY_ENV=staging"
```

### Get build console log

```bash
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/job/${JOB_PATH}/${BUILD_NUMBER}/consoleText"
```

### Get queue item status

After triggering a build, the `Location` header returns a queue URL. Poll it
to find the actual build number once Jenkins schedules the build:

```bash
QUEUE_URL="https://jenkins.example.com/queue/item/42/"
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${QUEUE_URL}api/json" \
  | jq '{why: .why, executable: .executable}'
```

When `.executable` is non-null, `.executable.number` is the build number.

### Stop a running build

```bash
CRUMB=$(curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/crumbIssuer/api/json" \
  | jq -r '"\(.crumbRequestField):\(.crumb)"')

curl -X POST "${JENKINS_BASE_URL}/job/${JOB_PATH}/${BUILD_NUMBER}/stop" \
  -u "${USER}:${JENKINS_TOKEN}" \
  -H "${CRUMB}"
```

### List pipeline stages (Blue Ocean API)

If the Blue Ocean plugin is installed:

```bash
curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/blue/rest/organizations/jenkins/pipelines/${PIPELINE_NAME}/runs/${BUILD_NUMBER}/nodes/" \
  | jq '[.[] | {displayName, result, durationInMillis}]'
```

## Job Path Conventions

Jenkins URLs encode job hierarchy using `/job/` separators:

| URL path                           | `JOB_PATH` value             |
|------------------------------------|------------------------------|
| `/job/my-pipeline/`                | `my-pipeline`                |
| `/job/my-folder/job/my-pipeline/`  | `my-folder/job/my-pipeline`  |
| `/job/a/job/b/job/c/`              | `a/job/b/job/c`              |

For example, the URL:

```text
https://build.randd.bis.sie.sony.com/job/SYS/job/EKS/job/khanh/job/Deploy-EKS-Cluster-V2-KN/398/console
```

has `JOB_PATH = SYS/job/EKS/job/khanh/job/Deploy-EKS-Cluster-V2-KN` and
`BUILD_NUMBER = 398`.

### Extracting `JOB_PATH` and `BUILD_NUMBER` from a URL

```bash
# Remove base URL prefix and strip trailing path components after the build number
# Works for both /job/.../NNN/ and /job/.../NNN/console style URLs
JENKINS_BASE_URL=$(echo "$JENKINS_URL" | awk -F/ '{print $1"//"$3}')
AFTER_BASE="${JENKINS_URL#${JENKINS_BASE_URL}/}"  # e.g. job/SYS/job/EKS/.../398/console

# Extract build number (first pure integer path segment)
BUILD_NUMBER=$(echo "$AFTER_BASE" | grep -oE '/[0-9]+/' | head -1 | tr -d '/')

# Strip leading "job/" and everything from the build number onward
JOB_PATH=$(echo "$AFTER_BASE" \
  | sed 's|^job/||' \
  | sed "s|/${BUILD_NUMBER}/.*||")
```

## Error Handling

### Common Status Codes

- **200 OK**: Successful GET
- **201 Created**: Build queued (includes `Location` header)
- **302 Found**: Redirect (follow with `-L`)
- **400 Bad Request**: Invalid parameters
- **401 Unauthorized**: Bad credentials
- **403 Forbidden**: CSRF crumb missing or invalid; or insufficient permissions
- **404 Not Found**: Job or build does not exist
- **503 Service Unavailable**: Jenkins is starting up or in quiet mode

### 403 on POST — CSRF

If a POST returns 403, re-fetch the crumb and retry. Crumbs are session-scoped
and expire.

```bash
# Re-fetch crumb then retry
CRUMB=$(curl -s -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/crumbIssuer/api/json" \
  | jq -r '"\(.crumbRequestField):\(.crumb)"')
```

### Error Response Format

Jenkins typically returns plain HTML on error. Use `-w "%{http_code}"` to
capture the status code separately:

```bash
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
  -u "${USER}:${JENKINS_TOKEN}" \
  "${JENKINS_BASE_URL}/job/${JOB_PATH}/api/json")

if [ "$HTTP_CODE" != "200" ]; then
  echo "Error: Jenkins returned HTTP ${HTTP_CODE}"
fi
```

## Environment Variables Reference

| Variable                | Description                                    | Required                |
|-------------------------|------------------------------------------------|-------------------------|
| `JENKINS_TOKEN_<ALIAS>` | API token for the named instance               | Yes (one per instance)  |
| `JENKINS_TOKEN_PROD`    | Default token when alias cannot be inferred    | Yes                     |
| `USER`                  | Shell username, set automatically by the shell | Yes (ask user if unset) |
