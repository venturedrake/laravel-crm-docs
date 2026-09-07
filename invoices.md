# Invoices

[[toc]]

## Overview

Invoices represent billing documents generated from [Orders](/orders). An invoice contains line items and tracks payment status including amount due, amount paid, and full payment date. Invoices support integration with external accounting systems like [Xero](/xero).

**Model:** `VentureDrake\LaravelCrm\Models\Invoice`
**Table:** `{prefix}invoices` (default: `crm_invoices`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `invoice_id` | `string` | Auto-generated invoice number (prefix + number) |
| `reference` | `string` | External reference |
| `currency` | `string` | Currency code |
| `subtotal` | `integer` | Subtotal (stored in cents) |
| `tax` | `integer` | Tax (stored in cents) |
| `total` | `integer` | Total (stored in cents) |
| `amount_due` | `integer` | Amount due (stored in cents, defaults to total) |
| `amount_paid` | `integer` | Amount paid (stored in cents) |
| `issue_date` | `datetime` | Issue date |
| `due_date` | `datetime` | Payment due date — **nullable** |
| `fully_paid_at` | `datetime` | Date fully paid |
| `person_id` | `integer` | Contact person |
| `organization_id` | `integer` | Organisation |
| `order_id` | `integer` | Source order |
| `user_owner_id` | `integer` | Owner user |
| `user_assigned_id` | `integer` | Assigned user |

> **Note:** All money fields are stored in cents. The `amount_due` accessor returns `total` if no explicit value is set.

> **Note:** `due_date` is nullable, and the [API](/api-invoices) accepts a null value for it. It is guarded on the invoices index, on the show page and in all five [PDF templates](/pdf-templates), and the emailed invoice body has a **no-due-date variant** rather than interpolating a formatted null. Before 2.4.1 a single null due date took down the entire invoices index for every user of the install, not only for whoever created the record.

## Computed Attributes

### title

Returns a formatted title with the monetary total and organisation/person name.

```php
$invoice->title; // "$1,500.00 - Acme Corp"
```

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Contact person |
| `organization()` | `belongsTo` | `Organization` | Organisation |
| `order()` | `belongsTo` | `Order` | Source order |
| `invoiceLines()` | `hasMany` | `InvoiceLine` | Line items |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `customFieldValues()` | `morphMany` | `FieldValue` | Custom field values |
| `xeroInvoice()` | `hasOne` | `XeroInvoice` | Xero integration |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Public Portal

Invoices have a public-facing portal page accessible at `/p/invoices/{external_id}`. This allows recipients to view invoices without needing a CRM login. The page renders the invoice's own [PDF template](/pdf-templates), so what the customer reads on screen is the document they download — see [Portal](/portal).

### Get link

A **Get link** button beside Preview, on the invoice show page and on index rows, hands over the same 14-day signed portal URL that gets emailed to the customer, with an optional **mark as sent** tick. Before 2.4.1 the only way to obtain that link was to send the invoice.

The modal is mounted once in the layout rather than once per row, and `confirm()` re-resolves the record through `Support\PortalLink`'s model whitelist and the invoice policy rather than trusting the component's own state — a tampered payload cannot mint a link to a record the caller may not view.

## Line Items

**Model:** `VentureDrake\LaravelCrm\Models\InvoiceLine`
**Table:** `{prefix}invoice_lines` (default: `crm_invoice_lines`)

Reached from the invoice via `invoiceLines()`.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used by the [API](/api-quotes#nested-line-items) |
| `product_id` | `integer` | The product being invoiced |
| `product_variation_id` | `integer` | Optional [product variation](/product-attributes) |
| `description` | `text` | Optional line description |
| `quantity` | `decimal(15,3)` | Quantity, to at most 3 decimal places |
| `price` | `integer` | Unit price (stored in cents) |
| `tax_rate` | `decimal` | Tax rate percentage applied to the line |
| `tax_amount` | `integer` | Tax on the line (stored in cents) |
| `amount` | `integer` | Line total (stored in cents) |
| `currency` | `string(3)` | Currency code |
| `comments` | `string` | Optional per-line note |

> **Note:** `quantity` is `decimal(15,3)`, so an invoice line can carry `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float`.

> **Note:** A **soft-deleted product stays readable** on the invoices that already reference it. `InvoiceLine::product()` resolves `withTrashed()`, so the line keeps its description on the show view, in all 14 PDF templates and in the [Xero](/xero) sync. The product pickers query `Product::` directly, so a deleted product stays out of selection lists. See [Products](/products).

When an invoice is raised from an [Order](/orders#drawing-down-an-order-line), the quantity is capped at the order line's outstanding remainder, recomputed server-side from the invoices already raised against it.

## PDF Generation

Invoices are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the invoice's own `pdf_template` column when set, otherwise the default chosen for **invoice** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the invoice create and edit form pins a template to the record; leaving it blank follows the Settings default.

Downloads, the emailed attachment and the [portal](/portal) render all resolve the same way, so all three agree.

The **"From"** contact block is filled from **Settings → Invoices → Contact details**, falling back to the shared **Settings → General → Document contact details** — see [Settings](/settings#document-contact-details). Invoices are the one document type whose From block renders on *every* template, `classic` included.

### Preview

A **Preview** action sits beside every download button — on the invoice show page and on index rows — rendering the real generated PDF in a slide-over with pdf.js, rather than sending you out through the browser's download tray to check a document before sending it.

Route `laravel-crm.invoices.preview`, carrying the same `can:view` guard as its download twin, so preview grants nothing download did not. The viewer chunk and its pdf.js worker are imported lazily, so a user who never opens a preview downloads none of it.

## Creating an Invoice

```php
use VentureDrake\LaravelCrm\Models\Invoice;

$invoice = Invoice::create([
    'currency' => 'USD',
    'subtotal' => 10000,
    'tax' => 1000,
    'total' => 11000,
    'issue_date' => '2026-01-15',
    'due_date' => '2026-02-15',
    'order_id' => $order->id,
    'person_id' => $person->id,
    'organization_id' => $organization->id,
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Searchable by `reference`, `invoice_id`, and associated person/organisation names. Filterable by `user_owner_id` and `labels.id`.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasGlobalSettings` | Global settings access |
