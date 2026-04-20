# Notion API Reference

Base URL: `https://api.notion.com/v1`  
Auth: `Authorization: Bearer $NOTION_API_KEY` + `Notion-Version: 2022-06-28` on all requests.

## Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/pages/{page_id}` | Get page metadata and properties |
| PATCH | `/pages/{page_id}` | Update page properties or archive |
| GET | `/pages/{page_id}/properties/{property_id}` | Get single property value |
| GET | `/databases/{database_id}` | Get database schema |
| POST | `/databases/{database_id}/query` | Query database rows |
| POST | `/pages` | Create new page (in DB or as child) |
| GET | `/blocks/{block_id}` | Get a block |
| PATCH | `/blocks/{block_id}` | Update a block |
| DELETE | `/blocks/{block_id}` | Delete a block |
| GET | `/blocks/{block_id}/children` | List child blocks |
| PATCH | `/blocks/{block_id}/children` | Append child blocks |
| POST | `/search` | Search pages and databases |

---

## Property Types (database columns)

| Type | Read path | Write shape |
|------|-----------|-------------|
| `title` | `.title[0].plain_text` | `{"title":[{"text":{"content":"..."}}]}` |
| `rich_text` | `.rich_text[0].plain_text` | `{"rich_text":[{"text":{"content":"..."}}]}` |
| `number` | `.number` | `{"number": 42}` |
| `select` | `.select.name` | `{"select":{"name":"Option"}}` |
| `multi_select` | `.multi_select[].name` | `{"multi_select":[{"name":"A"},{"name":"B"}]}` |
| `date` | `.date.start` | `{"date":{"start":"2024-01-15"}}` |
| `checkbox` | `.checkbox` | `{"checkbox": true}` |
| `url` | `.url` | `{"url":"https://..."}` |
| `email` | `.email` | `{"email":"user@example.com"}` |
| `phone_number` | `.phone_number` | `{"phone_number":"+33..."}` |
| `relation` | `.relation[].id` | `{"relation":[{"id":"page-id"}]}` |
| `people` | `.people[].name` | `{"people":[{"id":"user-id"}]}` |
| `formula` | `.formula.string` / `.formula.number` | read-only |
| `rollup` | `.rollup.array` / `.rollup.number` | read-only |
| `status` | `.status.name` | `{"status":{"name":"In Progress"}}` |

---

## Filter Syntax (database query)

Single filter:
```json
{ "filter": { "property": "Status", "select": { "equals": "Done" } } }
```

AND compound:
```json
{
  "filter": {
    "and": [
      { "property": "Status", "select": { "does_not_equal": "Done" } },
      { "property": "Priority", "select": { "equals": "High" } }
    ]
  }
}
```

OR compound:
```json
{
  "filter": {
    "or": [
      { "property": "Assignee", "people": { "contains": "user-id" } },
      { "property": "Tags", "multi_select": { "contains": "urgent" } }
    ]
  }
}
```

### Filter operators by type

| Type | Operators |
|------|-----------|
| text / rich_text / title | `equals`, `does_not_equal`, `contains`, `does_not_contain`, `starts_with`, `ends_with`, `is_empty`, `is_not_empty` |
| number | `equals`, `does_not_equal`, `greater_than`, `less_than`, `greater_than_or_equal_to`, `less_than_or_equal_to`, `is_empty`, `is_not_empty` |
| select | `equals`, `does_not_equal`, `is_empty`, `is_not_empty` |
| multi_select | `contains`, `does_not_contain`, `is_empty`, `is_not_empty` |
| checkbox | `equals` (true/false) |
| date | `equals`, `before`, `after`, `on_or_before`, `on_or_after`, `is_empty`, `is_not_empty`, `past_week`, `past_month`, `past_year`, `next_week`, `next_month`, `next_year` |
| relation | `contains`, `does_not_contain`, `is_empty`, `is_not_empty` |

---

## Sort Syntax

```json
{
  "sorts": [
    { "property": "Name", "direction": "ascending" },
    { "timestamp": "created_time", "direction": "descending" }
  ]
}
```

`direction`: `"ascending"` or `"descending"`  
`timestamp`: `"created_time"` or `"last_edited_time"`

---

## Block Types

| Type | Key | Notable fields |
|------|-----|----------------|
| `paragraph` | `paragraph` | `rich_text[]` |
| `heading_1` / `heading_2` / `heading_3` | `heading_X` | `rich_text[]` |
| `bulleted_list_item` | `bulleted_list_item` | `rich_text[]` |
| `numbered_list_item` | `numbered_list_item` | `rich_text[]` |
| `to_do` | `to_do` | `rich_text[]`, `checked` (bool) |
| `toggle` | `toggle` | `rich_text[]`, `children[]` |
| `code` | `code` | `rich_text[]`, `language` |
| `quote` | `quote` | `rich_text[]` |
| `divider` | `divider` | `{}` |
| `image` | `image` | `type: "external"`, `external.url` |
| `callout` | `callout` | `rich_text[]`, `icon` |
| `child_page` | `child_page` | `title` |
| `table` | `table` | `table_width`, `has_column_header` |

Rich text object:
```json
{ "type": "text", "text": { "content": "Hello", "link": null },
  "annotations": { "bold": false, "italic": false, "code": false, "color": "default" } }
```

---

## Common jq Recipes

```bash
# Extract all titles from a database query
jq '.results[] | .properties.Name.title[0].plain_text'

# Get id + title + select property
jq '.results[] | { id: .id, name: .properties.Name.title[0].plain_text, status: .properties.Status.select.name }'

# List all database property names and types
curl .../databases/$DB_ID | jq '.properties | to_entries[] | {name: .key, type: .value.type}'

# Get plain text from all paragraph blocks
curl .../blocks/$PAGE_ID/children | jq '.results[] | select(.type=="paragraph") | .paragraph.rich_text[].plain_text'
```
