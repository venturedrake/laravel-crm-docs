# Features

[[toc]]

## Overview

The Features module is a public feature-request and voting board, similar to a lightweight Canny or UserVoice. Customers and team members can submit ideas, vote on them, comment, and follow status changes as work progresses through your roadmap.

The module is enabled via the `features` entry in the [`modules`](/configuration#optional-modules) configuration array and is gated by the `@hasfeaturesenabled` Blade directive.

## Surfaces

| Surface | Route | Description |
|---|---|---|
| Admin index | `/crm/features` | List view with search and status filter |
| Admin board | `/crm/features/board` | Kanban board grouped by status |
| Admin show | `/crm/features/{external_id}` | Detail view with charts, voters, and comments |
| Admin create / edit | `/crm/features/create`, `/crm/features/{external_id}/edit` | CRUD forms |
| Public board | `/p/features` | Public roadmap visible to portal users |
| Public team board | `/p/features/team/{team_id}` | The same board addressed by team — the shareable link on a multi-tenant install |
| Public show | `/p/features/{external_id}` | Public detail with vote and comment |
| Public submit | `/p/features/submit` | Anyone signed in to the portal can suggest a feature |

The admin index carries a **Public board** button linking to the shareable board. On a [teams](/teams) install the link is team-scoped, so an admin copies a URL that works for someone with no account and no session. See [Portal → Portal Teams](/portal#portal-teams) for how a bare `/p/features` resolves which board to show.

> **Note:** Every team gets its own board. `LARAVEL_CRM_PORTAL_TEAM_ID` is optional — set it only to pin the portal to a single team and 404 everything outside it. A feature submitted through the portal is stamped with the **board's** team, not the submitter's, because a visitor who registered through `/p/register` holds no host-app team.

## Models

| Model | Table | Description |
|---|---|---|
| `Feature` | `{prefix}features` | The feature request itself |
| `FeatureStatus` | `{prefix}feature_statuses` | Roadmap statuses (Under Review, Planned, etc.) |
| `FeatureComment` | `{prefix}feature_comments` | Threaded comments on a feature |
| `FeatureVote` | `{prefix}feature_votes` | Pivot of users → features (one vote per user per feature) |
| `FeatureView` | `{prefix}feature_views` | Per-visit view-tracking row used to chart engagement |

## Feature

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in routes |
| `feature_id` | `string` | Human-readable ID (e.g. `FR0001`) |
| `title` | `string` | Short title |
| `description` | `text` | Long-form description |
| `is_public` | `boolean` | Visible on the public portal |
| `votes_count` | `integer` | Denormalised vote count |
| `comments_count` | `integer` | Denormalised comment count |
| `views_count` | `integer` | Denormalised view count |
| `feature_status_id` | `bigint` | FK to `FeatureStatus` |
| `submitted_by_user_id` | `bigint` | The user who submitted (may be a portal user) |
| `user_owner_id` | `bigint` | CRM owner |
| `user_assigned_id` | `bigint` | CRM user assigned to the request |

### Relationships

| Method | Type | Description |
|---|---|---|
| `status()` | `belongsTo` | `FeatureStatus` |
| `submittedBy()` | `belongsTo` | `User` who submitted |
| `comments()` | `hasMany` | `FeatureComment` |
| `views()` | `hasMany` | `FeatureView` |
| `voters()` | `belongsToMany` | `User` via `FeatureVote` pivot |
| `customFieldValues()` | `morphMany` | Custom field values |

### Traits

| Trait | Description |
|---|---|
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmActivities` | Activity timeline |
| `HasCrmFields` | Custom-field support |
| `SearchFilters` | Search and filter helpers |
| `SoftDeletes` | Soft delete support |

## Feature Status

Statuses model your roadmap columns. Each status has a `color` (used for badges), an `order` (controls kanban column order), and `is_default` / `is_closed` flags.

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Display label, e.g. *Planned* |
| `description` | `text` | Optional description |
| `color` | `string` | Hex colour (`#RRGGBB`) — sanitised on save |
| `order` | `tinyint` | Sort order on the kanban board |
| `is_default` | `boolean` | New submissions inherit this status |
| `is_closed` | `boolean` | Closed statuses (Completed, Declined) collapse on the board |

The package seeds five default statuses on install:

1. Under Review (default)
2. Planned
3. In Progress
4. Completed (closed)
5. Declined (closed)

Manage these under **Settings → Features → Statuses**.

> **Note:** A status cannot be deleted while features still point at it — the delete is refused with a message rather than orphaning them on the board. Re-assign those features first. Clearing the "default status" flag is scoped to the current team, so marking a default on one team no longer unsets every other team's.

## Voting

Each user (CRM user or portal user) can cast at most one vote per feature, enforced by a `unique(feature_id, user_id)` constraint on the `feature_votes` pivot. Voting is exposed on:

- The public feature show page (visitors must sign in to the portal)
- The admin show page (CRM users)

The denormalised `votes_count` on the feature is updated by the observer.

## Comments

Comments support an `is_admin_reply` flag so replies from your team can be visually distinguished on the public page. Comments are threaded via the optional `parent_id` column.

A `FeatureCommentPostedNotification` is dispatched to the feature owner and assignee when a new comment is created.

## View Tracking

Every public GET on `/p/features/{external_id}` records a `FeatureView` row containing either the authenticated user ID or a hashed IP for anonymous visitors. The denormalised `views_count` is updated on each new view.

To prevent inflated counts from refreshes, views are de-duplicated within a configurable window:

```env
LARAVEL_CRM_FEATURES_VIEW_DEDUP_MINUTES=60
```

Set to `0` to record every request.

## Charts

The admin show page renders **votes over time** and **views over time** charts side-by-side, with a period selector for the last **7, 30, 90, or 365 days**. Charts are powered by Chart.js v4.

A paginated **voters** card is shown alongside the charts so you can see who voted and when.

## Notifications

| Notification | Triggered when |
|---|---|
| `FeatureSubmittedNotification` | A new feature is created (sent to feature admins) |
| `FeatureStatusChangedNotification` | The feature's status changes (sent to voters and the submitter) |
| `FeatureCommentPostedNotification` | A new comment is posted (sent to the owner and assignee) |

Recipient resolution is centralised in the `ResolvesFeatureRecipients` trait so the same logic is reused across all three notifications.

## Configuration

```php
// config/laravel-crm.php
'features' => [
    'view_dedup_minutes' => env('LARAVEL_CRM_FEATURES_VIEW_DEDUP_MINUTES', 60),
],

'portal' => [
    'allow_registration' => env('LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_FEATURES_VIEW_DEDUP_MINUTES` | `60` | View de-duplication window in minutes |
| `LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION` | `false` | Allow visitors to self-register on the portal so they can vote and comment |

## Permissions

The package seeds the following permissions for the Features module:

- `view crm features`
- `create crm features`
- `edit crm features`
- `delete crm features`
- `manage crm feature statuses`

These are wired up in `FeaturePolicy`. Manager and Employee both hold the quad; only Manager holds `manage crm feature statuses`. See [Roles](/roles) and [Permissions](/permissions).
