# API · Products

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/crm/api/v2/products` | List (paginated). |
| `POST` | `/crm/api/v2/products` | Create. |
| `GET` | `/crm/api/v2/products/{uuid}` | Show. |
| `PUT` | `/crm/api/v2/products/{uuid}` | Update. |
| `DELETE` | `/crm/api/v2/products/{uuid}` | Soft-delete. |

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `name`, `code`. Prefix `-` for descending. |
| `user_owner_id` | Filter by owner (integer user ID). |
| `active` | `true` / `false`. |

## Resource shape

```json
{
  "id": "uuid",
  "name": "Annual subscription",
  "code": "SUB-1Y",
  "barcode": null,
  "description": null,
  "unit": "ea",
  "tax_rate": 10.0,
  "tax_rate_id": 3,
  "product_category_id": "uuid",
  "purchase_account": null,
  "sales_account": null,
  "active": true,
  "prices": [
    { "currency": "USD", "unit_price": 99.00, "default": true }
  ],
  "owner": { "id": 1, "name": "Jane Doe" }
}
```

## Writable fields

Accepted on `POST` / `PUT` (`*` = required on `POST`):

`name*`, `code`, `barcode`, `description`, `unit`, `unit_price`, `currency`, `tax_rate`, `tax_rate_id` (int), `product_category_id` (UUID), `purchase_account`, `sales_account`, `active` (bool), `user_owner_id` (int).

