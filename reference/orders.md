# Orders

[[toc]]

## Overview

Orders represent confirmed sales. An order can be created from a [Quote](/reference/quotes) or independently, and contains line items referencing [Products](/reference/products). Orders can generate [Invoices](/reference/invoices) and [Deliveries](/reference/deliveries), and the model tracks fulfillment completion for both.

**Model:** `VentureDrake\LaravelCrm\Models\Order`
**Table:** `{prefix}orders` (default: `crm_orders`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
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
| `organisation_id` | `integer` | Organisation |
| `client_id` | `integer` | Client |
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
| `organisation()` | `belongsTo` | `Organisation` | Organisation |
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

## Creating an Order

```php
use VentureDrake\LaravelCrm\Models\Order;

$order = Order::create([
    'currency' => 'USD',
    'subtotal' => 10000,
    'tax' => 1000,
    'total' => 11000,
    'person_id' => $person->id,
    'organisation_id' => $organisation->id,
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
