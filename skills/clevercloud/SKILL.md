---
name: clevercloud
description: >
  Interact with Clever Cloud via its REST API to retrieve application logs,
  monitor deployments, check metrics, and inspect infrastructure status. Use
  this skill whenever you need to: see application logs, check if a deployment
  succeeded, monitor resource usage, list apps or add-ons. Triggers: "logs
  Clever Cloud", "status de mon app", "dernier déploiement", "métriques
  application", "liste mes apps Clever Cloud".
---

# Clever Cloud Skill

Retrieve logs, deployments, and metrics from Clever Cloud via its REST API.

## Setup

```bash
source .env.integrations
# Required: CLEVER_TOKEN, CLEVER_SECRET
# Optional: CLEVER_ORG_ID (orga_xxx for org apps, empty for personal account)
```

### Getting your tokens

**Option A — From CLI (recommended):**
```bash
npm install -g clever-tools
clever login
# Tokens saved to ~/.config/clever-cloud/clever-tools.json
cat ~/.config/clever-cloud/clever-tools.json | jq '{token, secret}'
```

**Option B — From API:**
```bash
# OAuth1 flow — see references/api-reference.md for details
```

## Authentication — OAuth1

Clever Cloud uses **OAuth1** with HMAC-SHA256. The easiest approach is to use the `clever` CLI for auth-heavy operations and fall back to the REST API for reads.

For direct API calls, build the OAuth1 header:

```bash
# Using oauth1-signature tool (npm install -g oauth-1.0a-hmac-sha1-cli)
# Or use the clever CLI as a proxy where possible
```

**Shortcut: use `clever` CLI for most operations** — it handles OAuth automatically:

```bash
# List apps
clever applications list

# Get logs
clever logs --app <APP_ID> --lines 100

# Get recent deployments
clever deploy --list --app <APP_ID>

# SSH into app
clever ssh --app <APP_ID>
```

## REST API (direct)

For automation without the CLI, use these endpoints with your token.  
Base URL: `https://api.clever-cloud.com`

```bash
# The simplest auth approach for read operations:
# Use clever-tools CLI to generate a signed request, or use the token directly
# for endpoints that accept it.

CC_HEADERS=(-H "Authorization: Bearer $CLEVER_TOKEN")
```

> Note: Most Clever Cloud API endpoints require full OAuth1 signing. For simple reads,
> the `clever` CLI is more reliable. See references/api-reference.md for OAuth1 signing details.

### List your applications

```bash
# Personal account
curl -s "https://api.clever-cloud.com/v2/self/applications" "${CC_HEADERS[@]}" \
  | jq '.[] | {id, name, zone, instance: .instance.type, state}'

# Organisation
curl -s "https://api.clever-cloud.com/v2/organisations/$CLEVER_ORG_ID/applications" "${CC_HEADERS[@]}" \
  | jq '.[] | {id, name, zone, state}'
```

### Get application info

```bash
APP_ID=app_xxx
curl -s "https://api.clever-cloud.com/v2/self/applications/$APP_ID" "${CC_HEADERS[@]}"
```

### Get application logs

```bash
# Via CLI (recommended — handles real-time streaming)
clever logs --app $APP_ID --lines 200

# Via API (last N lines)
curl -s "https://api.clever-cloud.com/v2/logs/$APP_ID" "${CC_HEADERS[@]}" \
  | jq '.[] | {timestamp, message}'
```

### Get deployments

```bash
# Via CLI
clever deploy --list --app $APP_ID

# Via API
curl -s "https://api.clever-cloud.com/v2/applications/$APP_ID/deployments" "${CC_HEADERS[@]}" \
  | jq '.[] | {uuid, date, action, state, cause}' | head -20
```

### Get environment variables

```bash
clever env --app $APP_ID

# Or via API
curl -s "https://api.clever-cloud.com/v2/self/applications/$APP_ID/env" "${CC_HEADERS[@]}" \
  | jq '.[] | {name, value}'
```

### Set environment variable

```bash
clever env set KEY value --app $APP_ID

# Via API
curl -s "https://api.clever-cloud.com/v2/self/applications/$APP_ID/env" "${CC_HEADERS[@]}" \
  -X PUT \
  -H "Content-Type: application/json" \
  -d '[{"name":"MY_VAR","value":"my_value"}]'
```

### List add-ons

```bash
clever addons --app $APP_ID

# Via API
curl -s "https://api.clever-cloud.com/v2/self/addons" "${CC_HEADERS[@]}" \
  | jq '.[] | {id, name, plan: .plan.name, provider: .provider.id}'
```

### Restart an application

```bash
clever restart --app $APP_ID

# Via API
curl -s "https://api.clever-cloud.com/v2/self/applications/$APP_ID/instances" "${CC_HEADERS[@]}" \
  -X DELETE   # stops all instances, Clever Cloud restarts them
```

## Useful CLI Commands Quick Reference

```bash
# Auth
clever login
clever logout
clever profile

# App management
clever applications list
clever status --app $APP_ID
clever logs --app $APP_ID --lines 100
clever logs --app $APP_ID --follow          # tail -f style
clever deploy --app $APP_ID                 # deploy current git HEAD
clever restart --app $APP_ID
clever stop --app $APP_ID
clever scale --instances 2 --app $APP_ID

# Environment
clever env --app $APP_ID
clever env set KEY value --app $APP_ID
clever env rm KEY --app $APP_ID

# Add-ons
clever addons --app $APP_ID
clever addons create postgresql-addon --plan dev --link $APP_ID

# Domain
clever domain --app $APP_ID
```

## Reference

See [references/api-reference.md](references/api-reference.md) for full endpoint list, OAuth1 signing details, and response schemas.
