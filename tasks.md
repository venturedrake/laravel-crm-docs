# Tasks

[[toc]]

## Overview

Tasks are assignable to-do items that can be attached to any CRM entity (leads, deals, people, organisations, etc.) via a polymorphic relationship. They support due dates, completion tracking, and optional email/SMS reminder notifications.

**Model:** `VentureDrake\LaravelCrm\Models\Task`
**Table:** `{prefix}tasks` (default: `crm_tasks`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `name` | `string` | Task title |
| `description` | `text` | Optional detailed description |
| `due_at` | `datetime` | When the task is due |
| `completed_at` | `datetime` | When the task was marked complete |
| `reminder_email` | `boolean` | Send an email reminder before the due date |
| `reminder_sms` | `boolean` | Send an SMS reminder before the due date |
| `taskable_type` | `string` | Polymorphic parent type |
| `taskable_id` | `integer` | Polymorphic parent ID |
| `user_owner_id` | `integer` | User responsible for the task |
| `user_assigned_id` | `integer` | User the task is assigned to |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `taskable()` | `morphTo` | `Lead\|Deal\|Person\|*` | The entity this task belongs to |
| `activity()` | `morphOne` | `Activity` | Activity record on the parent entity's timeline |
| `ownerUser()` | `belongsTo` | `User` | Task owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |
| `createdByUser()` | `belongsTo` | `User` | Creator |
| `updatedByUser()` | `belongsTo` | `User` | Last updater |

## Usage

Tasks are created from the activity timeline on any entity's detail page. They appear in the **Activity** section of the CRM and can be filtered to show only tasks.

```php
use VentureDrake\LaravelCrm\Models\Task;

// Create a task attached to a lead
$task = Task::create([
    'name'           => 'Follow up call',
    'description'    => 'Call to discuss proposal',
    'due_at'         => now()->addDays(2),
    'reminder_email' => true,
    'user_assigned_id' => $user->id,
]);

$task->taskable()->associate($lead)->save();
```

## Reminders

When `reminder_email` or `reminder_sms` is set, the `laravelcrm:reminders` command (scheduled every minute) sends the relevant notification before the task's `due_at` time.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |

