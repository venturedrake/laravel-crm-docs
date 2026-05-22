# API · Quotes

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/crm/api/v2/quotes` | List (paginated). |
| `POST` | `/crm/api/v2/quotes` | Create. |
| `GET` | `/crm/api/v2/quotes/{uuid}` | Show. |
| `PUT` | `/crm/api/v2/quotes/{uuid}` | Update. |
| `DELETE` | `/crm/api/v2/quotes/{uuid}` | Soft-delete. |

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `title`, `total`. Prefix `-` for descending. |
| `user_owner_id` | Filter by owner (integer user ID). |

## Resource shape

```json
{
  "id": "uuid",
  "quote_id": "Q1001",
  "title": "Annual renewal quote",
  "description": null,
  "reference": "PO-9981",
  "currency": "USD",
  "issue_at":  "2026-05-01T00:00:00+00:00",
  "expire_at": "2026-06-01T00:00:00+00:00",
  "terms": null,
  "subtotal": 1000.00,
  "discount": 50.00,
  "tax": 95.00,
  "adjustments": 0.00,
  "total": 1045.00,
  "accepted_at": null,
  "rejected_at": null,
  "owner":        { "id": 1, "name": "Jane Doe" },
  "person":       { "id": "uuid", "name": "John Smith" },
  "organization": { "id": "uuid", "name": "Acme Co" },
  "labels":       [],
  "line_items":   [/* see "Nested line items" below */]
}
```

## Writable fields

Accepted on `POST` / `PUT` (`*` = required on `POST`):

`title*`, `description`, `reference`, `currency`, `issue_at`, `expire_at`, `terms`, `subtotal`, `discount`, `tax`, `adjustments`, `total`, `person_id` (UUID), `organization_id` (UUID), `lead_id` (UUID, `POST` only), `pipeline_stage_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs), `line_items[]`.

## Nested line items

`line_items` is accepted on `POST` and `PUT`.

**Output:**

```json
{
  "id": "uuid",
  "product_id": "uuid",
  "product":    { "id": "uuid", "name": "Annual subscription", "code": "SUB-1Y" },
  "quantity": 2,
  "unit_price": 100.00,
  "amount": 200.00,
  "tax_rate": 10.0,
  "tax_amount": 20.00,
  "currency": "USD",
  "comments": null,
  "order": 1
}
```

**Input:**

```json
{
  "id": "uuid (optional, update only)",
  "product_id": "uuid (required)",
  "quantity": 2,
  "unit_price": 100.00,
  "amount": 200.00,
  "comments": "Optional notes"
}
```

- **Create new line:** omit `id`. A new line is inserted.
- **Update in place:** include the existing line's `id` (UUID). The line is updated.
- **Replace lines:** on `PUT`, lines in the existing record that aren't matched by `id` are removed.
- `product_id`, `quantity`, `unit_price`, and `amount` are required on every line item.

