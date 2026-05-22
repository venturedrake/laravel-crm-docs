# API · Leads

[[toc]]

See [API overview](/api) for authentication, headers, pagination, sorting, conventions, error envelopes, and rate limits that apply to every endpoint below.

## Endpoints

| Verb | Path | Action |
|---|---|---|
| `GET` | `/crm/api/v2/leads` | List (paginated). |
| `POST` | `/crm/api/v2/leads` | Create. |
| `GET` | `/crm/api/v2/leads/{uuid}` | Show. |
| `PUT` | `/crm/api/v2/leads/{uuid}` | Update. |
| `DELETE` | `/crm/api/v2/leads/{uuid}` | Soft-delete. |

`{uuid}` is the lead's `external_id` (UUID), exposed as `id` in JSON responses.

## List parameters

| Param | Notes |
|---|---|
| `per_page` | `1`–`100`, default `25`. |
| `sort` | One of `created_at`, `updated_at`, `title`, `amount`. Prefix `-` for descending. Default `-created_at`. |
| `user_owner_id` | Filter by owner (integer user ID). |

## Resource shape

```json
{
  "id": "uuid",
  "lead_id": "L1001",
  "title": "Acme renewal",
  "description": null,
  "amount": 12500.00,
  "currency": "USD",
  "expected_close": "2026-09-30T00:00:00+00:00",
  "converted_at": null,
  "created_at": "2026-05-01T12:00:00+00:00",
  "updated_at": "2026-05-01T12:00:00+00:00",
  "deleted_at": null,
  "owner":        { "id": 1, "name": "Jane Doe" },
  "person":       { "id": "uuid", "name": "John Smith" },
  "organization": { "id": "uuid", "name": "Acme Co" },
  "labels":       [{ "id": "uuid", "name": "Hot" }]
}
```

## Writable fields

Accepted on `POST` / `PUT` (`*` = required on `POST`):

`title*`, `description`, `amount`, `currency` (3-letter), `expected_close` (ISO-8601), `person_id` (UUID), `organization_id` (UUID), `lead_source_id` (UUID), `pipeline_stage_id` (UUID), `user_owner_id` (int), `labels[]` (UUIDs).

## Example

```bash
TOKEN="1|abcdef..."

# Create
curl -s -X POST https://example.test/crm/api/v2/leads \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "New enterprise lead",
    "amount": 12500.00,
    "currency": "USD",
    "expected_close": "2026-09-30T00:00:00Z"
  }' | jq .

# List, newest first, filtered by owner
curl -s "https://example.test/crm/api/v2/leads?sort=-created_at&user_owner_id=1" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" | jq .
```

