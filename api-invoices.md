# API · Invoices

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/crm/api/v2/invoices` | List (paginated). |
| `POST` | `/crm/api/v2/invoices` | Create. |
| `GET` | `/crm/api/v2/invoices/{uuid}` | Show. |
| `PUT` | `/crm/api/v2/invoices/{uuid}` | Update. |
| `DELETE` | `/crm/api/v2/invoices/{uuid}` | Soft-delete. |

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `total`, `issue_date`, `due_date`. Prefix `-` for descending. |
| `user_owner_id` | Filter by owner (integer user ID). |

## Resource shape

```json
{
  "id": "uuid",
  "invoice_id": "I1001",
  "reference": null,
  "issue_date": "2026-05-01T00:00:00+00:00",
  "due_date":   "2026-05-31T00:00:00+00:00",
  "currency": "USD",
  "terms": null,
  "subtotal": 1000.00,
  "tax": 100.00,
  "total": 1100.00,
  "amount_due": 1100.00,
  "amount_paid": 0.00,
  "fully_paid_at": null,
  "owner":        { "id": 1, "name": "Jane Doe" },
  "person":       { "id": "uuid", "name": "John Smith" },
  "organization": { "id": "uuid", "name": "Acme Co" },
  "labels":       [],
  "line_items":   [/* see "Nested line items" below */]
}
```

## Writable fields

Accepted on `POST` / `PUT`:

`reference`, `issue_date`, `due_date`, `currency`, `terms`, `subtotal`, `tax`, `total`, `person_id` (UUID), `organization_id` (UUID), `order_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs), `line_items[]`.

## Nested line items

Same shape and rules as [Quotes → Nested line items](/api-quotes#nested-line-items).

