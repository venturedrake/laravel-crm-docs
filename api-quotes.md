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

`title*`, `description`, `reference`, `currency`, `issue_at`, `expire_at`, `terms`, `discount`, `tax`, `adjustments`, `person_id` (UUID), `organization_id` (UUID), `lead_id` (UUID, `POST` only), `pipeline_stage_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs), `line_items[]`.

> **Important:** `subtotal` and `total` are **not** writable. Both are computed from `line_items`, `discount`, `tax` and `adjustments`, and sending either is a `422`. They are still returned in the response shape above. `discount` and `tax` reject negative values — see [API → Conventions](/api#conventions).

## Nested line items

`line_items` is accepted on `POST` and `PUT`.

**Output:**

```json
{
  "id": "uuid",
  "product_id": "uuid",
  "product":    { "id": "uuid", "name": "Annual subscription", "code": "SUB-1Y" },
  "quantity": 3.5,
  "unit_price": 100.00,
  "amount": 350.00,
  "tax_rate": 10.0,
  "tax_amount": 35.00,
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
  "quantity": 3.5,
  "unit_price": 100.00,
  "amount": 350.00,
  "comments": "Optional notes"
}
```

- **Create new line:** omit `id`. A new line is inserted.
- **Update in place:** include the existing line's `id` (UUID). The line is updated.
- **`line_items` on a `PUT` is the complete set.** Any existing line whose `id` does not appear in the payload is deleted. To replace every line, send the new lines with no `id` on any of them — the old ones go. To leave the lines untouched, omit `line_items` from the payload entirely.
- `product_id`, `quantity`, `unit_price`, and `amount` are required on every line item. `quantity` accepts up to 3 decimal places — see below.

### Decimal quantity

`quantity` is stored as `decimal(15,3)`, so a product sold by weight or volume can be quoted as `3.5` Kg or `0.25` L. The validation rule is:

```
numeric | min:0.001 | max:999999999 | decimal:0,3
```

A value finer than three decimal places is a `422`, not a silent rounding.

`min:0.001` rather than `gt:0` because a quantity like `0.0001` would round to `0` on store and the line would then be discarded without an error.

> **Important:** This is a breaking change to the *response* type. `quantity` was previously cast to an integer in every response and is now a JSON number that may come back fractional. A client decoding it into an `int` field will truncate or fail. The request side is a widening — `integer|min:1` became the rule above, so every payload that was valid before is still valid.

