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
| `organization_id` | `integer` | Organisation |
| `client_id` | `integer` | Client |
| `deal_id` | `integer` | Source deal |
| `pipeline_id` | `integer` | Pipeline |
| `pipeline_stage_id` | `integer` | Pipeline stage |

> **Note:** All money fields are stored in cents. Mutators automatically multiply by 100 on set.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Contact person |
| `organization()` | `belongsTo` | `Organization` | Organisation |
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

## Line Items

**Model:** `VentureDrake\LaravelCrm\Models\QuoteProduct`
**Table:** `{prefix}quote_products` (default: `crm_quote_products`)

Reached from the quote via `quoteProducts()`.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used by the [API](/api-quotes#nested-line-items) |
| `product_id` | `integer` | The product being quoted |
| `product_variation_id` | `integer` | Optional [product variation](/product-attributes) |
| `quantity` | `decimal(15,3)` | Quantity, to at most 3 decimal places |
| `price` | `integer` | Unit price (stored in cents) |
| `tax_rate` | `decimal` | Tax rate percentage applied to the line |
| `tax_amount` | `integer` | Tax on the line (stored in cents) |
| `amount` | `integer` | Line total (stored in cents) |
| `currency` | `string(3)` | Currency code |
| `comments` | `string` | Optional per-line note |
| `order` | `integer` | Position of the line on the document |

> **Note:** `quantity` is `decimal(15,3)`, so a product sold by weight or volume can be quoted at `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float` — a whole quantity of 2 compares as `2.0`. Values are rounded to 3 decimal places on write, and a whole quantity still renders as `2` rather than `2.000`.

## PDF Generation

Quotes are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the quote's own `pdf_template` column when set, otherwise the default chosen for **quote** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the quote create and edit form pins a template to the record; leaving it blank follows the Settings default.

Downloads, the emailed attachment and the [portal](/portal) render all resolve the same way, so all three agree.

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
    'organization_id' => $organization->id,
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
