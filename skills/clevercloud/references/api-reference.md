# Clever Cloud API Reference

Base URL: `https://api.clever-cloud.com`  
Auth: OAuth1 HMAC-SHA256 (see below) or `clever` CLI.

---

## Authentication

### OAuth1 Flow

Clever Cloud uses OAuth 1.0a with HMAC-SHA256 signatures.

**Step 1 — Get request token:**
```bash
curl -s "https://api.clever-cloud.com/v1/oauth/request_token" \
  -X POST \
  -H "Authorization: OAuth oauth_consumer_key=\"$CLEVER_CONSUMER_KEY\",oauth_signature_method=\"HMAC-SHA256\",..."
```

**Step 2 — Redirect user to authorize:**
```
https://console.clever-cloud.com/oauth/authorize?oauth_token=<request_token>
```

**Step 3 — Exchange for access token:**
```bash
curl -s "https://api.clever-cloud.com/v1/oauth/access_token" -X POST ...
```

**Practical shortcut:** Use `clever login` (CLI) to complete the OAuth flow once and extract the resulting tokens from `~/.config/clever-cloud/clever-tools.json`:
```json
{ "token": "...", "secret": "..." }
```

### Using tokens for API calls

After obtaining `CLEVER_TOKEN` and `CLEVER_SECRET`, generate OAuth1 signatures per request. Tools:
- `oauth-1.0a` Node.js library
- `clever` CLI (wraps all auth automatically)

For simple GET reads, try Bearer token first (some endpoints accept it):
```bash
-H "Authorization: Bearer $CLEVER_TOKEN"
```

---

## Endpoints

### Self (personal account)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v2/self` | Get current user info |
| GET | `/v2/self/applications` | List personal applications |
| GET | `/v2/self/applications/{appId}` | Get app details |
| PUT | `/v2/self/applications/{appId}` | Update app config |
| DELETE | `/v2/self/applications/{appId}` | Delete app |
| GET | `/v2/self/applications/{appId}/instances` | List running instances |
| DELETE | `/v2/self/applications/{appId}/instances` | Stop all instances |
| GET | `/v2/self/applications/{appId}/env` | List env vars |
| PUT | `/v2/self/applications/{appId}/env` | Set env vars (replaces all) |
| PUT | `/v2/self/applications/{appId}/env/{varName}` | Set single env var |
| DELETE | `/v2/self/applications/{appId}/env/{varName}` | Delete env var |
| GET | `/v2/self/addons` | List add-ons |
| GET | `/v2/self/addons/{addonId}` | Get add-on details |

### Organisations

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v2/organisations/{orgId}` | Get org info |
| GET | `/v2/organisations/{orgId}/applications` | List org apps |
| GET | `/v2/organisations/{orgId}/applications/{appId}` | Get org app details |
| GET | `/v2/organisations/{orgId}/addons` | List org add-ons |
| GET | `/v2/organisations/{orgId}/members` | List org members |

### Logs

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v2/logs/{appId}` | Get recent logs |
| GET | `/v4/logs/applications/{appId}/logs` | Stream logs (Server-Sent Events) |
| GET | `/v2/logs/{appId}/drains` | List log drains |
| POST | `/v2/logs/{appId}/drains` | Add log drain |

Query params for logs:
- `limit` — number of lines (default 300)
- `before` — ISO timestamp, get logs before this date
- `after` — ISO timestamp, get logs after this date
- `filter` — regex filter on log lines

### Deployments

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v2/applications/{appId}/deployments` | List deployments |
| GET | `/v2/applications/{appId}/deployments/{deploymentId}` | Get deployment |

### Metrics

| Method | Path | Description |
|--------|------|-------------|
| GET | `/v4/metrics/organisations/{orgId}/applications/{appId}` | Get app metrics |

---

## Application Object

```json
{
  "id": "app_xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "name": "my-app",
  "description": "My web application",
  "zone": "par",
  "instance": {
    "type": "node",
    "version": "20",
    "minInstances": 1,
    "maxInstances": 2,
    "minFlavor": { "name": "XS", "cpus": 1, "mem": 256 }
  },
  "deployment": {
    "type": "git",
    "repoState": "master"
  },
  "state": "SHOULD_BE_UP",
  "vhosts": [{ "fqdn": "myapp.cleverapps.io" }]
}
```

`state` values: `SHOULD_BE_UP`, `WANTS_TO_BE_DOWN`, `MIGRATION_IN_PROGRESS`, `BOOTED`

## Deployment Object

```json
{
  "uuid": "deploy_xxx",
  "date": "2024-01-15T10:00:00.000Z",
  "action": "DEPLOY",
  "state": "OK",
  "cause": "HEAD commit pushed",
  "commit": "abc123def456..."
}
```

`action`: `DEPLOY`, `UNDEPLOY`, `RESTART`  
`state`: `OK`, `FAIL`, `CANCELLED`, `WIP`

---

## Common jq Recipes

```bash
# List app names and states
jq '.[] | {name, state, zone, url: .vhosts[0].fqdn}'

# Filter failed deployments
jq '.[] | select(.state=="FAIL") | {uuid, date, cause}'

# Get env var value
jq '.[] | select(.name=="DATABASE_URL") | .value'

# Logs: extract just the message text
jq '.[] | .message'
```

---

## Clever CLI Reference

Full docs: [developers.clever-cloud.com](https://developers.clever-cloud.com/doc/cli)

```bash
# Install
npm install -g clever-tools

# Auth
clever login                           # OAuth browser flow
clever profile                         # Current logged-in user

# App operations
clever applications list               # All apps across all orgs
clever status --app app_xxx            # App status
clever logs --app app_xxx              # Recent logs
clever logs --app app_xxx --follow     # Live log tail
clever restart --app app_xxx           # Restart
clever stop --app app_xxx             # Stop
clever scale --instances 2 --app app_xxx  # Scale

# Env
clever env --app app_xxx              # List all env vars
clever env set KEY value --app app_xxx
clever env rm KEY --app app_xxx

# Add-ons
clever addons --org $CLEVER_ORG_ID    # List add-ons
clever service link-addon $ADDON_ID --app $APP_ID

# Domains
clever domain --app app_xxx

# Deploy
clever link .                          # Link local folder to app
clever deploy                          # Deploy current HEAD
```
