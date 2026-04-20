---
name: odoo
description: >
  Interact with the Odoo CRM and ERP via its JSON-RPC 2.0 API. Use this skill
  whenever you need to read or write Odoo data: contacts, leads/opportunities,
  sales orders, invoices, projects, or any other Odoo model. Triggers: "liste
  les contacts Odoo", "crée un lead", "mets à jour l'opportunité", "cherche dans
  Odoo", "récupère les devis", or any request involving Odoo data.
---

# Odoo Skill

Interact with Odoo via JSON-RPC 2.0. All calls go to `$ODOO_URL`.

## Setup

```bash
source .env.integrations   # or: export ODOO_URL=... ODOO_DB=... ODOO_API_KEY=...
```

Required variables: `ODOO_URL`, `ODOO_DB`, and either `ODOO_API_KEY` or `ODOO_USER`+`ODOO_PASSWORD`.

## Authentication

### Preferred: API Key (Odoo 16+)

Use the API key directly as the password with user `__api__` — no session needed:

```bash
curl -s "$ODOO_URL/web/dataset/call_kw" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", "method": "call", "id": 1,
    "params": {
      "model": "res.partner",
      "method": "search_read",
      "args": [[["is_company", "=", true]]],
      "kwargs": {
        "fields": ["name", "email", "phone"],
        "limit": 10,
        "context": {"db": "'"$ODOO_DB"'"}
      }
    }
  }' \
  -u "__api__:$ODOO_API_KEY"
```

### Fallback: Username + Password (session cookie)

```bash
# 1. Authenticate and get session cookie
SESSION=$(curl -sc /tmp/odoo_cookie "$ODOO_URL/web/session/authenticate" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"db":"'"$ODOO_DB"'","login":"'"$ODOO_USER"'","password":"'"$ODOO_PASSWORD"'"}}')

# 2. Use cookie for subsequent calls
curl -sb /tmp/odoo_cookie "$ODOO_URL/web/dataset/call_kw" \
  -H "Content-Type: application/json" \
  -d '{...}'
```

## Core Pattern: call_kw

Every Odoo model operation follows the same structure:

```bash
curl -s "$ODOO_URL/web/dataset/call_kw" \
  -H "Content-Type: application/json" \
  -u "__api__:$ODOO_API_KEY" \
  -d '{
    "jsonrpc": "2.0", "method": "call", "id": 1,
    "params": {
      "model": "<MODEL>",
      "method": "<METHOD>",
      "args": [<DOMAIN_OR_IDS>],
      "kwargs": { <KWARGS> }
    }
  }'
```

## Common Operations

### Search and read records

```bash
# search_read: filter + fetch fields in one call
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"crm.lead","method":"search_read",
    "args":[[["stage_id.name","=","New"]]],"kwargs":{"fields":["name","partner_id","expected_revenue","probability"],"limit":20,"order":"expected_revenue desc"}}}'
```

### Create a record

```bash
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"crm.lead","method":"create",
    "args":[{"name":"New opportunity","partner_id":42,"expected_revenue":5000}],"kwargs":{}}}'
# Returns: {"result": <new_id>}
```

### Update a record

```bash
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"crm.lead","method":"write",
    "args":[[<ID>],{"probability":80,"expected_revenue":8000}],"kwargs":{}}}'
# Returns: {"result": true}
```

### Delete a record

```bash
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"crm.lead","method":"unlink",
    "args":[[<ID>]],"kwargs":{}}}'
```

## Domain Syntax (filters)

Domains are lists of triples `[field, operator, value]` combined with `&` (AND) or `|` (OR):

```json
[["is_company","=",true], ["country_id.code","=","FR"]]           // AND (default)
["|", ["email","ilike","@gmail"], ["email","ilike","@yahoo"]]      // OR
[["active","=",true], ["stage_id.probability",">=",50]]
```

Operators: `=`, `!=`, `>`, `>=`, `<`, `<=`, `like`, `ilike`, `in`, `not in`, `child_of`

## Key Models

| Model | Usage |
|-------|-------|
| `res.partner` | Contacts, clients, suppliers |
| `crm.lead` | Leads and opportunities |
| `sale.order` | Sales orders / devis |
| `account.move` | Invoices (type=`out_invoice`) |
| `project.project` | Projects |
| `project.task` | Tasks |
| `hr.employee` | Employees |
| `product.product` | Products |

## Parsing Responses

Odoo always returns `{"jsonrpc":"2.0","id":1,"result": <data>}`.  
On error: `{"jsonrpc":"2.0","id":1,"error":{"code":200,"message":"...","data":{"message":"..."}}}`.

Extract result with jq:
```bash
curl ... | jq '.result'
curl ... | jq '.result[] | {id, name, email}'
```

## Reference

See [references/api-reference.md](references/api-reference.md) for detailed endpoint docs and model field lists.
