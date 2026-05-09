# Tax Rates

[[toc]]

## Overview

Tax rates are applied to product line items on [Quotes](/quotes), [Orders](/orders), [Invoices](/invoices), and [Purchase Orders](/purchase-orders). Each [Product](/products) can have a default tax rate which is automatically pulled into a new line item, and the rate can be overridden per line.

**Model:** `VentureDrake\LaravelCrm\Models\TaxRate`
**Table:** `{prefix}tax_rates` (default: `crm_tax_rates`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `name` | `string` | Display name (e.g. `GST`, `VAT 20%`) |
| `description` | `text` | Optional description |
| `rate` | `decimal` | Percentage rate (e.g. `10.00` for 10%) |
| `tax_type` | `string` | Type/jurisdiction of the tax |
| `default` | `boolean` | Whether this is the default rate for new products |
| `team_id` | `integer` | Team scope when teams are enabled |

## Defaults

The package's [Configuration](/configuration) exposes two settings that control how tax behaves out of the box:

| Setting | Description |
|---|---|
| `LARAVEL_CRM_TAX_NAME` | Label used on quote/invoice line items (default `Tax`) |
| `LARAVEL_CRM_TAX_RATE` | Initial default tax rate percentage |

## Managing Tax Rates

Tax rates are managed through **Settings → Tax Rates** in the CRM UI. The default rate is automatically applied to new product line items but can be overridden on a per-line basis.

## Usage

```php
use VentureDrake\LaravelCrm\Models\TaxRate;

$taxRate = TaxRate::create([
    'name' => 'GST',
    'rate' => 10.00,
    'default' => true,
]);

// Assign as the default tax rate for a product
$product->update(['tax_rate_id' => $taxRate->id]);
```

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |

