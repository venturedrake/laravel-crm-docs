# Product Attributes

[[toc]]

## Overview

Product attributes define the characteristics that differentiate [Product](/reference/products) variations. For example, a "T-Shirt" product might have attributes for "Size" and "Color", with each combination creating a product variation.

**Model:** `VentureDrake\LaravelCrm\Models\ProductVariation`
**Table:** `{prefix}product_variations` (default: `crm_product_variations`)

## Product Variations

Product variations represent specific versions of a product with different attribute values. Each variation can have its own pricing.

### Attributes

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Variation name |
| `description` | `text` | Description |
| `product_id` | `integer` | Parent product |

### Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `product()` | `belongsTo` | `Product` | Parent product |
| `productPrices()` | `hasMany` | `ProductPrice` | Prices for this variation |

### Usage

```php
// Create a product variation
$variation = $product->productVariations()->create([
    'name' => 'Large / Red',
    'description' => 'Large size, red colour',
]);

// Add pricing
$variation->productPrices()->create([
    'unit_price' => 2500, // in cents
    'currency' => 'USD',
]);
```

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
