# Odoo API Reference — XML-RPC (Python)

Base URL: `$ODOO_URL`
Protocol: **XML-RPC** via `xmlrpc.client` (Python stdlib).
Never use curl or `/web/dataset/call_kw` — XML-RPC is the only working interface.

---

## Endpoints

| Endpoint | Usage |
|----------|-------|
| `$ODOO_URL/xmlrpc/2/common` | Auth + version info |
| `$ODOO_URL/xmlrpc/2/object` | All model operations (search, read, write…) |

---

## Authentication

```python
import xmlrpc.client, os

ODOO_URL      = os.environ["ODOO_URL"]
ODOO_DB       = os.environ["ODOO_DB"]
ODOO_USERNAME = os.environ["ODOO_USERNAME"]
ODOO_PASSWORD = os.environ["ODOO_PASSWORD"]

common = xmlrpc.client.ServerProxy(f"{ODOO_URL}/xmlrpc/2/common")
uid    = common.authenticate(ODOO_DB, ODOO_USERNAME, ODOO_PASSWORD, {})
# uid is an integer on success, 0/False on failure
models = xmlrpc.client.ServerProxy(f"{ODOO_URL}/xmlrpc/2/object")
```

---

## ORM Methods via `execute_kw`

```python
models.execute_kw(db, uid, password, model, method, args, kwargs)
```

| Method | args | kwargs | Returns |
|--------|------|--------|---------|
| `search_read` | `[domain]` | `fields`, `limit`, `offset`, `order` | list of dicts |
| `search` | `[domain]` | `limit`, `offset`, `order` | list of IDs |
| `read` | `[ids]` | `fields` | list of dicts |
| `create` | `[{vals}]` | — | new ID (int) |
| `write` | `[[ids], {vals}]` | — | `True` |
| `unlink` | `[[ids]]` | — | `True` |
| `fields_get` | `[]` | `attributes: ["string","type","required"]` | field metadata dict |

---

## Model Field Reference

### `crm.lead` (Leads & Opportunités)

```
id, name, partner_name, contact_name
email_from, phone, mobile
type: "lead" | "opportunity"
stage_id (Many2one crm.stage)
user_id (Many2one res.users — commercial assigné)
team_id (Many2one crm.team)
expected_revenue, planned_revenue, probability (0–100)
priority: "0" (normal) | "1" (bon) | "2" (très bon) | "3" (excellent)
kanban_state: "normal" | "done" | "blocked"
date_deadline, date_open, date_closed
description, tag_ids (Many2many crm.tag)
active (False = archivé)
city, country_id, company_id
```

### `crm.stage`

```
id, name, sequence, is_won, probability
```

### `mail.activity` (Activités)

```
id, res_id, res_model
summary, note
activity_type_id (Many2one mail.activity.type)
date_deadline, user_id, state: "overdue" | "today" | "planned"
```

### `res.partner` (Contacts)

```
id, name, email, phone, mobile, website
is_company (bool), company_id (Many2one res.partner)
street, city, zip, country_id
vat, ref, active, customer_rank, supplier_rank
```

### `sale.order` (Devis / Commandes)

```
id, name (ex: "S00042")
partner_id (Many2one res.partner)
state: "draft" | "sent" | "sale" | "done" | "cancel"
date_order, validity_date
amount_untaxed, amount_tax, amount_total
order_line (One2many sale.order.line)
user_id, team_id
```

### `account.move` (Factures)

```
id, name (ex: "INV/2024/00042")
move_type: "out_invoice" | "out_refund" | "in_invoice" | "in_refund"
partner_id, invoice_date, invoice_date_due
amount_untaxed, amount_tax, amount_total
payment_state: "not_paid" | "partial" | "paid" | "reversed"
state: "draft" | "posted" | "cancel"
invoice_line_ids (One2many account.move.line)
```

### `project.project`

```
id, name, partner_id, user_id
date_start, date (deadline)
task_count, task_ids
```

### `project.task`

```
id, name, project_id, user_ids
stage_id, priority: "0" | "1"
date_deadline, date_assign
description, tag_ids
```

---

## Query Examples

### Toutes les opportunités actives triées par revenu

```python
opps = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "crm.lead", "search_read",
    [[["type", "=", "opportunity"], ["active", "=", True]]],
    {"fields": ["name", "partner_name", "stage_id", "expected_revenue",
                "probability", "user_id", "date_deadline"],
     "order": "expected_revenue desc", "limit": 100}
)
```

### Deals bloqués (aucune activité depuis 14 jours)

```python
from datetime import datetime, timedelta
cutoff = (datetime.now() - timedelta(days=14)).strftime("%Y-%m-%d %H:%M:%S")

# Récupère les IDs d'opps avec activité récente
active_ids = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "mail.activity", "search",
    [[["res_model", "=", "crm.lead"], ["date_deadline", ">=", cutoff]]],
    {}
)
active_lead_ids = [a["res_id"] for a in models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "mail.activity", "read", [active_ids], {"fields": ["res_id"]}
)]

# Opps sans activité récente
blocked = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "crm.lead", "search_read",
    [[["type", "=", "opportunity"], ["active", "=", True],
      ["id", "not in", active_lead_ids]]],
    {"fields": ["name", "partner_name", "stage_id", "expected_revenue", "date_deadline"],
     "order": "date_deadline asc"}
)
```

### Deals créés cette semaine

```python
from datetime import datetime, timedelta
monday = (datetime.now() - timedelta(days=datetime.now().weekday())).strftime("%Y-%m-%d")

new_this_week = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "crm.lead", "search_read",
    [[["type", "=", "opportunity"], ["create_date", ">=", monday]]],
    {"fields": ["name", "partner_name", "expected_revenue", "stage_id", "user_id"]}
)
```

### Deals gagnés / perdus cette semaine

```python
won = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "crm.lead", "search_read",
    [[["active", "=", True], ["date_closed", ">=", monday],
      ["stage_id.is_won", "=", True]]],
    {"fields": ["name", "partner_name", "expected_revenue", "date_closed"]}
)

lost = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "crm.lead", "search_read",
    [[["active", "=", False], ["date_closed", ">=", monday]]],
    {"fields": ["name", "partner_name", "expected_revenue", "date_closed"]}
)
```

### Étapes du pipeline

```python
stages = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "crm.stage", "search_read",
    [[]],
    {"fields": ["id", "name", "sequence", "is_won"], "order": "sequence asc"}
)
```

### Inspecter les champs disponibles

```python
fields_info = models.execute_kw(ODOO_DB, uid, ODOO_PASSWORD,
    "crm.lead", "fields_get", [],
    {"attributes": ["string", "type", "required"]}
)
for fname, finfo in sorted(fields_info.items()):
    print(f"{fname:40} {finfo['type']:15} {finfo.get('string','')}")
```

---

## Error Handling

```python
try:
    result = models.execute_kw(...)
except xmlrpc.client.Fault as e:
    print(f"Odoo error {e.faultCode}: {e.faultString}")
except Exception as e:
    print(f"Connection error: {e}")
```

---

## Notes Odoo 17+/19

- `planned_revenue` peut être absent → toujours filtrer avec `fields_get` si doute
- Les deals perdus ont `active = False` → toujours inclure `["active", "in", [True, False]]` si nécessaire
- `date_closed` est renseigné sur les deals gagnés ET perdus
- `stage_id.is_won` = True identifie les étapes "Gagné"