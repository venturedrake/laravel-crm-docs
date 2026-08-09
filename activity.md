# Activity

[[toc]]

## Overview

Activities track interactions and events across the CRM. Each activity uses three polymorphic relationships to record who caused the activity, which entity's timeline it appears on, and what record was affected. This provides a comprehensive activity feed and history.

**Model:** `VentureDrake\LaravelCrm\Models\Activity`
**Table:** `{prefix}activities` (default: `crm_activities`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `description` | `string` | Activity description |
| `details` | `text` | Detailed notes |
| `causeable_type` | `string` | Who/what caused the activity |
| `causeable_id` | `integer` | Causeable ID |
| `timelineable_type` | `string` | Which entity's timeline |
| `timelineable_id` | `integer` | Timelineable ID |
| `recordable_type` | `string` | What record was affected |
| `recordable_id` | `integer` | Recordable ID |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `causeable()` | `morphTo` | `User\|*` | Who/what triggered the activity |
| `timelineable()` | `morphTo` | `Lead\|Deal\|*` | Entity whose timeline shows this activity |
| `recordable()` | `morphTo` | `*` | The record that was created/updated/deleted |

## How Activities Are Created

Activities are automatically created by models that use the `HasCrmActivities` trait. When a model is created, updated, or deleted, an activity record is generated linking the acting user, the parent timeline entity, and the affected record.

```php
// Activities are typically queried through the timeline
$lead->activities; // All activities on a lead's timeline
```

## Permissions

The `activities/*` route group is gated by `ActivityPolicy`, which maps onto the `create` / `view` / `edit` / `delete crm activities` permissions.

> **Note:** This group previously shipped with no gate at all — any user who could reach the CRM could hit every route under `/crm/activities` regardless of role. `ActivityPolicy` is new in 2.4.0. Manager and Employee both hold the full quad, so no stock role loses access. See [Permissions](/permissions) and the [Upgrade Guide](/upgrading#breaking-changes).

## Activity Types

The CRM tracks various activity types through dedicated Livewire components:

- **Tasks** — Assignable tasks with due dates
- **Notes** — Free-form notes attached to entities
- **Calls** — Phone call logs
- **Meetings** — Meeting records
- **Lunches** — Lunch meeting records
- **Files** — File attachments

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

