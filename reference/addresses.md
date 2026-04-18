# Addresses

[[toc]]

## Overview

Addresses are polymorphic records that can belong to any addressable entity — [People](/reference/people), [Organisations](/reference/organisations), [Leads](/reference/leads), [Orders](/reference/orders), [Deliveries](/reference/deliveries), etc. Each entity can have multiple addresses with different types (billing, shipping, primary, etc.).

**Model:** `VentureDrake\LaravelCrm\Models\Address`
**Table:** `{prefix}addresses` (default: `crm_addresses`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `address_type_id` | `integer` | Address type (5 = billing, 6 = shipping) |
| `line1` | `string` | Address line 1 |
| `line2` | `string` | Address line 2 |
| `line3` | `string` | Address line 3 |
| `city` | `string` | City |
| `state` | `string` | State/province |
| `code` | `string` | Postal/ZIP code |
| `country` | `string` | Country |
| `primary` | `boolean` | Whether this is the primary address |
| `addressable_type` | `string` | Polymorphic type |
| `addressable_id` | `integer` | Polymorphic ID |

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `addressable()` | `morphTo` | `*` | The parent entity |

## Usage

Addresses are accessed through the parent model's `addresses()` relationship:

```php
// Add an address to an organisation
$organisation->addresses()->create([
    'line1' => '123 Main St',
    'city' => 'New York',
    'state' => 'NY',
    'code' => '10001',
    'country' => 'US',
    'primary' => true,
]);

// Get primary address
$address = $organisation->getPrimaryAddress();

// Get billing/shipping addresses
$billing = $organisation->getBillingAddress();   // address_type_id = 5
$shipping = $organisation->getShippingAddress(); // address_type_id = 6
```
