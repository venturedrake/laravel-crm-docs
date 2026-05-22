# API · People

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/api/crm/v2/people` | List (paginated). |
| `POST` | `/api/crm/v2/people` | Create. |
| `GET` | `/api/crm/v2/people/{uuid}` | Show. |
| `PUT` | `/api/crm/v2/people/{uuid}` | Update. |
| `DELETE` | `/api/crm/v2/people/{uuid}` | Soft-delete. |

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `first_name`, `last_name`. Prefix `-` for descending. |
| `user_owner_id` | Filter by owner (integer user ID). |

## Resource shape

```json
{
  "id": "uuid",
  "title": "Mr",
  "first_name": "John",
  "middle_name": null,
  "last_name": "Smith",
  "name": "John Smith",
  "gender": "male",
  "birthday": "1985-04-12T00:00:00+00:00",
  "description": null,
  "owner":        { "id": 1, "name": "Jane Doe" },
  "organization": { "id": "uuid", "name": "Acme Co" },
  "labels":       []
}
```

## Writable fields

Accepted on `POST` / `PUT` (`*` = required on `POST`):

`title`, `first_name*`, `middle_name`, `last_name`, `gender`, `birthday` (date), `description`, `organization_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs).

