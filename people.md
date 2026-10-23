# People

[[toc]]

## Overview

People represent individual contacts in the CRM. A person can belong to an [Organisation](/organizations) and be associated with [Leads](/leads) and [Deals](/deals). Personal fields support optional encryption for privacy.

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
| `client()` | `morphOne` | `Customer` | v1 leftover; throws a `QueryException` — see [Customers](/customers#from-other-models) |
| `ownerUser()` | `belongsTo` | `User` | Owner |

## Related Contacts

The **Related organizations** card on a person's show page, and the **Related people** card on an organization's, list `Contact` records reached via `contacts()`. A link is stored in both directions, one row on each side.

Since 2.5.0, saving a lead, deal, quote, order, invoice or purchase order with both a contact person and an organization links them as related contacts, so each lists the other. This applies whether the person and organization are new or picked from the autocomplete. `PersonService::linkToOrganization()` makes the link:

```php
app(\VentureDrake\LaravelCrm\Services\PersonService::class)
    ->linkToOrganization($person, $organization);
```

- **An existing link is not duplicated.** Each side is a `firstOrCreate()`.
- **Nothing is unlinked.** Changing the person or organization on a record keeps the old link.
- **There is no backfill.** Records saved before 2.5.0 stay unlinked until they are saved again.

Before 2.5.0, saving such a record never linked the two ([#94](https://github.com/venturedrake/laravel-crm/issues/94)).

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
