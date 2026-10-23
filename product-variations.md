# Product Variations

[[toc]]

## Overview

A [Product](/products) can have named variations, such as "Small" or "Gold / 18in". Each variation has its own SKU, barcode, price and active flag, and you can pick one on deal, quote, order, invoice and purchase-order lines. Variations are new in 2.5.0. A product with no variations behaves exactly as before.

**Model:** `VentureDrake\LaravelCrm\Models\ProductVariation`
**Table:** `{prefix}product_variations` (default: `crm_product_variations`)

> **Note:** The table, `crm_product_prices.product_variation_id` and the `product_variation_id` line columns have been in the schema since 2.0, but no code read or wrote them before 2.5.0. The 2.5.0 upgrade adds the remaining columns — see [Upgrading to 2.5.0](/upgrading#upgrading-to-250).

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used by the [API](/api-products#variations) (set by `ProductVariationObserver`) |
| `product_id` | `integer` | Parent product |
| `name` | `string` | Variation name |
| `code` | `string` | SKU (indexed). Printed on lines in place of the product code when set |
| `barcode` | `string` | Barcode |
| `description` | `text` | Description (not shown in the UI) |
| `order` | `integer` | Display order |
| `active` | `boolean` | Whether the variation can be picked on new lines (default `true`) |
| `team_id` | `integer` | Team (multi-tenant installs) |

The `user_created_id`, `user_updated_id`, `user_deleted_id` and `user_restored_id` columns are filled by `ProductVariationObserver`.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `product()` | `belongsTo` | `Product` | Parent product |
| `productPrices()` | `hasMany` | `ProductPrice` | Prices for this variation |

From the product side, `Product::productVariations()` returns every variation in display order, and `Product::activeProductVariations()` only the active ones.

## Pricing

A variation has **at most one price per currency**. Variation prices live in `crm_product_prices` alongside product prices, with `product_variation_id` set.

- **A variation without a price sells at the product price.** `Product::unitPriceFor($variation, $currency)` returns the variation's price in the document currency, and falls back to the product's default price when the variation has none in that currency.
- **The product form edits one price per variation, in the product's own price currency.** A product priced only in another currency keeps its variation prices in that currency too.
- **`$product->productPrices` returns product-level prices only.** Since 2.5.0 it excludes rows that carry a `product_variation_id`, so a variation price can no longer be read back as the product price by `getDefaultPrice()`, the API product payload or the [Xero](/xero) product sync. Reach variation prices through `$variation->productPrices`, or `$variation->priceFor($currency)`.

```php
$variation->priceFor('USD');       // ProductPrice|null
$variation->getDefaultPrice();     // the install currency's price, or null
$product->unitPriceFor($variation, 'USD'); // cents; falls back to the product price
```

## Managing Variations

### Product form

The product create and edit form has a **Variations** card. Use it to add variations, edit their name, SKU, barcode and price, reorder them with the arrow buttons, turn them off with the **Active** toggle, and remove them. A note under the card reminds you that the product price applies to any variation without its own price.

On save, `ProductService::syncVariations()` matches the submitted rows to the product's variations:

- A row with an id updates that variation. The id is looked up within the product, so a tampered id cannot move another product's variation across.
- A row without an id creates a variation.
- A variation that is no longer in the form is **soft-deleted**. Its prices are kept, and lines that already use it keep their label.
- The order of the rows becomes the display order.

The product show page lists the variations with their SKU, barcode, price and active flag. A variation without its own price shows the product price, greyed out. Searching the products index by a variation's SKU finds the parent product.

### In code

```php
use VentureDrake\LaravelCrm\Services\ProductService;

$service = app(ProductService::class);

// unit_price is in the major unit (dollars), and optional
$variation = $service->createVariation($product, [
    'name' => 'Large',
    'code' => 'WDG-L',
    'unit_price' => 12.50,
], 'USD');

// Only the keys you pass change. A blank unit_price removes the price in that currency.
$service->updateVariation($variation, ['active' => false]);

$service->deleteVariation($variation); // soft delete
```

Creating a variation through Eloquent directly works too, but you then have to set the price row yourself, including its `product_id`:

```php
$variation = $product->productVariations()->create(['name' => 'Large / Red']);

$variation->productPrices()->create([
    'product_id' => $product->id,
    'unit_price' => 25.00, // the mutator stores 2500 (cents)
    'currency' => 'USD',
]);
```

## Variations on Line Items

Choosing a variation is optional. Once a product with active variations is chosen on a deal, quote, order, invoice or purchase-order line, a variation select appears next to it. The options show the variation's SKU in brackets when it has one.

- **Picking a variation prices the line** from `Product::unitPriceFor()`, in the document's currency. The quote, order, invoice and purchase-order forms pass their currency to `crm-model-products` for this.
- **Changing the product clears the variation.**
- **The services only store a valid variation.** `ResolvesProductVariation` keeps a submitted variation id only if it belongs to the line's product in the current team. A deleted variation is kept only on a line that already pointed at it, so existing lines keep their label, but a new line cannot pick a deleted variation.
- **Deactivated and deleted variations stay on lines that hold them.** The select still lists them for that line, so the line shows what it holds.

### Conversions

Deal → quote, quote → order, order → invoice and order → purchase order all carry the variation across, even if it has since been deleted. A delivery line has no product of its own and prints the variation of the order line it delivers.

### How a line prints

Line models (`DealProduct`, `QuoteProduct`, `OrderProduct`, `InvoiceLine`, `PurchaseOrderLine`) use the `HasProductLineName` trait:

| Method | Returns |
|---|---|
| `displayName()` | `Product — Variation`, or the product name alone when the line has no variation |
| `displayCode()` | The variation's SKU when it has one, otherwise the product code |

`DeliveryProduct` has the same two methods and reads them from its order line. Both relations resolve `withTrashed()`, so a line keeps its label after the product or variation is deleted.

The show pages, all five [PDF templates](/pdf-templates), the emailed documents and the [portal](/portal) print `displayName()`. The quote, order, invoice, purchase-order and delivery show pages also print `displayCode()`. Eager-load `product` and `productVariation` wherever you list lines in your own code.

### Xero

Invoice and purchase-order lines send the variation name in `Description`. They send the variation's SKU as `ItemCode` only when a synced Xero item already has that code, and otherwise send the parent product's item code. Variations are never created as Xero items. See [Xero → Line items](/xero#line-items).

## API

Product responses include a `variations` array, and the `products/{product}/variations` sub-resource creates, reads, updates and deletes them. Quote, order and invoice `line_items` accept `product_variation_id`. See [API · Products → Variations](/api-products#variations).

## Permissions

There are no variation permissions. Managing variations needs `edit crm products` on the parent product, as part of the product form or through the API. Picking a variation on a line is gated in the same way as the line itself — see [Roles](/roles#why-manager-and-employee-can-still-build-a-quote-without-crm-products).

## Product Attributes (removed)

Before 2.5.0 this page documented a **Settings → Product Attributes** screen. It was an unfinished v1 stub with no table and no views, so every route errored with `Table 'product_attributes' doesn't exist`. 2.5.0 removed the page, its `laravel-crm.product-attributes.*` routes, `ProductAttributeController`, the `ProductAttribute` model, `ProductAttributePolicy` and the four `crm product attributes` permissions.

To record product details such as material, brand or warranty, attach [custom fields](/custom-fields) to Products. See [Upgrading to 2.5.0](/upgrading#product-attributes-removed).

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
