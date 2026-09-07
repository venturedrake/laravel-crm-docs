# Upgrade Guide

[[toc]]

Two things to read before any upgrade: the **version-specific notes** for the release you are moving to, and [Upgrading Within 2.x](#upgrading-within-2-x), which is the same every release. Read the version notes **first** — some releases change who is allowed to do what, and the failure mode is a `403` for a user who could previously click the button.

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

**None.** Neither change in this release touches `resources/views`, so a published view cannot hide either fix. If you are coming from 2.4.0, the 2.4.1 table below still applies to you.

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

Version 2.3.0 introduces two new optional modules — a public **Features** voting board and an **uptime / SSL Monitoring** module — plus a redesigned public portal and quality-of-life improvements to file uploads and the installer. There are no schema-breaking changes; follow the standard [Upgrading Within 2.x](#upgrading-within-2-x) steps.

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

Version 2.2.0 introduces a JSON REST API and adds page titles throughout the UI. There are no schema-breaking changes — follow the standard [Upgrading Within 2.x](#upgrading-within-2-x) steps.

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

### Step 1. Update Package

```bash
composer require venturedrake/laravel-crm
```

### Step 2. Publish & Migrate

```bash
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="migrations"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="config"
php artisan vendor:publish --provider="VentureDrake\LaravelCrm\LaravelCrmServiceProvider" --tag="assets" --force
php artisan migrate
```

### Step 3. Run the v2 Migration Helper

The package ships a one-shot command that backfills new 2.x columns, normalises existing data, and seeds new lookup tables required by the rewritten UI:

```bash
php artisan laravelcrm:v2
```

Run this **once** after migrating from a 1.x installation.

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
