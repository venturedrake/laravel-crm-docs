# Orders

[[toc]]

## Overview

Orders represent confirmed sales. An order can be created from a [Quote](/quotes) or independently, and contains line items referencing [Products](/products). Orders can generate [Invoices](/invoices) and [Deliveries](/deliveries), and the model tracks fulfillment completion for both.

**Model:** `VentureDrake\LaravelCrm\Models\Order`
**Table:** `{prefix}orders` (default: `crm_orders`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `order_id` | `string` | Auto-generated order number (prefix + number) |
| `reference` | `string` | External reference |
| `description` | `text` | Description |
| `currency` | `string` | Currency code |
| `subtotal` | `integer` | Subtotal (stored in cents) |
| `discount` | `integer` | Discount (stored in cents) |
| `tax` | `integer` | Tax (stored in cents) |
| `adjustments` | `integer` | Adjustments (stored in cents) |
| `total` | `integer` | Total (stored in cents) |
| `person_id` | `integer` | Contact person |
| `organization_id` | `integer` | Organisation |
| `client_id` | `integer` | Client |
| `lead_id` | `integer` | Source lead |
| `deal_id` | `integer` | Source deal |
| `quote_id` | `integer` | Source quote |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |

> **Note:** All money fields are stored in cents. Mutators automatically multiply by 100 on set.

## Computed Attributes

### title

Returns a formatted title with the monetary total and client/organisation name.

```php
$order->title; // "$1,500.00 - Acme Corp"
```

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Contact person |
| `organization()` | `belongsTo` | `Organization` | Organisation |
| `client()` | `belongsTo` | `Client` | Client |
| `deal()` | `belongsTo` | `Deal` | Source deal |
| `quote()` | `belongsTo` | `Quote` | Source quote |
| `orderProducts()` | `hasMany` | `OrderProduct` | Line items |
| `invoices()` | `hasMany` | `Invoice` | Invoices for this order |
| `deliveries()` | `hasMany` | `Delivery` | Deliveries for this order |
| `purchaseOrders()` | `hasMany` | `PurchaseOrder` | Purchase orders |
| `addresses()` | `morphMany` | `Address` | Billing/shipping addresses |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `customFieldValues()` | `morphMany` | `FieldValue` | Custom field values |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Helper Methods

### invoiceComplete()

Returns `true` if all order products have been fully invoiced.

```php
if ($order->invoiceComplete()) {
    // All items have been invoiced
}
```

### deliveryComplete()

Returns `true` if all order products have been fully delivered.

```php
if ($order->deliveryComplete()) {
    // All items have been shipped
}
```

### getBillingAddress()

Returns the billing address (address type 5).

### getShippingAddress()

Returns the shipping address (address type 6).

## Line Items

**Model:** `VentureDrake\LaravelCrm\Models\OrderProduct`
**Table:** `{prefix}order_products` (default: `crm_order_products`)

Reached from the order via `orderProducts()`.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used by the [API](/api-quotes#nested-line-items) |
| `product_id` | `integer` | The product being ordered |
| `product_variation_id` | `integer` | Optional [product variation](/product-attributes) |
| `quote_product_id` | `integer` | The quote line this was drawn from, if any |
| `quantity` | `decimal(15,3)` | Quantity, to at most 3 decimal places |
| `price` | `integer` | Unit price (stored in cents) |
| `tax_rate` | `decimal` | Tax rate percentage applied to the line |
| `tax_amount` | `integer` | Tax on the line (stored in cents) |
| `amount` | `integer` | Line total (stored in cents) |
| `currency` | `string(3)` | Currency code |
| `comments` | `string` | Optional per-line note |
| `order` | `integer` | Position of the line on the document |

> **Note:** `quantity` is `decimal(15,3)`, so an order line can carry `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float`.

### Drawing down an order line

The Order → Invoice and Order → Delivery forms let you invoice or deliver part of an order line, and the quantity control is a **bounded number input** rather than a dropdown — a dropdown built by an integer loop cannot express 3.5, so an order line of 2.5 could only ever be invoiced as 2, leaving 0.5 outstanding forever.

The cap is enforced **server-side**. On submit the remainder is recomputed from the order line and the invoices or deliveries already raised against it, and the submitted quantity is checked against that — not against the row's own `quantity_max`, which is a public Livewire property and therefore whatever the caller sends back. Previously the cap existed only in the browser, so an over-invoice was reachable by posting the form directly. The delivery form, which ran no validation at all, now validates its quantities too.

`invoiceComplete()` and `deliveryComplete()` compare the drawn-down totals within half the smallest storable unit rather than with `> 0`, so floating-point residue from a split fulfilment cannot leave a document reading as "not fully invoiced" forever.

## PDF Generation

Orders are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the order's own `pdf_template` column when set, otherwise the default chosen for **order** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the order create and edit form pins a template to the record; leaving it blank follows the Settings default.

## Creating an Order

```php
use VentureDrake\LaravelCrm\Models\Order;

$order = Order::create([
    'currency' => 'USD',
    'subtotal' => 10000,
    'tax' => 1000,
    'total' => 11000,
    'person_id' => $person->id,
    'organization_id' => $organization->id,
    'quote_id' => $quote->id,
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Searchable by `reference`, `order_id`, and associated person/organisation names. Filterable by `user_owner_id` and `labels.id`.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
