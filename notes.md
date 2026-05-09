# Notes

[[toc]]

## Overview

Notes are free-form text entries that can be attached to any CRM entity (leads, deals, people, organisations, etc.) via a polymorphic relationship. They appear on the entity's activity timeline and support pinning for quick visibility.

**Model:** `VentureDrake\LaravelCrm\Models\Note`
**Table:** `{prefix}notes` (default: `crm_notes`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `content` | `text` | The note body |
| `noted_at` | `datetime` | When the note was recorded |
| `pinned` | `boolean` | Whether the note is pinned to the top of the timeline |
| `noteable_type` | `string` | Polymorphic parent type |
| `noteable_id` | `integer` | Polymorphic parent ID |
| `related_note_id` | `integer` | Optional reference to a related note |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `noteable()` | `morphTo` | `Lead\|Deal\|Person\|*` | The entity this note belongs to |
| `relatedNote()` | `belongsTo` | `Note` | A related/parent note |
| `activity()` | `morphOne` | `Activity` | Activity record on the parent entity's timeline |
| `createdByUser()` | `belongsTo` | `User` | Creator |
| `updatedByUser()` | `belongsTo` | `User` | Last updater |

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

