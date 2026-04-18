# Users

[[toc]]

## Overview

Laravel CRM integrates with your application's existing `User` model through two traits. These traits provide CRM access control and team membership capabilities without requiring a separate user model.

## Traits

### HasCrmAccess

Add this trait to your `User` model to enable CRM access. It provides the ability to check if a user has been granted access to the CRM.

```php
use VentureDrake\LaravelCrm\Traits\HasCrmAccess;

class User extends Authenticatable
{
    use HasCrmAccess;
    // ...
}
```

### HasCrmTeams

Add this trait to your `User` model to enable team membership. It provides relationships and methods for managing which [Teams](/reference/teams) a user belongs to.

```php
use VentureDrake\LaravelCrm\Traits\HasCrmTeams;

class User extends Authenticatable
{
    use HasCrmTeams;
    // ...
}
```

## User Audit Relations

Most CRM models track which user created, updated, deleted, and restored records via standard foreign key relationships:

| Relationship | Foreign Key | Description |
|---|---|---|
| `createdByUser()` | `user_created_id` | User who created the record |
| `updatedByUser()` | `user_updated_id` | User who last updated the record |
| `deletedByUser()` | `user_deleted_id` | User who deleted the record |
| `restoredByUser()` | `user_restored_id` | User who restored the record |
| `ownerUser()` | `user_owner_id` | Record owner |
| `assignedToUser()` | `user_assigned_id` | Assigned user |

Models using the `HasCrmUserRelations` trait get these relationships automatically.

## Setup

Add both traits to your application's `User` model:

```php
namespace App;

use Illuminate\Foundation\Auth\User as Authenticatable;
use VentureDrake\LaravelCrm\Traits\HasCrmAccess;
use VentureDrake\LaravelCrm\Traits\HasCrmTeams;

class User extends Authenticatable
{
    use HasCrmAccess;
    use HasCrmTeams;
}
```
