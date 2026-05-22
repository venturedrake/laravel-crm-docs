# API · Deals

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/api/crm/v2/deals` | List (paginated). |
| `POST` | `/api/crm/v2/deals` | Create. |
| `GET` | `/api/crm/v2/deals/{uuid}` | Show. |
| `PUT` | `/api/crm/v2/deals/{uuid}` | Update. |
| `DELETE` | `/api/crm/v2/deals/{uuid}` | Soft-delete. |

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `title`, `amount`. Prefix `-` for descending. |
| `user_owner_id` | Filter by owner (integer user ID). |

## Resource shape

```json
{
  "id": "uuid",
  "deal_id": "D1001",
  "title": "Acme renewal — year 2",
  "description": null,
  "amount": 12500.00,
  "currency": "USD",
  "expected_close": "2026-09-30T00:00:00+00:00",
  "closed_at": null,
  "owner":        { "id": 1, "name": "Jane Doe" },
  "person":       { "id": "uuid", "name": "John Smith" },
  "organization": { "id": "uuid", "name": "Acme Co" },
  "lead":         { "id": "uuid", "title": "Acme renewal" },
  "labels":       []
}
```

## Writable fields

Accepted on `POST` / `PUT` (`*` = required on `POST`):

`title*`, `description`, `amount`, `currency`, `expected_close`, `lead_id` (UUID), `person_id` (UUID), `organization_id` (UUID), `pipeline_stage_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs).

