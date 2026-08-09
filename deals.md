# Deals

[[toc]]

## Overview

Deals represent qualified business opportunities with an estimated value and expected close date. Deals are typically created from converted [Leads](/leads) and can progress through pipeline stages. Deals can have associated [Products](/products), and can generate [Quotes](/quotes).

**Model:** `VentureDrake\LaravelCrm\Models\Deal`
**Table:** `{prefix}deals` (default: `crm_deals`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `deal_id` | `string` | Unique deal identifier |
| `title` | `string` | Deal title |
| `amount` | `integer` | Deal value (stored in cents) |
| `currency` | `string` | Currency code |
| `description` | `text` | Detailed description |
| `expected_close` | `datetime` | Expected close date |
| `closed_at` | `datetime` | Actual close date |
| `closed_status` | `string` | Won/lost status |
| `person_id` | `integer` | Associated person |
| `organization_id` | `integer` | Associated organisation |
| `client_id` | `integer` | Associated client |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |
| `pipeline_id` | `integer` | Pipeline |
| `pipeline_stage_id` | `integer` | Current pipeline stage |

> **Note:** The `amount` attribute is stored in cents. The mutator automatically multiplies by 100 on set.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Contact person |
| `organization()` | `belongsTo` | `Organization` | Organisation |
| `client()` | `belongsTo` | `Client` | Client |
| `dealProducts()` | `hasMany` | `DealProduct` | Line items |
| `pipeline()` | `belongsTo` | `Pipeline` | Pipeline |
| `pipelineStage()` | `belongsTo` | `PipelineStage` | Pipeline stage |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `customFieldValues()` | `morphMany` | `FieldValue` | Custom field values |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Creating a Deal

```php
use VentureDrake\LaravelCrm\Models\Deal;

$deal = Deal::create([
    'title' => 'Website Redesign',
    'amount' => 10000, // Stored as 1000000 (cents)
    'currency' => 'USD',
    'expected_close' => '2026-06-30',
    'person_id' => $person->id,
    'organization_id' => $organization->id,
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Deals are searchable by `deal_id`, `title`, and associated person/organisation names. Filterable by `user_owner_id` and `labels.id`.

## Views

Deals support both **list view** (paginated table) and **board view** (Kanban-style pipeline stages). Toggle between views using the view switcher.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasGlobalSettings` | Global settings access |
