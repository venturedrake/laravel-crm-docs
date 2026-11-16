# Products

[[toc]]

## Overview

Products represent goods or services that can be sold. Products can have multiple price points (per currency), [variations](/product-variations), and belong to [Product Categories](/product-categories). Products are referenced as line items on [Deals](/deals), [Quotes](/quotes), [Orders](/orders), [Invoices](/invoices) and [Purchase Orders](/purchase-orders).

**Model:** `VentureDrake\LaravelCrm\Models\Product`
**Table:** `{prefix}products` (default: `crm_products`)

## Saving document line items

On quote, order, invoice, purchase-order and delivery forms, **Save captures the values currently entered in the product editor**, including changes made immediately before clicking Save or pressing Enter. You do not need to wait for the totals preview to refresh. The form pauses editing while it prepares and saves the lines. If validation fails or a request cannot complete, you can correct the entries or retry without re-entering them.

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
| `productPrices()` | `hasMany` | `ProductPrice` | Product-level prices per currency (excludes variation prices since 2.5.0) |
| `productVariations()` | `hasMany` | `ProductVariation` | [Variations](/product-variations), in display order |
| `activeProductVariations()` | `hasMany` | `ProductVariation` | Active variations only |
| `productCategory()` | `belongsTo` | `ProductCategory` | Category |
| `taxRate()` | `belongsTo` | `TaxRate` | Default tax rate |
| `ownerUser()` | `belongsTo` | `User` | Owner |

## Helper Methods

### getDefaultPrice()

Returns the default price for the configured system currency.

```php
$price = $product->getDefaultPrice();
```

### getPrimaryPrice()

Returns the default price, or failing that whichever product-level price row the product has. The edit form, its save and the show page all read this row. A product priced only in another currency therefore keeps that price on edit, rather than gaining a second, empty one.

### unitPriceFor()

Returns the unit price, in cents, for a line of this product. A [variation's](/product-variations#pricing) price in the given currency wins. Otherwise the line falls back to the product's default price, exactly as a line with no variation.

```php
$cents = $product->unitPriceFor($variation, 'USD'); // $variation may be null
```

> **Note:** Variation prices live in the same `crm_product_prices` table. Since 2.5.0, `productPrices()` and the `ProductPrice::default()` scope exclude rows with a `product_variation_id`. Before that, `getDefaultPrice()`, the API product payload and the [Xero](/xero) product sync read the first matching row, which could be a variation's price. The `default()` scope also compared the currency against a `Setting` model rather than its value.

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

## Variations

The product create and edit form has a **Variations** card for adding, editing, reordering, deactivating and removing variations, and the show page lists them. Each variation has its own SKU, barcode, price and active flag. See [Product Variations](/product-variations).

## Line-item product picker

The product select on each line of the quote, order, invoice, purchase-order and deal forms is searched on the server. Since 2.7.0 it returns up to 20 matching products for the term you type, instead of rendering the whole catalogue into every row. A deleted product stays out of the results.

## Deleting a Product

Deleting a product **soft-deletes** it. The distinction matters because a product is referenced by every quote, order, invoice, purchase order, delivery and deal line ever raised against it:

- **It stays readable on the documents that already reference it.** The line-item relations resolve `withTrashed()`, so the historic line description still renders on the show views, in all 14 [PDF templates](/pdf-templates) and in the [Xero](/xero) sync.
- **It stays out of the product pickers.** Those query `Product::` directly, so a deleted product cannot be chosen on a new document.

Readable on the documents that already reference it, not choosable on new ones. Before 2.4.1 deleting a product made every document that referenced it permanently unviewable — the foreign-key column was still populated but the relation resolved to null, so reading the product's name fatalled on a quote or invoice that had been fine for months.

## Searching & Filtering

Searchable by `name`. Since 2.5.0 the products index also matches a [variation's](/product-variations) SKU, so searching for it finds the parent product. Filterable by `user_owner_id` and `labels.id`.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
