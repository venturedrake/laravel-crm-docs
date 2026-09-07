# Portal

[[toc]]

## Overview

The CRM ships a small public-facing portal that hosts:

- The **public feature board** at `/p/features` — see [Features](/features)
- **Signed quote, invoice, and purchase-order pages** at `/p/quotes/{id}`, `/p/invoices/{id}`, and `/p/purchase-orders/{id}` — your customers receive these links via email
- The **chat widget** at `/p/chat/{publicKey}` — see [Chat](/chat)
- Optional **portal authentication** (login + register) so portal users can vote on features and post comments

Portal pages are rendered with the same Tailwind v4 + DaisyUI v5 + MaryUI stack as the CRM admin UI but use a separate, cleaner layout: a centred main container with toast notifications, and a full-width footer pinned to the bottom of the page rather than floating under a short document.

The layout splits in two. The **auth, feature-board and invite pages** keep the navbar — logo, theme toggle and a **Login** button — because they are pages you navigate. The **document pages** (quote, invoice, purchase order) render chrome-free and pin `data-theme="light"`: they are documents you read, opened from an emailed link by someone who has no account to log in to, and they should look like the printed PDF they mirror rather than inheriting whatever theme the visitor's browser happens to prefer.

## Routes

Portal routes live in `src/Http/portal-routes.php` and are registered **outside** the CRM `auth.laravel-crm` middleware group so customers do not need a CRM account to view signed documents or browse the feature board.

| Route | Description |
|---|---|
| `/p/features` | Public feature board (read-only without sign-in) |
| `/p/features/team/{id}` | The same board, addressed by team — the shareable link on a multi-tenant install |
| `/p/features/{external_id}` | Public feature show page (vote / comment requires sign-in) |
| `/p/features/submit` | Public submission form (requires sign-in) |
| `/p/quotes/{external_id}` | Customer-facing quote |
| `/p/invoices/{external_id}` | Customer-facing invoice |
| `/p/purchase-orders/{external_id}` | Supplier-facing purchase order |
| `/p/chat/{publicKey}` | Embeddable chat-widget iframe |
| `/p/login`, `/p/register`, `/p/logout` | Portal auth pages (when enabled) |
| `/p/email/o/{token}.gif` | Email campaign open-tracking pixel |
| `/p/email/c/{token}` | Email campaign click tracker |
| `/p/email/u/{token}` | Email campaign unsubscribe form (`GET`) and confirmation (`POST`) |
| `/p/sms/c/{token}` | SMS campaign click tracker |
| `/p/sms/u/{token}` | SMS campaign unsubscribe form (`GET`) and confirmation (`POST`) |

> **Note:** The `/p/email/*` and `/p/sms/*` tracking routes live in `src/Http/email-tracking-routes.php` and are registered outside the `web` middleware group as well, so a tracking pixel or a one-click unsubscribe works from any mail client without session or CSRF interference. See [Email Marketing](/email-marketing) and [SMS Marketing](/sms-marketing).

## Portal Documents

A portal document page renders **the record's own PDF template** — the customer sees the same document on screen that the Download button hands them, not a second hand-maintained HTML layout that drifts away from it.

`Support\PortalDocument` renders the view `PdfTemplateRegistry::viewForModel()` returns — the same call the download path makes — with the same view data, and embeds the result in a self-sizing sandboxed iframe. Template resolution is therefore identical on both surfaces: the record's `pdf_template` column, the **Settings → Templates** default for that document type, a published view, then `modern`. See [PDF Templates](/pdf-templates).

**The sandbox deliberately withholds `allow-scripts`.** PDF templates emit record content unescaped, which is safe in a PDF renderer and is not safe in a browser document. Nothing in a template needs to run script, so the capability is simply not granted.

