# Lunches

[[toc]]

## Overview

Lunches log lunch-meeting interactions and can be attached to any CRM entity (leads, deals, people, organisations, etc.) via a polymorphic relationship. They appear on the entity's activity timeline and support start/end times and optional email/SMS reminders.

**Model:** `VentureDrake\LaravelCrm\Models\Lunch`
**Table:** `{prefix}lunches` (default: `crm_lunches`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `name` | `string` | Lunch subject or title |
| `description` | `text` | Notes about the lunch |
| `start_at` | `datetime` | When the lunch starts |
| `finish_at` | `datetime` | When the lunch ends |
| `reminder_email` | `boolean` | Send an email reminder before `start_at` |
| `reminder_sms` | `boolean` | Send an SMS reminder before `start_at` |
| `lunchable_type` | `string` | Polymorphic parent type |
| `lunchable_id` | `integer` | Polymorphic parent ID |
| `user_owner_id` | `integer` | User responsible |
| `user_assigned_id` | `integer` | User the lunch is assigned to |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `lunchable()` | `morphTo` | `Lead\|Deal\|Person\|*` | The entity this lunch belongs to |
| `contacts()` | `morphMany` | `Contact` | Contacts associated with the lunch |
| `activity()` | `morphOne` | `Activity` | Activity record on the parent entity's timeline |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |
| `createdByUser()` | `belongsTo` | `User` | Creator |
| `updatedByUser()` | `belongsTo` | `User` | Last updater |

## Reminders

When `reminder_email` or `reminder_sms` is set, the `laravelcrm:reminders` command (scheduled every minute) sends the relevant notification before the lunch's `start_at` time.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |

