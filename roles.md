# Roles

[[toc]]

## Overview

Roles extend [Spatie Permission](https://spatie.be/docs/laravel-permission) to provide role-based access control within the CRM. CRM-specific roles are distinguished from application roles via the `crm_role` flag.

**Model:** `VentureDrake\LaravelCrm\Models\Role`
**Extends:** `Spatie\Permission\Models\Role`

## Default Roles

`LaravelCrmTablesSeeder` creates four CRM roles, each with `crm_role = 1` and `team_id = null` — including when teams are enabled, which is why role queries have to allow for a null team.

| Role | Permissions granted |
|---|---|
| **Owner** | `Permission::all()` |
| **Admin** | `Permission::all()` |
| **Manager** | An explicit list of 97 permissions |
| **Employee** | An explicit list of 82 permissions |

### What Manager and Employee hold

Both hold create / view / edit / delete on the core CRM entities: leads, deals, quotes, orders, invoices, deliveries, purchase orders, people, organisations, contacts, activities, tasks, notes, calls, meetings, lunches, files, pipelines and features.

They diverge on the rest:

| Permission family | Manager | Employee |
|---|---|---|
| `crm customers` | No | Yes |
| `crm email-campaigns`, `crm email-templates` | Yes | No |
| `crm sms-campaigns`, `crm sms-templates` | Yes | No |
| `view` / `reply crm chat` | Yes | Yes |
| `delete crm chat`, `manage crm chat widgets` | Yes | No |
| `manage crm feature statuses` | Yes | No |
| `crm products`, `crm product categories`, `crm product attributes` | No | No |
| `crm monitors` | No | No |
| `crm users`, `crm teams`, `crm roles`, `crm permissions` | No | No |
| `crm settings`, `view crm updates` | No | No |
| `crm tax rates`, `crm labels`, `crm lead sources`, `crm fields`, `crm integrations` | No | No |

> **Important:** Manager holding no `crm customers` while Employee does is a long-standing quirk of the seeder, not a design statement. Neither holds any `crm products` or `crm monitors` permission. If your Managers or Employees are expected to run campaigns, manage monitors, edit the product catalogue or edit customers, grant those permissions explicitly under **Settings → Roles**.

### Why Manager and Employee can still build a quote without `crm products`

Line items on a deal, quote or order are gated on a `manageProducts` **policy ability**, not on a seeded permission. `QuotePolicy::manageProducts()` and its siblings check the *parent's* edit permission — `edit crm quotes` — so anyone who may edit the quote may edit the lines inside it. The `crm products` quad governs the standalone product catalogue at `/crm/products`, which is a different thing.

That is also why the deal / quote / order `products` sub-resource route groups carry `can:manageProducts,<Model>` as their only gate. They previously carried a per-route `can:view,{param}` underneath it, which resolved to `view crm <entity>` — so a custom role holding edit-but-not-view could open the form and then `403` on the line items embedded in it.

## Scopes

### crm

Filters to only CRM roles.

```php
$roles = Role::crm()->get();
```

### crmNotOwner

Filters to CRM roles excluding the Owner role.

```php
$roles = Role::crmNotOwner()->get();
```

### assignable

CRM roles valid for the current team. Keeps `Owner` in the set so ownership transfer stays possible.

```php
$roles = Role::assignable()->get();
```

### assignableBy

CRM roles the caller is entitled to hand out — `assignable()`, minus `Owner` for anyone who is not already an Owner. A null caller (console, queue, unauthenticated) is treated as not an Owner.

```php
$roles = Role::assignableBy()->get();     // the authenticated caller
$roles = Role::assignableBy($user)->get(); // an explicit caller
```

> **Important:** Every site that turns user input into a role assignment goes through `assignableBy()` — the role dropdowns, the `AssignableRole` validation rule, [invitations](/users#invitations) and CSV import — so the options offered and the values accepted cannot diverge. Use it rather than `assignable()` or `crmNotOwner()` for anything driven by user input. See [Security](/security#role-escalation).

## Usage

```php
use VentureDrake\LaravelCrm\Models\Role;

// Get all CRM roles
$roles = Role::crm()->get();

// Assign a role to a user
$user->assignRole('Admin');

// Check role
$user->hasRole('Owner');
```

## Seeding Roles

The four roles and their grants are created by `LaravelCrmTablesSeeder`, which `laravelcrm:install` and `laravelcrm:update` both run. It is `firstOrCreate` + `givePermissionTo` throughout, so re-running matches existing rows rather than duplicating them, adds any permission introduced since the last seeding, revokes nothing, and does not touch custom roles.

```bash
php artisan laravelcrm:update

# …or the seeder on its own
php artisan db:seed --class="VentureDrake\LaravelCrm\Database\Seeders\LaravelCrmTablesSeeder" --force
```

On a multi-tenant install, follow it with:

```bash
php artisan laravelcrm:permissions
```

> **Important:** `laravelcrm:permissions` creates no roles and no permissions, despite the name. It copies the global CRM roles and their *existing* grants down to each team, so it only does anything when `laravel-crm.teams = true` — on a single-tenant install it prints `Teams config for multi-tenant support is not enabled.` and exits. It is not a substitute for running the seeder.

