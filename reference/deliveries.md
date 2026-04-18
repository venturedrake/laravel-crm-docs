# Deliveries

[[toc]]

## Overview

Deliveries track the physical shipment of goods from an [Order](/reference/orders). Each delivery contains delivery products that reference order line items, allowing partial deliveries and tracking of fulfillment status.

**Model:** `VentureDrake\LaravelCrm\Models\Delivery`
**Table:** `{prefix}deliveries` (default: `crm_deliveries`)

> **Note:** The Delivery model extends `Illuminate\Database\Eloquent\Model` directly rather than the CRM base model.

## Attributes

| Attribute | Type | Description |
|---|---|---|
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

## Creating a Delivery

```php
use VentureDrake\LaravelCrm\Models\Delivery;

$delivery = Delivery::create([
    'order_id' => $order->id,
    'delivery_expected' => '2025-02-01',
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
