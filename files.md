# Files

[[toc]]

## Overview

Files are file attachments that can be associated with any CRM entity (leads, deals, people, organisations, etc.) via a polymorphic relationship. They appear on the entity's activity timeline and are stored on a configurable Laravel disk.

**Model:** `VentureDrake\LaravelCrm\Models\File`
**Table:** `{prefix}files` (default: `crm_files`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `file` | `string` | Stored file path on the disk |
| `name` | `string` | Original filename |
| `title` | `string` | Optional display title |
| `format` | `string` | MIME type or file extension |
| `filesize` | `string` | File size (human-readable) |
| `disk` | `string` | Laravel filesystem disk (default `local`) |
| `fileable_type` | `string` | Polymorphic parent type |
| `fileable_id` | `integer` | Polymorphic parent ID |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `fileable()` | `morphTo` | `Lead\|Deal\|Person\|*` | The entity this file belongs to |
| `relatedFile()` | `belongsTo` | `File` | A related/parent file record |
| `activity()` | `morphOne` | `Activity` | Activity record on the parent entity's timeline |
| `createdByUser()` | `belongsTo` | `User` | Uploader |
| `updatedByUser()` | `belongsTo` | `User` | Last updater |

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

