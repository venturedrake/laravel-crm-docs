# Organisations

[[toc]]

## Overview

Organisations represent companies or business entities. An organisation can have multiple [People](/people) associated with it, and can be linked to [Leads](/leads), [Deals](/deals), [Orders](/orders), and other entities. The organisation name supports optional encryption for privacy.

**Model:** `VentureDrake\LaravelCrm\Models\Organisation`
**Table:** `{prefix}organisations` (default: `crm_organisations`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `name` | `string` | Organisation name — encryptable |
| `description` | `text` | Notes |
| `annual_revenue` | `integer` | Annual revenue (stored in cents) |
| `total_money_raised` | `integer` | Total funding raised (stored in cents) |
| `number_of_employees` | `integer` | Employee count |
| `organisation_type_id` | `integer` | Organisation type |
| `timezone_id` | `integer` | Timezone |
| `user_owner_id` | `integer` | Owner user |

> **Note:** Money fields (`annual_revenue`, `total_money_raised`) are stored in cents. The `name` field is encrypted at rest when `encrypt_db_fields` is enabled.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `people()` | `hasMany` | `Person` | Associated people |
| `deals()` | `hasMany` | `Deal` | Associated deals |
| `emails()` | `morphMany` | `Email` | Email addresses |
| `phones()` | `morphMany` | `Phone` | Phone numbers |
| `addresses()` | `morphMany` | `Address` | Physical addresses |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `contacts()` | `morphMany` | `Contact` | Contact records |
| `client()` | `morphOne` | `Client` | Client record |
| `organisationType()` | `belongsTo` | `OrganisationType` | Type classification |
| `timezone()` | `belongsTo` | `Timezone` | Timezone |
| `ownerUser()` | `belongsTo` | `User` | Owner |

## Helper Methods

| Method | Returns | Description |
|---|---|---|
| `getPrimaryEmail()` | `Email\|null` | Primary email address |
| `getPrimaryPhone()` | `Phone\|null` | Primary phone number |
| `getPrimaryAddress()` | `Address\|null` | Primary address |
| `getBillingAddress()` | `Address\|null` | Billing address (type 5) |
| `getShippingAddress()` | `Address\|null` | Shipping address (type 6) |

## Creating an Organisation

```php
use VentureDrake\LaravelCrm\Models\Organisation;

$org = Organisation::create([
    'name' => 'Acme Corp',
    'description' => 'Software company',
    'annual_revenue' => 1000000, // Stored in cents
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Searchable by `name`. Filterable by `user_owner_id` and `labels.id`. Sortable by `id`, `name`, `created_at`, and `updated_at`.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `LaravelEncryptableTrait` | Field-level encryption |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `Sortable` | Column sorting |
| `HasCrmActivities` | Activity timeline tracking |
| `HasCrmUserRelations` | Standard user tracking relations |
