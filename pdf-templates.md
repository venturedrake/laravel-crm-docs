# PDF Templates

[[toc]]

## Overview

Every document the CRM can render as a PDF — [Quotes](/quotes), [Orders](/orders), [Invoices](/invoices), [Deliveries](/deliveries) and [Purchase Orders](/purchase-orders) — renders through one of five shipped templates rather than the single hardcoded layout each document type used to carry.

| Template | Slug |
|---|---|
| Modern (default) | `modern` |
| Classic | `classic` |
| Bold | `bold` |
| Compact | `compact` |
| Professional | `professional` |

The five document types are `invoice`, `order`, `purchase-order`, `delivery` and `quote`. Every template renders every document type, so any of the 25 combinations is valid.

Templates are defined by `VentureDrake\LaravelCrm\Support\PdfTemplateRegistry`, which is the single source of truth for the slug list (`PdfTemplateRegistry::SLUGS`), the document types (`DOC_TYPES`) and the resolution rules below.

> **Note:** `classic` is a thin `@include` of the pre-2.4.0 layouts. If you had published and edited those views, choosing **Classic** keeps your customisation while still opting the record into the picker.

## Settings → Templates

The admin picker lives at `/crm/settings/templates` (route `laravel-crm.settings.templates.edit`) and sets the **default** template for each document type.

- One tab per document type, each showing the five templates as selectable cards with packaged thumbnails.
- A **live full-page preview** streams a real PDF for any template × document type pair, rendered from `Support\PdfSampleData`. Nothing is written to the database to produce a preview.
- The chosen tab is query-string synced, so `?tab=order` deep-links cleanly.

