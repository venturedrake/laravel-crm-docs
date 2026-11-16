# Upgrade Guide

[[toc]]

Two things to read before any upgrade: the **version-specific notes** for the release you are moving to, and [Upgrading Within 2.x](#upgrading-within-2x), which is the same every release. Read the version notes **first** — some releases change who is allowed to do what, and the failure mode is a `403` for a user who could previously click the button.

## Upgrading to 2.7.0

A minor release. It adds [email campaign audiences](/email-marketing#audiences), [part payments on invoices](/invoices#payments), [pipeline stage ordering](/pipelines#managing-pipelines) and [paged boards](/pipelines#board-view), and fixes a [stored XSS](/security#stored-markup-escaping) in notes, task descriptions and PDF free text. It is a minor rather than a patch release because it adds a configuration key (`board.per_column`), new columns and new features. The upgrade is the usual

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

but **take a database backup first**. Five of its six migrations change existing rows and have no `down()`. Three rewrite invoice and lead data, two of them lossily (see the table), and the other two soft-delete rows. Only the email campaign audience migration can be rolled back. If you have published views, the XSS fix only reaches you once you re-publish the affected ones (see [Views to re-publish](#views-to-re-publish)).

**Still on CRM 1.x?** Follow [Upgrading from v1](#upgrading-from-v1-go-straight-to-260) with `^2.7.0` in place of `^2.6.0`.

### Six migrations

All six live in `database/updates/` and run with a plain `php artisan migrate` (or `laravelcrm:update`). Each is guarded, so running it again changes nothing.

| Migration | What it does | Reversible |
|---|---|---|
| `2026_11_01_000001_backfill_amount_due_on_laravel_crm_invoices_table` | Gives every invoice with a `NULL` balance one | No (see below) |
| `2026_11_01_000002_settle_legacy_paid_invoices_on_laravel_crm_invoices_table` | Records invoices marked paid as paid in full | **No, lossy** (see below) |
| `2026_11_06_000001_reopen_leads_whose_deals_were_deleted_on_laravel_crm_leads_table` | Clears `converted_at` on leads whose only deals were deleted | **No, lossy** (see below) |
| `2026_11_08_000001_remove_orphaned_custom_field_links_on_laravel_crm_field_models_table` | Soft-deletes custom field links and values whose field is gone | No |
| `2026_11_11_000001_remove_country_only_addresses_on_laravel_crm_addresses_table` | Soft-deletes order and delivery addresses that hold only a country | No |
| `2026_11_12_000001_add_audience_to_laravel_crm_email_campaigns_table` | Adds `audience_type` (default `all`) and `audience_ids` to `crm_email_campaigns` | Yes |

The version bump to `2.7.0` re-arms the [settings-seeding pass](/updates#settings-seeding) and the update banner, as usual.

### New config key

```php
'board' => [
    'per_column' => env('LARAVEL_CRM_BOARD_PER_COLUMN', 20),
],
```

It reaches an existing install through `mergeConfigFrom`, so you do not need to re-publish `config/laravel-crm.php`. See [Configuration → Kanban Boards](/configuration#kanban-boards) and [Boards load one page per column](#boards-load-one-page-per-column).

### Two invoice balance migrations, neither reversible

Each only fills in or corrects stored values, and each has an empty `down()`. The values they replace carried no information worth restoring, so a rollback leaves the corrected ones in place.

- `2026_11_01_000001_backfill_amount_due_on_laravel_crm_invoices_table` gives every invoice with a `NULL` balance one: `amount_due = total - amount_paid`, with `amount_paid` set to `0` where it was `NULL`. Form-created invoices never had a balance before, so the Pay drawer could not take a payment against them and the dashboard's outstanding total skipped them.
- `2026_11_01_000002_settle_legacy_paid_invoices_on_laravel_crm_invoices_table` records as paid in full any invoice marked paid with nothing recorded against it (`amount_paid = total`, `amount_due = 0`). The old Pay drawer only ever accepted a payment of `0`, so that is how every invoice paid through it was stored. It also raises any `amount_due` below zero to `0`. **This is the lossy one**: the previous `amount_paid` (always `0` or `NULL`) is overwritten and is not kept anywhere.

### Paid invoices keep their total

Once a payment is recorded against an invoice, an edit that would change its subtotal or tax is refused, through the form, the API (`422` on `line_items`) and the legacy controller alike. A line re-sent with the product, price and quantity it was saved with keeps its saved amount and tax, so edits that leave the lines alone still save, even after a product's tax rate has changed. An edit that changes a line is repriced and then judged. Before, such an edit could leave a part-paid invoice with no way to close it, or reopen a paid one and lose the date it was paid on. See [Invoices → Payments](/invoices#payments).

### Boards load one page per column

The leads, deals and quotes boards now render 20 cards per column with a **Load more** button, and each column header shows its full count with the value of the whole column beneath it, one line per currency. Set `LARAVEL_CRM_BOARD_PER_COLUMN` to change the page size. Search by contact name on the boards now works with `LARAVEL_CRM_ENCRYPT_DB_FIELDS=true`. See [Pipelines → Board View](/pipelines#board-view).

The quote and invoice lists and the quote board share one Send drawer per page, filled when it opens. If you have published the package's views, republish or update these:

| File | What goes wrong otherwise |
|---|---|
| `livewire/kanban-board/stage.blade.php`, `record.blade.php` | No paging controls, and the board's Send item calls a method that no longer exists |
| `livewire/kanban-board/sortable.blade.php` | Drags still save, but a failed drag or one that overlaps another update leaves the board out of date until the next full render, and card order can go wrong if another session moved a card |
| `livewire/leads/lead-board.blade.php`, `livewire/deals/deal-board.blade.php` | Load more does nothing |
| `livewire/quotes/quote-board.blade.php` | Load more and the board's Send item do nothing |
| `livewire/quotes/quote-send.blade.php`, `livewire/invoices/invoice-send.blade.php` | The Send drawer opens with empty fields |
| `livewire/quotes/quote-index.blade.php`, `livewire/invoices/invoice-index.blade.php` | Each row keeps its own Send component, so the lists are heavier than they need to be |

If you extend `LeadBoard`, `DealBoard` or `QuoteBoard`, they now extend `PipelineBoard`, which pages its own queries:

- `records()` is no longer called, and returns nothing if you call it through `parent`, so the cards a subclass filtered out come back. Move that filter to `boardScope()`, calling the parent first, and qualify its columns with `$this->column('title')`. A search joins the people and organizations tables, which share columns such as `title`, `user_owner_id` and `team_id`, so a bare column name fails as ambiguous as soon as someone searches.
- `onStageSorted()` and `onStageChanged()` take new optional arguments. An override with the old signature is a fatal error, so add `$stageId = null, $lastRender = null` and `$fromStage = null, $lastRender = null` respectively.

Boards that extend `KanbanBoard` directly are not paged. One thing changes for a quote board: a card's Send item now dispatches `quote-send` to a shared drawer, instead of calling the board's `send()` method and mounting a drawer per card. `livewire/kanban-board/board.blade.php` mounts that drawer for a quote board, so republish it if you published the views. A board with its own view needs `<livewire:crm-quote-send type="menu-item" />` in it once, or Send does nothing.

### Leads whose deal was deleted are reopened

A converted lead can no longer be converted again, and deleting its last deal now clears `converted_at` so that it can. `2026_11_06_000001_reopen_leads_whose_deals_were_deleted_on_laravel_crm_leads_table` does the same for deals deleted before this release. Those leads would otherwise show as converted with no deal, no **Convert** button, and no place on the list or board. See [Leads → Converting a Lead](/leads#converting-a-lead).

It clears `converted_at` on every lead that has a deleted deal and no remaining deal. Leads with no deals at all, leads with a deal left, and deleted leads are left alone. Expect the following after it runs:

- The reopened leads reappear on the lead list and board, and count as open again.
- Dashboard conversion figures drop by the same leads.
- Each lead keeps its current stage, such as **Converted** or **Won**, because nothing records the stage before it. Move them by hand.
- A lead is reopened even if you replaced its deal with a new one that isn't linked to it, or deleted the deal on purpose, because a deleted deal doesn't say why it went. Delete the lead, or convert it again, if it shouldn't be open.

**This migration is lossy.** `down()` does nothing, because the cleared dates can't be recovered. Take a database backup first. To see which leads it will reopen before you upgrade:

```sql
SELECT l.id, l.title, l.converted_at
FROM crm_leads l
WHERE l.converted_at IS NOT NULL
  AND l.deleted_at IS NULL
  AND EXISTS (SELECT 1 FROM crm_deals d WHERE d.lead_id = l.id AND d.deleted_at IS NOT NULL)
  AND NOT EXISTS (SELECT 1 FROM crm_deals d WHERE d.lead_id = l.id AND d.deleted_at IS NULL);
```

Use your own table prefix if `db_table_prefix` isn't the default `crm_`.

### Deleted custom fields stop breaking record creation

Deleting a custom field used to leave its links to models live, so creating any record of a linked model failed. `FieldObserver` now removes a field's links and values on delete, and `2026_11_08_000001_remove_orphaned_custom_field_links_on_laravel_crm_field_models_table` soft-deletes the links and values left behind by earlier deletes. Nothing visible is lost: the field they belonged to is already gone. See [Custom Fields → Deleting a Field](/custom-fields#deleting-a-field).

### Country-only order and delivery addresses are removed

The order and delivery forms prefill the country, and an order or delivery saved with its address left blank got an address that was only a country. Those are no longer saved, and `2026_11_11_000001_remove_country_only_addresses_on_laravel_crm_addresses_table` soft-deletes the existing ones. Team (settings) addresses and purchase orders' copies of them are left alone. The purchase-order form no longer offers a team address that has no line 1 or city.

### Email campaigns: existing campaigns target everyone, new ones save as drafts

Existing campaigns get `audience_type = all`, so they keep sending to all subscribed contacts. A new campaign saves as a draft unless you choose **Send now** and confirm it, and a campaign with an empty audience can't be sent. See [Email Marketing → Audiences](/email-marketing#audiences).

### Quotes, orders and contacts are harder to delete or change

- An accepted quote, or one with an order, is read-only, and a quote with an order can't be deleted or unaccepted. See [Quotes → Locked Quotes](/quotes#locked-quotes).
- An order with invoices, deliveries or purchase orders can't be deleted. See [Orders → Deleting an Order](/orders#deleting-an-order).
- A tax rate that products still use can't be deleted. See [Tax Rates](/tax-rates#managing-tax-rates).
- Deleting a person or organization warns which records still link to it, and those records show "Name (deleted)".
- `/crm/pipelines/create` is a 404. The pipeline stage create page is back at `/crm/pipeline-stages/create`. See [Removed](#removed).

The REST API returns `422` or `403` for the same cases. See [API → Conventions](/api#conventions).

### Stricter validation on contact details and dates

Person, organization, lead, deal, quote and order forms reject badly formed email addresses and phone numbers (`ContactEmail` and `PhoneNumber` rules), and so do the person, organization, lead and deal API endpoints. Stored values are not changed, and an address or number already saved on the record passes as it is, so an older invalid value does not block an unrelated edit. See [People → Email and Phone Validation](/people#email-and-phone-validation).

Invoices, quotes and purchase orders reject an end date before the start date: an invoice due date before its issue date, a quote expiry before its issue date, and a PO delivery date before its issue date. The forms check all three, the invoice and quote API endpoints return a `422`, and the legacy invoice and purchase-order requests check it too. Activity forms reject a finish time before the start time.

### Removed

The v1 pipeline and pipeline stage create/store routes ([#107](https://github.com/venturedrake/laravel-crm/issues/107)). They rendered views v2 never shipped and returned a 500. `/crm/pipelines/create` is now a 404, and the `StorePipelineRequest` and `StorePipelineStageRequest` classes are gone. The pipeline stage create page is back as a v2 Livewire component. See [Pipelines → Managing Pipelines](/pipelines#managing-pipelines).

### Smaller fixes

- **New invoices and purchase orders showed "Sent" before they were sent**, and sending them never set `sent`.
- **An invalid recipient address in the quote, invoice or PO Send drawer was a 500.** Addresses are validated (comma or semicolon separated) before the PDF is rendered, and a mail transport failure shows an error toast and keeps the drawer open.
- **The quote email subject and body printed a blank reference** ("Quote  from …"). They use the quote number.
- **The legacy Send Quote / Invoice / Purchase Order components wrote PDFs to `storage/app/private` on Laravel 11+** instead of `storage/app`.
- **The update check in `UpdateController` had no timeout.** It has one now, and the version-check client is resolved from a container binding.
- **Seeded campaign templates left placeholder braces in the preview** ([#137](https://github.com/venturedrake/laravel-crm/issues/137)). `laravelcrm:sample-data` wrote `{{first_name}}` where the editor and renderers use `{first_name}`. The seeder now writes `{key}`, and the email and SMS renderers also accept `{{key}}`, so records already seeded that way render correctly. The unsupported placeholders in existing demo data (`{{product_name}}`, `{{discount}}`, `{{order_id}}`) still show literally until you run `php artisan laravelcrm:sample-data --fresh`.
- **Email campaign subjects were sent with their placeholders unreplaced**, and the campaign and template preview showed no subject. See [Email Marketing → Placeholders](/email-marketing#placeholders).
- **The monitor form's scheme selector and URL field disagreed** ([#143](https://github.com/venturedrake/laravel-crm/issues/143)). The URL is built from the selected scheme plus the hostname, and the unsupported TCP option is gone. See [Monitoring](/monitoring#monitor).
- **The Xero integrations page threw a jQuery error** ([#145](https://github.com/venturedrake/laravel-crm/issues/145)) and showed no organization name when connected.
- **The product categories page failed on any re-render** ([#126](https://github.com/venturedrake/laravel-crm/issues/126)), and person and other show pages raised `ucfirst(null)` deprecations ([#125](https://github.com/venturedrake/laravel-crm/issues/125)). The person birthday uses the `date_format` setting.
- **Twelve toast messages showed raw lang keys** ([#112](https://github.com/venturedrake/laravel-crm/issues/112)), and invoice overpayment and a missing pipeline on a stage showed raw validation messages ([#127](https://github.com/venturedrake/laravel-crm/issues/127)).
- **Line-item edits made just before Save could be lost** ([#134](https://github.com/venturedrake/laravel-crm/issues/134), [#142](https://github.com/venturedrake/laravel-crm/issues/142), [#146](https://github.com/venturedrake/laravel-crm/issues/146), [#147](https://github.com/venturedrake/laravel-crm/issues/147)). Save sends the current rows with the request, a variation picked just before Save is priced as that variation, and a price typed alongside a variation pick is kept.
- **Board fixes.** The filter drawer's Reset button was a 500 on the lead, deal and quote boards, card menus were cut off by the board, and a card could be moved to a stage outside the board's pipeline. The team list's Owner filter queried a missing column.
- **Table sorts on display-only columns errored.** The product, order, invoice, purchase-order, delivery and team lists, and the tax rate, pipeline stage, role, custom field group and product category settings lists, now sort only on real columns (`SortsTable`).
- **The generated lead, deal and quote title overwrote a title typed by hand** when the contact name changed.
- **The invoice list search crashed** on a missing trait.
- **Saves are atomic.** Quote, order, invoice, delivery and purchase-order saves, with the contacts and custom field values made alongside them, run in one transaction, and the Xero push runs after commit.

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder uses your frozen copy instead of the package's. This release changes more than a hundred views. The ones that matter most:

| View | What you miss if you keep the old copy |
|---|---|
| `livewire/notes/note-item.blade.php`, `livewire/tasks/task-item.blade.php` | **The XSS fix.** A frozen copy still runs script stored in a note or task description |
| `quotes/pdf.blade.php`, `orders/pdf.blade.php`, `invoices/pdf.blade.php`, `purchase-orders/pdf.blade.php`, `deliveries/pdf.blade.php` and every file under `pdfs/` | **The XSS fix** for PDF free text, which the public portal also renders in a browser |
| `livewire/model-products.blade.php`, and the quote, order, invoice, purchase-order and delivery `-create` / `-edit` views | The searchable product picker, and Save sending the current lines. **With a frozen copy, an edit made just before Save can still be lost** |
| `livewire/invoices/invoice-pay.blade.php`, `invoice-show.blade.php`, `invoice-index.blade.php`, `invoice-related-index.blade.php`, `portal/invoices/show.blade.php` | Part payments, and the amount paid and balance due |
| `livewire/kanban-board/*`, `livewire/leads/lead-board.blade.php`, `livewire/deals/deal-board.blade.php`, `livewire/quotes/quote-board.blade.php` | Board paging and the shared Send drawer (see the table under [Boards load one page per column](#boards-load-one-page-per-column)) |
| `livewire/quotes/quote-show.blade.php`, `quote-index.blade.php`, `livewire/orders/order-show.blade.php`, `order-index.blade.php` | The disabled Edit and Delete buttons on locked quotes and orders, with the reason |
| `livewire/email-campaigns/email-campaign-create.blade.php`, `email-campaign-edit.blade.php`, `email-campaign-show.blade.php` | Audience selection and draft-by-default sends |
| `livewire/purchase-orders/purchase-order-form.blade.php` | The Pickup default when the team has no delivery address |
| `livewire/settings/pipelines/pipeline-show.blade.php`, `livewire/settings/pipeline-stages/pipeline-stage-index.blade.php` | Stage ordering and the **Create pipeline stage** button |
| `livewire/settings/integrations/xero/xero-connect.blade.php` | The fix for the jQuery error on the Xero page |
| `livewire/monitors/_monitor-form.blade.php` | The scheme selector fix |
| `mail/templates/send-quote/subject.blade.php`, `message.blade.php` | The quote number in the email instead of a blank reference |

New views (`components/board-card-menu`, `components/record-link`, `livewire/email-campaigns/_audience-send`, `_confirm-send`, `livewire/kanban-board/records`, `scripts`, `livewire/settings/pipeline-stages/pipeline-stage-create`, `settings/pipeline-stages/create`) are read from the package unless you publish them. Run `git diff 2.6.0 2.7.0 --stat -- resources/views` in the package for the full list.

If you publish lang files, many keys were added to `en` and `fa`.

### Development dependencies

The package's dev dependencies moved to Pest 3/4 and Testbench 10/11, because Composer now blocks every Laravel 10/11 release on security advisories and `pest-plugin-laravel ^2` no longer resolved. Host apps are unaffected.

## Upgrading to 2.6.0

A minor release. It adds three groups of configuration keys: [order form options](/orders#form-options), [delivery options](/deliveries#delivery-options) and a fixed [route domain](/configuration#route-domain). Every key defaults to the current behaviour, so an install that sets nothing sees no change. It also fixes user deletion, product-line comments and ordering, invoice line tax on update, and the v1 → v2 upgrade (`laravelcrm:v2`). The `laravelcrm:v2` fixes were prepared as 2.5.1, which was never tagged, so 2.6.0 is the first release that has them.

**Still on CRM 1.x?** Read [Upgrading from v1: go straight to 2.6.0](#upgrading-from-v1-go-straight-to-260) first. On v2, the upgrade is the usual

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

### No migrations

2.6.0 adds no tables or columns. Run `laravelcrm:update` anyway: it advances the `db_version` marker, and without it the system check reports the database as behind the code. The version bump to `2.6.0` re-arms the [settings-seeding pass](/updates#settings-seeding) and the update banner, as usual.

One migration you have already published is fixed in place: `add_extra_fields_to_laravel_crm_organizations_table` now adds and drops only the columns it needs to. `laravelcrm:update` re-publishes it over your copy. The file name stays the same, so a migration that has already run does not run again.

### New config keys

The new keys reach an existing install through `mergeConfigFrom`, so you do not need to re-publish `config/laravel-crm.php`. Clear a stale `config:cache` (the composer hook does this). The merge only covers top-level keys, but every key is read with its default, so if you add an `orders` block to your published config and leave out a `form` key, that key keeps its default.

```php
'route_domain' => env('LARAVEL_CRM_ROUTE_DOMAIN'), // e.g. crm.example.com.au

'orders' => [
    'form' => [
        'description' => true,        // false hides the description textarea
        'labels' => true,             // false hides the labels picker
        'custom_fields' => true,      // false hides both custom field blocks
        'unallocated_owner' => true,  // false removes "Unallocated" from the owner select
        'addresses' => ['billing', 'shipping'], // which address tabs to show and save
    ],
],

'deliveries' => [
    'delivered_on' => true,                           // false hides "Delivered on" everywhere
    'delivery_expected_label' => 'delivery_expected', // lang key used to label delivery_expected
],
```

All three are install-wide, including on teams installs. They are not in the Settings UI. The REST API ignores the `orders` and `deliveries` keys, but `route_domain` binds the API routes too, as described below.

**Hiding a field never changes stored data.** Validation and save skip a hidden field rather than clearing it, so an order's description, labels, custom field values and any address of an unlisted type survive an edit, and so does a delivery's `delivered_on`. See [Orders → Form options](/orders#form-options) and [Deliveries → Delivery options](/deliveries#delivery-options) for the details.

### Subdomain installs: set `route_domain`

`route_domain` is now the recommended setting for a subdomain install:

```dotenv
LARAVEL_CRM_ROUTE_DOMAIN=crm.example.com.au
LARAVEL_CRM_ROUTE_PREFIX=
```

When it is set, it wins over `route_subdomain`:

- The CRM, login and API route groups are bound to exactly that host. Before, the domain was worked out from the host of whatever request booted the app, so `php artisan route:cache` and queue workers registered routes from the CLI environment. A cached route list now matches an uncached boot. Check it with `php artisan route:cache && php artisan route:list --name=laravel-crm.dashboard`.
- The public `/p` routes (quote and invoice portal, chat widget embed, email and SMS tracking) stay unbound, as before.
- **New redirect.** With `route_prefix` empty, `GET` and `HEAD` requests for the bare domain or `www.` go to the same path and query on `route_domain`, including URLs that match no CRM route. It is registered as global HTTP middleware (`RedirectToRouteDomain`). `POST` and other methods, `/api/*` and `/p/*` are never redirected, and nothing is redirected while `route_prefix` is set. If you added your own middleware to redirect the bare domain to the subdomain, remove it once `route_domain` is set.
- `RouteSubdomain` compares the full host with `route_domain` and 404s any other host.

Without `route_domain`, one thing changes. `RouteSubdomain` used to 404 a two-label host such as `example.com`, so the login page rendered on the bare domain and every page behind it 404'd. It now redirects `GET` and `HEAD` there to `{route_subdomain}.{host}`, keeping the path and query, and other methods still 404. It also runs on the login routes now, and before `Authenticate`, so a guest is sent to the subdomain before the login page. See [Configuration → Route Domain](/configuration#route-domain).

### Deleting a user now releases their CRM records

Deleting a user used to fail with an integrity violation once they had a personal setting, had created a CRM record or had a team seat. The CRM tables reference users with foreign keys that have no `ON DELETE` action. The delete now goes through:

- The user's personal `crm_settings` rows and their `crm_team_user` rows are deleted.
- Every nullable CRM column with a foreign key to the user is set to `NULL`. That includes `user_owner_id`, so records the user owned become unallocated, `user_assigned_id`, and the `user_created_id`, `user_updated_id`, `user_deleted_id` and `user_restored_id` audit columns. **Reassign the user's records first** if you want to keep an owner on them.
- A required reference still blocks the delete. The usual one is a team owner (`crm_teams.user_id`). In that case nothing is released, and the delete fails as before.
- A soft delete of the user leaves every reference in place.

This runs from `UserObserver::deleting()`, so it also applies to users deleted by your own code, for example a Filament `DeleteAction`. See [Users → Deleting a user](/users#deleting-a-user).

### Smaller fixes

- **Product line comments were not saved on Livewire 4.** The comments input on quote, order, invoice and purchase-order lines used `wire:model.blur`, which on Livewire 4 only updates the browser copy, so the comment never reached the form's save. Lines carried over from a quote or order keep their read-only comment.
- **Line comments print inside the product's cell** on the quote, order, invoice and purchase-order show pages, as on the PDFs, instead of in a table row of their own.
- **Drag-and-drop ordering of product lines works again** on the quote, order, invoice and purchase-order forms, and purchase-order PDFs print their lines in that order.
- **Updating an invoice changed the tax on its existing lines.** `InvoiceService::update()` looked up the tax rate with the invoice line's own id instead of the product id, so an existing line took its tax from whichever product had that id, or got no tax. New lines were not affected. Invoices edited on an earlier version keep the wrong tax until their lines are saved again.
- **The chat show page crashed when the lead a conversation was converted to had been deleted.** It now renders without the lead link, and the conversation can be converted again. See [Chat](/chat#crm-inbox).

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder prefers your frozen copy, and this release changes:

| View | What you miss if you keep the old copy |
|---|---|
| `livewire/orders/order-form.blade.php` | The order form options. **A frozen copy still shows every field**, but a save skips the fields you hide, so an edit to a field that is still on screen is silently not saved |
| `livewire/orders/partials/order-address-fields.blade.php` | New in 2.6.0. The address fields `order-form` now includes for each tab. Publish it along with `order-form` |
| `livewire/deliveries/delivery-form.blade.php`, `livewire/deliveries/delivery-show.blade.php` | `delivered_on` and `delivery_expected_label`. A frozen form still shows "Delivered on" when it is off, and a save does not write it |
| `deliveries/pdf.blade.php` (the `classic` delivery template) and `delivery.blade.php` under `pdfs/{modern,bold,compact,professional}/` | `delivery_expected_label` on the PDFs. A frozen copy keeps "Delivery date" |
| `livewire/model-products.blade.php` | Drag-and-drop ordering of product lines, and saving line comments on Livewire 4. **With a frozen copy, line comments are still not saved on Livewire 4** |
| `purchase-orders/pdf.blade.php` (the `classic` purchase-order template) and `purchase-order.blade.php` under `pdfs/{modern,bold,compact,professional}/` | Purchase-order lines printed in their saved order |
| `livewire/quotes/quote-show.blade.php`, and the same file under `.../orders`, `.../invoices`, `.../purchase-orders` | Line comments printed inside the product's cell instead of in a row of their own |
| `livewire/chat/chat-index.blade.php`, `livewire/chat/chat-show.blade.php` | The fix for a conversation whose lead was deleted. **A frozen `chat-show` still crashes** on such a conversation |

The delivery index and related index columns are built in PHP, so they follow the delivery options even if you keep frozen views. The user deletion and invoice tax fixes are PHP-only too.

If you publish lang files, `billing_address` was added to `en` and `fa`, and `delivery_expected` and `delivery_date` were added to `en_au` and `en_gb`.

### Upgrading from v1: go straight to 2.6.0

If you are already on v2, the `laravelcrm:v2` changes below do not affect you.

If you are still on CRM 1.x, upgrade directly to 2.6.0 and do **not** run 2.5.0's `laravelcrm:v2`. 2.5.0 left v1 class names in ten columns and could abort part-way. The procedure is:

1. Take a full database backup.
2. `composer require venturedrake/laravel-crm:^2.6.0`
3. `php artisan laravelcrm:v2 --force`

Do not run `php artisan migrate` or `php artisan laravelcrm:update` before step 3. `laravelcrm:v2` renames the v1 tables first and then runs `laravelcrm:update` itself. See [Upgrading from 1.x to 2.x](#upgrading-from-1x-to-2x) for the full steps.

#### `laravelcrm:v2 --force`

`--force` skips the backup prompt and passes `--force` on to `laravelcrm:update`, so the command can run from a deploy script. Without it, `laravelcrm:v2` still asks you to confirm that you have taken a backup.

#### Exit codes

`laravelcrm:v2` now exits `1` when:

- you decline the backup prompt (it used to exit `0`)
- it runs non-interactively without `--force` (it used to treat that as declining)
- the nested `laravelcrm:update` fails (it used to ignore the failure and print "Laravel CRM is now updated to version 2.")

On a failure it tells you to fix the error and re-run `php artisan laravelcrm:v2`. Re-running it is safe: every step skips work that is already done.

#### `migrate` is blocked on a v1 database

`php artisan migrate` and `php artisan laravelcrm:update` now refuse to start while any of `crm_organisations`, `crm_organisation_types` or `crm_clients` exists (with your table prefix). They stop with a message that tells you to take a backup and run `php artisan laravelcrm:v2` first. Running the v2 migrations first creates empty v2 tables next to the v1 ones, and that used to strand the v1 data. `migrate:status` and other commands are not affected, and neither are the `migrate` and `laravelcrm:update` calls that `laravelcrm:v2` makes itself. The check uses the default database connection only.

If you added an `App\Listeners\GuardCrmV2Upgrade` listener to work around this, you can delete it once you are on 2.6.0. Keeping it is harmless.

#### Stranded v2 tables are healed

If `migrate` already ran before `laravelcrm:v2`, `laravelcrm:v2` now finds the v2 tables and columns sitting next to their v1 counterparts before it changes anything:

- A v2 table with no rows (`crm_customers`, `crm_organization_types`, `crm_organizations`) is dropped, and the v1 table is renamed into its place.
- A v2 column that is NULL in every row (for example `crm_leads.customer_id` next to `client_id`) is dropped, together with its indexes, and the v1 column is renamed into its place.

If a v2 table has rows, or a v2 column has values, the command lists each one with both names and both counts, prints "Nothing has been changed" and exits `1`. Decide which side holds the data you want. Usually that is the v1 side, unless people have used the CRM since the early `migrate`. Then remove or empty the other side, or restore your backup, and re-run `php artisan laravelcrm:v2`.

#### Duplicate column on v1 hosts

v2 republishes seven v1 migrations under new names, and the nested `migrate` ran them again. `add_extra_fields_to_laravel_crm_organizations_table` has no guards, so it failed on `vat_number`. The stub now adds and drops only the columns it needs to. `laravelcrm:v2` also marks the v2 name of each renamed v1 migration as run, in the v1 row's batch, and keeps the v1 row.

#### If you already ran 2.5.0's `laravelcrm:v2`

2.5.0 rewrote `Organisation` and `Client` class names only in the emails, phones, addresses, field values, notes, contacts, files and customers tables. Labels, tasks, calls, meetings and lunches on an organization, activity timeline entries, custom-field model assignments and the v1 `audits.auditable_type` still pointed at classes that no longer exist, so those records disappeared from organization pages.

Check the affected columns with this query. It assumes the default `crm_` prefix, so adjust it if yours differs, and leave out the `audits` line if you never had that table. MySQL needs the doubled backslashes in string literals; on PostgreSQL and SQLite, use single ones.

```sql
SELECT 'crm_labelables.crm_labelable_type' AS col, COUNT(*) AS n FROM crm_labelables WHERE crm_labelable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_tasks.taskable_type', COUNT(*) FROM crm_tasks WHERE taskable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_calls.callable_type', COUNT(*) FROM crm_calls WHERE callable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_meetings.meetingable_type', COUNT(*) FROM crm_meetings WHERE meetingable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_lunches.lunchable_type', COUNT(*) FROM crm_lunches WHERE lunchable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_activities.timelineable_type', COUNT(*) FROM crm_activities WHERE timelineable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_activities.recordable_type', COUNT(*) FROM crm_activities WHERE recordable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_activities.causeable_type', COUNT(*) FROM crm_activities WHERE causeable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'crm_field_models.model', COUNT(*) FROM crm_field_models WHERE model IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client')
UNION ALL SELECT 'audits.auditable_type', COUNT(*) FROM audits WHERE auditable_type IN ('VentureDrake\\LaravelCrm\\Models\\Organisation', 'VentureDrake\\LaravelCrm\\Models\\Client');
```

If any count is above zero, take a backup and re-run `php artisan laravelcrm:v2 --force` on 2.6.0. It is idempotent: on a database that is already on v2 it renames nothing, rewrites the class names in the newly covered columns, marks the renamed migrations as run, and runs `laravelcrm:update`. The rewrite matches exactly, so `Person` rows are untouched, and `audits.old_values` / `new_values` are history and are not rewritten.

## Upgrading to 2.5.0

A minor release. It adds [product variations](/product-variations), with a new `products/{product}/variations` [API sub-resource](/api-products#variations). It also removes two route groups that never worked in v2 (Product Attributes and Customers), and fixes purchase-order lines, contact linking and three observers. It is a minor rather than a patch release because of the new feature and API sub-resource.

### Three migrations, no new config keys

The upgrade is still

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

but it runs three new migrations from `database/updates`. You do not need to publish anything, and a plain `php artisan migrate` runs them too:

| Migration | What it does |
|---|---|
| `add_code_and_audit_columns_to_laravel_crm_product_variations_table` | Adds `code` (indexed), `barcode`, `order`, `active` (default `true`) and the `user_created_id` / `user_updated_id` / `user_deleted_id` / `user_restored_id` columns to `crm_product_variations` |
| `add_comments_and_order_product_id_to_laravel_crm_purchase_order_lines_table` | Adds `comments` and `order_product_id` to `crm_purchase_order_lines`. See [Purchase-order lines](#purchase-order-lines-now-save) below |
| `remove_product_attribute_permissions_from_laravel_crm` | Deletes the four `crm product attributes` permissions. See [Product Attributes removed](#product-attributes-removed) |

The two column migrations check for each column before adding it. They are safe on installs that added any of these columns by hand, running them twice does nothing, and neither changes existing rows.

The version bump to `2.5.0` re-arms the [settings-seeding pass](/updates#settings-seeding) and the update banner, as usual.

### Product Attributes removed

The unfinished **Settings → Product Attributes** page is gone. It was a v1 stub with no table and no views, and every route errored with `Table 'product_attributes' doesn't exist` ([#53](https://github.com/venturedrake/laravel-crm/issues/53)). Removed with it: the `laravel-crm.product-attributes.*` routes, `ProductAttributeController`, the `ProductAttribute` model, `ProductAttributePolicy`, the empty index view, both Settings nav entries and the unused `product_attribute*` lang keys.

**Host code that calls `route('laravel-crm.product-attributes.index')`, or any sibling, now throws `RouteNotFoundException`.** Search your app for `product-attributes` and `ProductAttribute` before you deploy.

The seeder no longer creates the four `crm product attributes` permissions. On existing installs, the role form lists every permission, so they showed up as checkboxes that granted nothing. The `remove_product_attribute_permissions_from_laravel_crm` migration deletes them, along with their `role_has_permissions` / `model_has_permissions` rows, and clears Spatie's permission cache. It does not touch a `product_attributes` table your application may have created itself. **Its `down()` is lossy:** it recreates the permissions, but cannot restore which roles held them. The seeded total drops from 156 to 152 — see [Permissions](/permissions#inventory).

To record product details such as material, brand or warranty, attach [custom fields](/custom-fields) to Products. Product variations are a separate feature and are not affected.

### Customers routes removed

The leftover v1 `/crm/customers` pages are gone, along with their `laravel-crm.customers.*` routes, `CustomerController`, the empty `customers/index` view and the unused `StoreClientRequest` / `UpdateClientRequest` classes. They never worked in v2: the index failed with `Call to undefined method Illuminate\Database\Eloquent\Builder::sortable()` ([#93](https://github.com/venturedrake/laravel-crm/issues/93)), and the other pages pointed at views that no longer exist.

**Host code that calls `route('laravel-crm.customers.index')`, or any sibling, now throws `RouteNotFoundException`.** Use People and Organizations instead.

No migration runs. The `customers` table, the `Customer` model, its observer and policy, and the `crm customers` permissions all stay, because the V1 → V2 upgrade still migrates that table and leads, deals, quotes and orders still carry `customer_id`. See [Customers](/customers).

### Product variations

A product can now have named variations, each with its own SKU, barcode, price and active flag, and you can pick one on deal, quote, order, invoice and purchase-order lines. **A product with no variations behaves exactly as before**, and no line changes unless someone picks a variation. See [Product Variations](/product-variations) for the details: pricing, conversions, how lines print, and the [Xero](/xero#line-items) behaviour.

**One change can affect host code.** `$product->productPrices` now returns product-level prices only. Variation prices share the `crm_product_prices` table and are reached through `$variation->productPrices`. The `ProductPrice::default()` scope excludes them too. Before this release, `getDefaultPrice()`, the API product payload and the Xero product sync read the first matching row, which could be a variation's price. If you query `crm_product_prices` directly, add `whereNull('product_variation_id')` to get the same result.

The REST API is a widening: product responses gain a `variations` array, quote, order and invoice `line_items` accept an optional `product_variation_id`, and line responses gain `product_variation_id` and `variation`. A variation that belongs to another product or team is a `422`. See [API · Products → Variations](/api-products#variations).

### Purchase-order lines now save

- **Saving a purchase order with lines failed on a real database.** `PurchaseOrderService::create()` wrote `comments` and `order_product_id` to `crm_purchase_order_lines`, but no migration had added either column. The test schema had `comments` but no test saved a purchase order with lines, so the suite passed. The migration above adds both.
- **Edits to purchase-order lines were silently lost.** `PurchaseOrderService::update()` read `$request->purchaseOrderLines`, which the form never sends. It now reads `products`, with the same row keys as `create()`: it updates matched lines, adds new ones and removes lines that are no longer submitted. It only matches lines that belong to the purchase order being edited.
- A dynamic product typed into the purchase-order form is named after the typed text again.

Line edits made to a purchase order before this release were not saved. Check any purchase order whose lines your users believe they changed after creating it.

### Contact person and organization are now linked on save

Saving a lead, deal, quote, order, invoice or purchase order with both a contact person and an organization now links the two as related contacts, so each lists the other on its show page ([#94](https://github.com/venturedrake/laravel-crm/issues/94)). An existing link is not duplicated, and changing the person or organization keeps the old link. The fix is PHP-only, so it applies even if you keep published views. **It does not backfill:** records saved before this release stay unlinked until they are saved again. See [People → Related contacts](/people#related-contacts).

### Smaller fixes

- **Creating a quote from a deal now keeps the deal's products.** The quote form starts with the deal's lines (product, variation, quantity and price), and they stay editable. See [Quotes](/quotes#creating-a-quote-from-a-deal).
- **A restored product, lunch or file no longer shows who deleted it.** `ProductObserver::restored()`, `LunchObserver::restored()` and `FileObserver::restored()` wrote the restoring user into `user_deleted_id`. They now clear `user_deleted_id` and set `user_restored_id`, as the other observers do. Rows restored on an earlier version keep the wrong value.
- **The currency select on the quote, order, invoice and purchase-order forms now updates live.** A product or variation picked after you change the currency is priced in the new currency. Lines already on the form keep the price they have.

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder prefers your frozen copy, and this release changes:

| View | What you miss if you keep the old copy |
|---|---|
| `livewire/products/product-form.blade.php`, `livewire/products/product-show.blade.php` | The **Variations** card on the form and the variations table on the show page. With a frozen form you cannot manage variations in the UI at all |
| `livewire/model-products.blade.php`, `livewire/deals/deal-form.blade.php` | The variation select on line items. A frozen copy never shows it, so no line can be put on a variation from the UI |
| `livewire/quotes/quote-form.blade.php`, and the same file under `.../orders`, `.../invoices`, `.../purchase-orders` (`order-form`, `invoice-form`, `purchase-order-form`) | They now pass the document currency to `crm-model-products`. **A frozen copy prices variations in the install currency, not the document's.** The quote form also starts a quote created from a deal with the deal's lines; a frozen copy starts it empty |
| `livewire/deals/deal-show.blade.php`, `livewire/quotes/quote-show.blade.php`, and the same file under `.../orders`, `.../invoices`, `.../purchase-orders`, `.../deliveries` | The variation name (`displayName()`) and SKU (`displayCode()`) on each line. A frozen copy prints the product name and code without the variation |
| Every PDF view: `quotes/pdf.blade.php`, `orders/pdf.blade.php`, `invoices/pdf.blade.php`, `purchase-orders/pdf.blade.php`, `deliveries/pdf.blade.php`, and all 20 under `pdfs/{modern,bold,compact,professional}/` | The variation name on each line, in downloads, emailed attachments and portal pages alike. A frozen copy prints the product name without the variation. See [PDF Templates → Line item names](/pdf-templates#line-item-names) |

`customers/index.blade.php` and `settings/product-attributes/product-attribute-index.blade.php` were deleted. A published copy of either is now unused and can be removed.

The contact linking, purchase-order and observer fixes are PHP-only. They apply even if you keep frozen views.

`php artisan laravelcrm:upgrade` names your drifted published views on every deploy (added in 2.4.1), so this table is a cross-check rather than the only signal you will get.

## Upgrading to 2.4.7

A one-fix patch release. If you picked a person or organization from an autocomplete and then edited the name, the form stayed linked to the record you picked. Picking "Paul Clark" and changing the name to "Paul" kept the link to Paul Clark: the **New** badge stayed hidden, and saving attached the existing Paul Clark instead of creating "Paul". The link is now cleared as soon as the trimmed name stops matching the linked record. The check compares against the decrypted name, so it also works with `encrypt_db_fields` on.

### No migrations, no new config keys

This release adds no tables, columns or configuration keys, so

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

is the whole upgrade. Run `laravelcrm:update` even though there is nothing to migrate. It still advances the `db_version` marker, and without it the system check reports the database as behind the code.

### Views to re-publish: none

**None.** The fix is PHP-only (in the `HasPersonSuggest` and `HasOrganizationSuggest` traits), so it applies even if you keep published views. If you are coming from 2.4.5 or earlier, the 2.4.6 table below still applies to you.

## Upgrading to 2.4.6

A bug-fix patch release:

- **Clicking an autocomplete suggestion now links the record.** On the **Related organizations** and **Related people** cards, and on the person and organization fields of the lead, deal, quote, order, invoice and purchase-order forms, the input closed its dropdown on blur. Clicking a suggestion blurred the input first, so the dropdown was gone before the click registered and nothing was linked. The dropdown now opens on focus and input and closes on a click outside, Esc or Tab. The input also sets `autocomplete="off"`, so the browser's own suggestion list no longer covers it.
- **Saving a team no longer removes all of its members.** The edit form opened with no members checked, and saving synced that empty list. See [Teams](/teams).
- **Person and organization autocompletes now work with field encryption on.** They used to run `LIKE` against the ciphertext and found nothing. With `encrypt_db_fields` on, they now match and sort on decrypted values, as the index searches already did. See [Security → Field Encryption](/security#field-encryption).
- **The Send quote, invoice and purchase-order subjects no longer end in a dangling "for".** When the document has no organization, the subject falls back to the person's name, and drops the clause when there is neither.
- **`<!--[if BLOCK]><![endif]-->` no longer appears in the Send invoice message.** These are Livewire morph markers, and they were reaching the email as literal text.
- **The user create/edit form no longer prints a stray `>`** under the card.

### No migrations, no new config keys

This release adds no tables, columns or configuration keys, so

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

is the whole upgrade. Run `laravelcrm:update` even though there is nothing to migrate. It still advances the `db_version` marker, and without it the system check reports the database as behind the code.

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder prefers your frozen copy, and this release changes:

| View | What you miss if you keep the old copy |
|---|---|
| `livewire/related-organizations.blade.php`, `livewire/related-people.blade.php` | The autocomplete fix. **A frozen copy stays broken**: it keeps `wire:blur`, so clicking a suggestion still does not link the organization or person |
| `livewire/leads/lead-form.blade.php` and the same file under `.../deals`, `.../quotes`, `.../orders`, `.../invoices`, `.../purchase-orders` (`deal-form`, `quote-form`, `order-form`, `invoice-form`, `purchase-order-form`) | The same fix for the person and organization fields on the create and edit forms. A frozen copy still loses the click on a suggestion |
| `mail/templates/send-quote/subject.blade.php`, `mail/templates/send-invoice/subject.blade.php`, `mail/templates/send-purchase-order/subject.blade.php` | The person-name fallback. A frozen copy still ends the subject in a dangling "for" when the document has no organization |
| `mail/templates/send-invoice/message.blade.php` | The morph-marker fix. A frozen copy still puts `<!--[if BLOCK]><![endif]-->` into the message body |
| `livewire/users/user-form.blade.php` | Cosmetic only: a frozen copy still prints a stray `>` under the form |

The team-members fix and the encrypted autocomplete search are PHP-only. They apply even if you keep frozen views. `hidePeople()` and `hideOrganizations()` stay on the suggest traits so that published copies of the old blades, which still call them, do not throw.

`php artisan laravelcrm:upgrade` names your drifted published views on every deploy (added in 2.4.1), so this table is a cross-check rather than the only signal you will get.

## Upgrading to 2.4.5

The [Xero integration](/xero) is now optional. `dcblogdev/laravel-xero` has moved from this package's `require` to `suggest` plus `require-dev`, and `guzzlehttp/guzzle` now allows `^8.0`. **If you use Xero, you have to require the package yourself.** Read [the deploy-order note](#if-you-use-the-xero-integration-require-it-yourself-before-you-deploy) below before you upgrade.

### No migrations, no new config keys

This release changes only which packages composer installs and how the Xero integration behaves when one of them is absent. It adds no tables, columns or configuration keys, so

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

is the whole upgrade. `laravelcrm:update` still advances the `db_version` marker, so run it even though there is nothing to migrate — otherwise the system check reports the database as behind the code.

### If you use the Xero integration, require it yourself *before* you deploy

Nothing pulls `dcblogdev/laravel-xero` into your application any more. A production `composer install --no-dev` will not bring it in from this package's `require-dev` either. Add it to your own `composer.json`:

```bash
composer require dcblogdev/laravel-xero:1.1.3
```

Do that **in the same change as the upgrade**, not after it. If you upgrade first, the package is gone between the two deploys, and the integration stops syncing in that window without saying so.

**In the app, nothing errors when it is missing — it goes quiet.** Every entry point now checks `XeroIntegration::installed()`. With the package absent, the `XeroToken` observer is never registered and `XeroTenant` is never pushed onto the `web` or `crm-api` middleware groups. `Product` / `Invoice` / `PurchaseOrder` sync becomes a no-op, and `php artisan laravelcrm:xero` exits non-zero with the install command instead of a fatal error. There is no banner. The one visible sign in the UI is **Settings → Integrations → Xero**, where an alert naming the missing package replaces the "Connect to Xero" button.

**The scheduler is the exception.** The Xero syncs are scheduled whenever `xero.clientId` and `xero.clientSecret` are set, without checking that the package is installed. If you drop the package but keep a published `config/xero.php` with credentials set, `xero:keep-alive` fails as an undefined command every 5 minutes, and `laravelcrm:xero contacts` / `products` fail every 10. If you are not keeping Xero, remove `XERO_CLIENT_ID` and `XERO_CLIENT_SECRET` from your `.env`.

Your data is not touched. Stored tokens and the `crm_xero_*` mirror rows survive, so installing the package restores the connection without re-authorising, as long as the stored refresh token has not expired in the meantime. See [Xero → When the package is not installed](/xero#when-the-package-is-not-installed).

### Why: the package would not install on Guzzle 8

`composer require venturedrake/laravel-crm` failed outright in any app that had resolved `guzzlehttp/guzzle` 8. That includes a stock Laravel 13.32+ app, since `laravel/framework` moved to `^7.8.2 || ^8.0` at that version. Composer named this package's own `^6.0|^7.0` constraint first, so it read as a one-line bump. But the binding cap came in transitively from `dcblogdev/laravel-xero`. **No version of it has ever allowed Guzzle 8**: 1.1.3 requires `^6|^7`, and later releases `^7.9.3`. There is no upstream release of it compatible with Guzzle 8.

So the choice is yours rather than ours, and it is a real one:

| You want | Do this | Consequence |
|---|---|---|
| Xero sync | `composer require dcblogdev/laravel-xero:1.1.3` | Your app stays on Guzzle 7 until that package ships a Guzzle 8 release |
| Guzzle 8 / Laravel 13.32+ | Nothing | The Xero integration is inert, as described above |

You cannot have both today.

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder prefers your frozen copy, and this release changes one of them:

| View | What you miss if you keep the old copy |
|---|---|
| `livewire/settings/integrations/xero/xero-connect.blade.php` | The "not installed" alert. A frozen copy still renders a **Connect to Xero** button with the package absent. It is not fatal — the route redirects back to this same screen — but it loops silently instead of naming what to install, which removes the only in-app signal this release gives you |

`php artisan laravelcrm:upgrade` names your drifted published views on every deploy (added in 2.4.1), so this table is a cross-check rather than the only signal you will get.

### If you copied blades out of `resources/v1/` during a v1 → v2 migration

This release also gated `resources/v1/views/invoices/partials/fields.blade.php` and its purchase-order twin. Nothing in the package loads or publishes that tree; it is there for v1-era apps that copied the blades into their own `resources/views`. If yours is one of them, a copy taken before this release calls `\Dcblogdev\Xero\Facades\Xero::isConnected()` directly. With the package absent, that is a hard `Class "Dcblogdev\Xero\Facades\Xero" not found` on the invoice and purchase-order forms. Replace the call with `\VentureDrake\LaravelCrm\Support\XeroIntegration::connected()`, which works the same way, or keep `dcblogdev/laravel-xero` installed.

## Upgrading to 2.4.4

A performance patch release. The Owner filter used to send the whole users table to the filter drawer. On a large install, that made a single index page tens of megabytes and seconds to render. It now searches on the server.

It also fixes a `500` on nine screens. Selecting an owner on the deal, lead and quote index pages and boards, and on the order, invoice and purchase-order index pages, threw an ambiguous-column `QueryException`. Those queries join `crm_people` and `crm_organizations`, and both tables have their own `user_owner_id`. The column is now table-qualified.

### No migrations, no new config keys

This release adds no tables or columns and no configuration keys, so

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

is the whole upgrade. `laravelcrm:update` still advances the `db_version` marker, so run it even though there is nothing to migrate — otherwise the system check reports the database as behind the code.

### The Owner filter no longer has a "Select all"

Type a name in the Owner filter and the matches come back from the database, up to 20 at a time. Before, you scrolled a list that was fully rendered on every request.

The trade-off is that **Select all** is gone from that filter. It could only ever select the options that were rendered. On any install big enough for this change to matter, it was already selecting a subset rather than "all owners". **Clear** replaces it and empties the filter in one click. Owners you have already selected stay visible as chips, even when they fall outside the current search.

No action is required unless you have published views; if you have, see the table below. Tell your support and sales people, since this is the one change in this release they will notice. See [Overview → List Views](/overview#list-views).

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder prefers your frozen copy. This release changes the Owner `<x-mary-choices>` in 15 of them:

| View | What you miss if you keep the old copy |
|---|---|
| The 12 index blades (`livewire/people/person-index.blade.php`, `livewire/organizations/organization-index.blade.php`, `livewire/leads/lead-index.blade.php`, and the same file under `.../deals`, `.../quotes`, `.../orders`, `.../invoices`, `.../deliveries`, `.../purchase-orders`, `.../products`, `.../tasks`, `.../teams`) | The `searchable` / `search-function="searchUsers"` / `clearable` Owner filter. **This one does not error — it goes quietly wrong.** A frozen copy keeps `allow-all` and has no search box, while the component behind it now returns at most 20 users. The filter lists only the first 20 owners alphabetically with no way to reach the rest, and **Select all** selects those 20 rather than everyone. On an install with more than 20 users, you must re-publish these |
| The 3 board blades (`livewire/leads/lead-board.blade.php`, `livewire/deals/deal-board.blade.php`, `livewire/quotes/quote-board.blade.php`) | The same change, and the same silent failure |

`php artisan laravelcrm:upgrade` names your drifted published views on every deploy (added in 2.4.1), so this table is a cross-check rather than the only signal you will get.

The reverse combination *does* throw: MaryUI rejects `allow-all` together with `searchable`. That is why **Select all** could not stay alongside the new search.

## Upgrading to 2.4.3

A performance patch release. CRM index pages now run a bounded number of queries. Before, a 25-row page ran anywhere from 75 to nearly 500, because each table cell lazy-loaded its relation per row. On the demo database, `/people` drops from 171 queries to 15 and `/orders` from 486 to 22. The `getPrimaryEmail()` / `getPrimaryPhone()` / `getPrimaryAddress()` helpers keep their signatures and read eager-loaded relations when they are present, so your own callers get the saving without code changes.

There are also two visible changes: the always-blank `next_activity` column is gone from the people and organizations index tables, and the header version badge now compares versions correctly.

### No migrations, no new config keys

This release adds no tables or columns and no configuration keys, so

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

is the whole upgrade. `laravelcrm:update` still advances the `db_version` marker, so run it even though there is nothing to migrate — otherwise the system check reports the database as behind the code.

### Settings seeding runs once per version

The settings-seeding pass used to run on every CRM page: 36 `crm_settings` queries plus an `information_schema` probe. It now sits behind a cache flag, which is the one new behaviour operators will see. The flag is stamped with the package version (`crm.settings-seeded.2.4.3`), so **the version bump re-arms it on its own**. The first CRM request after the upgrade runs the seeding pass once, and every request after that skips it. You do not have to clear anything by hand. The version in the key is what re-arms the pass. `laravelcrm:upgrade` clears only the config, route and view caches, not the application cache that holds this flag.

The flag also has a 24-hour TTL, so a settings row deleted by hand comes back on the next day's first request rather than staying missing until the next release.

**On a teams install the flag is partitioned by team.** Most of what the pass seeds is team-scoped — `organization_name`, `currency`, the document prefixes — so each team's first request after a deploy runs its own seeding pass, once. With a single shared flag, only the team that made the first request would have got its rows. See [Updates → Settings seeding](/updates#settings-seeding).

### The version phone-home now has timeouts

The CRM's version check posts to `api.laravelcrm.com`. Guzzle defaults both `connect_timeout` and `timeout` to "wait forever", and the install id is only recorded from a successful response. So **an install with outbound traffic to that host blocked put a blocking outbound POST with no timeout in front of every CRM page load.** The call is now capped at 2s connect / 3s total. An hour-long backoff is set before the request goes out, so a failure costs at most one attempt an hour rather than one per page view. The check also now runs inside the settings-seeding pass above, so it is only considered when that pass runs.

No action is needed. This may explain a noticeable drop in page load times after the upgrade. If you had blocked the CRM's own checks to work around the stall, you no longer need to. See [Security → Outbound version check](/security#outbound-version-check).

### `crm_usage_requests.visitor` values change format

The visitor pseudonym was hashed with `crypt($ip, config('hashing.encryption_key'))`. That is not a stock Laravel config key, so the salt was null and PHP emitted a deprecation on every page view. The pseudonym is now a sha256 over your application key.

**Rows written before the upgrade will not match rows written after it for the same visitor.** There is no migration and no backfill, and none is offered: the column is a pseudonym, not a key, and nothing joins on it. As a result, any report that counts *distinct* visitors over a window spanning the upgrade double-counts anyone who appears on both sides of it. If the number has to be exact, pick a window that starts after the deploy.

### The version badge uses `version_compare()`

The header popover compared version strings with `<`, and `'2.2.0' < '2.10.0'` is false when compared as strings. Once the minor version reached double digits, an install would show *"Latest Version"* even when a newer release existed. It now uses `version_compare()`, as `SystemCheckService` already did. The popover also reads its two settings from the cached settings map instead of running two raw queries per page.

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder prefers your frozen copy, and this release changes:

| View | What you miss if you keep the old copy |
|---|---|
| `livewire/people/person-index.blade.php`, `livewire/organizations/organization-index.blade.php` | The open / lost / won deal cells now read `open_deals_count`, `lost_deals_count` and `won_deals_count`, which the components fill with `withCount`. The components no longer eager-load `deals`, so **a frozen copy is slower than before the upgrade**: its `$person->deals->…` cells lazy-load every row's deals, about 25 extra queries on a 25-row page. Re-publish these. (The `next_activity` column is removed in the components, so it goes away whether or not you re-publish.) |
| `livewire/quotes/quote-index.blade.php`, `livewire/users/user-index.blade.php` | Reads of the newly eager-loaded relations: `$quote->orders->count()` in place of `$quote->orders()->count()`, and `$user->roles->first()` in place of `$user->roles()->first()`. A frozen copy keeps those per-row queries |
| `layouts/app.blade.php` | The `version_compare()` badge fix and the cached settings read. A frozen copy keeps two raw queries on every page, and shows the wrong badge once the minor version passes 2.10 |
| `mail/templates/send-quote/*`, `send-invoice/*`, `send-purchase-order/*` (both `message` and `subject`) | The memoised settings read. The rendered output is identical but the query count is not, and these render once per row behind the index send buttons |

`php artisan laravelcrm:upgrade` names your drifted published views on every deploy (added in 2.4.1), so this table is a cross-check rather than the only signal you will get.

## Upgrading to 2.4.2

A two-fix patch release. Both fixes matter to someone: every PDF preview was failing on nginx and older Apache, and the public portal was rendering one tenant's branding on another tenant's document.

### Republish assets — the pdf.js worker filename changed

The pdf.js worker is now emitted as `pdf.worker.min-<hash>.js` where it was `pdf.worker.min-<hash>.mjs`. `.mjs` is in the default MIME map of neither nginx nor older Apache, so those servers handed it over as `application/octet-stream` and the browser's strict module-script MIME check refused to run it — every PDF preview failed with *"setting up fake worker failed"*, on a build and a publish that were both correct.

`manifest.json` names the new filename, so **a host that upgrades without republishing assets has a manifest pointing at a file that is no longer on disk, and every preview fails.** Nothing manual is needed — the composer hook fires `laravelcrm:upgrade`, which republishes the assets and prunes the stale content-hashed file — but this is the release where `composer update` on its own is not enough. If you have not added the hook, see [After upgrading](#after-upgrading) below, or run `php artisan laravelcrm:upgrade` by hand.

### No migrations; one config key removed

No tables and no columns. Unlike 2.4.1 this is *not* a "no new config keys" release: `portal.team_id` is **removed** — see below. Run

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

as usual. `laravelcrm:update` still advances the `db_version` marker, so run it even though there is nothing to migrate — otherwise the system check reports the database as behind the code.

### Multi-tenant installs: the portal leaked branding across teams

Before this release the public portal rendered a document's page and PDF from **unscoped** settings. The portal is anonymous — a signed link, no login — so `BelongsToTeamsScope` never engaged there and the settings query returned every team's rows; `pluck()` keys by name, so whichever team the database listed last supplied the organisation name, ABN, contact block and logo on every tenant's quotes, invoices and purchase orders alike. On a `laravel-crm.teams` install, a customer opening one team's emailed invoice link could be shown another team's branding.

**No data migration is needed.** The fix is in the scope and the cache key — the portal controllers pin the settings service to the document's own team before rendering — so upgrading and clearing the application cache is sufficient. Single-tenant installs were never affected. See [Security](/security) and [Portal → Portal Settings and Teams](/portal#portal-settings-and-teams).

### Removing the pinned portal team

`LARAVEL_CRM_PORTAL_TEAM_ID` and `config('laravel-crm.portal.team_id')` are **removed**. The variable sat ahead of every other portal team signal as a hard single-tenant lock, which was harmless while the portal served only roadmaps and wrong once the portal started answering "whose branding does this invoice carry?" — a document states its own owner, and one env var was silently overruling it for every team on the install. Every portal team signal is now derived from the request or the record.

**If you had that variable set,** delete it from your `.env` and drop the `team_id` line from `config/laravel-crm.php` if you have published the config. Nothing reads it any more, so leaving it in place is inert rather than harmful — but the portal will stop behaving as a single-tenant lock, which is the point. A team whose board you do *not* want public should have its features marked non-public rather than relying on the other teams being locked out.

Boards stay reachable per team at `/p/features/team/{id}`, and bare `/p/features` resolves as it has since 2.4.0: the team in the URL, the board remembered in the visitor's session, the signed-in user's current team, and finally — when exactly one team has a public board — that team. See [Portal → Portal Teams](/portal#portal-teams).

### Views to re-publish: none

**None.** Neither change in this release touches `resources/views`, so a published view cannot hide either fix. If you are coming from 2.4.0, the 2.4.1 table below still applies to you. Upgrading past 2.4.2, check the 2.4.3 – 2.4.6 and 2.5.0 tables above as well.

## Upgrading to 2.4.1

A patch release that is mostly additive: a PDF preview drawer on every document, a **Get link** button on the three document types that have a portal page, portal pages that render the record's own PDF template, and a tabbed Settings → General screen.

### What's new

- **A PDF preview drawer** beside every download button on quotes, orders, deliveries, invoices and purchase orders, rendering the real generated PDF with pdf.js rather than an HTML approximation of it.
- **A "Get link" button** on invoice, quote and purchase-order show pages and index rows, handing over the same 14-day signed portal URL that gets emailed to the customer.
- **Portal document pages render the record's selected PDF template**, so the page a customer opens from an emailed link and the PDF they download from it are the same document. See [Portal](/portal).
- **Settings → General is split across tabs** — one per entity, in the order records flow through the CRM. See [Settings](/settings).
- **A shared `pdf_contact_details` setting** filling the "From" block on quote, order, delivery and invoice PDFs. See [Settings → Document contact details](/settings#document-contact-details).
- **The settings logo can be deleted**, not only replaced.
- **`laravelcrm:upgrade` warns about drifted published views** on every deploy. See [Updates](/updates).

### No migrations, no new config keys

This is a patch release. It adds no tables or columns and no configuration keys, so

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

is the whole upgrade. `laravelcrm:update` still advances the `db_version` marker, so run it even though there is nothing to migrate — otherwise the system check reports the database as behind the code.

### Multi-tenant installs: the settings cache leaked across teams

Before this release the settings cache was global while the query behind it was team-scoped, so on a `laravel-crm.teams` install whichever team warmed the cache served its organisation name, ABN, address and logo to every other team until the next settings write — on the settings screen and on the documents those settings are rendered into.

**No data migration is needed.** The fix is in the cache key, so upgrading and clearing the application cache is sufficient. Single-tenant installs were never affected. See [Security](/security) and [Teams → Data Scoping](/teams#data-scoping).

### Views to re-publish

If you have published views into `resources/views/vendor/laravel-crm`, the view finder prefers your frozen copy, and this release changes:

| View | What you miss if you keep the old copy |
|---|---|
| `livewire/settings/setting-edit.blade.php` | The flat single-column settings page persists — harmless, and it keeps saving correctly |
| `livewire/kanban-board/record.blade.php`, `livewire/kanban-board/sortable.blade.php` | The client-side half of the drag-and-drop fix — the `data-record-id` marking. The server side resolves, filters and renumbers regardless, so the 500 and the authorization gap are closed either way |
| The quote / order / delivery / invoice / purchase-order `*-index`, `*-related-index`, `*-show` and `*-form` views | The Preview and **Get link** buttons — the routes exist, but nothing renders a link to them |
| `portal/quotes/show`, `portal/invoices/show`, `portal/purchase-orders/show` | The portal page keeps its own hand-built layout instead of rendering the record's PDF template |
| `layouts/app.blade.php`, `layouts/portal.blade.php` | The Get-link modal mount, the chrome-free portal document pages and the footer fix. `layouts/partials/nav-integrations.blade.php` no longer exists in the package at all |
| `pdfs/{modern,bold,compact,professional}/*` and the five classic `*/pdf.blade.php` | The null-date guards, the Bold header inset and its logo alignment |
| `mail/templates/send-invoice/message.blade.php` | The no-due-date variant of the emailed invoice body |

`php artisan laravelcrm:upgrade` now names your drifted published views on every deploy, so you no longer have to work this out by hand — it md5-compares each published blade against the packaged one and warns on both drifted views and views the package no longer ships. It only warns; it never fails the run. See [Updates](/updates).

**The Settings → Templates thumbnails come back on their own.** 2.4.0 shipped without the five template SVGs, so the picker rendered broken images; the artwork has moved out of the build output directory and is republished by `laravelcrm:upgrade`, which `laravelcrm:update` calls first. Nothing manual is needed. See [PDF Templates → Thumbnails](/pdf-templates#thumbnails).

**If you published the portal views**, note the three `crm-portal-*-line-items` Livewire components are gone. A published portal view still referencing one will throw — remove the reference, or re-publish the view.

### New routes

Five preview routes are added: `laravel-crm.quotes.preview` and its order, delivery, invoice and purchase-order siblings. Each carries the same `can:view` guard as the download route it mirrors, so they grant nothing your existing roles did not already allow.

## Upgrading from 2.3.x to 2.4.0

Version 2.4.0 is the largest release since the 2.x rewrite. It enforces authorization on every mutating action, widens line item quantities to decimals, gives every team its own lookup data on a multi-tenant install, and adds a new `laravelcrm:upgrade` command with a composer hook behind it.

**It carries breaking changes.** Read [Breaking changes](#breaking-changes) below in full before you deploy, and on a teams install **take a database backup** — the per-team backfill rewrites foreign keys and has no `down()`.

The whole sequence, in order:

```bash
# --- Still on 2.3.x ---
# 1. Back up the database. On a teams install this is not optional.
# 2. Create any permission rows this install is missing, and audit custom roles.
php artisan laravelcrm:update
php artisan laravelcrm:permissions        # teams installs only

# --- Deploy 2.4.0 ---
composer update venturedrake/laravel-crm  # or composer install --no-dev on a deploy

# 3. Apply the migrations, seeders and backfills.
php artisan laravelcrm:update --force

# 4. Fan the roles out to each team again, now that 2.4.0 has run.
php artisan laravelcrm:permissions        # teams installs only
```

Steps 2 and 4 look redundant and are not: step 2 fixes missing permissions **while the old code is still running**, so nobody sees a `403` in the window between deploying and seeding. Step 4 picks up anything the 2.4.0 seeder added. Both commands are idempotent.

`laravelcrm:update` calls `laravelcrm:permissions` itself on a teams install, so step 4 is already covered if you ran step 3 — it is listed so the ordering is explicit and so you can run it alone.

### What's new

- **User invitations** — invite someone by email instead of creating the account and handing over a password out of band. See [Users → Invitations](/users#invitations).
- **Five PDF templates**, pickable per document type under **Settings → Templates** and per record on the document form. See [PDF Templates](/pdf-templates).
- **Per-team lookup data and pipelines** on a teams install, plus a host-team switcher in the CRM header. See [Teams](/teams).
- **Decimal line item quantities** — a product can be sold by weight or volume (3.5 Kg, 0.25 L).
- **A start date and time on tasks** (`start_at`). See [Tasks](/tasks).
- **`laravelcrm:upgrade`**, a system-check banner and a rebuilt updates page. See [Updates](/updates).
- **Every public feature board is team-aware**, and `LARAVEL_CRM_PORTAL_TEAM_ID` is now optional — and **removed in 2.4.2**, see [Removing the pinned portal team](#removing-the-pinned-portal-team) above. See [Portal](/portal).
- **Per-team API foreign-key scoping**, and per-account throttling on `POST /auth/token`. See [API](/api).
- **Persian (`fa`) translations**.

### Before you upgrade

**1. Back up your database.** Always, but especially on a teams install: the `db_update_1201` backfill rewrites `pipeline_stage_id` on seven tables and cannot be reversed.

**2. Re-run the permission seeding — before deploying the new code.** This is the most likely cause of unexpected 403s after upgrade, and it is entirely preventable.

An install that has been upgraded over time without re-seeding may be missing permissions added in later releases. Those permissions did not matter before, because the actions they gate were not enforced on the server. They matter now. The six families most commonly missing on long-lived installs:

| Permission family | Added for |
|---|---|
| `crm monitors` | Uptime / SSL monitoring |
| `crm features` | Feature voting & feedback portal |
| `crm email-campaigns` | Email marketing |
| `crm sms-campaigns` | SMS marketing |
| `crm chat` | Live chat |
| `crm activities` | Activity timeline |

If the permission row does not exist, no role holds it, and every action it gates returns `403` — **including for Owner and Admin**, because those roles are granted `Permission::all()` *as it existed at the moment they were seeded*.

```bash
# 1. Creates any missing permission rows and re-grants them to the stock roles.
php artisan laravelcrm:update

# …or, to re-run only the seeder without also migrating:
php artisan db:seed --class="VentureDrake\LaravelCrm\Database\Seeders\LaravelCrmTablesSeeder" --force

# 2. Multi-tenant (laravel-crm.teams = true) installs ONLY — run after step 1.
#    laravelcrm:update runs this for you; it is listed separately so the
#    ordering is explicit and so you can run it on its own.
php artisan laravelcrm:permissions
```

Step 1 is the step that fixes missing permissions, and it is safe to re-run: every permission is created with `firstOrCreate()` and every role grant is additive, so existing rows are matched rather than duplicated. Re-seeding revokes nothing and does not touch custom roles.

> **Important:** `laravelcrm:permissions` is **not** a substitute for step 1. Despite the name it creates no permissions — it copies the global CRM roles and their *existing* grants down to each team, so it only does anything when `laravel-crm.teams = true`. On a single-tenant install it prints `Teams config for multi-tenant support is not enabled.` and exits without changing anything. If that message is all you saw, the missing permissions are still missing.

Verify before you deploy:

```bash
php artisan tinker
>>> Spatie\Permission\Models\Permission::where('name', 'like', '%crm monitors%')->count();  // expect 4
>>> Spatie\Permission\Models\Role::where('name', 'Owner')->first()->permissions->count();   // expect all
```

**3. Audit your custom roles.** See breaking change 2 below.

### Breaking changes

**1. Authorization is now enforced on every mutating action.**

The UI has always advertised permissions through Blade `@can` directives, but the Livewire components behind those buttons did not re-check them on the server. Anyone who could reach a CRM page could invoke its actions directly over the Livewire endpoint, regardless of role.

`$this->authorize(...)` is now called on **165 mutating actions across 117 Livewire components** — leads, deals, quotes, orders, invoices, deliveries, purchase orders, products, tasks, people, organisations, notes / calls / meetings / lunches / files, settings lookups, chat, imports, templates and campaigns. `can:` middleware was added to the previously ungated `activities/*` route group and the deal / quote / order `products` sub-resource groups. Mutating buttons and menu items are now hidden rather than shown-then-denied, and kanban cards are no longer draggable without the matching `edit` permission.

**No new permission name is introduced anywhere in this change.** Every guard reuses a permission that already ships in the seeder, and no stock role loses access it previously exercised — provided the permission rows exist, which is what the seeding step above guarantees. See [Security](/security) and [Roles](/roles).

**2. Custom view-only roles lose actions they could previously perform.**

This is correct behaviour rather than a bug, but it will generate support traffic. If your team built a custom role under **Settings → Roles** — a "Read only" or "Support" role granted `view crm leads` but not `edit crm leads`, say — those users could previously still edit, delete and reorder records using the on-screen controls. The UI hid some of the buttons; the server did not check. Now it does.

```bash
php artisan tinker
>>> Spatie\Permission\Models\Role::where('crm_role', 1)
...     ->whereNotIn('name', ['Owner', 'Admin', 'Manager', 'Employee'])
...     ->get()
...     ->mapWithKeys(fn ($r) => [$r->name => $r->permissions->pluck('name')]);
```

For each custom role, decide whether the people holding it are *expected* to perform the actions they have been performing. If yes, grant the matching `create` / `edit` / `delete` permission. If no, the upgrade is the fix. Either way, tell those users first.

**3. A trimmed `config('laravel-crm.modules')` now 403s the disabled module — for everyone, including Owner.**

Every policy gates its methods on an `isEnabled()` helper that returns `true` when the module is listed, `true` when `modules` is falsy, and `null` when `modules` **is** an array that does not contain the module. Policy methods read `if ($this->isEnabled() && $user->hasPermissionTo(...))`, so a disabled module denies the action no matter which permissions the user holds.

Before this release that only affected surfaces which already called `authorize()`. Now it affects every mutating action in the module.

```bash
php artisan tinker
>>> config('laravel-crm.modules');
```

If a module is absent from the array and you still expect people to use it, add it back. If it is absent deliberately, confirm the module's navigation is also hidden. See [Configuration → Optional Modules](/configuration#optional-modules).

**4. Line item quantities become `decimal(15,3)` — six `ALTER TABLE`s.**

`quantity` widens from `integer` to `decimal(15,3)` on six tables:

| Table |
|---|
| `crm_quote_products` |
| `crm_order_products` |
| `crm_deal_products` |
| `crm_invoice_lines` |
| `crm_purchase_order_lines` |
| `crm_delivery_products` |

**No data is lost.** `decimal(15,3)` strictly contains the old `INT` range, so every existing row widens exactly and NULLs stay NULL. Nothing needs backfilling.

**Plan for a brief write lock.** On MySQL each `ALTER` rewrites the table; Postgres is likewise a rewrite. On a small CRM that is a fraction of a second. On an install with millions of invoice lines, budget for it or run the migration in a maintenance window.

**Rolling back truncates.** `down()` puts the column back to `integer`, discarding the decimal part of any quantity entered since. There is no lossless inverse — export those rows first if you need to go back.

Verify afterwards:

```sql
SHOW COLUMNS FROM crm_quote_products LIKE 'quantity';   -- decimal(15,3), Null: YES
```

**5. The `db_update_1201` per-team backfill rewrites `pipeline_stage_id` on seven tables.**

**Teams installs only** (`laravel-crm.teams = true`) — single-tenant installs skip this entirely.

`laravelcrm:update` runs a one-time backfill that gives every pre-existing team its own copy of the CRM lookup data and pipelines — pipelines, stages, labels, tax rates, industries and the three type lookups — then re-points existing records at that team's pipeline stages. Without it, `/leads/create` on a teams install renders against an empty per-team pipeline. The rewrite touches `pipeline_stage_id` on `crm_leads`, `crm_deals`, `crm_quotes`, `crm_orders`, `crm_invoices`, `crm_deliveries` and `crm_purchase_orders`.

Precisely what it does:

- **Matching is by stage name**, within the same pipeline model — a global stage on the Lead pipeline maps only to the per-team Lead stage of the same name, never to a Deal stage that happens to share it.
- **Only rows belonging to the team being backfilled are touched.**
- **A stage name with no per-team counterpart is left alone.** Nothing is nulled and nothing errors — the record keeps pointing at the global stage. If you renamed stages per team, add or rename the missing stage first and re-run.
- **There is no `down()`.** This is a data migration, not a schema one, so rolling the package back does not put the ids back. **Restore from a backup if you need to reverse it.**
- **It is safe to re-run.** The lookup copy upserts on the team plus the row's own name, and the re-point matches on the global stage id, which the first run has already replaced.

**Take a database backup before upgrading a teams install.**

**6. `laravelcrm:update` now fails loudly.**

It used to catch migration and seeder exceptions, downgrade them to warnings, and still print `Laravel CRM is now updated.` with exit code 0 — so a broken upgrade and a clean one looked identical in a deploy log. Both are now fatal: the command prints an error and returns a failure exit code, and `db_version` is stamped only on the success path.

**A deploy script's `&&` chain that previously carried on over a half-applied schema will now stop.** That is the point, but check your pipeline for it. The command also gained `--force` for non-interactive use.

**7. Migrations no longer need publishing — but `migrate` alone is still not enough.**

Migrations added from this release ship as real `.php` files inside the package, in `database/updates`, loaded with `loadMigrationsFrom`. The existing `.stub` publish array is frozen and still published, so **existing hosts are unaffected and keep the filenames they already have**.

The seven migrations this release adds all arrive this way, so there is nothing to publish for any of them:

| Migration | What it does |
|---|---|
| `add_perf_notified_at_to_laravel_crm_monitors_table` | Performance-alert dedup timestamp |
| `add_recovered_notified_at_to_laravel_crm_monitors_table` | Recovery-alert dedup timestamp |
| `create_crm_user_invitations_table` | The user invitation lifecycle |
| `add_soft_deletes_and_last_sent_at_to_crm_user_invitations_table` | Resend + revoke support |
| `add_pdf_template_to_laravel_crm_tables` | Per-document PDF template choice |
| `add_start_at_to_laravel_crm_tasks_table` | Task start time |
| `change_quantity_to_decimal_on_laravel_crm_tables` | Decimal line item quantities (see above) |

> **Important:** `composer update && php artisan migrate` is **not** enough. It runs no seeders, no data backfills, and stamps no `db_version` — and a stale `manifest.json` will point at asset filenames that no longer exist on disk. Run `php artisan laravelcrm:update`.

Newly published stubs are also now stamped from a fixed `2024_01_01` epoch rather than the moment of publishing, so a fresh install orders them correctly against the package-loaded migrations. This only affects stubs that have never been published on a given host.

**8. REST API changes.**

- **`subtotal` and `total` are rejected on quote / order / invoice writes.** Both are computed from `line_items`, `discount`, `tax` and `adjustments`. Sending either is now a `422` naming the cause, where a pre-release build silently ignored them and returned recomputed numbers. `prohibited` passes for an absent or `null` value, so a payload that never sent them is unaffected. **Remove both fields from your write payloads.**
- **`discount` and `tax` gained `min:0`.** A negative value previously passed validation and inflated the computed total past the sum of the line items. Send an `adjustments` value instead.
- **`quantity` is now a JSON number that may come back fractional.** It was previously cast to an integer in every response. Clients decoding it into an `int` field will truncate. The request side is a widening — every previously valid payload still passes.
- **Cross-team ids are now a `422`.** Every foreign key on a write is validated against the caller's team. Single-tenant installs are unaffected.

See [API → Conventions](/api#conventions) and [API · Quotes → Nested line items](/api-quotes#nested-line-items).

**9. Two removals.**

- **The `laravel-crm.users.sendinvite` route is gone.** It is the only named route dropped since 2.3.0 — invitations run through the `crm_user_invitations` table and its Livewire surface now. `route('laravel-crm.users.sendinvite')` throws `RouteNotFoundException`, so grep your app for it.
- **The `SystemCheck` middleware class is gone.** It was pushed onto the `crm` middleware group and reported through flash messages; the banner is the `crm-system-check` Livewire component backed by `SystemCheckService` now. If you referenced `VentureDrake\LaravelCrm\Http\Middleware\SystemCheck` in your own stack, remove the reference.

**10. The users index swapped its query-string filters.**

The `#[Url]` properties `user_id` and `label_id` are replaced by `role_id` (array) and `crm_access` (nullable string), matching the filters the page now offers — neither of the old two matched anything a user row actually carries. A bookmarked or generated `?user_id=` / `?label_id=` link is **ignored rather than erroring**.

**11. Host code that extends, publishes or integrates against these things.**

None of these affect a stock install.

- **`ModelProducts` renamed the `quantities` row key to `quantity_max`.** The per-row array carried a `quantities` array of `<select>` options; it now carries a single `quantity_max` number, because the control is a bounded number input rather than a dropdown. A **published** copy of `resources/views/vendor/laravel-crm/livewire/model-products.blade.php` still iterates `$products[$index]['quantities']` and will render an empty control. Re-publish that view with `--force` and re-apply your edits.
- **`$lineItem->quantity` now reads back as a PHP `float`, not an `int`.** The `HasDecimalQuantity` trait casts it on `QuoteProduct`, `OrderProduct`, `DealProduct`, `InvoiceLine`, `PurchaseOrderLine` and `DeliveryProduct`. A whole quantity of 2 now compares as `2.0`, so `$line->quantity === 2` is `false` and `is_int($line->quantity)` is `false`. Loose `==` and arithmetic are unaffected. Search your app for strict comparisons and `is_int` / `gettype` checks against a line item quantity; your own `(int)` casts still work but will silently truncate a fractional quantity.
- **`CheckAmount`'s `subTotal()` / `tax()` / `total()` return a real `bool`.** They previously returned `true` on a match and fell off the end returning `null` on a mismatch. Code doing `=== false` against them never matched and now does; `=== null` no longer matches.
- **`LiveRelatedContactOrganisation` was renamed to `LiveRelatedContactOrganization`.**

### After upgrading

**Existing installs: add the composer hook once.** New installs get this from `laravelcrm:install`.

```json
"scripts": {
    "post-autoload-dump": [
        "@php artisan package:discover --ansi",
        "@php artisan laravelcrm:upgrade --ansi"
    ]
}
```

Then run `composer dump-autoload` to confirm it fires. From that point on, every `composer install` and `composer update` republishes CRM assets and clears caches on its own. The line must come **after** `package:discover` — that is what makes the package's artisan commands resolvable.

See [Updates](/updates) for what the two commands do and how the system-check banner decides you need them.

## Upgrading Within 2.x

Follow these steps when upgrading between 2.x releases. Read the version-specific notes above first.

### Local

```bash
composer update venturedrake/laravel-crm
php artisan laravelcrm:update
```

That is the whole procedure. The first command republishes assets and clears caches via the composer hook; the second applies database changes.

> **Important:** `composer update && php artisan migrate` is **not** enough. Migrations published before 2.4.0 ship as `.stub` files that have to be published into your `database/migrations` before the migrator can see them, and a stale `manifest.json` will point at asset filenames that no longer exist on disk. `laravelcrm:update` does both, and also runs the seeders and data backfills.

### Production deploy

```bash
composer install --no-dev --optimize-autoloader   # post-autoload-dump fires laravelcrm:upgrade
php artisan laravelcrm:update --force             # migrations + backfills, no prompts
php artisan config:cache && php artisan route:cache && php artisan view:cache
```

`--force` skips the production confirmation prompt. `laravelcrm:update` exits non-zero if migrations or seeding fail, so `&&` chains and CI steps stop where they should.

### Zero-downtime deploys

On Envoyer, Deployer, Vapor and anything else that builds each release in its own directory, the two commands belong in different hooks:

| Command | Hook | Why |
|---|---|---|
| `laravelcrm:upgrade` | **Build / install** — it fires automatically from `composer install` | It writes into *that release's* `public/`, so it has to run per release directory, before the symlink flips |
| `laravelcrm:update` | **Activate / after-deploy**, once | It touches the shared database. Running it per server would run the same migrations concurrently |

Nothing extra is needed for the first — the composer hook handles it. Just make sure the second is not in a per-server hook.

### What each command does

| | `laravelcrm:upgrade` | `laravelcrm:update` |
|---|---|---|
| Republishes built assets (JS/CSS/images) | Yes | Yes (calls `upgrade` first) |
| Prunes stale content-hashed build files | Yes | Yes |
| Clears cached config, routes, views | Yes | Yes |
| Publishes Flasher assets | Yes | Yes |
| Warns about drifted published views | Yes | Yes |
| Publishes migration stubs | No | Yes |
| Runs `migrate` | No | Yes |
| Runs seeders and data backfills | No | Yes |
| Stamps the `db_version` marker | No | Yes |
| Prompts | Never | Only on an interactive production console, unless `--force` |
| Exits non-zero on failure | Only if asset publishing itself errors | On any failure |

**`laravelcrm:upgrade` never opens a database connection.** That is deliberate: it runs from a composer hook, which can fire during a build when the database is unreachable, mid-migration, or belongs to a different release. All database work is in `laravelcrm:update`, which you run explicitly.

`laravelcrm:update` also runs the lookup-data seeders this guide used to tell you to run by hand — `laravelcrm:lead-sources`, and on teams installs `laravelcrm:permissions`, `laravelcrm:labels`, `laravelcrm:addresstypes`, `laravelcrm:contacttypes` and `laravelcrm:organizationtypes`. All are idempotent.

### Caveats

**Published views are frozen at the version you published them.** If you ran `vendor:publish --tag=views`, your copies in `resources/views/vendor/laravel-crm` shadow the package's and will *not* pick up template changes from a release. Re-publish with `--force` and re-apply your edits, or diff the package's `resources/views` against your copies before upgrading. The same applies to published lang files. One deliberate exception: an edited PDF view keeps rendering your layout — see [PDF Templates](/pdf-templates#customising-via-a-published-view).

**New config keys arrive automatically, but a cached config hides them.** Keys added to `config/package.php` and `config/laravel-crm.php` reach your app through `mergeConfigFrom`, so you do not need to re-publish the config. A stale `php artisan config:cache` from the previous release will hide them — the composer hook runs `config:clear` for exactly this reason. Keys you have overridden in your published `config/laravel-crm.php` stay as you set them.

**The database can be behind the code without anything looking wrong.** The CRM stamps a `db_version` setting when `laravelcrm:update` completes and reports a banner when the installed code is ahead of it, or when one of *this package's* migrations has not run. Migrations belonging to your own application or to other packages are not counted. If you see *"Your Laravel CRM version requires some database updates"*, run `php artisan laravelcrm:update`. See [Updates](/updates).

**Rolling back the package does not roll back the database.** Migrations are not reversed by downgrading the composer constraint. Restore from a backup if you need to go back.

**Remove the composer hook before you remove the package.** `post-autoload-dump` fires on `composer remove venturedrake/laravel-crm` too, at which point `laravelcrm:upgrade` no longer exists and composer reports the script returned a non-zero exit code. Delete the `@php artisan laravelcrm:upgrade --ansi` line from your `composer.json` first.

## Upgrading from 2.2.x to 2.3.0

Version 2.3.0 introduces two new optional modules — a public **Features** voting board and an **uptime / SSL Monitoring** module — plus a redesigned public portal and quality-of-life improvements to file uploads and the installer. There are no schema-breaking changes; follow the standard [Upgrading Within 2.x](#upgrading-within-2x) steps.

### What's new

- **Features** — public roadmap board with voting, comments, status tracking, view analytics, and email notifications. See [Features](/features).
- **Monitoring** — uptime and SSL monitoring for HTTP/HTTPS endpoints, with response-time charts, sparklines, SSL expiry alerts, and SSRF protection. See [Monitoring](/monitoring).
- **Portal redesign** — public quote, invoice, purchase-order, and feature pages rebuilt on Tailwind v4 + DaisyUI v5 + MaryUI with a top navbar, theme toggle, and toast notifications. See [Portal](/portal).
- **File-upload improvements** — drag-and-drop dropzone, upload progress bar, deferred upload, and per-component max-size / allowed-types validation.
- **Installer module selection** — `laravelcrm:install` now prompts which modules to enable, or accepts `--modules=all` / `--modules=leads,deals,...` for non-interactive installs.

### Enabling the new modules

The new modules are added to the `modules` array in the published config file. When you run `laravelcrm:update` the migrations for `crm_features*`, `crm_monitors`, and `crm_monitor_checks` will be applied.

To enable them, ensure the following entries exist in `config/laravel-crm.php`:

```php
'modules' => [
    // ... existing modules
    'features',
    'monitoring',
],
```

Optional environment variables (all have sensible defaults):

```env
# Features
LARAVEL_CRM_FEATURES_VIEW_DEDUP_MINUTES=60
LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION=false

# Monitoring
LARAVEL_CRM_MONITORING_DEFAULT_FREQUENCY_MINUTES=5
LARAVEL_CRM_MONITORING_DEFAULT_SSL_DAYS_BEFORE_EXPIRY_ALERT=14
LARAVEL_CRM_MONITORING_REQUEST_TIMEOUT_SECONDS=15
LARAVEL_CRM_MONITORING_SSL_RECHECK_HOURS=12
LARAVEL_CRM_MONITORING_ALLOW_PRIVATE_TARGETS=false
```

If you use the Monitoring module, make sure Laravel's scheduler is running (`* * * * * php artisan schedule:run`) so monitor checks fire on their configured intervals.

### Breaking changes

None. PHP 8.2+ and Laravel 11+ requirements introduced in 2.2.0 are unchanged.

## Upgrading from 2.1.x to 2.2.0

Version 2.2.0 introduces a JSON REST API and adds page titles throughout the UI. There are no schema-breaking changes — follow the standard [Upgrading Within 2.x](#upgrading-within-2x) steps.

### What's new

- **REST API** — Sanctum-authenticated JSON API at `/crm/api/v2` exposing 8 resourceful entities (`leads`, `products`, `organizations`, `people`, `deals`, `quotes`, `orders`, `invoices`) plus auth endpoints. See [API](/api).
- **Page titles** — Every CRM page now sets a descriptive `<title>` tag for better browser tabs, history, and SEO.

### Breaking changes

- **PHP requirement** — Minimum PHP version is now **8.2** (was 8.1).
- **Laravel requirement** — Minimum Laravel version is now **11** (was 10).

If you're on PHP 8.1 or Laravel 10, upgrade those first before pulling 2.2.0.

### Optional: enable the REST API

If you want to use the new API, install Sanctum in the host application — see [API → Installation](/api).

## Upgrading from 1.x to 2.x

Version 2.x is a major rewrite of the user interface and frontend stack. The backend API and model layer remain largely compatible.

### Breaking Changes

- **UI stack**: Bootstrap 4 + jQuery replaced with Tailwind CSS v4 + DaisyUI v5 + MaryUI
- **Livewire**: Upgraded from Livewire 2 to Livewire 3 or 4. All Livewire components have been rewritten.
- **PHP requirement**: Minimum PHP version is now 8.1 (was 7.3)
- **Laravel requirement**: Minimum Laravel version is now 10 (was 6)
- **Views**: All Blade views have been rewritten. If you published and customized views, you will need to re-apply customizations to the new templates.

### New Modules

- **Chat** — Live chat with embeddable visitor widget — see [Chat](/chat)
- **Email Marketing** — Campaign and template management with open/click tracking — see [Email Marketing](/email-marketing)
- **SMS Marketing** — Campaign and template management via ClickSend — see [SMS Marketing](/sms-marketing)

> **Important:** Upgrade straight to **2.6.0 or later**. 2.5.0's `laravelcrm:v2` left v1 class names in ten columns and could abort part-way. If you already ran it, see [If you already ran 2.5.0's `laravelcrm:v2`](#if-you-already-ran-250s-laravelcrmv2).

### Step 1. Back Up and Update Package

Take a full database backup, then:

```bash
composer require venturedrake/laravel-crm:^2.6.0
```

### Step 2. Publish Config & Assets

```bash
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="config"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="assets" --force
```

Do **not** run `php artisan migrate` or `php artisan laravelcrm:update` yet. Since 2.6.0 both refuse to start while a v1 table exists, because running the v2 migrations first creates empty v2 tables next to the v1 ones and strands the v1 data. See [`migrate` is blocked on a v1 database](#migrate-is-blocked-on-a-v1-database).

### Step 3. Run the v2 Migration Helper

The package ships a one-shot command that renames the v1 tables and columns, rewrites stored v1 class names, and then publishes and runs the v2 migrations through `laravelcrm:update`:

```bash
php artisan laravelcrm:v2
```

It asks you to confirm that you have taken a backup. Pass `--force` to skip the prompt, for example from a deploy script; without `--force` a non-interactive run exits `1`. If it fails, fix the error and run it again: every step skips work that is already done. See [Upgrading from v1: go straight to 2.6.0](#upgrading-from-v1-go-straight-to-260) for the exit codes and how it handles v2 tables created by an early `migrate`.

### Step 4. Update Permissions & Custom Fields

```bash
php artisan laravelcrm:permissions
php artisan laravelcrm:fields
```

### Step 5. Clear Caches

```bash
php artisan cache:clear
php artisan config:clear
php artisan view:clear
```

## General Upgrade Tips

- **Back up your database** before upgrading to any new version.
- **Review the [changelog](https://github.com/venturedrake/laravel-crm/blob/master/CHANGELOG.md)** for breaking changes and new features.
- **Clear caches** after upgrading:

```bash
php artisan cache:clear
php artisan config:clear
php artisan view:clear
```

- **Run tests** to ensure your customisations still work with the updated package.
