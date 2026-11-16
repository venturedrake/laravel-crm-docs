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
| `customer_id` | `integer` | v1 [customer](/customers), if any (`client_id` before the v2 upgrade) |
| `lead_id` | `integer` | Source lead |
| `deal_id` | `integer` | Source deal |
| `quote_id` | `integer` | Source quote |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |

> **Note:** All money fields are stored in cents. Mutators automatically multiply by 100 on set.

## Computed Attributes

### title

Returns a formatted title with the monetary total and the organisation name, or the person name when there is no organisation.

```php
$order->title; // "$1,500.00 - Acme Corp"
```

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Contact person |
| `organization()` | `belongsTo` | `Organization` | Organisation |
| `client()` | `belongsTo` | `Customer` | v1 leftover; always `null` — see [Customers](/customers#from-other-models) |
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
| `product_variation_id` | `integer` | Optional [product variation](/product-variations) |
| `quote_product_id` | `integer` | The quote line this was drawn from, if any |
| `quantity` | `decimal(15,3)` | Quantity, to at most 3 decimal places |
| `price` | `integer` | Unit price (stored in cents) |
| `tax_rate` | `decimal` | Tax rate percentage applied to the line |
| `tax_amount` | `integer` | Tax on the line (stored in cents) |
| `amount` | `integer` | Line total (stored in cents) |
| `currency` | `string(3)` | Currency code |
| `comments` | `string` | Optional per-line note, printed inside the product's cell on the show page and PDF |
| `order` | `integer` | Position of the line on the document, set by dragging lines on the form |

> **Note:** `quantity` is `decimal(15,3)`, so an order line can carry `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float`.

> **Note:** A **soft-deleted product stays readable** on the orders that already reference it. `OrderProduct::product()` resolves `withTrashed()`, so the line keeps its description on the show view, in all 14 PDF templates and in the [Xero](/xero) sync. The product pickers query `Product::` directly, so a deleted product stays out of selection lists. See [Products](/products).

> **Note:** A line can carry a [product variation](/product-variations). The variation select appears once a product with active variations is chosen, and picking one prices the line in the order's currency. The show page and PDF print `Product — Variation`, and the show page also prints the variation's SKU when it has one. The variation is carried across from the quote, and on to invoices, purchase orders and delivery lines.

### Drawing down an order line

The Order → Invoice and Order → Delivery forms let you invoice or deliver part of an order line, and the quantity control is a **bounded number input** rather than a dropdown — a dropdown built by an integer loop cannot express 3.5, so an order line of 2.5 could only ever be invoiced as 2, leaving 0.5 outstanding forever.

The cap is enforced **server-side**. On submit the remainder is recomputed from the order line and the invoices or deliveries already raised against it, and the submitted quantity is checked against that — not against the row's own `quantity_max`, which is a public Livewire property and therefore whatever the caller sends back. Previously the cap existed only in the browser, so an over-invoice was reachable by posting the form directly. The delivery form, which ran no validation at all, now validates its quantities too.

Since 2.7.0 an invoice raised from a part-invoiced line is priced from the quantity still outstanding, and the final one bills the exact remainder. See [Invoices → Line Items](/invoices#line-items).

`invoiceComplete()` and `deliveryComplete()` compare the drawn-down totals within half the smallest storable unit rather than with `> 0`, so floating-point residue from a split fulfilment cannot leave a document reading as "not fully invoiced" forever.

The order's `subtotal`, `tax` and `total` are summed from its saved lines, as on [quotes](/quotes#document-totals).

## Addresses

Since 2.7.0 an address left blank on the order form is not saved, even though the form prefills its country, and blanking an address on edit removes it ([#115](https://github.com/venturedrake/laravel-crm/issues/115)). The 2.7.0 upgrade soft-deletes the country-only addresses saved before, see the [Upgrade Guide](/upgrading#country-only-order-and-delivery-addresses-are-removed).

## Deleting an Order

Since 2.7.0 an order with invoices, deliveries or purchase orders can't be deleted ([#102](https://github.com/venturedrake/laravel-crm/issues/102)). `OrderPolicy::delete` refuses with the reason, the **Delete** button is disabled with it, and the [API](/api-orders) returns a `403`. Soft-deleted invoices, deliveries and purchase orders do not count. An invoice, delivery or purchase order whose source order was deleted earlier shows the order's number with a **deleted** badge instead of a broken link.

## Form Options

Since 2.6.0 you can trim the order create and edit form with the `orders.form` keys in `config/laravel-crm.php`. Every key defaults to the full form, so an install that sets nothing sees no change.

```php
'orders' => [
    'form' => [
        'description' => true,        // false hides the description textarea
        'labels' => true,             // false hides the labels picker
        'custom_fields' => true,      // false hides both custom field blocks
        'unallocated_owner' => true,  // false removes "Unallocated" from the owner select
        'addresses' => ['billing', 'shipping'], // which address tabs to show and save
    ],
],
```

- **`unallocated_owner`** set to `false` removes "Unallocated" from the owner select, presets an order with no owner to the signed-in user, and makes the owner required.
- **`addresses`** lists the address tabs to show and save, by address type name (`billing`, `shipping`), not id. With one listed it is shown without the tab switcher. An empty array hides the addresses card.

**Hiding a field never changes stored data.** Validation and save skip a hidden field rather than clearing it, so an existing description, labels, custom field values and an address of an unlisted type survive an edit. An order created from a [quote](/quotes) still copies the quote's description when the field is hidden.

The keys are install-wide, including on teams installs, and are not in the Settings UI. The REST API and the legacy controller are not affected: `OrderService::create()` and `update()` take the options as an optional `FormOptions` argument and save every field when it is not passed. The quote, invoice and purchase-order forms are not covered yet; they will take the same `form` shape under their own key.

For a host that only ships to customers and does not use descriptions, labels or custom fields on orders:

```php
'orders' => [
    'form' => [
        'description' => false,
        'labels' => false,
        'custom_fields' => false,
        'unallocated_owner' => false,
        'addresses' => ['shipping'],
    ],
],
```

> **Note:** If you have published `livewire/orders/order-form.blade.php`, re-publish it along with the new `livewire/orders/partials/order-address-fields.blade.php`. A frozen copy still shows every field, but a save skips the hidden ones, so an edit to a field that is still on screen is silently not saved. See the [Upgrade Guide](/upgrading#upgrading-to-260).

## PDF Generation

Orders are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the order's own `pdf_template` column when set, otherwise the default chosen for **order** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the order create and edit form pins a template to the record; leaving it blank follows the Settings default.

The **"From"** contact block on the themed templates is filled from **Settings → General → Document contact details**, or from an order-specific override — see [Settings](/settings#document-contact-details). `classic` renders no From block on orders.

### Preview

A **Preview** action sits beside every download button — on the order show page and on index rows — rendering the real generated PDF in a slide-over with pdf.js, rather than sending you out through the browser's download tray to check a document before sending it.

Route `laravel-crm.orders.preview`, carrying the same `can:view` guard as its download twin, so preview grants nothing download did not. The viewer chunk and its pdf.js worker are imported lazily, so a user who never opens a preview downloads none of it.

> **Note:** Orders have a PDF preview but no portal page, so there is no **Get link** button on them.

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

Since 2.7.0 the list finds an order by its number with or without the prefix, in any case, and by its reference. `%` and `_` in a search term are matched literally.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