> **Important:** Saving the form writes a choice for **all five** document types at once, not just the tab you are looking at. That matters if you are relying on a published-view override — see [Customising via a published view](#customising-via-a-published-view).

Each choice is stored as a setting named `pdf_template_{docType}`. The hyphen in `purchase-order` is kept, so the key is `pdf_template_purchase-order`.

## Per-record template

Each of the five document tables carries a nullable `pdf_template` column:

| Table |
|---|
| `crm_invoices` |
| `crm_orders` |
| `crm_purchase_orders` |
| `crm_deliveries` |
| `crm_quotes` |

The create and edit form for each document renders a **PDF template** select. Leaving it blank means the record has pinned nothing and follows the Settings default; the blank option is labelled with the template that choice currently resolves to (for example *Default (Bold)*).

Records created before 2.4.0 carry a `null` `pdf_template` and so keep tracking Settings, which is why the picker is not pre-filled with the resolved default — doing that would silently pin a template onto every record the first time its form was saved.

Only the document forms carry the picker. Every other writer — the [REST API](/api), the legacy controllers, the multi-purchase-order split — submits nothing for this field and leaves whatever the record already carries alone.

## Resolution order

`PdfTemplateRegistry::viewForModel()` answers one question — which Blade view renders this document — in this order:

1. The **record's own** `pdf_template`, when it holds a slug this package still ships.
2. The **Settings → Templates default** for that document type, when one has been saved.
3. A **published view the host has customised** — see below.
4. `modern`.

Steps 1 and 2 are *explicit choices*. Step 3 only applies when nobody has chosen anything, which is what lets a hand-customised view act as the default without overriding a real choice.

An unknown or removed slug falls back to `modern` rather than erroring, so a persisted preference for a template that no longer ships never 500s a download route.

## Customising via a published view

Before 2.4.0 the only way to restyle a PDF was to publish and edit the document's view — `resources/views/vendor/laravel-crm/invoices/pdf.blade.php` and its four siblings. Those hosts keep their layout.

The check is a content comparison, not a presence check: the published file counts as an override only **while it differs from the packaged copy** (`md5_file()` on both). A byte-identical `vendor:publish --tag=views` is not an override, because publishing copies the whole views directory and honouring presence alone would pin a host that published last week to the old layout forever.

| Legacy view | Document type |
|---|---|
| `invoices/pdf.blade.php` | `invoice` |
| `orders/pdf.blade.php` | `order` |
| `purchase-orders/pdf.blade.php` | `purchase-order` |
| `deliveries/pdf.blade.php` | `delivery` |
| `quotes/pdf.blade.php` | `quote` |

An explicit choice — on the record or in Settings — still wins over the published view. The Templates page warns on any tab where an override is currently in effect, because saving the form retires it for every document type at once.

> **Tip:** To keep your customised view *and* opt into the picker, choose **Classic**. It `@include`s the original views, so a published override of those files still applies.

## Where templates are applied

Every surface that renders a document PDF resolves through the registry, so a downloaded PDF, an emailed one and a customer-facing one always agree:

| Surface | Resolved by |
|---|---|
| Document downloads | `QuoteController`, `OrderController`, `InvoiceController`, `DeliveryController`, `PurchaseOrderController` |
| The **PDF preview drawer** | The same five controllers, through the shared `Concerns\ServesPdfDocuments` |
| Emailed PDF attachments | `SendQuote` / `SendInvoice` / `SendPurchaseOrder`, and their Livewire equivalents |
| [Portal](/portal) downloads | `Portal\QuoteController`, `Portal\InvoiceController`, `Portal\PurchaseOrderController` |
| [Portal](/portal) document pages | `Support\PortalDocument`, rendering the same view into an iframe |

> **Note:** Before 2.4.0 the send components loaded `laravel-crm::quotes.pdf` and friends directly, so an emailed attachment could differ from the document the sender had just downloaded. They no longer can.

> **Note:** Preview and download build **one** PDF served two ways — `ServesPdfDocuments::buildPdf()` generates it, and the two actions differ only in their `Content-Disposition`. They cannot drift apart as the templates change. Likewise the portal page and the portal download resolve through the same `viewForModel()` call with the same view data.

## The "From" contact block

The themed templates print a **From** contact block above the document body, filled from **Settings → General → Document contact details** — a shared `pdf_contact_details` setting, overridable per document type. See [Settings → Document contact details](/settings#document-contact-details) for the resolution chain and for how to clear a field.

Where the block renders is not uniform, and the gaps are deliberate:

| Template | Quote | Order | Delivery | Invoice | Purchase order |
|---|---|---|---|---|---|
| `classic` | — | — | — | Yes | — |
| `modern` | Yes | Yes | Yes | Yes | — |
| `bold` | Yes | Yes | Yes | Yes | — |
| `compact` | Yes | Yes | Yes | Yes | — |
| `professional` | Yes | Yes | Yes | Yes | — |

`classic` reproduces the pre-2.4.0 layouts unchanged, where only the invoice blade ever carried a From block. Purchase-order layouts pair a **Supplier** column with a **Delivery details** one rather than From/To, so no purchase-order blade reads the value on any template.

That is the answer to *"why is my quote's From block empty on Classic?"* — the setting is resolving fine; that template does not render it.

## Thumbnails

The picker's artwork is served through `laravel-crm.settings.templates.thumbnail`, which prefers the host's published copy in `public/vendor/laravel-crm/img/pdf-templates` and falls back to the copy inside the package. A host whose published assets predate the artwork still sees the thumbnails without re-publishing.

The SVGs ship in `resources/assets/img/pdf-templates` and publish to `public/vendor/laravel-crm/img/pdf-templates`.

> **Note:** **2.4.0 shipped with no thumbnail artwork at all**, so the picker rendered five broken images on every install. The SVGs then lived in `public/vendor/laravel-crm/img/pdf-templates` — which is the Vite build output directory, and `vite.config.js` sets `emptyOutDir: true`, so the `npm run build` that preceded the tag deleted them. That also made the "fall back to the copy inside the package" path inert on arrival: neither copy existed, so the route 404'd for every slug. 2.4.1 moved the artwork to a publish source the build cannot reach, and `laravelcrm:upgrade` restores it. **Nothing manual is needed** — see [Updates](/updates).

## Permissions

| Surface | Gate |
|---|---|
| Settings → Templates page, preview and thumbnail routes | `can:update` against `VentureDrake\LaravelCrm\Models\Setting` — that is, `edit crm settings` |
| The **Templates** item in the Settings sidebar | `view crm settings` |

All three routes carry the same gate as General Settings, so the pages that read a template preference and the page that writes it are governed by one permission. See [Permissions](/permissions).
