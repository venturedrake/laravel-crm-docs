# Quotes

[[toc]]

## Overview

Quotes represent formal proposals sent to clients with pricing and terms. A quote can be created from a [Deal](/deals) and, once accepted, can be converted into an [Order](/orders). Quotes progress through pipeline stages and track acceptance/rejection status.

**Model:** `VentureDrake\LaravelCrm\Models\Quote`
**Table:** `{prefix}quotes` (default: `crm_quotes`)

## Attributes

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used in URLs (auto-generated) |
| `quote_id` | `string` | Auto-generated quote number (prefix + number) |
| `title` | `string` | Quote title |
| `reference` | `string` | External reference |
| `description` | `text` | Description |
| `currency` | `string` | Currency code |
| `subtotal` | `integer` | Subtotal (stored in cents) |
| `discount` | `integer` | Discount amount (stored in cents) |
| `tax` | `integer` | Tax amount (stored in cents) |
| `adjustments` | `integer` | Adjustments (stored in cents) |
| `total` | `integer` | Total (stored in cents) |
| `issue_at` | `datetime` | Issue date |
| `expire_at` | `datetime` | Expiry date |
| `accepted_at` | `datetime` | When accepted |
| `rejected_at` | `datetime` | When rejected |
| `person_id` | `integer` | Contact person |
| `organization_id` | `integer` | Organisation |
| `customer_id` | `integer` | v1 [customer](/customers), if any (`client_id` before the v2 upgrade) |
| `deal_id` | `integer` | Source deal |
| `pipeline_id` | `integer` | Pipeline |
| `pipeline_stage_id` | `integer` | Pipeline stage |

> **Note:** All money fields are stored in cents. Mutators automatically multiply by 100 on set.

