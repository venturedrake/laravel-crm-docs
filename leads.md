# Leads

[[toc]]

## Overview

Leads represent potential business opportunities that have not yet been qualified. A lead is typically the first point of contact in the sales pipeline, and can be associated with a [Person](/people), [Organisation](/organizations), or both. Once qualified, a lead can be converted into a [Deal](/deals).

**Model:** `VentureDrake\LaravelCrm\Models\Lead`
**Table:** `{prefix}leads` (default: `crm_leads`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `lead_id` | `string` | Unique lead identifier (e.g. `L1001`) |
| `title` | `string` | Lead title/description |
| `amount` | `integer` | Estimated value (stored in cents) |
| `currency` | `string` | Currency code |
| `description` | `text` | Detailed description |
| `converted_at` | `datetime` | When the lead was converted to a deal |
| `lead_status_id` | `integer` | Foreign key to lead status |
| `lead_source_id` | `integer` | Foreign key to lead source |
| `person_id` | `integer` | Associated person |
| `organization_id` | `integer` | Associated organisation |
| `customer_id` | `integer` | v1 [customer](/customers), if any (`client_id` before the v2 upgrade) |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |
| `pipeline_id` | `integer` | Pipeline the lead belongs to |
| `pipeline_stage_id` | `integer` | Current pipeline stage |

> **Note:** The `amount` attribute is stored in cents. When setting the value, the mutator automatically multiplies by 100. For example, setting `$lead->amount = 150` stores `15000` in the database.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Associated contact person |
| `organization()` | `belongsTo` | `Organization` | Associated organisation |
| `customer()` | `belongsTo` | `Customer` | v1 [customer](/customers), on `customer_id` |
| `leadStatus()` | `belongsTo` | `LeadStatus` | Current status |
| `leadSource()` | `belongsTo` | `LeadSource` | Source of the lead |
| `pipeline()` | `belongsTo` | `Pipeline` | Pipeline |
| `pipelineStage()` | `belongsTo` | `PipelineStage` | Pipeline stage |
| `emails()` | `morphMany` | `Email` | Email addresses |
| `phones()` | `morphMany` | `Phone` | Phone numbers |
| `addresses()` | `morphMany` | `Address` | Addresses |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `customFieldValues()` | `morphMany` | `FieldValue` | Custom field values |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |
| `createdByUser()` | `belongsTo` | `User` | Creator |
| `updatedByUser()` | `belongsTo` | `User` | Last updater |
| `deal()` | `hasOne` | `Deal` | The deal the lead was converted to (the latest, if there are several) |

## Helper Methods

### getPrimaryEmail()

Returns the primary email for the lead. If the lead has an associated person, it returns the person's primary email instead.

```php
$email = $lead->getPrimaryEmail();
```

### getPrimaryPhone()

Returns the primary phone number. Falls back to the associated person's primary phone.

```php
$phone = $lead->getPrimaryPhone();
```

### getPrimaryAddress()

Returns the primary address. Falls back to the associated organisation's primary address.

```php
$address = $lead->getPrimaryAddress();
```

## Creating a Lead

```php
use VentureDrake\LaravelCrm\Models\Lead;

$lead = Lead::create([
    'title' => 'New website project',
    'amount' => 5000, // Stored as 500000 (cents)
    'currency' => 'USD',
    'description' => 'Potential website redesign project',
    'person_id' => $person->id,
    'organization_id' => $organization->id,
    'lead_source_id' => $source->id,
    'user_owner_id' => auth()->id(),
]);
```

## Converting a Lead

A lead is converted to a [Deal](/deals) from its show page. Since 2.7.0:

- **The lead moves to a converted stage.** `LeadService::markConverted()` sets `converted_at` and moves the lead to a **Converted**, **Closed Won** or **Won** stage when its pipeline has one.
- **The show page shows a dated Converted badge** with a link to the deal, through `Lead::deal()`.
- **A lead can only be converted once.** The show page, a deal form left open on a lead that has since been converted, `DealService::create()` and the [API](/api-deals) (`422` on `lead_id`) all refuse a second deal. Before, a converted lead could be converted again into a second deal.
- **Deleting the lead's last deal reopens it.** `converted_at` is cleared, so the lead returns to the list and board and can be converted again. The 2.7.0 upgrade reopens leads whose deals were deleted earlier, see the [Upgrade Guide](/upgrading#leads-whose-deal-was-deleted-are-reopened).

A person created from the lead form saves their email with the **work** type ([#144](https://github.com/venturedrake/laravel-crm/issues/144)).

## Searching & Filtering

Leads are searchable by `lead_id`, `title`, and associated person/organisation names. They can be filtered by `user_owner_id` and `labels.id`.

Since 2.7.0 the list and board find a lead by its number with or without the prefix, in any case. `%` and `_` in a search term are matched literally.

## Views

Leads support both **list view** (paginated table) and **board view** (Kanban-style pipeline stages). Toggle between views using the view switcher. The board renders a page of cards per column with **Load more**, see [Pipelines → Board View](/pipelines#board-view).

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
