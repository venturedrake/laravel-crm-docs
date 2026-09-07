# Deliveries

[[toc]]

## Overview

Deliveries track the physical shipment of goods from an [Order](/orders). Each delivery contains delivery products that reference order line items, allowing partial deliveries and tracking of fulfillment status.

**Model:** `VentureDrake\LaravelCrm\Models\Delivery`
**Table:** `{prefix}deliveries` (default: `crm_deliveries`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `delivery_id` | `string` | Delivery identifier |
| `reference` | `string` | External reference |
| `description` | `text` | Description |
| `delivery_expected` | `date` | Expected delivery date |
| `delivered_on` | `date` | Actual delivery date |
| `order_id` | `integer` | Source order |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `order()` | `belongsTo` | `Order` | Source order |
| `deliveryProducts()` | `hasMany` | `DeliveryProduct` | Line items |
| `addresses()` | `morphMany` | `Address` | Shipping addresses |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Helper Methods

### getShippingAddress()

Returns the shipping address (address type 6).

```php
$address = $delivery->getShippingAddress();
```

## Computed Attributes

### title

Returns the order total and client/organisation name.

```php
$delivery->title; // "$1,500.00 - Acme Corp"
```

## Line Items

**Model:** `VentureDrake\LaravelCrm\Models\DeliveryProduct`
**Table:** `{prefix}delivery_products` (default: `crm_delivery_products`)

Reached from the delivery via `deliveryProducts()`.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID |
| `order_product_id` | `integer` | The order line this delivery draws down |
| `quantity` | `decimal(15,3)` | Quantity delivered, to at most 3 decimal places |
| `order` | `integer` | Position of the line on the document |

A delivery line references an [order line](/orders#drawing-down-an-order-line) rather than a product directly, which is what makes partial deliveries possible.

> **Note:** `quantity` is `decimal(15,3)`, so a delivery can record `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float`.

> **Note:** A **soft-deleted product stays readable** on the deliveries that already reference it. `DeliveryProduct::orderProduct()` resolves `withTrashed()`, and the order line's own product relation does too, so the line keeps its description on the show view and in all 14 PDF templates. See [Products](/products).

The quantity control on the Order → Delivery form is a bounded number input, and the cap is enforced **server-side**: on submit the remainder is recomputed from the order line and the deliveries already raised against it. The delivery form previously ran no quantity validation at all.

## PDF Generation

Deliveries are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the delivery's own `pdf_template` column when set, otherwise the default chosen for **delivery** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the delivery create and edit form pins a template to the record; leaving it blank follows the Settings default.

The **"From"** contact block on the themed templates is filled from **Settings → General → Document contact details**, or from a delivery-specific override — see [Settings](/settings#document-contact-details). `classic` renders no From block on deliveries.

### Preview

A **Preview** action sits beside every download button — on the delivery show page and on index rows — rendering the real generated PDF in a slide-over with pdf.js, rather than sending you out through the browser's download tray to check a document before sending it.

Route `laravel-crm.deliveries.preview`, carrying the same `can:view` guard as its download twin, so preview grants nothing download did not. The viewer chunk and its pdf.js worker are imported lazily, so a user who never opens a preview downloads none of it.

> **Note:** Deliveries have a PDF preview but no portal page, so there is no **Get link** button on them.

## Creating a Delivery

```php
use VentureDrake\LaravelCrm\Models\Delivery;

$delivery = Delivery::create([
    'order_id' => $order->id,
    'delivery_expected' => '2026-02-01',
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Searchable by `reference`, `delivery_id`, and associated person/organisation names.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasGlobalSettings` | Global settings access |

