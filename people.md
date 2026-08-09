# People

[[toc]]

## Overview

People represent individual contacts in the CRM. A person can belong to an [Organisation](/organizations), be associated with [Leads](/leads) and [Deals](/deals), and become a [Customer](/customers) via the polymorphic Client model. Personal fields support optional encryption for privacy.

**Model:** `VentureDrake\LaravelCrm\Models\Person`
**Table:** `{prefix}people` (default: `crm_people`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `title` | `string` | Title (Mr, Mrs, etc.) — encryptable |
| `first_name` | `string` | First name — encryptable |
| `middle_name` | `string` | Middle name — encryptable |
| `last_name` | `string` | Last name — encryptable |
| `maiden_name` | `string` | Maiden name — encryptable |
| `birthday` | `date` | Date of birth |
| `description` | `text` | Notes |
| `organization_id` | `integer` | Parent organisation |
| `user_owner_id` | `integer` | Owner user |

> **Note:** Fields marked "encryptable" are encrypted at rest when `encrypt_db_fields` is enabled in the [configuration](/configuration).

## Computed Attributes

### name

Returns the full name (`first_name last_name`).

```php
$person->name; // "John Smith"
```

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `organization()` | `belongsTo` | `Organization` | Parent organisation |
| `deals()` | `hasMany` | `Deal` | Associated deals |
| `emails()` | `morphMany` | `Email` | Email addresses |
| `phones()` | `morphMany` | `Phone` | Phone numbers |
| `addresses()` | `morphMany` | `Address` | Physical addresses |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `contacts()` | `morphMany` | `Contact` | Contact records |
| `client()` | `morphOne` | `Client` | Client record |
| `ownerUser()` | `belongsTo` | `User` | Owner |

## Helper Methods

| Method | Returns | Description |
|---|---|---|
| `getPrimaryEmail()` | `Email\|null` | Primary email address |
| `getPrimaryPhone()` | `Phone\|null` | Primary phone number |
| `getPrimaryAddress()` | `Address\|null` | Primary address |

## Creating a Person

```php
use VentureDrake\LaravelCrm\Models\Person;

$person = Person::create([
    'first_name' => 'John',
    'last_name' => 'Smith',
    'organization_id' => $organization->id,
    'user_owner_id' => auth()->id(),
]);

// Add an email
$person->emails()->create([
    'address' => 'john@example.com',
    'type' => 'work',
    'primary' => true,
]);

// Add a phone number
$person->phones()->create([
    'number' => '+1234567890',
    'type' => 'work',
    'primary' => true,
]);
```

## Searching & Filtering

Searchable by `first_name`, `middle_name`, `last_name`, and `maiden_name`. Filterable by `user_owner_id` and `labels.id`. Sortable by `id`, `first_name`, `last_name`, `created_at`, and `updated_at`.

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
| `HasGlobalSettings` | Global settings access |
