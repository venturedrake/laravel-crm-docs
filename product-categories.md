# Product Categories

[[toc]]

## Overview

Product categories provide a way to organise [Products](/products) into logical groups. Each product can belong to one category.

**Model:** `VentureDrake\LaravelCrm\Models\ProductCategory`
**Table:** `{prefix}product_categories` (default: `crm_product_categories`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Category name |
| `description` | `text` | Description |
| `order` | `integer` | Display order |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `products()` | `hasMany` | `Product` | Products in this category |

## Usage

```php
use VentureDrake\LaravelCrm\Models\ProductCategory;

$category = ProductCategory::create([
    'name' => 'Web Services',
    'description' => 'Website design and development services',
]);

// Get all products in a category
$products = $category->products;
```

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |

