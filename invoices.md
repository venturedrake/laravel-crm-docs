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

> **Note:** All money fields are stored in cents. Since 2.7.0 a new invoice is saved with `amount_paid = 0` and `amount_due = total`, and the 2.7.0 upgrade gives every older invoice with a `NULL` balance one (see the [Upgrade Guide](/upgrading#two-invoice-balance-migrations-neither-reversible)). The `amount_due` accessor returns `total` only when the stored value is negative.

> **Note:** Since 2.7.0 a `due_date` on an earlier day than `issue_date` is rejected, in the form, the [API](/api#conventions) and the legacy request ([#106](https://github.com/venturedrake/laravel-crm/issues/106)).

> **Note:** `due_date` is nullable, and the [API](/api-invoices) accepts a null value for it. It is guarded on the invoices index, on the show page and in all five [PDF templates](/pdf-templates), and the emailed invoice body has a **no-due-date variant** rather than interpolating a formatted null. Before 2.4.1 a single null due date took down the entire invoices index for every user of the install, not only for whoever created the record.

## Recording Payments

On the invoice page, use **Pay** to record all or part of the outstanding balance. After a payment, **Amount paid** and **Balance due** appear immediately below **Total**, in the invoice's currency. These rows remain hidden until a payment is recorded or the invoice is marked fully paid.

A partially paid invoice shows a **Part paid** badge alongside its due or overdue badge, when it has a due date. Once settled, the header shows **Paid** and the balance is zero. The amounts and badges refresh immediately after recording a payment; reloading the page shows the same values.

The displayed balance is calculated from the total minus cumulative payments, with a minimum of zero, rather than the stored `amount_due` value on older invoices. A fully paid invoice always displays a zero balance.

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

## Payments

Payments are recorded from the **Pay** drawer on the invoice. Since 2.7.0 it takes part payments:

- **Any amount up to the balance** is accepted and recorded in cents. The toast says how much is still due, e.g. "Payment of $100.00 recorded. $120.00 still due." An amount over the balance is refused with a validation message ([#127](https://github.com/venturedrake/laravel-crm/issues/127)). Reopening the drawer after a part payment no longer shows an empty amount.
- **The amount paid and balance due** show on the invoice show page, the invoice lists and the [portal](#public-portal) invoice page ([#111](https://github.com/venturedrake/laravel-crm/issues/111), [#116](https://github.com/venturedrake/laravel-crm/issues/116)), and the lists refresh after a payment ([#136](https://github.com/venturedrake/laravel-crm/issues/136)).
- **Concurrent payments are serialised.**

Before 2.7.0 a part payment was always refused: the drawer checked the amount against a `NULL` balance and subtracted `amount_paid` twice, so only a payment of `0` went through.

### Paid invoices keep their total

Once a payment is recorded, an edit that would change the invoice's subtotal or tax is refused, in the form, the [API](/api-invoices) (`422` on `line_items`) and the legacy controller. A line re-sent with the product, price and quantity it was saved with keeps its saved amount and tax, so an edit that leaves the lines alone still saves, even after a product's tax rate has changed.

## Public Portal

Invoices have a public-facing portal page accessible at `/p/invoices/{external_id}`. This allows recipients to view invoices without needing a CRM login. The page renders the invoice's own [PDF template](/pdf-templates), so what the customer reads on screen is the document they download — see [Portal](/portal).

### Get link

A **Get link** button beside Preview, on the invoice show page and on index rows, hands over the same 14-day signed portal URL that gets emailed to the customer, with an optional **mark as sent** tick. Before 2.4.1 the only way to obtain that link was to send the invoice.

Since 2.7.0 a new invoice shows no **Sent** badge until it has been sent, and sending it from the Send drawer sets `sent`. The Send drawer validates recipient addresses (comma or semicolon separated) before the PDF is rendered, and keeps the drawer open with an error toast if the mail transport fails. The invoice list shares one Send drawer per page.

The modal is mounted once in the layout rather than once per row, and `confirm()` re-resolves the record through `Support\PortalLink`'s model whitelist and the invoice policy rather than trusting the component's own state — a tampered payload cannot mint a link to a record the caller may not view.

## Line Items

**Model:** `VentureDrake\LaravelCrm\Models\InvoiceLine`
**Table:** `{prefix}invoice_lines` (default: `crm_invoice_lines`)

Reached from the invoice via `invoiceLines()`.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used by the [API](/api-quotes#nested-line-items) |
| `product_id` | `integer` | The product being invoiced |
| `product_variation_id` | `integer` | Optional [product variation](/product-variations) |
| `description` | `text` | Optional line description |
| `quantity` | `decimal(15,3)` | Quantity, to at most 3 decimal places |
| `price` | `integer` | Unit price (stored in cents) |
| `tax_rate` | `decimal` | Tax rate percentage applied to the line |
| `tax_amount` | `integer` | Tax on the line (stored in cents) |
| `amount` | `integer` | Line total (stored in cents) |
| `currency` | `string(3)` | Currency code |
| `comments` | `string` | Optional per-line note, printed inside the product's cell on the show page and PDF |

> **Note:** `quantity` is `decimal(15,3)`, so an invoice line can carry `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float`.

> **Note:** A **soft-deleted product stays readable** on the invoices that already reference it. `InvoiceLine::product()` resolves `withTrashed()`, so the line keeps its description on the show view, in all 14 PDF templates and in the [Xero](/xero) sync. The product pickers query `Product::` directly, so a deleted product stays out of selection lists. See [Products](/products).

> **Note:** A line can carry a [product variation](/product-variations). The variation select appears once a product with active variations is chosen, and picking one prices the line in the invoice's currency. The show page, PDF, emailed invoice and portal print `Product — Variation`, and an invoice raised from an order keeps the order line's variation. The [Xero](/xero#line-items) sync sends the variation name in the line description.

When an invoice is raised from an [Order](/orders#drawing-down-an-order-line), the quantity is capped at the order line's outstanding remainder, recomputed server-side from the invoices already raised against it.

Since 2.7.0 a follow-up invoice from a part-invoiced order is priced from the quantity still outstanding, and skips lines already invoiced in full. The final invoice for a line bills the exact remainder, so invoicing 1.25 and then 1.25 of a 2.5 line worth 833.33 comes to 833.33 rather than 833.32. Before, a follow-up invoice billed the whole order line again.

The invoice's `subtotal`, `tax` and `total` are summed from its saved lines, as on [quotes](/quotes#document-totals). An invoice line's `amount` is `price × quantity`, rounded to the cent.

> **Note:** Before 2.6.0, updating an invoice changed the tax on its existing lines. `InvoiceService::update()` looked up the tax rate with the invoice line's own id instead of the product id, so an existing line took its tax from whichever product had that id, or got no tax when there was none. New lines were not affected. An invoice edited on an earlier version keeps the wrong tax until its lines are saved again.

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

Since 2.7.0 the list finds an invoice by its number with or without the prefix, in any case, and by its reference. `%` and `_` in a search term are matched literally.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasGlobalSettings` | Global settings access |
