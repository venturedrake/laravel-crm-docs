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
| `customer_id` | `integer` | v1 [customer](/customers), if any (`client_id` before the v2 upgrade) |
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
| `client()` | `belongsTo` | `Customer` | v1 leftover; always `null` — see [Customers](/customers#from-other-models) |
| `dealProducts()` | `hasMany` | `DealProduct` | Line items |
| `pipeline()` | `belongsTo` | `Pipeline` | Pipeline |
| `pipelineStage()` | `belongsTo` | `PipelineStage` | Pipeline stage |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `customFieldValues()` | `morphMany` | `FieldValue` | Custom field values |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Products

A deal's products are `DealProduct` lines, reached via `dealProducts()`. Since 2.5.0 a line can carry a [product variation](/product-variations): once a product with active variations is chosen on the deal form, a variation select appears beside it, and picking one prices the line from the variation's price in the deal's currency. The deal show page prints the line as `Product — Variation`.

Creating a [Quote](/quotes#creating-a-quote-from-a-deal) from the deal starts the quote with the deal's lines, variations included.

A deal saved without products shows no products section, and a blank product row on the form is not saved as a line ([#139](https://github.com/venturedrake/laravel-crm/issues/139)). The product picker on each line searches the catalogue on the server, see [Products → Line-item product picker](/products#line-item-product-picker).

## Converting from a Lead

A deal made from a [lead](/leads#converting-a-lead) sets `lead_id`. Since 2.7.0 `DealService::create()` refuses a lead that is already converted, with a validation error on `lead_id` ("This lead has already been converted to a deal"), and the [API](/api-deals) returns it as a `422`. Deleting a lead's last deal clears the lead's `converted_at`, so it can be converted again.

## Linked Contacts

Since 2.5.0, saving a deal with both a contact person and an organization links the two as related contacts. See [People → Related contacts](/people#related-contacts).

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

Since 2.7.0 the list and board find a deal by its number with or without the prefix, in any case. `%` and `_` in a search term are matched literally.

## Views

Deals support both **list view** (paginated table) and **board view** (Kanban-style pipeline stages). Toggle between views using the view switcher. The board renders a page of cards per column with **Load more**, see [Pipelines → Board View](/pipelines#board-view).

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasGlobalSettings` | Global settings access |
