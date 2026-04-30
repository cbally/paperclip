---
name: odoo
description: >
  Interact with the Odoo CRM and ERP via its XML-RPC API (Python). Use this skill
  whenever you need to read or write Odoo data: contacts, leads/opportunities,
  sales orders, invoices, projects, or any other Odoo model. Triggers: "liste
  les contacts Odoo", "crée un lead", "mets à jour l'opportunité", "cherche dans
  Odoo", "récupère les devis", or any request involving Odoo data.
  IMPORTANT: Always use Python + xmlrpc.client. Never use curl or JSON-RPC HTTP calls.
---

# Odoo Skill — XML-RPC (Python)

**Always use Python with `xmlrpc.client`. Never use curl, requests, or `/web/dataset/call_kw`.**

## Setup

```python
import xmlrpc.client, os

ODOO_URL      = os.environ["ODOO_URL"]       # e.g. https://myodoo.com
ODOO_DB       = os.environ["ODOO_DB"]
ODOO_USERNAME = os.environ["ODOO_USERNAME"]
ODOO_PASSWORD = os.environ["ODOO_PASSWORD"]
```

## Authentication

```python
common = xmlrpc.client.ServerProxy(f"{ODOO_URL}/xmlrpc/2/common")
uid    = common.authenticate(ODOO_DB, ODOO_USERNAME, ODOO_PASSWORD, {})
if not uid:
    raise PermissionError("Authentification Odoo échouée.")

models = xmlrpc.client.ServerProxy(f"{ODOO_URL}/xmlrpc/2/object")
```

## Core Helper

```python
def x(model, method, domain_or_ids, **kwargs):
    """Wrapper universel pour tous les appels ORM Odoo."""
    return models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD, model, method, [domain_or_ids], kwargs)
```

## Common Operations

### Search and read records

```python
leads = x("crm.lead", "search_read",
    [["active", "=", True], ["type", "=", "opportunity"]],
    fields=["name", "stage_id", "expected_revenue", "probability",
            "date_deadline", "partner_name", "user_id", "date_open"],
    limit=100,
    order="expected_revenue desc"
)
```

### Search (IDs only)

```python
ids = x("crm.lead", "search",
    [["type", "=", "opportunity"], ["active", "=", True]],
    limit=50
)
```

### Read specific records by ID

```python
records = x("crm.lead", "read",
    [1, 2, 3],
    fields=["name", "stage_id", "expected_revenue"]
)
```

### Create a record

```python
new_id = x("crm.lead", "create",
    {"name": "Nouvelle opportunité", "partner_name": "Acme", "expected_revenue": 5000},
)
# Returns: integer (new record ID)
```

### Update a record

```python
success = x("crm.lead", "write",
    [[42], {"probability": 80, "expected_revenue": 8000}]
)
# Returns: True
```

### Delete a record

```python
success = x("crm.lead", "unlink", [[42]])
```

### Inspect available fields on any model

```python
fields_info = models.execute_kw(
    ODOO_DB, uid, ODOO_PASSWORD,
    "crm.lead", "fields_get", [],
    {"attributes": ["string", "type", "required"]}
)
available_fields = list(fields_info.keys())
print(available_fields)
```

## Domain Syntax (filters)

Domains are lists of triples `[field, operator, value]`, AND by default, OR with `"|"` prefix:

```python
[["is_company", "=", True], ["country_id.code", "=", "FR"]]        # AND (default)
["|", ["email", "ilike", "@gmail"], ["email", "ilike", "@yahoo"]]   # OR
[["active", "=", True], ["probability", ">=", 50]]
[["active", "in", [True, False]]]                                   # include archived
```

Operators: `=`, `!=`, `>`, `>=`, `<`, `<=`, `like`, `ilike`, `in`, `not in`, `child_of`

## Defensive Field Handling

**Always filter fields against available ones before a search_read** if there is any doubt
(especially on Odoo 17+/19 where field names may differ):

```python
wanted_fields = ["name", "stage_id", "expected_revenue", "planned_revenue", "date_deadline"]
available     = set(models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD, "crm.lead", "fields_get", [],
                    {"attributes": ["string"]}).keys())
fields        = [f for f in wanted_fields if f in available]
missing       = [f for f in wanted_fields if f not in available]
if missing:
    print(f"[info] Champs absents : {missing}")
```

## Key Models

| Model | Usage |
|-------|-------|
| `res.partner` | Contacts, clients, fournisseurs |
| `crm.lead` | Leads et opportunités |
| `crm.stage` | Étapes du pipeline CRM |
| `mail.activity` | Activités planifiées (relances, appels…) |
| `sale.order` | Devis / commandes |
| `account.move` | Factures (`move_type="out_invoice"`) |
| `project.project` | Projets |
| `project.task` | Tâches |
| `res.users` | Utilisateurs |

## Reference

See [references/api-reference.md](references/api-reference.md) for model field lists and query examples.