One correction is applied on the way out, because DomPDF and a browser genuinely disagree: DomPDF multiplies the font's own height by a unitless `line-height` rather than treating it as the line box height, so a template declaring `1.2` renders at roughly `1.79em` there and `1.2em` in a browser. `PortalDocument` rewrites the declared line heights for `modern`, `bold`, `compact` and `professional` so the page's line pitch matches the PDF's. `classic` declares no unitless line-height and is excluded. The correction lives in `PortalDocument` rather than in the shared templates, so the generated PDFs are byte-for-byte unaffected.

> **Note:** The three `crm-portal-quote-line-items`, `crm-portal-invoice-line-items` and `crm-portal-purchase-order-line-items` Livewire components were removed in 2.4.1, dead once the pages render the template itself. A published portal view still referencing one will throw — see the [Upgrade Guide](/upgrading#views-to-re-publish).

## Portal Authentication

Portal authentication is **opt-in**. By default, anonymous visitors can browse the feature board but cannot vote, comment, or submit new feature requests. Enable registration to let visitors create an account on your portal:

```env
LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION=true
```

When enabled, `/p/register` writes rows to the host application's `users` table and dispatches Laravel's `Registered` event so any normal Laravel signup hooks (welcome email, etc.) still fire. Portal users are regular host-app users without CRM access, but they can interact with public surfaces such as the feature board.

## Portal Teams

A public roadmap is read by a team's customers. They are anonymous and carry no `currentTeam`, so on a multi-tenant install the team behind a board cannot be inferred from the session. `Support\PortalTeam` resolves it instead, in this order:

1. **The team named in the URL** — `/p/features/team/{id}`. This is what makes a board shareable with people who have no account. Opening it also remembers the board for the rest of the visitor's session.
2. **The board remembered in the session**, so "back to the board", voting, commenting and submitting all stay on the board the visitor arrived at.
3. **The signed-in user's current team** — the natural default for staff.
4. **The only team that has a public board**, when there is exactly one. This is what makes the common "teams enabled, one team" install work with no configuration at all.

If none of these answers and teams are enabled, the request 404s rather than guessing. When teams are off the whole mechanism is skipped.

Admins can copy the right link from the **Public board** button on `/crm/features` — see [Features](/features).

Every signal is derived from the request or the record. There is deliberately **no configured override**: `LARAVEL_CRM_PORTAL_TEAM_ID` used to sit ahead of all four as a hard single-tenant lock, and was removed in 2.4.2 — see the [Upgrade Guide](/upgrading#removing-the-pinned-portal-team).

> **Note:** A public feature is reachable by its own link whichever team owns it, and opening one moves the visitor onto that board for the rest of the session.

> **Important:** A feature submitted through the portal is stamped with the **board's** team, not the submitter's. The submit path used to require the submitter's `currentTeam` to match the board's team, which `403`'d every visitor who registered through `/p/register` — they hold no host-app team, which is the entire population the portal exists for.

## Portal Settings and Teams

A document's branding comes from **the document's own team**, not from whoever happens to be reading it and not from whichever team's settings row the database listed last.

The portal is anonymous by design, so `BelongsToTeamsScope` never engages there. Before 2.4.2 that meant every settings read on a portal document page was unscoped, and on a multi-tenant install a customer opening one team's emailed invoice could be shown another team's organisation name, ABN, contact block and logo — on the page and in the PDF behind the Download button.

The portal controllers now call `SettingService::forTeam()` with the document's own `team_id` before rendering anything, which corrects every downstream reader of the shared scoped instance — the contact block, the logo and the settings view composer included. The cached settings map is partitioned on the same answer, so warming the page with one team's invoice cannot serve its branding to the next team's for the rest of the TTL.

A record that predates teams — no `team_id` at all — renders a blank From block rather than borrowing another team's. See [Teams](/teams) and [Security](/security).

## Configuration

```php
// config/laravel-crm.php
'portal' => [
    'allow_registration' => env('LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION` | `false` | Allow self-registration on the portal |

See [Configuration → Portal](/configuration#portal).
