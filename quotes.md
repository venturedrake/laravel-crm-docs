# Quotes

[[toc]]

## Overview

Quotes represent formal proposals sent to clients with pricing and terms. A quote can be created from a [Deal](/deals) and, once accepted, can be converted into an [Order](/orders). Quotes progress through pipeline stages and track acceptance/rejection status.

**Model:** `VentureDrake\LaravelCrm\Models\Quote`
**Table:** `{prefix}quotes` (default: `crm_quotes`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `quote_id` | `string` | Auto-generated quote number (prefix + number) |
| `title` | `string` | Quote title |
| `reference` | `string` | External reference |
| `description` | `text` | Description |
| `currency` | `string` | Currency code |
| `subtotal` | `integer` | Subtotal (stored in cents) |
| `discount` | `integer` | Discount amount (stored in cents) |
| `tax` | `integer` | Tax amount (stored in cents) |
| `adjustments` | `integer` | Adjustments (stored in cents) |
| `total` | `integer` | Total (stored in cents) |
| `issue_at` | `datetime` | Issue date |
| `expire_at` | `datetime` | Expiry date |
| `accepted_at` | `datetime` | When accepted |
| `rejected_at` | `datetime` | When rejected |
| `person_id` | `integer` | Contact person |
| `organisation_id` | `integer` | Organisation |
| `client_id` | `integer` | Client |
| `deal_id` | `integer` | Source deal |
| `pipeline_id` | `integer` | Pipeline |
| `pipeline_stage_id` | `integer` | Pipeline stage |

> **Note:** All money fields are stored in cents. Mutators automatically multiply by 100 on set.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Contact person |
| `organisation()` | `belongsTo` | `Organisation` | Organisation |
| `client()` | `belongsTo` | `Client` | Client |
| `deal()` | `belongsTo` | `Deal` | Source deal |
| `quoteProducts()` | `hasMany` | `QuoteProduct` | Line items |
| `orders()` | `hasMany` | `Order` | Orders generated from this quote |
| `pipeline()` | `belongsTo` | `Pipeline` | Pipeline |
| `pipelineStage()` | `belongsTo` | `PipelineStage` | Pipeline stage |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `customFieldValues()` | `morphMany` | `FieldValue` | Custom field values |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Helper Methods

### orderComplete()

Returns `true` if all quote products have been fully converted to order products across all related orders.

```php
if ($quote->orderComplete()) {
    // All items have been ordered
}
```

## Public Portal

Quotes have a public-facing portal page accessible at `/p/quotes/{external_id}`. This allows recipients to view and accept or reject quotes without needing a CRM login.

## PDF Generation

Quotes can be exported as PDF documents using `barryvdh/laravel-dompdf`.

## Views

Quotes support both **list view** (paginated table) and **board view** (Kanban-style pipeline stages).

## Creating a Quote

```php
use VentureDrake\LaravelCrm\Models\Quote;

$quote = Quote::create([
    'title' => 'Website Redesign Proposal',
    'currency' => 'USD',
    'subtotal' => 10000,
    'tax' => 1000,
    'total' => 11000,
    'issue_at' => '2026-01-15',
    'expire_at' => '2026-02-15',
    'person_id' => $person->id,
    'organisation_id' => $organisation->id,
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Searchable by `quote_id`, `title`, `reference`, and associated person/organisation names. Filterable by `user_owner_id` and `labels.id`.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasGlobalSettings` | Global settings access |
