# API · Organizations

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/crm/api/v2/organizations` | List (paginated). |
| `POST` | `/crm/api/v2/organizations` | Create. |
| `GET` | `/crm/api/v2/organizations/{uuid}` | Show. |
| `PUT` | `/crm/api/v2/organizations/{uuid}` | Update. |
| `DELETE` | `/crm/api/v2/organizations/{uuid}` | Soft-delete. |

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `name`. Prefix `-` for descending. |
| `user_owner_id` | Filter by owner (integer user ID). |

## Resource shape

```json
{
  "id": "uuid",
  "name": "Acme Co",
  "description": null,
  "vat_number": null,
  "linkedin": null,
  "number_of_employees": 50,
  "annual_revenue": 5000000.00,
  "total_money_raised": null,
  "organization_type_id": 1,
  "industry_id": 4,
  "timezone_id": 12,
  "owner":  { "id": 1, "name": "Jane Doe" },
  "labels": []
}
```

## Writable fields

Accepted on `POST` / `PUT` (`*` = required on `POST`):

`name*`, `description`, `vat_number`, `linkedin`, `number_of_employees` (int), `annual_revenue`, `total_money_raised`, `organization_type_id` (int), `industry_id` (int), `timezone_id` (int), `user_owner_id` (int).

