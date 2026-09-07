# Teams

[[toc]]

## Overview

"Team" means two different things in this package, and they are worth separating before anything else on this page.

| | Host teams | CRM teams |
|---|---|---|
| What it is | The Jetstream or Spark tenant a user is currently switched to | A grouping of CRM users inside a single tenant |
| Model | The host application's own, typically `App\Models\Team` | `VentureDrake\LaravelCrm\Models\Team` |
| Table | The host's, typically `teams` | `crm_teams` |
| Turned on by | `LARAVEL_CRM_TEAMS=true` | The `teams` entry in [`modules`](/configuration#optional-modules) |
| What it does | **Multi-tenant data isolation.** Every model using `BelongsToTeams` is scoped to the signed-in user's `currentTeam` | Organises users; carries no data-scoping of its own |

The `BelongsToTeams` trait and the `team_id` column on CRM tables both refer to the **host** team. The CRM's own `Team` model is itself scoped by `BelongsToTeams`, so each tenant has its own set of CRM teams.

The rest of this page covers the CRM `Team` model first, then the host-team surfaces the CRM provides.

**Model:** `VentureDrake\LaravelCrm\Models\Team`
**Table:** `crm_teams`

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Team name |
| `user_id` | `integer` | User who created the team |
| `personal_team` | `boolean` | Whether this is a personal team |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `userCreated()` | `belongsTo` | `User` | Team creator |
| `users()` | `belongsToMany` | `User` | Team members (via `crm_team_user` pivot) |

## Usage

```php
use VentureDrake\LaravelCrm\Models\Team;

// Create a team
$team = Team::create([
    'name' => 'Sales Team',
    'user_id' => auth()->id(),
    'personal_team' => false,
]);

// Add a user to the team
$team->users()->attach($user->id);
```

## Data Scoping

All models that use the `BelongsToTeams` trait are automatically scoped to the signed-in user's **host** team (`auth()->user()->currentTeam`), giving data isolation between tenants without manual query filtering. The trait also stamps `team_id` on create.

The scope is inert unless `laravel-crm.teams` is on and the user has a current team, so a single-tenant install behaves as though it were not there.

### Settings

[Settings](/settings) are team-scoped too — each team has its own organisation name, logo, ID prefixes and document terms — and the settings **cache** is partitioned per team to match, with a generation counter so a write still invalidates every team's entry on cache drivers that cannot tag or scan. Switching teams flushes the cache and unsets the now-stale `currentTeam` relation.

The settings services bind as `scoped` rather than `singleton`, so memoised state cannot outlive a queued job or survive between requests under Octane.

The [portal](/portal) is the one surface that reads settings with no signed-in user, so it pins the settings service to the document's own team before rendering — see [Portal → Portal Settings and Teams](/portal#portal-settings-and-teams).

## Enabling Teams

Set the environment variable:

```env
LARAVEL_CRM_TEAMS=true
```

> **Important:** This requires Jetstream or Spark teams to be installed, and Spatie Permissions v5+ with teams support enabled.

## Team Switcher

With `laravel-crm.teams` on, the CRM header carries a dropdown listing the host teams the signed-in user belongs to, with a check mark against the current one. Switching posts to `PUT /crm/current-team` (route `current-team.update`) and reloads in the new tenant's context, so an operator no longer has to leave the CRM to change team.

- It works on hosts **without Jetstream**. Team detection prefers the user's `allTeams()` method and falls back to the `crmTeams()` relation, and the current team falls back the same way.
- Tenant teams are grouped under **Enterprises** in the dropdown, to distinguish them from the CRM's own user teams.
- The dropdown is gated on `config('laravel-crm.teams')`, not on the `teams` module — a host running the CRM single-tenant does not see a switcher for teams it does not have.

## Creating a Team from the CRM

The switcher's **+ New enterprise** item opens a quick-create form inside the CRM:

| Route name | Path | Method |
|---|---|---|
| `laravel-crm.host-teams.create` | `/crm/new-team` | `GET` |
| `laravel-crm.host-teams.store` | `/crm/new-team` | `POST` |

`HostTeamController` writes the row through the host application's own team model and switches the user onto it. This exists because a host whose `teams.create` route sits behind its own middleware — Jetstream's `hasNoTeam`, for instance — would otherwise block the link.

The model is auto-detected via the user's `ownedTeams()` relationship. Set [`host_team_model`](/configuration#host-team-model) to name it explicitly. If neither the CRM route nor a host `teams.create` route exists, the item is not rendered.

## Per-Team Lookup Data

Before 2.4.0 the CRM's lookup data was global on a teams install, so no tenant could tailor its own pipeline stages and `/leads/create` rendered against an empty per-team pipeline. Each team now gets its own copy of:

- Pipelines and pipeline stages
- Labels
- Tax rates
- Industries
- Organisation types, address types and contact types

A team created while the CRM is installed is seeded by `TeamObserver` on creation. Teams that already existed are handled by the one-time `db_update_1201` backfill, which `laravelcrm:update` runs: it copies the lookup data to every team, then re-points existing records at that team's pipeline stages, rewriting `pipeline_stage_id` on `crm_leads`, `crm_deals`, `crm_quotes`, `crm_orders`, `crm_invoices`, `crm_deliveries` and `crm_purchase_orders`.

Every block upserts on the team plus the row's own natural key, so re-running adds no duplicates.

> **Important:** The backfill has no `down()`. **Take a database backup before upgrading a teams install** — see the [Upgrade Guide](/upgrading#breaking-changes) for exactly what it touches and how unmatched stage names are handled.

> **Note:** `PipelineStageProbability` rows are not copied per team. A copied stage keeps the same `pipeline_stage_probability_id` as the global stage it came from.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

