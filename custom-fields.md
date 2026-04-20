# Custom Fields

[[toc]]

## Overview

Custom fields allow you to extend CRM entities with additional data fields without modifying the database schema. Fields belong to [Custom Field Groups](/custom-field-groups) and store their values via a polymorphic `FieldValue` model. Supported on [Leads](/leads), [Deals](/deals), [Orders](/orders), [Quotes](/quotes), [Invoices](/invoices), and other entities that use the `HasCrmFields` trait.

## Models

### Field

**Model:** `VentureDrake\LaravelCrm\Models\Field`
**Table:** `{prefix}fields` (default: `crm_fields`)

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Field name |
| `handle` | `string` | Unique handle for programmatic access |
| `type` | `string` | Field type (text, textarea, select, etc.) |
| `required` | `boolean` | Whether the field is required |
| `order` | `integer` | Display order |
| `field_group_id` | `integer` | Parent field group |

#### Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `fieldGroup()` | `belongsTo` | `FieldGroup` | Parent group |
| `fieldOptions()` | `hasMany` | `FieldOption` | Options for select-type fields |

### FieldValue

**Model:** `VentureDrake\LaravelCrm\Models\FieldValue`
**Table:** `{prefix}field_values` (default: `crm_field_values`)

Stores the actual custom field data for each entity instance.

| Attribute | Type | Description |
|---|---|---|
| `field_id` | `integer` | The field definition |
| `value` | `text` | The stored value |
| `custom_field_valueable_type` | `string` | Polymorphic type |
| `custom_field_valueable_id` | `integer` | Polymorphic ID |

#### Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `field()` | `belongsTo` | `Field` | The field definition |
| `fieldValueable()` | `morphTo` | `*` | The parent entity |

## Usage

```php
// Access custom field values on an entity
$lead->customFieldValues;

// Get a specific custom field value
$value = $lead->customFieldValues()
    ->whereHas('field', fn ($q) => $q->where('handle', 'source_url'))
    ->first();
```

## Traits

All custom field models use:

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

