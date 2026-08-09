# Portal

[[toc]]

## Overview

The CRM ships a small public-facing portal that hosts:

- The **public feature board** at `/p/features` — see [Features](/features)
- **Signed quote, invoice, and purchase-order pages** at `/p/quotes/{id}`, `/p/invoices/{id}`, and `/p/purchase-orders/{id}` — your customers receive these links via email
- The **chat widget** at `/p/chat/{publicKey}` — see [Chat](/chat)
- Optional **portal authentication** (login + register) so portal users can vote on features and post comments

Portal pages are rendered with the same Tailwind v4 + DaisyUI v5 + MaryUI stack as the CRM admin UI but use a separate, cleaner layout: a top navbar with logo and theme toggle, a centred main container with toast notifications, and a footer.

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

## Portal Authentication

Portal authentication is **opt-in**. By default, anonymous visitors can browse the feature board but cannot vote, comment, or submit new feature requests. Enable registration to let visitors create an account on your portal:

```env
LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION=true
```

When enabled, `/p/register` writes rows to the host application's `users` table and dispatches Laravel's `Registered` event so any normal Laravel signup hooks (welcome email, etc.) still fire. Portal users are regular host-app users without CRM access, but they can interact with public surfaces such as the feature board.

## Portal Teams

A public roadmap is read by a team's customers. They are anonymous and carry no `currentTeam`, so on a multi-tenant install the team behind a board cannot be inferred from the session. `Support\PortalTeam` resolves it instead, in this order:

1. `laravel-crm.portal.team_id`, when set. Anyone who configured it wanted a single-tenant portal, so it is a hard lock and every other signal is ignored.
2. **The team named in the URL** — `/p/features/team/{id}`. This is what makes a board shareable with people who have no account. Opening it also remembers the board for the rest of the visitor's session.
3. **The board remembered in the session**, so "back to the board", voting, commenting and submitting all stay on the board the visitor arrived at.
4. **The signed-in user's current team** — the natural default for staff.
5. **The only team that has a public board**, when there is exactly one. This is what makes the common "teams enabled, one team" install work with no configuration at all.

If none of these answers and teams are enabled, the request 404s rather than guessing. When teams are off the whole mechanism is skipped.

Admins can copy the right link from the **Public board** button on `/crm/features` — see [Features](/features).

> **Note:** A public feature is reachable by its own link whichever team owns it, and opening one moves the visitor onto that board for the rest of the session. An install with `portal.team_id` set keeps 404ing everything outside that team.

> **Important:** A feature submitted through the portal is stamped with the **board's** team, not the submitter's. The submit path used to require the submitter's `currentTeam` to match the board's team, which `403`'d every visitor who registered through `/p/register` — they hold no host-app team, which is the entire population the portal exists for.

## Configuration

```php
// config/laravel-crm.php
'portal' => [
    'team_id' => env('LARAVEL_CRM_PORTAL_TEAM_ID'),
    'allow_registration' => env('LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION', false),
],
```

| Environment Variable | Default | Description |
|---|---|---|
| `LARAVEL_CRM_PORTAL_TEAM_ID` | `null` | Pin the portal to a single team |
| `LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION` | `false` | Allow self-registration on the portal |

`portal.team_id` is **optional**. Leave it unset and every team gets its own portal, resolved as described above. Set it and the portal behaves as the single-tenant lock it always was: every feature outside that team 404s. It is ignored entirely when `laravel-crm.teams` is off.

See [Configuration → Portal](/configuration#portal).
