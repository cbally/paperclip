---
name: notion
description: >
  Interact with Notion via its REST API v1. Use this skill to read pages, query
  databases, create or update content in Notion workspaces. Triggers: "liste les
  pages Notion", "cherche dans ma base Notion", "crée une page", "mets à jour
  une entrée Notion", "récupère les tâches Notion", or any request involving
  Notion content.
---

# Notion Skill

Interact with Notion via the REST API. All requests use `NOTION_API_KEY`.

## Rules

- ALWAYS include `children` when creating a page — never create empty pages
- To write a full article, use `children` with heading_1, heading_2, paragraph, bulleted_list_item blocks
- Never use `jq` to filter curl responses during creation — log the raw response to verify success
- If a database ID is in the issue URL (notion.so/xxx/DATABASE_ID?v=...), extract it directly without searching
- Write JSON payloads to a temp file when the body is large, then pass with `-d @/tmp/notion_payload.json`

## Setup

```bash
source .env.integrations   # or: export NOTION_API_KEY=secret_xxx
```

Required: `NOTION_API_KEY`.  
Get one at [notion.so/my-integrations](https://www.notion.so/my-integrations) → create integration → copy "Internal Integration Secret".  
Then share your database/page with the integration (Share → Invite → select your integration).

## Base Headers

Every request needs:
```
Authorization: Bearer $NOTION_API_KEY
Notion-Version: 2022-06-28
Content-Type: application/json
```

Shorthand for curl:
```bash
NOTION_HEADERS=(-H "Authorization: Bearer $NOTION_API_KEY" -H "Notion-Version: 2022-06-28" -H "Content-Type: application/json")
```

## Core Operations

### Get a page

```bash
curl -s "https://api.notion.com/v1/pages/$PAGE_ID" "${NOTION_HEADERS[@]}"
```

### Get a database

```bash
curl -s "https://api.notion.com/v1/databases/$DATABASE_ID" "${NOTION_HEADERS[@]}"
```

### Query a database

```bash
curl -s "https://api.notion.com/v1/databases/$DATABASE_ID/query" "${NOTION_HEADERS[@]}" \
  -X POST \
  -d '{
    "filter": {
      "property": "Status",
      "select": { "equals": "In Progress" }
    },
    "sorts": [{ "property": "Created", "direction": "descending" }],
    "page_size": 20
  }'
```

### Create a page in a database

```bash
curl -s "https://api.notion.com/v1/pages" "${NOTION_HEADERS[@]}" \
  -X POST \
  -d '{
    "parent": { "database_id": "'"$DATABASE_ID"'" },
    "properties": {
      "Name": { "title": [{ "text": { "content": "My new entry" } }] },
      "Status": { "select": { "name": "Todo" } },
      "Tags": { "multi_select": [{ "name": "urgent" }] }
    }
  }'
```

### Create a page with content (properties + blocks)

**Always use this form — never create pages without `children`.**

For large payloads, write to a file first:

```bash
cat > /tmp/notion_payload.json << 'EOF'
{
  "parent": { "database_id": "DATABASE_ID_HERE" },
  "properties": {
    "Name": { "title": [{ "text": { "content": "Mon article" } }] },
    "Status": { "select": { "name": "Draft" } }
  },
  "children": [
    {
      "object": "block",
      "type": "heading_1",
      "heading_1": {
        "rich_text": [{ "type": "text", "text": { "content": "Titre principal" } }]
      }
    },
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": {
        "rich_text": [{ "type": "text", "text": { "content": "Introduction" } }]
      }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": {
        "rich_text": [{ "type": "text", "text": { "content": "Contenu de l'introduction ici." } }]
      }
    },
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": {
        "rich_text": [{ "type": "text", "text": { "content": "Points clés" } }]
      }
    },
    {
      "object": "block",
      "type": "bulleted_list_item",
      "bulleted_list_item": {
        "rich_text": [{ "type": "text", "text": { "content": "Premier point" } }]
      }
    },
    {
      "object": "block",
      "type": "bulleted_list_item",
      "bulleted_list_item": {
        "rich_text": [{ "type": "text", "text": { "content": "Deuxième point" } }]
      }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": {
        "rich_text": [{ "type": "text", "text": { "content": "Conclusion ici." } }]
      }
    }
  ]
}
EOF

curl -s "https://api.notion.com/v1/pages" "${NOTION_HEADERS[@]}" \
  -X POST \
  -d @/tmp/notion_payload.json
```

After creation, always verify the page is not empty:

```bash
PAGE_ID="<id from response>"
curl -s "https://api.notion.com/v1/blocks/$PAGE_ID/children" "${NOTION_HEADERS[@]}" | jq '.results | length'
# Must be > 1, otherwise the page is empty — use "Append blocks" to fix it
```

### Update page properties

```bash
curl -s "https://api.notion.com/v1/pages/$PAGE_ID" "${NOTION_HEADERS[@]}" \
  -X PATCH \
  -d '{ "properties": { "Status": { "select": { "name": "Done" } } } }'
```

### Get page blocks (content)

```bash
curl -s "https://api.notion.com/v1/blocks/$PAGE_ID/children" "${NOTION_HEADERS[@]}"
```

### Append blocks to a page

```bash
curl -s "https://api.notion.com/v1/blocks/$PAGE_ID/children" "${NOTION_HEADERS[@]}" \
  -X PATCH \
  -d '{
    "children": [
      {
        "object": "block",
        "type": "paragraph",
        "paragraph": {
          "rich_text": [{ "type": "text", "text": { "content": "New paragraph added." } }]
        }
      }
    ]
  }'
```

### Search across workspace

```bash
curl -s "https://api.notion.com/v1/search" "${NOTION_HEADERS[@]}" \
  -X POST \
  -d '{ "query": "project roadmap", "filter": { "value": "page", "property": "object" }, "page_size": 10 }'
```

## Pagination

Responses include `has_more` and `next_cursor`. Loop to fetch all results:

```bash
CURSOR=""
while true; do
  BODY='{"page_size":100}'
  [ -n "$CURSOR" ] && BODY='{"page_size":100,"start_cursor":"'"$CURSOR"'"}'
  RESP=$(curl -s "https://api.notion.com/v1/databases/$DATABASE_ID/query" "${NOTION_HEADERS[@]}" -X POST -d "$BODY")
  echo "$RESP" | jq '.results[]'
  HAS_MORE=$(echo "$RESP" | jq -r '.has_more')
  [ "$HAS_MORE" != "true" ] && break
  CURSOR=$(echo "$RESP" | jq -r '.next_cursor')
done
```

## Parsing Responses

```bash
# List all page titles from a database query
curl ... | jq '.results[] | .properties.Name.title[0].plain_text'

# Get all text content from a page's blocks
curl ... | jq '.results[] | select(.type=="paragraph") | .paragraph.rich_text[].plain_text'
```

## Reference

See [references/api-reference.md](references/api-reference.md) for property types, filter syntax, and block types.
