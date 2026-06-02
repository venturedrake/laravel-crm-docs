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
| `/p/features/{external_id}` | Public feature show page (vote / comment requires sign-in) |
| `/p/features/submit` | Public submission form (requires sign-in) |
| `/p/quotes/{external_id}` | Customer-facing quote |
| `/p/invoices/{external_id}` | Customer-facing invoice |
| `/p/purchase-orders/{external_id}` | Supplier-facing purchase order |
| `/p/chat/{publicKey}` | Embeddable chat-widget iframe |
| `/p/login`, `/p/register`, `/p/logout` | Portal auth pages (when enabled) |

## Portal Authentication

Portal authentication is **opt-in**. By default, anonymous visitors can browse the feature board but cannot vote, comment, or submit new feature requests. Enable registration to let visitors create an account on your portal:

```env
LARAVEL_CRM_PORTAL_ALLOW_REGISTRATION=true
```

When enabled, `/p/register` writes rows to the host application's `users` table and dispatches Laravel's `Registered` event so any normal Laravel signup hooks (welcome email, etc.) still fire. Portal users are regular host-app users without CRM access, but they can interact with public surfaces such as the feature board.

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
