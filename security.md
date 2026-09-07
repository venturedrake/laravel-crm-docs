# Security

[[toc]]

## Overview

Laravel CRM is designed with security and data privacy best practices. Several features provide layers of protection for sensitive data.

## Authentication

The CRM uses your application's existing authentication system. All CRM routes are protected by the `auth.laravel-crm` middleware. Users must have the `crm_access` attribute set to `true` on their user model.

Three surfaces sit deliberately outside that middleware:

- **The [REST API](/api)** authenticates with Laravel Sanctum personal access tokens instead. `POST /crm/api/v2/auth/token` is public, throttled per IP and per account, and returns the same `422` for bad credentials, an unknown email and a user without `crm_access`, so it cannot be used to enumerate addresses.
- **The [portal](/portal)** — signed quote / invoice / purchase-order pages, the public feature board and the chat widget — is reachable by anonymous visitors by design. Portal self-registration is off by default.
- **[Invitation acceptance](/users#accepting)** (`/crm/users/invitations/{code}/accept`), because a logged-out invitee has to reach the emailed link and carries `crm_access = 0` until they accept.

## Authorization

Every CRM entity has a corresponding Laravel Policy in `src/Policies/`. Permissions are managed via the [Spatie Permission](https://spatie.be/docs/laravel-permission) package with CRM-specific roles and permissions.

See [Roles](/roles) and [Permissions](/permissions) for details.

### Enforced at the action layer, not only in Blade

Until 2.4.0 the UI advertised permissions through Blade `@can` directives, but the Livewire components behind those buttons did not re-check on the server. Any user who could reach a CRM page could invoke its actions directly over the Livewire endpoint, regardless of role.

Every mutating action now calls `$this->authorize(...)` against the same permission the UI advertises — 165 actions across 117 components — and the previously ungated `activities/*` and deal / quote / order `products` route groups carry `can:` middleware. Mutating controls are hidden rather than shown-then-denied, and kanban cards are not draggable without the matching `edit` permission.

No new permission name was introduced. A regression test fails the suite by name if a mutating Livewire action ships without a guard or a documented exemption.

> **Important:** On an install upgraded over time without re-seeding, permissions added in later releases may be missing — and every action they gate will now `403`, including for Owner. Re-run the permission seeding **before** deploying. See the [Upgrade Guide](/upgrading#before-you-upgrade).

### Role escalation

`Role::assignableBy()` is the single predicate deciding which CRM roles a caller may hand out. It layers an Owner check onto the team and `crm_role` filter, so a caller who is not an Owner is never offered `Owner` — and a null caller (console, queue, unauthenticated) is treated as not an Owner. Role dropdowns, the `AssignableRole` validation rule, the invitation form and CSV import all go through it, so the options offered and the values accepted cannot diverge.

CSV import previously resolved roles with a bare name lookup, which meant anyone holding `create crm users` could import `role=Owner`, a host-application role such as `super-admin`, or a role belonging to another tenant. An unassignable role is now dropped rather than failing the row. The role is also resolved and vetted **before** the user row is written, so a blocked escalation no longer leaves an orphaned role-less user behind.

Deleting a user outside your current team is refused, and the users listing is scoped to the current team so the visible set matches the actionable set.

### Kanban reordering

Dragging a card on a [pipeline](/pipelines) board posts the new sibling order. `onStageSorted` used to authorize only the **first** resolvable record in that batch and then update every id it had been handed, so a record the user had no right to edit was reordered as long as an editable one led the list. Every record in the batch is authorized now, and the batch is resolved, filtered and renumbered in a transaction.

### Portal link minting

The **Get link** modal on quotes, invoices and purchase orders is mounted once in the layout and receives a type slug and a record id over the wire. `confirm()` re-resolves the record through `Support\PortalLink`'s model whitelist — three slugs mapped to three model classes, so a raw class name never crosses the request — and then through the record's policy, rather than trusting the component's own state. A tampered payload cannot mint a signed link to a record the caller may not view.

## Team Data Scoping

### The settings cache is partitioned per team

Until 2.4.1 the settings cache was global while the query behind it was team-scoped. On a `laravel-crm.teams` install, whichever team warmed the cache served its organisation name, ABN, address and logo to every other team until the next settings write — on the settings screen itself and on every invoice, quote and PDF those settings render into.

`SettingService::cacheKey()` partitions by the current team, with a generation counter so a write still reaches every team's entry on cache drivers that cannot tag or scan. Alongside it, the settings view composer dropped its own second, never-invalidated cache; the three team-switch sites flush the cache and unset the now-stale `currentTeam` relation; and the settings services bind as `scoped` rather than `singleton`, so memoised state cannot outlive a queued job or survive between requests under Octane. See [Teams → Data Scoping](/teams#data-scoping).

### The portal rendered another tenant's branding

Until 2.4.2 the public [portal](/portal) rendered a document's page and PDF from **unscoped** settings. The portal is anonymous by design — a signed link, no login — so `BelongsToTeamsScope` never engaged there and every settings read returned every team's rows; `pluck()` keys by name, so whichever team the database listed last supplied the organisation name, ABN, contact block and logo for every tenant's documents alike. A customer opening one team's emailed invoice could be shown another team's branding.

The portal controllers now pin `SettingService` to the **document's own team** via `forTeam()` before rendering anything, which corrects every downstream reader of the shared scoped instance. The cache is partitioned on the same answer. A document that predates teams renders a blank From block rather than borrowing whoever sorted last.

### Team switching verifies membership

`CurrentTeamController`'s non-Jetstream fallback switched a user into any team id it was handed. It verifies membership of the target team before switching now.

### Signed document routes stand down the team scope

`BelongsToTeamsScope` does **not** engage on the three signed portal document routes. Those links are authorised by their signature and the recipient has no account at all, so who happens to be logged in should not decide whether a signed link opens — a staff member signed in to team A was getting a 404 on team B's valid link, because route-model binding filtered the record out.

This narrows nothing: the signature is the authorisation, and the settings the page renders are pinned to the document's own team regardless of the session.

## API Data Scoping

Every foreign key on an API write is validated against the caller's team by the `ScopedExists` rule. The bare Laravel `exists` rule queries the database directly and so bypasses the team scope, which let an authenticated caller reference another tenant's `external_id` and have it accepted. A table with no `team_id` column falls back to a bare `exists` rather than producing a SQL error, and a caller holding no current team is failed rather than passed.

Cross-team ids present as a `422` on the field. Single-tenant installs are unaffected. See [API → Conventions](/api#conventions).

## API Token Throttling

`POST /crm/api/v2/auth/token` is throttled twice: `throttle:6,1` per IP, and a per-account counter keyed on the submitted email (5 failed attempts per 10 minutes by default), so credential stuffing spread across many IPs against one address does not get unlimited attempts. See [API → Rate limits](/api#rate-limits).

## Monitoring SSRF Guard

Any CRM admin can create a [monitor](/monitoring), which makes an outbound HTTP request to a URL they supply. Monitor URLs that resolve to loopback, private, link-local or reserved addresses are rejected before the request is made, and DNS resolution is pinned so cURL cannot re-resolve to a different address than the guard validated — a defence against DNS rebinding. Set `LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS=true` only if you genuinely need to monitor services on your own private network.

## Field Encryption

Sensitive personal data (names, emails, phone numbers) can be encrypted at rest in the database. Enable via:

```env
LARAVEL_CRM_ENCRYPT_DB_FIELDS=true
```

Then run the encryption command:

```bash
php artisan laravelcrm:encrypt
```

Encrypted fields are declared in each model's `$encryptable` array and handled transparently by the `LaravelEncryptableTrait`.

## Reporting Vulnerabilities

If you discover a security vulnerability, please email [andrew@laravelcrm.com](mailto:andrew@laravelcrm.com). All security vulnerabilities will be promptly addressed.

