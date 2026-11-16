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

`reference`, `issue_date`, `due_date`, `currency`, `terms`, `tax`, `person_id` (UUID), `organization_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs), `line_items[]`.

Accepted on `POST` only — the source order is set at creation and cannot be re-pointed:

`order_id` (UUID).

> **Important:** `subtotal` and `total` are **not** writable. Both are computed from the saved `line_items` and their tax, and sending either is a `422`. They are still returned in the response shape above. Since 2.7.0 the header `tax` is the sum of the lines' tax: it is still accepted, and rejects negatives, but is ignored. See [API → Conventions](/api#conventions).

> **Important:** Since 2.7.0, on invoices:
>
> - **A line's `amount` is derived.** It is stored as `unit_price × quantity`, rounded to the cent. `line_items.*.amount` is still required, but a posted amount that differs is ignored; send the reduced `unit_price` for a discounted line.
> - **A paid invoice's totals are locked.** Once it has any `amount_paid` or a `fully_paid_at`, a `PUT` whose `line_items` would change its subtotal or tax is a `422` on `line_items` and nothing is saved. Lines re-sent unchanged with their `id`, and a `PUT` that omits `line_items`, still save.
> - **`due_date` can't be before `issue_date`.** An earlier day is a `422`. Both keep only the date.
>
> See [API → Conventions](/api#conventions) and [Invoices → Payments](/invoices#payments).

> **Note:** Invoices have no `discount` or `adjustments` field — quotes and orders do, invoices do not. That is deliberate, not an omission from this list: `tax` is the only rate-style field an invoice write accepts, and it rejects negative values.

## Nested line items

Same shape and rules as [Quotes → Nested line items](/api-quotes#nested-line-items), except that the stored `amount` is computed from `unit_price × quantity`: the posted `amount` is validated but not stored.

