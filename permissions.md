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
$user->can('view leads');

// Grant permission via role
$role->givePermissionTo('edit deals');
```

## Permission Naming Convention

CRM permissions follow the pattern `{action} {entity}`:

- `view leads`, `create leads`, `edit leads`, `delete leads`
- `view deals`, `create deals`, `edit deals`, `delete deals`
- And so on for each entity type

## Seeding Permissions

```bash
php artisan laravelcrm:permissions
```

