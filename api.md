# API

[[toc]]

## Overview

Laravel CRM ships with a JSON REST API for partner developers and external integrations. The API is versioned, authenticated with [Laravel Sanctum](https://laravel.com/docs/sanctum) personal access tokens, and mounted at `/crm/api/v2`.

- **Base URL:** `https://your-app.test/crm/api/v2`
- **Content type:** `application/json`
- **Authentication:** Sanctum bearer tokens
- **Current version:** `v2`

The API exposes **8 resourceful entities** with full CRUD (list, create, show, update, soft-delete), plus 3 auth endpoints for issuing, inspecting, and revoking tokens.

### Entity references

Each entity has its own reference page covering endpoints, list parameters, JSON shape, and writable fields:

- [Leads](/api-leads)
- [Products](/api-products)
- [Organizations](/api-organizations)
- [People](/api-people)
- [Deals](/api-deals)
- [Quotes](/api-quotes)
- [Orders](/api-orders)
- [Invoices](/api-invoices)

## Requirements

- Laravel CRM installed and migrated.
- Laravel Sanctum installed in the host application.
- A CRM user with `crm_access` granted, to issue tokens against.

## Installation

The API ships with the package — you only need to wire Sanctum into the host application.

### 1. Publish and run Sanctum migrations

```bash
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

This creates the `personal_access_tokens` table.

### 2. Add `HasApiTokens` to your User model

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
    // ...
}
```

### 3. Verify the API routes are registered

```bash
php artisan route:list --path=crm/api
```

You should see 8 resourceful entities (5 verbs each), plus the 3 auth routes.

## Authentication

The API uses Sanctum personal access tokens. Tokens can be issued through the API itself, or via an artisan command for ops use.

### Issue a token via the API

```http
POST /crm/api/v2/auth/token
Content-Type: application/json

{
  "email": "user@example.com",
  "password": "secret",
  "device_name": "Mobile App"
}
```

**Response (`201 Created`):**

```json
{
  "token": "1|abcdef1234...",
  "user": {
    "id": 1,
    "name": "Jane Doe",
    "email": "user@example.com"
  }
}
```

- Returns `422` on bad credentials, an unknown email, or when the user lacks `crm_access`. The response is intentionally indistinguishable across these cases to avoid leaking which emails belong to real users.
- `device_name` is optional and defaults to the request's `User-Agent` or `api-token`.
- The plaintext token is shown **once** — store it securely.

### Issue a token via artisan

```bash
php artisan laravel-crm:api-token user@example.com --name="Mobile App"
```

The plaintext token is printed once. The command exits non-zero if the user does not exist or lacks `crm_access`.

> **Note:** `laravel-crm:api-token` is hyphenated. It is the one exception — every other artisan command this package ships is namespaced `laravelcrm:` with no hyphen (`laravelcrm:install`, `laravelcrm:update`, `laravelcrm:upgrade`). This is not a typo in the docs.

### Making authenticated requests

Pass the token in the `Authorization` header:

```http
GET /crm/api/v2/leads HTTP/1.1
Authorization: Bearer 1|abcdef1234...
Accept: application/json
```

### Inspecting the current user

```http
GET /crm/api/v2/auth/me
```

**Response (`200 OK`):**

```json
{
  "user": {
    "id": 1,
    "name": "Jane Doe",
    "email": "user@example.com"
  }
}
```

### Revoking the current token

```http
DELETE /crm/api/v2/auth/token
```

Returns `204 No Content` and deletes the personal access token used to authenticate the request.

## Headers

| Header | Required | Purpose |
|---|---|---|
| `Authorization: Bearer <token>` | Yes (except `POST /auth/token`) | Sanctum personal access token. |
| `Accept: application/json` | Recommended | Forces JSON responses (the `laravel-crm.api.json` middleware sets this automatically when missing). |
| `Content-Type: application/json` | Yes (for `POST` / `PUT`) | Request body is JSON. |
| `X-Team-ID: <team-id>` | Optional | Overrides the authenticated user's active team for the request. Must be a team the user belongs to; otherwise the API returns `403`. Only relevant when `laravel-crm.teams=true`. |

### Multi-tenancy notes

When the host app runs in teams mode (`config('laravel-crm.teams', false)` — teams are **off** by default):

- Without `X-Team-ID`, requests are scoped to the user's `current_team_id`.
- With `X-Team-ID`, list / store / update / delete endpoints run in the context of that team.
- `GET /{resource}/{uuid}` resolves the route-bound model using the user's **default** current team, because Laravel's `SubstituteBindings` middleware runs before the team-context middleware. Use the list endpoints (filtered by `X-Team-ID`) to discover the correct UUIDs for the active team.
- **A token whose user has no current team cannot reference anything.** Every foreign key on a write whose table is team-scoped — `person_id`, `organization_id`, `pipeline_stage_id`, `labels[]`, `line_items.*.product_id`, and the rest — is validated against the active team. When teams are on and the token's user has no `currentTeam`, and none was supplied via `X-Team-ID`, the rule matches nothing rather than falling back to unscoped, so **every** such id comes back `422` at once. It presents as "all my ids are suddenly invalid". The fix is on the user record, not the payload: give the user a current team, or send `X-Team-ID`. Service accounts created outside the host app's normal registration flow are the usual cause.

## Endpoint summary

### Auth

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/crm/api/v2/auth/token` | Public | Issue a personal access token. |
| `GET` | `/crm/api/v2/auth/me` | Bearer | Return the authenticated user. |
| `DELETE` | `/crm/api/v2/auth/token` | Bearer | Revoke the current token. |

### Entities

All entity endpoints follow the same RESTful shape:

| Verb | Path | Action |
|---|---|---|
| `GET` | `/{resource}` | List (paginated). |
| `POST` | `/{resource}` | Create. |
| `GET` | `/{resource}/{uuid}` | Show. |
| `PUT` | `/{resource}/{uuid}` | Update. |
| `DELETE` | `/{resource}/{uuid}` | Soft-delete. |

`{uuid}` is the entity's `external_id` (UUID), exposed as `id` in JSON responses.

| Resource | Path | Reference |
|---|---|---|
| Lead | `/crm/api/v2/leads` | [Leads](/api-leads) |
| Product | `/crm/api/v2/products` | [Products](/api-products) |
| Organization | `/crm/api/v2/organizations` | [Organizations](/api-organizations) |
| Person | `/crm/api/v2/people` | [People](/api-people) |
| Deal | `/crm/api/v2/deals` | [Deals](/api-deals) |
| Quote | `/crm/api/v2/quotes` | [Quotes](/api-quotes) |
| Order | `/crm/api/v2/orders` | [Orders](/api-orders) |
| Invoice | `/crm/api/v2/invoices` | [Invoices](/api-invoices) |

## Conventions

- **IDs are UUIDs.** The JSON `id` is always the entity's `external_id`. Integer primary keys are never exposed.
- **Foreign keys are also UUIDs** for CRM entities — e.g. `person_id`, `organization_id`, `lead_id`, `pipeline_stage_id`, `lead_source_id`, `product_category_id`, and every `labels[]` entry must be a UUID (the related row's `external_id`).
- **Some lookups are integer IDs.** Host-app users (`user_owner_id`) and a handful of small reference tables that aren't UUID-backed use integer primary keys: `tax_rate_id`, `organization_type_id`, `industry_id`, `timezone_id`.
- **Human-readable IDs are returned, not accepted.** Entities also expose a sequential identifier (`lead_id`, `deal_id`, `quote_id`, `order_id`, `invoice_id`, e.g. `L1001`, `D1001`) in the response. These are read-only and assigned automatically.
- **Money is dollars in JSON; cents in storage.** All amount / price / total / subtotal / discount / tax / adjustments fields are sent and returned as decimal dollars (e.g. `1500.50`). The package converts to integer cents on write.
- **Timestamps are ISO-8601** and must include a timezone offset or the `Z` UTC suffix on input — e.g. `2026-07-15T10:00:00+00:00` or `2026-07-15T10:00:00Z`. Other formats are rejected with a `422`.
- **Pagination:** `?per_page=N` (1–100, default 25). Responses use Laravel's standard pagination envelope (`data`, `meta`, `links`).
- **Sorting:** `?sort=field` ascending; `?sort=-field` descending. Unknown columns are silently ignored. Default sort is `-created_at`.
- **Soft deletes:** `DELETE` returns `204` and soft-deletes the row. Subsequent `GET`s return `404`.
- **`subtotal` and `total` are computed and rejected on input.** On quotes and orders both are derived from `line_items`, `discount`, `tax` and `adjustments`; on invoices, which carry no `discount` or `adjustments` field, from `line_items` and `tax`. Sending either on a `POST` or `PUT` is a `422` naming the cause, rather than being silently ignored and recomputed. An absent or `null` value passes, so a payload that never sent them is unaffected. Both are still returned in responses.
- **`discount` and `tax` reject negatives.** `tax` carries `min:0` on quote / order / invoice writes; `discount` carries it on quote / order writes, invoices having no such field. A negative discount was a way to inflate a total past the sum of the line items; on a quote or order, send a negative `adjustments` value instead.
- **Referenced ids must belong to the active team.** Every UUID reference on a write is checked against the request's team as well as the table. An id belonging to another team is a `422` on that field, where it previously validated and produced a cross-team record. Single-tenant installs (`laravel-crm.teams = false`) are unaffected — the check is skipped entirely — as are package-wide lookup tables that carry no `team_id` column, which are still checked against the table alone.

## Errors

The API returns standard Laravel error envelopes.

### `422 Unprocessable Entity` (validation)

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "title": ["The title field is required."],
    "person_id": ["The selected person_id is invalid."]
  }
}
```

### `401 Unauthorized`

```json
{ "message": "Unauthenticated." }
```

### `403 Forbidden`

```json
{ "message": "This action is unauthorized." }
```

For the `X-Team-ID` non-member case:

```json
{ "message": "You are not a member of the requested team." }
```

### `404 Not Found`

Returned when a UUID does not resolve to a model (or it has been soft-deleted).

### `429 Too Many Requests`

Returned when the rate limit is exceeded. Standard Laravel rate-limit headers are included: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `Retry-After`.

## Rate limits

The API enforces a single named rate limiter, `laravel-crm-api`:

| Caller | Limit |
|---|---|
| Authenticated (Sanctum) | **60 requests / minute / user** |
| Unauthenticated | **30 requests / minute / IP** |

Exceeding the limit returns `429 Too Many Requests` with `Retry-After` in seconds.

### `POST /auth/token` is throttled twice

On top of the per-IP limit above, token issuing carries a fixed `throttle:6,1` (6 attempts / minute / IP) **and** a per-account counter on failed attempts, so credential stuffing spread across many IPs against one email address does not get unlimited attempts.

| Setting | Default | Environment Variable |
|---|---|---|
| `laravel-crm.api.token_attempts_per_account` | `5` | `LARAVEL_CRM_API_TOKEN_ATTEMPTS_PER_ACCOUNT` |
| `laravel-crm.api.token_attempts_decay_seconds` | `600` | `LARAVEL_CRM_API_TOKEN_ATTEMPTS_DECAY_SECONDS` |

The per-account counter is keyed on the submitted email, incremented only on a failed attempt, and cleared on success. Once it trips, the endpoint returns `429` with the error under `errors.email` rather than the usual `429` envelope:

```json
{
  "message": "Too many login attempts. Try again in 540 seconds.",
  "errors": { "email": ["Too many login attempts. Try again in 540 seconds."] }
}
```

> **Important:** `POST /auth/token` could not return `429` before 2.4.0. A client that re-issues a token on every request, or retries on failure without backing off, will start seeing it — issue a token once and reuse it, and back off on `429`. Both limits key on IP or email, so one misbehaving integration can lock out a shared address; raise the two config keys if your deployment legitimately issues tokens in bursts. See [Configuration → API](/configuration#api).

## Worked example

Issue a token, list leads, create a lead, then revoke the token.

```bash
# 1. Issue a token
curl -s -X POST https://example.test/crm/api/v2/auth/token \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"secret","device_name":"curl"}' \
  | jq .

# 2. List leads
TOKEN="1|abcdef..."
curl -s https://example.test/crm/api/v2/leads \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" \
  | jq .

# 3. Create a lead
curl -s -X POST https://example.test/crm/api/v2/leads \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "New enterprise lead",
    "amount": 12500.00,
    "currency": "USD",
    "expected_close": "2026-09-30T00:00:00Z"
  }' \
  | jq .

# 4. Revoke the token
curl -s -X DELETE https://example.test/crm/api/v2/auth/token \
  -H "Authorization: Bearer $TOKEN" \
  -i
```

