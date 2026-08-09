# Purchase Orders

[[toc]]

## Overview

Purchase orders manage supplier orders linked to [Orders](/orders). They track what needs to be purchased from suppliers to fulfill customer orders.

**Model:** `VentureDrake\LaravelCrm\Models\PurchaseOrder`
**Table:** `{prefix}purchase_orders` (default: `crm_purchase_orders`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `purchase_order_id` | `string` | Auto-generated PO number |
| `reference` | `string` | External reference |
| `description` | `text` | Description |
| `currency` | `string` | Currency code |
| `subtotal` | `integer` | Subtotal (stored in cents) |
| `discount` | `integer` | Discount (stored in cents) |
| `tax` | `integer` | Tax (stored in cents) |
| `adjustments` | `integer` | Adjustments (stored in cents) |
| `total` | `integer` | Total (stored in cents) |
| `order_id` | `integer` | Related customer order |
| `organization_id` | `integer` | Supplier organisation |
| `person_id` | `integer` | Supplier contact |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |

> **Note:** All money fields are stored in cents.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `order()` | `belongsTo` | `Order` | Related customer order |
| `organization()` | `belongsTo` | `Organization` | Supplier |
| `person()` | `belongsTo` | `Person` | Supplier contact |
| `purchaseOrderLines()` | `hasMany` | `PurchaseOrderLine` | Line items |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Line Items

**Model:** `VentureDrake\LaravelCrm\Models\PurchaseOrderLine`
**Table:** `{prefix}purchase_order_lines` (default: `crm_purchase_order_lines`)

Reached from the purchase order via `purchaseOrderLines()`.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID |
| `product_id` | `integer` | The product being purchased |
| `product_variation_id` | `integer` | Optional [product variation](/product-attributes) |
| `description` | `text` | Optional line description |
| `quantity` | `decimal(15,3)` | Quantity, to at most 3 decimal places |
| `price` | `integer` | Unit price (stored in cents) |
| `tax_rate` | `decimal` | Tax rate percentage applied to the line |
| `tax_amount` | `integer` | Tax on the line (stored in cents) |
| `amount` | `integer` | Line total (stored in cents) |
| `currency` | `string(3)` | Currency code |
| `order` | `integer` | Position of the line on the document |

> **Note:** `quantity` is `decimal(15,3)`, so a purchase order line can carry `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float`.

## PDF Generation

Purchase orders are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the purchase order's own `pdf_template` column when set, otherwise the default chosen for **purchase-order** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the purchase order create and edit form pins a template to the record; leaving it blank follows the Settings default.

Downloads, the emailed attachment and the [portal](/portal) render all resolve the same way, so all three agree.

> **Note:** The Settings key for this document type keeps its hyphen — `pdf_template_purchase-order`.

## Creating a Purchase Order

```php
use VentureDrake\LaravelCrm\Models\PurchaseOrder;

$po = PurchaseOrder::create([
    'currency' => 'USD',
    'subtotal' => 5000,
    'tax' => 500,
    'total' => 5500,
    'order_id' => $order->id,
    'organization_id' => $supplier->id,
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Searchable by `reference`, `purchase_order_id`, and associated organisation names.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |

