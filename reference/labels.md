# Labels

[[toc]]

## Overview

Labels provide a tagging system for categorising CRM entities. Labels use a polymorphic many-to-many relationship and can be applied to [Leads](/reference/leads), [Deals](/reference/deals), [People](/reference/people), [Organisations](/reference/organisations), and other entities. Each label has a colour (hex code) for visual identification.

**Model:** `VentureDrake\LaravelCrm\Models\Label`
**Table:** `{prefix}labels` (default: `crm_labels`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Label name |
| `hex` | `string` | Hex colour code (stored with `#` prefix) |
| `color` | `string` | Named colour |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `leads()` | `morphedByMany` | `Lead` | Leads with this label |
| `deals()` | `morphedByMany` | `Deal` | Deals with this label |
| `people()` | `morphedByMany` | `Person` | People with this label |
| `organisations()` | `morphedByMany` | `Organisation` | Organisations with this label |

## Usage

```php
use VentureDrake\LaravelCrm\Models\Label;

// Create a label
$label = Label::create([
    'name' => 'Hot Lead',
    'hex' => 'ff0000',  // Stored as #ff0000
    'color' => 'red',
]);

// Attach a label to a lead
$lead->labels()->attach($label->id);

// Get all leads with a label
$hotLeads = $label->leads;
```

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
