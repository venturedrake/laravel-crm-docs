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
| `organisation_id` | `integer` | Supplier organisation |
| `person_id` | `integer` | Supplier contact |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |

> **Note:** All money fields are stored in cents.

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `order()` | `belongsTo` | `Order` | Related customer order |
| `organisation()` | `belongsTo` | `Organisation` | Supplier |
| `person()` | `belongsTo` | `Person` | Supplier contact |
| `purchaseOrderProducts()` | `hasMany` | `PurchaseOrderProduct` | Line items |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Creating a Purchase Order

```php
use VentureDrake\LaravelCrm\Models\PurchaseOrder;

$po = PurchaseOrder::create([
    'currency' => 'USD',
    'subtotal' => 5000,
    'tax' => 500,
    'total' => 5500,
    'order_id' => $order->id,
    'organisation_id' => $supplier->id,
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

