# Custom Field Groups

[[toc]]

## Overview

Custom field groups organise [Custom Fields](/reference/custom-fields) into logical sections. Each group can contain multiple fields and is scoped to a specific entity type (leads, deals, etc.).

**Model:** `VentureDrake\LaravelCrm\Models\FieldGroup`
**Table:** `{prefix}field_groups` (default: `crm_field_groups`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Group name |
| `handle` | `string` | Unique handle |
| `model` | `string` | Entity class this group applies to |
| `order` | `integer` | Display order |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `fields()` | `hasMany` | `Field` | Fields in this group |

## Usage

```php
use VentureDrake\LaravelCrm\Models\FieldGroup;

// Create a field group for leads
$group = FieldGroup::create([
    'name' => 'Additional Info',
    'handle' => 'additional_info',
    'model' => 'VentureDrake\LaravelCrm\Models\Lead',
    'order' => 1,
]);

// Add a field to the group
$group->fields()->create([
    'name' => 'Source URL',
    'handle' => 'source_url',
    'type' => 'text',
    'required' => false,
    'order' => 1,
]);
```

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
