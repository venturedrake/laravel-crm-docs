# Settings

[[toc]]

## Overview

**Settings → General** at `/crm/settings` is the account-wide configuration screen: organisation identity, localisation, tax defaults, ID prefixes, per-document terms, and the contact block printed on document PDFs. It is a single Livewire component, `Livewire\Settings\SettingEdit`, writing rows to the `{prefix}settings` table.

The sibling screen **Settings → Templates** picks the PDF layout for each document type — see [PDF Templates](/pdf-templates).

> **Note:** Settings are configuration, not records. They are read through the `laravel-crm.settings` service, which caches the whole map, so a save clears the cache rather than expiring individual keys.

## Tabs

The page is split across eight tabs, in the order records flow through the CRM:

| Tab | Holds |
|---|---|
| **General** | Everything that belongs to no single entity — see below |
| **Leads** | Lead ID prefix |
| **Deals** | Deal ID prefix |
| **Quotes** | Quote ID prefix, quote terms |
| **Orders** | Order ID prefix |
| **Invoices** | Invoice ID prefix, invoice contact details, invoice terms, payment instructions |
| **Deliveries** | Delivery ID prefix |
| **Purchase orders** | Purchase order ID prefix, terms, delivery instructions |

Each entity tab holds that entity's ID prefix beside its own terms, so a tab is always the one place that entity's settings live. Leads and deals carry only a prefix today; a near-empty tab beats a shared one an admin has to learn the contents of.

Before 2.4.1 all of this was one flat column of roughly 27 unrelated controls, so finding "invoice terms" meant scrolling past everything else.

### General

**General** keeps everything that belongs to no single entity:

- **Identity** — organisation name, VAT / ABN number, logo
- **Localisation** — country, language, currency, timezone, date format, time format
- **Tax defaults** — tax name and default rate (see [Tax Rates](/tax-rates))
- **Document contact details** — the shared "From" block, [described below](#document-contact-details)
- **Two account-wide toggles** — show related activity, and dynamic products
- **Contact panels** — the organisation's own phone numbers, email addresses and [addresses](/addresses)

### Tab state

The visible tab is query-string synced, so `/crm/settings?tab=invoices` deep-links — the same contract Settings → Templates carries.

An unrecognised tab name, an empty string, or a real tab whose module is switched off all fall back to **General** rather than erroring. That includes the retired `?tab=documents`: a Documents tab briefly held the shared contact-details block on its own, and a bookmark pointing at it lands on General instead of a 404.

### Saving

**Nothing about how the page saves changed when it gained tabs.** It is still one form and one atomic save covering every panel, whichever tab is showing — `save()` does not branch on the visible tab.

A validation failure jumps to the tab holding the offending field, rather than leaving Save looking inert while the error sits on a hidden panel.

### Module gating

A tab is hidden when its module is switched off. The check goes through `VentureDrake\LaravelCrm\Support\Modules`, which is the same rule the `@has*enabled` Blade directives apply — including the non-obvious case that an **empty** `modules` array means every module is enabled, not none.

See [Configuration → Optional Modules](/configuration#optional-modules).

## Document contact details

The shared `pdf_contact_details` setting fills the **"From"** block on quote, order, delivery and invoice PDFs. Before 2.4.1 only invoices had such a field, which is why the other document types had no way to populate the block the themed templates render.

Each render resolves a chain, through one `Support\PdfContactDetails` helper:

```
{doc_type}_contact_details  →  pdf_contact_details  →  null
```

The per-doc-type key is checked first, so an install that had filled **Settings → Invoices → Contact details** keeps its exact invoice output.

### Where the block actually renders

The gaps are deliberate — closing either would change the visible output of a document that never carried the block:

| Template | Quote | Order | Delivery | Invoice | Purchase order |
|---|---|---|---|---|---|
| `classic` | — | — | — | Yes | — |
| `modern` | Yes | Yes | Yes | Yes | — |
| `bold` | Yes | Yes | Yes | Yes | — |
| `compact` | Yes | Yes | Yes | Yes | — |
| `professional` | Yes | Yes | Yes | Yes | — |

**Purchase orders render no contact block on any template.** Their layouts pair a **Supplier** column with a **Delivery details** one rather than From/To, so `purchase_order_contact_details` resolves through the same chain but prints nowhere.

**`classic` renders the block on invoices only.** It reproduces the pre-2.4.0 layout unchanged, where only the invoice blade ever had a From block.

So if your quote's From block is empty on Classic, that is why — pick another template, or fill the block on a template that renders it. See [PDF Templates](/pdf-templates).

### Clearing a field

Both fields in the chain can be **cleared**. They save on an explicit "was this field submitted" check rather than the truthy check the neighbouring prefix and terms fields use, so an empty textarea writes an empty row rather than being skipped.

That matters because an empty row is a meaningful value here: `PdfContactDetails` reads with `filled()`, so a cleared row resolves to null and falls through to the next link in the chain. Without it, `invoice_contact_details` — which *shadows* the shared value — would be write-once, and an admin who had ever filled it could never drop back to the shared block. A field never filled on an install still writes no row at all.

## Logo

The logo on **General** can be uploaded, previewed and deleted.

- The preview is framed like the fields around it and bounded on both axes, so a tall or very wide upload does not push the rest of the form down the page.
- `deleteLogo()` removes whichever logo is on screen. A **pending** upload discards back to the saved logo rather than destroying artwork that is still printing on every PDF; only a second click removes the saved one.
- Both are guarded with `authorize('update', Setting::class)`, matching the rest of the settings form.

## Integrations

The [Xero](/xero) and [ClickSend](/clicksend) settings pages share an `integration-tabs` component over the same DaisyUI lifted-tab panel as Settings → General and Settings → Templates, so the tab strip, the active state and the card edge match the rest of the settings area.

## Team scoping

On a [teams](/teams) install, settings are team-scoped: each team has its own organisation name, logo, prefixes and terms. The settings cache is partitioned per team, with a generation counter so a write still invalidates every team's entry on cache drivers that cannot tag or scan.

The [portal](/portal) is the one surface that reads settings without a signed-in user, and it pins the settings service to the **document's** team before rendering — see [Portal → Portal Settings and Teams](/portal#portal-settings-and-teams).

## Permissions

| Surface | Gate |
|---|---|
| Opening `/crm/settings` (`GET`) and saving it (`POST`) | `can:update` against `VentureDrake\LaravelCrm\Models\Setting` — that is, `edit crm settings` |
| Uploading or deleting the logo | The same, re-checked server-side with `authorize('update', Setting::class)` |

General Settings is an edit screen throughout: there is no read-only view of it, so both the `GET` and the `POST` carry `edit crm settings`. [Settings → Templates](/pdf-templates#permissions) carries the same gate. See [Permissions](/permissions).
