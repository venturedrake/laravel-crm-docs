# Customers

[[toc]]

## Overview

Customers (internally called "Clients") represent the polymorphic bridge between a [Person](/people) or [Organisation](/organizations) and their role as a customer. This allows both people and organisations to be treated uniformly as customers throughout the system — on [Leads](/leads), [Deals](/deals), [Orders](/orders), etc.

**Model:** `VentureDrake\LaravelCrm\Models\Client`
**Table:** `{prefix}clients` (default: `crm_clients`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `clientable_type` | `string` | Polymorphic type (`Person` or `Organization`) |
| `clientable_id` | `integer` | Polymorphic ID |

## Computed Attributes

### name

Returns the name of the underlying person or organisation.

```php
$client->name; // "John Smith" or "Acme Corp"
```

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `clientable()` | `morphTo` | `Person\|Organization` | The underlying entity |

## Polymorphic Usage

A Person or Organisation becomes a customer via their `client()` relationship:

```php
// Create a client from a person
$client = $person->client()->create();

// Create a client from an organisation
$client = $organization->client()->create();

// Access the underlying entity
$client->clientable; // Returns Person or Organisation instance
```

## Searching & Filtering

Searchable by the underlying person or organisation names.

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

