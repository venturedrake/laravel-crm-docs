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
  "variations": [/* see "Variations" below */],
  "owner": { "id": 1, "name": "Jane Doe" }
}
```

`variations` (since 2.5.0) lists every variation of the product, active and inactive, in display order, in the same shape as the [variations sub-resource](#variations). `prices` holds the product's own prices only: since 2.5.0 it never includes variation prices, which share the same table.

## Writable fields

Accepted on `POST` / `PUT` (`*` = required on `POST`):

`name*`, `code`, `barcode`, `description`, `unit`, `unit_price`, `currency`, `tax_rate`, `tax_rate_id` (int), `product_category_id` (UUID), `purchase_account`, `sales_account`, `active` (bool), `user_owner_id` (int).

**Product writes leave variations alone.** `POST` / `PUT` on `/products` neither accepts nor changes `variations`. Use the sub-resource below.

## Variations

Since 2.5.0 a product's [variations](/product-variations) (sizes, colours and so on) are a sub-resource of the product:

| Verb | Path | Permission | Action |
|---|---|---|---|
| `GET` | `/crm/api/v2/products/{product}/variations` | `view crm products` | List all the product's variations, active and inactive, ordered by `order`. Not paginated. |
| `POST` | `/crm/api/v2/products/{product}/variations` | `edit crm products` | Create. Returns `201`. |
| `GET` | `/crm/api/v2/products/{product}/variations/{variation}` | `view crm products` | Show. |
| `PUT` / `PATCH` | `/crm/api/v2/products/{product}/variations/{variation}` | `edit crm products` | Update. Only the fields sent change. |
| `DELETE` | `/crm/api/v2/products/{product}/variations/{variation}` | `edit crm products` | Soft-delete. Returns `204`. |

There are no variation permissions. Reading a variation needs `view` on its product, and creating, updating or deleting one needs `update` (edit) on the product. `{product}` and `{variation}` are both UUIDs. The variation is looked up **within** the URL's product, so a variation of another product, or a deleted one, is a `404`.

### Variation fields

| Field | Rules | Notes |
|---|---|---|
| `name` | required on `POST`; string, max 255 | |
| `code` | nullable string, max 255 | The variation's SKU. Printed on lines in place of the product code when set. |
| `barcode` | nullable string, max 255 | |
| `active` | boolean | Defaults to `true`. |
| `order` | nullable integer ≥ 0 | Defaults to after the last variation. |
| `unit_price` | nullable number ≥ 0, in dollars | Omitted on `PUT`: the price is left alone. Sent as `null`: the variation's price in `currency` is removed, and lines on it use the product price. |
| `currency` | nullable 3-letter code | The currency `unit_price` applies to. Defaults to the product's price currency. |

### Variation shape

```json
{
  "data": {
    "id": "6b0e...variation-uuid",
    "name": "Large",
    "code": "WDG-L",
    "barcode": null,
    "active": true,
    "order": 2,
    "prices": [
      { "currency": "USD", "unit_price": 12.50 }
    ],
    "created_at": "2026-10-22T09:00:00+00:00",
    "updated_at": "2026-10-22T09:00:00+00:00"
  }
}
```

In the CRM forms, a line on a variation with no price in the document currency is priced from the product. The API does not price lines for you. To put a variation on a quote, order or invoice line, send `product_variation_id` with the `unit_price` you want — see [Quotes → Nested line items](/api-quotes#product-variation-id).