> **Note:** Since 2.7.0 an `expire_at` on an earlier day than `issue_at` is rejected, in the form and the [API](/api#conventions) ([#106](https://github.com/venturedrake/laravel-crm/issues/106)).

## Relationships

| Method | Type | Related Model | Description |
|---|---|---|---|
| `person()` | `belongsTo` | `Person` | Contact person |
| `organization()` | `belongsTo` | `Organization` | Organisation |
| `client()` | `belongsTo` | `Customer` | v1 leftover; always `null` — see [Customers](/customers#from-other-models) |
| `deal()` | `belongsTo` | `Deal` | Source deal |
| `quoteProducts()` | `hasMany` | `QuoteProduct` | Line items |
| `orders()` | `hasMany` | `Order` | Orders generated from this quote |
| `pipeline()` | `belongsTo` | `Pipeline` | Pipeline |
| `pipelineStage()` | `belongsTo` | `PipelineStage` | Pipeline stage |
| `labels()` | `morphToMany` | `Label` | Labels/tags |
| `customFieldValues()` | `morphMany` | `FieldValue` | Custom field values |
| `ownerUser()` | `belongsTo` | `User` | Owner |
| `assignedToUser()` | `belongsTo` | `User` | Assigned user |

## Helper Methods

### orderComplete()

Returns `true` if all quote products have been fully converted to order products across all related orders.

```php
if ($quote->orderComplete()) {
    // All items have been ordered
}
```

## Public Portal

Quotes have a public-facing portal page accessible at `/p/quotes/{external_id}`. This allows recipients to view and accept or reject quotes without needing a CRM login. The page renders the quote's own [PDF template](/pdf-templates), so what the customer reads on screen is the document they download — see [Portal](/portal).

Since 2.7.0 the portal refuses a response to a quote that is already accepted or rejected, or has an order. An expired quote can't be accepted, but it can still be rejected. The expiry day itself still counts as valid.

### Get link

A **Get link** button beside Preview, on the quote show page and on index rows, hands over the same 14-day signed portal URL that gets emailed to the customer. Before 2.4.1 the only way to obtain that link was to send the quote.

The modal is mounted once in the layout rather than once per row, and `confirm()` re-resolves the record through `Support\PortalLink`'s model whitelist and the quote policy rather than trusting the component's own state — a tampered payload cannot mint a link to a record the caller may not view.

> **Note:** The **mark as sent** tick offered on invoices and purchase orders is not offered on quotes: quotes carry no `sent` column, so there is nothing to tick.

## Line Items

**Model:** `VentureDrake\LaravelCrm\Models\QuoteProduct`
**Table:** `{prefix}quote_products` (default: `crm_quote_products`)

Reached from the quote via `quoteProducts()`.

| Attribute | Type | Description |
|---|---|---|
| `external_id` | `string` | UUID used by the [API](/api-quotes#nested-line-items) |
| `product_id` | `integer` | The product being quoted |
| `product_variation_id` | `integer` | Optional [product variation](/product-variations) |
| `quantity` | `decimal(15,3)` | Quantity, to at most 3 decimal places |
| `price` | `integer` | Unit price (stored in cents) |
| `tax_rate` | `decimal` | Tax rate percentage applied to the line |
| `tax_amount` | `integer` | Tax on the line (stored in cents) |
| `amount` | `integer` | Line total (stored in cents) |
| `currency` | `string(3)` | Currency code |
| `comments` | `string` | Optional per-line note, printed inside the product's cell on the show page and PDF |
| `order` | `integer` | Position of the line on the document, set by dragging lines on the form |

> **Note:** `quantity` is `decimal(15,3)`, so a product sold by weight or volume can be quoted at `3.5` Kg or `0.25` L. The `HasDecimalQuantity` trait casts it, which means `$line->quantity` reads back as a PHP `float` — a whole quantity of 2 compares as `2.0`. Values are rounded to 3 decimal places on write, and a whole quantity still renders as `2` rather than `2.000`.

> **Note:** A **soft-deleted product stays readable** on the quotes that already reference it. `QuoteProduct::product()` resolves `withTrashed()`, so the line keeps its description on the show view, in all 14 PDF templates and in the [Xero](/xero) sync. The product pickers query `Product::` directly, so a deleted product stays out of selection lists — readable on the documents that already reference it, not choosable on new ones. See [Products](/products).

### Variations

When a line's product has active [variations](/product-variations), a variation select appears beside the product. Picking one prices the line from the variation's price in the quote's currency, falling back to the product price. The show page, the PDF, the emailed quote and the portal print the line as `Product — Variation` (`displayName()`), and the show page prints the variation's SKU when it has one (`displayCode()`). Converting the quote to an order carries the variation across.

### Document totals

Since 2.7.0 a quote's `subtotal`, `tax` and `total` come from the lines it saved. `QuoteService::create()` and `update()` sum the stored lines' `amount` and `tax_amount` in cents (`Support\DocumentTotals::fromLines()`) instead of storing the totals the form or API posted, so a line with quantity 0 no longer leaves its amount in the header. Line amounts are computed in integer cents in one place (`Support\LineAmount`), so the form and the "check amounts" badge agree on half-cent boundaries (333.33 × 2.5 = 833.33). Orders, invoices and purchase orders work the same way.

The product picker on each line searches the catalogue on the server, see [Products → Line-item product picker](/products#line-item-product-picker). Save sends the current rows with the request, so a quantity, price or variation changed just before Save is kept.

### Creating a quote from a deal

Since 2.5.0 a quote created from a [Deal](/deals) starts with the deal's lines — product, variation, quantity and price — and they stay editable. Before, the quote form started empty and the deal's products were dropped.

## PDF Generation

Quotes are exported as PDF documents through `barryvdh/laravel-dompdf`, rendered with one of the five shipped [PDF Templates](/pdf-templates).

The template is resolved per record: the quote's own `pdf_template` column when set, otherwise the default chosen for **quote** under **Settings → Templates**, otherwise a PDF view the host has published and customised, otherwise `modern`. A **PDF template** select on the quote create and edit form pins a template to the record; leaving it blank follows the Settings default.

Downloads, the emailed attachment and the [portal](/portal) render all resolve the same way, so all three agree.

The **"From"** contact block on the themed templates is filled from **Settings → General → Document contact details**, or from a quote-specific override — see [Settings](/settings#document-contact-details). `classic` renders no From block on quotes.

### Preview

A **Preview** action sits beside every download button — on the quote show page and on index rows — rendering the real generated PDF in a slide-over with pdf.js, rather than sending you out through the browser's download tray to check a document before sending it.

Route `laravel-crm.quotes.preview`, carrying the same `can:view` guard as its download twin, so preview grants nothing download did not. The viewer chunk and its pdf.js worker are imported lazily, so a user who never opens a preview downloads none of it.

## Sending and Status

A quote is sent from the **Send** drawer on its show page, the quote list or the board. The list and the board share one drawer per page. Recipient addresses are validated before the PDF is rendered, and several can be given separated by commas or semicolons. If the mail transport fails, an error toast shows and the drawer stays open. The email subject and body name the quote by its number.

Since 2.7.0 a quote's pipeline stage follows its status:

- Sending a **Draft** quote moves it to **Sent** once the email has gone.
- Accepting, rejecting, unaccepting and unrejecting, from the CRM or the [portal](#public-portal), all go through `QuoteStateService`, which updates the stage too.

### Locked Quotes

An accepted quote, or one with an order made from it, is read-only:

- The **Edit** button is disabled with the reason, and the edit route redirects to the show page.
- `QuoteService::update()` refuses the change, and the [API](/api-quotes) returns a `422`.
- A quote with an order shows an **Ordered** badge, and can't be unaccepted or deleted (the API returns a `403` on delete).

To change an accepted quote that has no order yet, unaccept it first. Before 2.7.0 accepted and ordered quotes could still be edited, unaccepted or deleted. An order or invoice whose source quote was deleted shows the quote's number with a **deleted** badge instead of a broken link.

## Views

Quotes support both **list view** (paginated table) and **board view** (Kanban-style pipeline stages). The board renders a page of cards per column with **Load more**, see [Pipelines → Board View](/pipelines#board-view).

## Creating a Quote

```php
use VentureDrake\LaravelCrm\Models\Quote;

$quote = Quote::create([
    'title' => 'Website Redesign Proposal',
    'currency' => 'USD',
    'subtotal' => 10000,
    'tax' => 1000,
    'total' => 11000,
    'issue_at' => '2026-01-15',
    'expire_at' => '2026-02-15',
    'person_id' => $person->id,
    'organization_id' => $organization->id,
    'user_owner_id' => auth()->id(),
]);
```

## Searching & Filtering

Searchable by `quote_id`, `title`, `reference`, and associated person/organisation names. Filterable by `user_owner_id` and `labels.id`.

Since 2.7.0 the list and board find a quote by its number with or without the prefix, in any case, so "QU-3002", "3002" and "qu-3002" all match. `%` and `_` in a search term are matched literally.

## Traits

| Trait | Description |
|---|---|
| `SoftDeletes` | Soft delete support |
| `BelongsToTeams` | Multi-tenant team scoping |
| `HasCrmFields` | Custom field support |
| `SearchFilters` | Search and filter capabilities |
| `HasCrmActivities` | Activity timeline tracking |
| `HasGlobalSettings` | Global settings access |
