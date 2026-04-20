# GitHub API Reference

Base URL: `https://api.github.com`  
Auth: `Authorization: Bearer $GITHUB_TOKEN` + `Accept: application/vnd.github+json` + `X-GitHub-Api-Version: 2022-11-28`

## Key Endpoints

### Repositories

| Method | Path | Description |
|--------|------|-------------|
| GET | `/user/repos` | List authenticated user's repos |
| GET | `/orgs/{org}/repos` | List org repos |
| GET | `/repos/{owner}/{repo}` | Get repo details |
| GET | `/repos/{owner}/{repo}/contents/{path}` | Get file content (base64) |
| PUT | `/repos/{owner}/{repo}/contents/{path}` | Create or update file |
| DELETE | `/repos/{owner}/{repo}/contents/{path}` | Delete file |

### Issues

| Method | Path | Description |
|--------|------|-------------|
| GET | `/repos/{owner}/{repo}/issues` | List issues (`?state=open/closed/all`) |
| GET | `/repos/{owner}/{repo}/issues/{number}` | Get issue |
| POST | `/repos/{owner}/{repo}/issues` | Create issue |
| PATCH | `/repos/{owner}/{repo}/issues/{number}` | Update/close issue |
| GET | `/repos/{owner}/{repo}/issues/{number}/comments` | List comments |
| POST | `/repos/{owner}/{repo}/issues/{number}/comments` | Add comment |
| POST | `/repos/{owner}/{repo}/issues/{number}/labels` | Add labels |

### Pull Requests

| Method | Path | Description |
|--------|------|-------------|
| GET | `/repos/{owner}/{repo}/pulls` | List PRs |
| GET | `/repos/{owner}/{repo}/pulls/{number}` | Get PR |
| POST | `/repos/{owner}/{repo}/pulls` | Create PR |
| PATCH | `/repos/{owner}/{repo}/pulls/{number}` | Update PR |
| PUT | `/repos/{owner}/{repo}/pulls/{number}/merge` | Merge PR |
| GET | `/repos/{owner}/{repo}/pulls/{number}/reviews` | List reviews |
| GET | `/repos/{owner}/{repo}/pulls/{number}/files` | List changed files |

### Commits & Branches

| Method | Path | Description |
|--------|------|-------------|
| GET | `/repos/{owner}/{repo}/commits` | List commits (`?sha=branch&per_page=N`) |
| GET | `/repos/{owner}/{repo}/commits/{sha}` | Get commit details |
| GET | `/repos/{owner}/{repo}/branches` | List branches |
| GET | `/repos/{owner}/{repo}/branches/{branch}` | Get branch |
| POST | `/repos/{owner}/{repo}/git/refs` | Create branch |

### GitHub Actions

| Method | Path | Description |
|--------|------|-------------|
| GET | `/repos/{owner}/{repo}/actions/workflows` | List workflows |
| GET | `/repos/{owner}/{repo}/actions/workflows/{id}/runs` | List runs |
| GET | `/repos/{owner}/{repo}/actions/runs/{run_id}` | Get run details |
| POST | `/repos/{owner}/{repo}/actions/workflows/{id}/dispatches` | Trigger dispatch |
| POST | `/repos/{owner}/{repo}/actions/runs/{run_id}/rerun` | Re-run failed jobs |
| GET | `/repos/{owner}/{repo}/actions/runs/{run_id}/jobs` | List jobs |
| GET | `/repos/{owner}/{repo}/actions/runs/{run_id}/logs` | Download logs (zip URL) |

### Releases

| Method | Path | Description |
|--------|------|-------------|
| GET | `/repos/{owner}/{repo}/releases` | List releases |
| GET | `/repos/{owner}/{repo}/releases/latest` | Get latest release |
| POST | `/repos/{owner}/{repo}/releases` | Create release |

### Search

| Method | Path | Description |
|--------|------|-------------|
| GET | `/search/repositories` | Search repos (`?q=...`) |
| GET | `/search/issues` | Search issues/PRs |
| GET | `/search/code` | Search code |
| GET | `/search/commits` | Search commits |

Search qualifiers: `repo:owner/repo`, `is:issue`, `is:pr`, `is:open`, `label:bug`, `author:user`, `language:python`, `created:>2024-01-01`

---

## Issue Object

```json
{
  "number": 42,
  "title": "Fix login bug",
  "state": "open",
  "body": "Description...",
  "user": { "login": "username" },
  "labels": [{ "name": "bug", "color": "ee0701" }],
  "assignees": [{ "login": "username" }],
  "milestone": { "title": "v1.0" },
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-16T12:00:00Z",
  "closed_at": null,
  "html_url": "https://github.com/owner/repo/issues/42"
}
```

## Pull Request Object

```json
{
  "number": 15,
  "title": "Add feature X",
  "state": "open",
  "draft": false,
  "user": { "login": "username" },
  "head": { "ref": "feature/x", "sha": "abc123" },
  "base": { "ref": "main", "sha": "def456" },
  "mergeable": true,
  "merged": false,
  "review_decision": "APPROVED",
  "checks_url": "...",
  "html_url": "https://github.com/owner/repo/pull/15"
}
```

## Workflow Run Object

```json
{
  "id": 123456789,
  "name": "CI",
  "status": "completed",
  "conclusion": "success",
  "head_branch": "main",
  "head_sha": "abc123",
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-15T10:05:00Z",
  "html_url": "https://github.com/owner/repo/actions/runs/123456789"
}
```

`status`: `queued`, `in_progress`, `completed`  
`conclusion`: `success`, `failure`, `cancelled`, `skipped`, `neutral`, `timed_out`, `action_required`

---

## Common jq Recipes

```bash
# Issues: number + title + labels
jq '.[] | {number, title, labels: [.labels[].name]}'

# PRs: branch info + review status
jq '.[] | {number, title, from: .head.ref, into: .base.ref, draft}'

# Commits: short sha + message first line
jq '.[] | {sha: .sha[0:8], msg: (.commit.message | split("\n")[0]), author: .commit.author.name}'

# Workflow runs: latest status per workflow
jq 'group_by(.name) | map({workflow: .[0].name, latest: .[0] | {status, conclusion, created_at}})'

# Get file content (decode base64)
curl .../contents/README.md | jq -r '.content' | base64 --decode
```
