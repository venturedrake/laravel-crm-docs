# Teams

[[toc]]

## Overview

Teams provide multi-tenant data isolation within the CRM. Every user belongs to one or more teams, and most CRM entities are scoped to a team via the `BelongsToTeams` trait. Each user has a personal team created automatically, and can be added to shared teams.

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

All models that use the `BelongsToTeams` trait are automatically scoped to the user's current team. This ensures data isolation between teams without manual query filtering.

## Enabling Teams

Set the environment variable:

```env
LARAVEL_CRM_TEAMS=true
```

> **Important:** This requires Jetstream or Spark teams to be installed, and Spatie Permissions v5+ with teams support enabled.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

