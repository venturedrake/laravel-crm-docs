# Roles

[[toc]]

## Overview

Roles extend [Spatie Permission](https://spatie.be/docs/laravel-permission) to provide role-based access control within the CRM. CRM-specific roles are distinguished from application roles via the `crm_role` flag.

**Model:** `VentureDrake\LaravelCrm\Models\Role`
**Extends:** `Spatie\Permission\Models\Role`

## Default Roles

The CRM ships with the following default roles:

- **Owner** — Full access to all CRM features
- **Admin** — Administrative access
- **Manager** — Management-level access
- **User** — Standard user access

## Scopes

### crm

Filters to only CRM roles.

```php
$roles = Role::crm()->get();
```

### crmNotOwner

Filters to CRM roles excluding the Owner role (useful for role assignment dropdowns).

```php
$roles = Role::crmNotOwner()->get();
```

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

```bash
php artisan laravelcrm:permissions
```

