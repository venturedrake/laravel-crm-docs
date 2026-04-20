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
| `organisation_id` | `integer` | Associated organisation |
| `client_id` | `integer` | Associated client |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |
| `pipeline_id` | `integer` | Pipeline the lead belongs to |
| `pipeline_stage_id` | `integer` | Current pipeline stage |

> **Note:** The `amount` attribute is stored in cents. When setting the value, the mutator automatically multiplies by 100. For example, setting `$lead->amount = 150` stores `15000` in the database.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Associated contact person |
| `organisation()` | `belongsTo` | `Organisation` | Associated organisation |
| `client()` | `belongsTo` | `Client` | Associated client |
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
    'organisation_id' => $organisation->id,
    'lead_source_id' => $source->id,
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Leads are searchable by `lead_id`, `title`, and associated person/organisation names. They can be filtered by `user_owner_id` and `labels.id`.

## Views

Leads support both **list view** (paginated table) and **board view** (Kanban-style pipeline stages). Toggle between views using the view switcher.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
