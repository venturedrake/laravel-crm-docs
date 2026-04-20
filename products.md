# Products

[[toc]]

## Overview

Products represent goods or services that can be sold. Products can have multiple price points (per currency), variations, and belong to [Product Categories](/product-categories). Products are referenced as line items on [Deals](/deals), [Quotes](/quotes), [Orders](/orders), and [Invoices](/invoices).

**Model:** `VentureDrake\LaravelCrm\Models\Product`
**Table:** `{prefix}products` (default: `crm_products`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `name` | `string` | Product name |
| `code` | `string` | Product code/SKU |
| `description` | `text` | Description |
| `unit` | `string` | Unit of measure |
| `active` | `boolean` | Whether the product is active |
| `tax_rate_id` | `integer` | Default tax rate |
| `product_category_id` | `integer` | Category |
| `user_owner_id` | `integer` | Owner user |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `productPrices()` | `hasMany` | `ProductPrice` | Prices per currency |
| `productVariations()` | `hasMany` | `ProductVariation` | Product variations |
| `productCategory()` | `belongsTo` | `ProductCategory` | Category |
| `taxRate()` | `belongsTo` | `TaxRate` | Default tax rate |
| `ownerUser()` | `belongsTo` | `User` | Owner |

## Helper Methods

### getDefaultPrice()

Returns the default price for the configured system currency.

```php
$price = $product->getDefaultPrice();
```

## Scopes

### active

Filters to only active products.

```php
$products = Product::active()->get();
```

## Creating a Product

```php
use VentureDrake\LaravelCrm\Models\Product;

$product = Product::create([
    'name' => 'Web Design Package',
    'code' => 'WDP-001',
    'description' => 'Full website design package',
    'unit' => 'each',
    'active' => true,
    'user_owner_id' => auth()->id(),
]);

// Add a price
$product->productPrices()->create([
    'unit_price' => 5000, // in cents
    'currency' => 'USD',
]);
```

## Searching & Filtering

Searchable by `name`. Filterable by `user_owner_id` and `labels.id`.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
