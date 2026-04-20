# Odoo API Reference

Base URL: `$ODOO_URL`  
Auth: `-u "__api__:$ODOO_API_KEY"` on all requests (Odoo 16+).

## Endpoints

### `POST /web/session/authenticate`
Session login (username/password fallback).

```json
{ "jsonrpc":"2.0","method":"call","id":1,
  "params":{"db":"$ODOO_DB","login":"$ODOO_USER","password":"$ODOO_PASSWORD"} }
```
Returns `uid` (integer) on success, `false` on failure.

---

### `POST /web/dataset/call_kw`
Main entry point for all model operations.

```json
{
  "jsonrpc": "2.0", "method": "call", "id": 1,
  "params": {
    "model": "<model_name>",
    "method": "<orm_method>",
    "args": [...],
    "kwargs": { ... }
  }
}
```

#### ORM Methods

| Method | args | kwargs | Returns |
|--------|------|--------|---------|
| `search_read` | `[domain]` | `fields`, `limit`, `offset`, `order` | list of dicts |
| `search` | `[domain]` | `limit`, `offset`, `order` | list of IDs |
| `read` | `[ids]` | `fields` | list of dicts |
| `create` | `[{vals}]` | — | new ID |
| `write` | `[[ids], {vals}]` | — | `true` |
| `unlink` | `[[ids]]` | — | `true` |
| `fields_get` | `[]` | `attributes: ["string","type","required"]` | field metadata |

---

### `POST /web/dataset/call_button`
Trigger button/action methods (e.g. confirm a sale order).

```json
{ "jsonrpc":"2.0","method":"call","id":1,
  "params":{"model":"sale.order","method":"action_confirm","args":[[<id>]],"kwargs":{}} }
```

---

## Model Field Reference

### `res.partner` (Contacts)

```
id, name, email, phone, mobile, website
is_company (bool), company_id (Many2one res.partner)
street, city, zip, country_id (Many2one res.country)
vat, ref, active, customer_rank, supplier_rank
```

### `crm.lead` (Leads & Opportunities)

```
id, name, partner_id (Many2one res.partner)
type: "lead" | "opportunity"
stage_id (Many2one crm.stage), user_id (Many2one res.users, salesperson)
expected_revenue, probability (0-100)
email_from, phone, mobile
date_deadline, date_closed
tag_ids (Many2many crm.tag)
description, active
```

### `sale.order` (Devis / Sales Orders)

```
id, name (SO ref e.g. "S00042")
partner_id (Many2one res.partner)
state: "draft" | "sent" | "sale" | "done" | "cancel"
date_order, validity_date
amount_untaxed, amount_tax, amount_total
order_line (One2many sale.order.line)
user_id (salesperson), team_id (sales team)
```

### `sale.order.line`

```
id, order_id, product_id (Many2one product.product)
name (description), product_uom_qty, price_unit, price_subtotal
tax_id (Many2many account.tax)
```

### `account.move` (Invoices)

```
id, name (invoice ref e.g. "INV/2024/00042")
move_type: "out_invoice" | "out_refund" | "in_invoice" | "in_refund"
partner_id, invoice_date, invoice_date_due
amount_untaxed, amount_tax, amount_total
payment_state: "not_paid" | "partial" | "paid" | "reversed"
state: "draft" | "posted" | "cancel"
invoice_line_ids (One2many account.move.line)
```

### `project.project`

```
id, name, partner_id, user_id (manager)
date_start, date (deadline)
task_count, task_ids (One2many project.task)
```

### `project.task`

```
id, name, project_id, user_ids (assignees)
stage_id, priority: "0" (normal) | "1" (urgent)
date_deadline, date_assign
description, tag_ids
```

---

## Common Query Examples

### Get all companies in France
```bash
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"res.partner","method":"search_read",
    "args":[[["is_company","=",true],["country_id.code","=","FR"]]],"kwargs":{"fields":["name","email","phone","city"],"limit":50}}}'
```

### Get open opportunities sorted by revenue
```bash
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"crm.lead","method":"search_read",
    "args":[[["type","=","opportunity"],["active","=",true]]],"kwargs":{"fields":["name","partner_id","stage_id","expected_revenue","probability","user_id"],"limit":20,"order":"expected_revenue desc"}}}'
```

### Get unpaid invoices
```bash
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"account.move","method":"search_read",
    "args":[[["move_type","=","out_invoice"],["payment_state","=","not_paid"],["state","=","posted"]]],"kwargs":{"fields":["name","partner_id","amount_total","invoice_date_due"],"order":"invoice_date_due asc"}}}'
```

### Inspect available fields on any model
```bash
MODEL=crm.lead
curl -s "$ODOO_URL/web/dataset/call_kw" -H "Content-Type: application/json" -u "__api__:$ODOO_API_KEY" \
  -d '{"jsonrpc":"2.0","method":"call","id":1,"params":{"model":"'"$MODEL"'","method":"fields_get",
    "args":[],"kwargs":{"attributes":["string","type","required"]}}}' | jq 'to_entries[] | {key, label:.value.string, type:.value.type}'
```

---

## Error Codes

| Code | Meaning |
|------|---------|
| 200 | ORM / business logic error (check `data.message`) |
| 100 | Invalid JSON |
| 403 | Access denied (check user permissions) |
| 404 | Model not found |
