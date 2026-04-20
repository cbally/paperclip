---
name: github
description: >
  Interact with GitHub via its REST API v3. Use this skill to manage
  repositories, issues, pull requests, commits, branches, releases, and GitHub
  Actions workflows. Triggers: "liste les issues GitHub", "crée une PR", "status
  des workflows", "récupère les commits récents", "cherche un repo", "clone",
  or any request involving GitHub data or operations.
---

# GitHub Skill

Interact with GitHub via the REST API v3. All calls use `$GITHUB_TOKEN`.

## Setup

```bash
source .env.integrations   # or: export GITHUB_TOKEN=ghp_xxx GITHUB_DEFAULT_OWNER=myorg
```

Required: `GITHUB_TOKEN` (fine-grained PAT or classic token).  
Scopes needed: `repo` (full), `workflow` (Actions), `read:org` (org data).

## Base Headers

```bash
GH_HEADERS=(-H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" -H "X-GitHub-Api-Version: 2022-11-28")
```

## Core Operations

### List repositories

```bash
# Your repos
curl -s "https://api.github.com/user/repos?sort=updated&per_page=20" "${GH_HEADERS[@]}" | jq '.[] | {name, full_name, language, updated_at, open_issues_count}'

# Org repos
curl -s "https://api.github.com/orgs/$GITHUB_DEFAULT_OWNER/repos?sort=updated&per_page=20" "${GH_HEADERS[@]}" | jq '.[] | {name, full_name}'
```

### List issues

```bash
OWNER=${GITHUB_DEFAULT_OWNER}
REPO=my-repo

curl -s "https://api.github.com/repos/$OWNER/$REPO/issues?state=open&per_page=20" "${GH_HEADERS[@]}" \
  | jq '.[] | {number, title, state, labels: [.labels[].name], assignee: .assignee.login}'
```

### Get a single issue

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/issues/$ISSUE_NUMBER" "${GH_HEADERS[@]}"
```

### Create an issue

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/issues" "${GH_HEADERS[@]}" \
  -X POST \
  -d '{
    "title": "Bug: something is broken",
    "body": "Description of the issue",
    "labels": ["bug"],
    "assignees": ["username"]
  }'
```

### Update/close an issue

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/issues/$ISSUE_NUMBER" "${GH_HEADERS[@]}" \
  -X PATCH \
  -d '{ "state": "closed", "state_reason": "completed" }'
```

### List pull requests

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/pulls?state=open&per_page=20" "${GH_HEADERS[@]}" \
  | jq '.[] | {number, title, user: .user.login, head: .head.ref, base: .base.ref}'
```

### Get recent commits

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/commits?per_page=10" "${GH_HEADERS[@]}" \
  | jq '.[] | {sha: .sha[0:8], message: .commit.message, author: .commit.author.name, date: .commit.author.date}'
```

### List branches

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/branches" "${GH_HEADERS[@]}" | jq '.[].name'
```

### GitHub Actions — list workflows

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/actions/workflows" "${GH_HEADERS[@]}" \
  | jq '.workflows[] | {id, name, state, path}'
```

### GitHub Actions — list runs for a workflow

```bash
WORKFLOW_ID=12345   # or filename e.g. "ci.yml"
curl -s "https://api.github.com/repos/$OWNER/$REPO/actions/workflows/$WORKFLOW_ID/runs?per_page=10" "${GH_HEADERS[@]}" \
  | jq '.workflow_runs[] | {id, status, conclusion, head_branch, created_at}'
```

### Trigger a workflow dispatch

```bash
curl -s "https://api.github.com/repos/$OWNER/$REPO/actions/workflows/$WORKFLOW_ID/dispatches" "${GH_HEADERS[@]}" \
  -X POST \
  -d '{ "ref": "main", "inputs": { "environment": "staging" } }'
```

### Search across GitHub

```bash
# Search issues/PRs
curl -s "https://api.github.com/search/issues?q=repo:$OWNER/$REPO+is:issue+is:open+label:bug&per_page=10" "${GH_HEADERS[@]}" \
  | jq '.items[] | {number, title}'

# Search code
curl -s "https://api.github.com/search/code?q=TODO+repo:$OWNER/$REPO&per_page=10" "${GH_HEADERS[@]}" \
  | jq '.items[] | {path: .path, url: .html_url}'
```

## Pagination

GitHub uses `Link` headers and also accepts `?page=N&per_page=100`:

```bash
PAGE=1
while true; do
  RESP=$(curl -si "https://api.github.com/repos/$OWNER/$REPO/issues?state=open&per_page=100&page=$PAGE" "${GH_HEADERS[@]}")
  echo "$RESP" | grep -v '^[a-z]' | jq '.[] | {number, title}'
  echo "$RESP" | grep '^link:' | grep -q 'rel="next"' || break
  PAGE=$((PAGE + 1))
done
```

## Rate Limits

```bash
curl -s "https://api.github.com/rate_limit" "${GH_HEADERS[@]}" | jq '.rate'
# { "limit": 5000, "used": 42, "remaining": 4958, "reset": 1716000000 }
```

## Reference

See [references/api-reference.md](references/api-reference.md) for full endpoint list and response schemas.
