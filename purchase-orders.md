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

> **Note:** A **soft-deleted product stays readable** on the purchase orders that already reference it. `PurchaseOrderLine::product()` resolves `withTrashed()`, so the line keeps its description on the show view, in all 14 PDF templates and in the [Xero](/xero) sync. The product pickers query `Product::` directly, so a deleted product stays out of selection lists. See [Products](/products).

## PDF Generation

Purchase orders are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the purchase order's own `pdf_template` column when set, otherwise the default chosen for **purchase-order** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the purchase order create and edit form pins a template to the record; leaving it blank follows the Settings default.

Downloads, the emailed attachment and the [portal](/portal) render all resolve the same way, so all three agree.

> **Note:** The Settings key for this document type keeps its hyphen — `pdf_template_purchase-order`.

> **Note:** Purchase orders render **no "From" contact block** on any of the five templates. Their layouts pair a **Supplier** column with a **Delivery details** one rather than From/To, so `purchase_order_contact_details` resolves through the same chain as the other document types but prints nowhere. See [Settings → Document contact details](/settings#document-contact-details).

### Preview

A **Preview** action sits beside every download button — on the purchase order show page and on index rows — rendering the real generated PDF in a slide-over with pdf.js, rather than sending you out through the browser's download tray to check a document before sending it.

Route `laravel-crm.purchase-orders.preview`, carrying the same `can:view` guard as its download twin, so preview grants nothing download did not. The viewer chunk and its pdf.js worker are imported lazily, so a user who never opens a preview downloads none of it.

## Public Portal

Purchase orders have a supplier-facing portal page at `/p/purchase-orders/{external_id}`, so a supplier can read and respond to one without a CRM login. The page renders the purchase order's own [PDF template](/pdf-templates), so what the supplier reads on screen is the document they download — see [Portal](/portal).

### Get link

A **Get link** button beside Preview, on the purchase order show page and on index rows, hands over the same 14-day signed portal URL that gets emailed to the supplier, with an optional **mark as sent** tick. Before 2.4.1 the only way to obtain that link was to send the purchase order.

The modal is mounted once in the layout rather than once per row, and `confirm()` re-resolves the record through `Support\PortalLink`'s model whitelist and the purchase order policy rather than trusting the component's own state — a tampered payload cannot mint a link to a record the caller may not view.

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

