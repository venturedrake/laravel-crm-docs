# API · Orders

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/crm/api/v2/orders` | List (paginated). |
| `POST` | `/crm/api/v2/orders` | Create. |
| `GET` | `/crm/api/v2/orders/{uuid}` | Show. |
| `PUT` | `/crm/api/v2/orders/{uuid}` | Update. |
| `DELETE` | `/crm/api/v2/orders/{uuid}` | Soft-delete. |

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `total`. Prefix `-` for descending. |
| `user_owner_id` | Filter by owner (integer user ID). |

## Resource shape

```json
{
  "id": "uuid",
  "order_id": "O1001",
  "reference": null,
  "description": null,
  "currency": "USD",
  "terms": null,
  "subtotal": 1000.00,
  "discount": 0.00,
  "tax": 100.00,
  "adjustments": 0.00,
  "total": 1100.00,
  "owner":        { "id": 1, "name": "Jane Doe" },
  "person":       { "id": "uuid", "name": "John Smith" },
  "organization": { "id": "uuid", "name": "Acme Co" },
  "labels":       [],
  "line_items":   [/* see "Nested line items" below */]
}
```

## Writable fields

Accepted on `POST` / `PUT`:

`reference`, `description`, `currency`, `terms`, `discount`, `tax`, `adjustments`, `person_id` (UUID), `organization_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs), `line_items[]`.

Accepted on `POST` only — the source links are set at creation and cannot be re-pointed:

`lead_id` (UUID), `deal_id` (UUID), `quote_id` (UUID).

> **Important:** `subtotal` and `total` are **not** writable. Both are computed from `line_items`, `discount`, `tax` and `adjustments`, and sending either is a `422`. They are still returned in the response shape above. `discount` and `tax` reject negative values — see [API → Conventions](/api#conventions).

## Nested line items

Same shape and rules as [Quotes → Nested line items](/api-quotes#nested-line-items).

