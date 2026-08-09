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

Add this trait to your `User` model to enable team membership. It provides relationships and methods for managing which [Teams](/teams) a user belongs to.

```php
use VentureDrake\LaravelCrm\Traits\HasCrmTeams;

class User extends Authenticatable
{
    use HasCrmTeams;
    // ...
}
```

## User Tracking Relations

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

`laravelcrm:install` patches your application's `User` model for you, adding both traits and their `use` statements. It prompts before editing the file, prints the traits if it cannot find the model, and skips silently when they are already present — so on a normal install there is nothing to do here.

The end state it produces:

```php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;
use VentureDrake\LaravelCrm\Traits\HasCrmAccess;
use VentureDrake\LaravelCrm\Traits\HasCrmTeams;

class User extends Authenticatable
{
    use HasRoles;
    use HasCrmAccess;
    use HasCrmTeams;
}
```

Add them by hand if you declined the prompt or your `User` model lives somewhere the installer could not find. If you also want to use the [REST API](/api), add Sanctum's `HasApiTokens` alongside them — the installer does not add that one.

## Adding Users

```bash
php artisan laravelcrm:add-user
```

## Invitations

Rather than creating an account and handing over a password out of band, a CRM admin can invite someone by email. The **Invite** button on `/crm/users` sends a `UserInvitationNotification` carrying an accept link. The invitee follows it and either signs in as an existing host user or sets a password and gets a new one, landing with the CRM role and team the inviter chose.

### The invitation record

**Model:** `VentureDrake\LaravelCrm\Models\UserInvitation`
**Table:** `{prefix}user_invitations` (default: `crm_user_invitations`)

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID, stamped on `creating` by the observer |
| `code` | `string(64)` | Unique accept code — this is the model's route key |
| `team_id` | `bigint` | Team the invitee will land in |
| `email` | `string` | Address invited |
| `role_id` | `bigint` | CRM role the invitee will be granted |
| `invited_by` | `bigint` | User who sent the invitation |
| `expires_at` | `timestamp` | Expiry — nullable, and never set by the package |
| `accepted_at` | `timestamp` | When the invitation was accepted |
| `last_sent_at` | `timestamp` | Stamped each time the invitation is resent |
| `deleted_at` | `timestamp` | Soft delete — this is what "revoke" writes |

The model is route-keyed on `code` rather than `id`, and exposes four state predicates:

| Method | True when |
|---|---|
| `isAccepted()` | `accepted_at` is set |
| `isExpired()` | `expires_at` is set **and** in the past |
| `isPending()` | Neither accepted nor expired |
| `isValid()` | Same as `isPending()` |

> **Important:** Invitations do not expire by default. The package never writes `expires_at` — the column exists so a host can set an expiry itself, and until it does, `isExpired()` is always `false` and an accept link stays valid indefinitely. Revoke an invitation you no longer want honoured rather than waiting for it to lapse.

### Accepting

```
GET  /crm/users/invitations/{code}/accept
POST /crm/users/invitations/{code}/accept
```

Both routes sit **outside** the `auth.laravel-crm` middleware and the CRM-access check, deliberately: a logged-out invitee has to be able to reach the emailed link, and an invited user carries `crm_access = 0` until they accept. Accepting is what grants them access. The `team_user` insert on accept is de-duplicated, so accepting cannot add a second pivot row.

### Pending Invitations tab

The users index carries two tabs — **Registered Users** and **Pending Invitations** — with resend (stamping `last_sent_at`) and revoke (soft delete) row actions on the second. The tab and the Invite button beside it are both gated on `create crm users`.

### Role escalation

Every role dropdown that can hand out a CRM role — user create, user edit, invite, and CSV import — is filtered through `Role::assignableBy()`. It layers an Owner check onto the team/`crm_role` filter, so a caller who is not an Owner is never offered `Owner` and cannot have it accepted if they post it anyway. A null caller (console, queue, unauthenticated) is treated as not an Owner. The same predicate backs the `AssignableRole` validation rule, so the options offered and the values accepted cannot diverge. See [Security](/security).

> **Note:** The `laravel-crm.users.sendinvite` route was removed in 2.4.0. It predates this flow and did none of the above. `route('laravel-crm.users.sendinvite')` now throws `RouteNotFoundException` — see the [Upgrade Guide](/upgrading#breaking-changes).

## Filters

The users index filters on **role** and **CRM access**, backed by the `role_id` (array) and `crm_access` query-string parameters.

> **Note:** These replaced the owner and label filters the page had inherited from the CRM entity indexes, neither of which matched anything a user row carries. A bookmarked `?user_id=` or `?label_id=` link is ignored rather than erroring.

The listing itself is scoped to the current team, so the visible set matches the actionable set — deleting a user outside your current team is refused.

