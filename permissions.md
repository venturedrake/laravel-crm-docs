# Permissions

[[toc]]

## Overview

Permissions extend [Spatie Permission](https://spatie.be/docs/laravel-permission) to provide granular access control within the CRM. CRM-specific permissions are distinguished via the `crm_permission` flag and are assigned to [Roles](/roles).

**Model:** `VentureDrake\LaravelCrm\Models\Permission`
**Extends:** `Spatie\Permission\Models\Permission`

## Scopes

### crm

Filters to only CRM permissions.

```php
$permissions = Permission::crm()->get();
```

## Usage

```php
use VentureDrake\LaravelCrm\Models\Permission;

// Get all CRM permissions
$permissions = Permission::crm()->get();

// Check a permission
$user->can('view crm leads');

// Grant permission via role
$role->givePermissionTo('edit crm deals');
```

## Permission Naming Convention

CRM permissions follow the pattern `{action} crm {entity}` — the `crm` segment is part of the name, and distinguishes these from any permissions the host application defines.

```
view crm leads      create crm leads      edit crm leads      delete crm leads
```

## Inventory

The seeder creates **156** permissions: 37 entities with a full create / view / edit / delete quad, plus 8 that do not follow the pattern.

### The quads

Each of these has all four of `create crm …`, `view crm …`, `edit crm …` and `delete crm …`:

`crm activities`, `crm calls`, `crm contacts`, `crm customers`, `crm deals`, `crm deliveries`, `crm email-campaigns`, `crm email-templates`, `crm features`, `crm fields`, `crm files`, `crm integrations`, `crm invoices`, `crm labels`, `crm lead sources`, `crm leads`, `crm lunches`, `crm meetings`, `crm monitors`, `crm notes`, `crm orders`, `crm organizations`, `crm people`, `crm permissions`, `crm pipelines`, `crm product attributes`, `crm product categories`, `crm products`, `crm purchase orders`, `crm quotes`, `crm roles`, `crm sms-campaigns`, `crm sms-templates`, `crm tasks`, `crm tax rates`, `crm teams`, `crm users`

### The rest

| Permission | Governs |
|---|---|
| `view crm settings` | Reading the Settings section, including the [Templates](/pdf-templates) sidebar item |
| `edit crm settings` | Writing any setting — the gate behind `can:update` against the `Setting` model |
| `view crm updates` | The [updates page](/updates) and the system-check banner |
| `view crm chat` | Reading [chat](/chat) conversations |
| `reply crm chat` | Replying to a conversation |
| `delete crm chat` | Deleting a conversation |
| `manage crm chat widgets` | Creating and configuring chat widgets |
| `manage crm feature statuses` | Managing the [feature board's](/features) roadmap statuses |

Note there is no `create crm chat` or `edit crm chat`: a conversation is created by a visitor, not by a CRM user.

## Policies

Every CRM entity has a policy in `src/Policies/` that maps these permission names onto Laravel's ability names. Two are worth calling out:

- **`ActivityPolicy`** is new in 2.4.0. The `activities/*` route group previously shipped with no gate at all.
- **`ProductAttributePolicy`** existed but was never registered, so every `ProductAttribute` authorization check silently fell through to deny. It is registered now — see [Product Attributes](/product-attributes).

`MonitorCheckPolicy` carries no permissions of its own; it delegates each method to `MonitorPolicy` against the check's parent monitor.

Every policy also gates its methods on the module being enabled. A module absent from `config('laravel-crm.modules')` denies the action regardless of the permissions the user holds — see the [Upgrade Guide](/upgrading#breaking-changes).

## Seeding Permissions

Permissions are created by `LaravelCrmTablesSeeder`, which `laravelcrm:install` and `laravelcrm:update` both run:

```bash
php artisan laravelcrm:update
```

On a multi-tenant install, follow it with `php artisan laravelcrm:permissions`, which fans the global CRM roles and their existing grants out to each team. That command creates no permissions of its own. See [Roles → Seeding Roles](/roles#seeding-roles).

